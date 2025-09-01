# Rozdział 9: Architektura mikrousług

Mikrousługi w Go to nie tylko podział aplikacji na mniejsze części - to zupełnie inne podejście do projektowania systemów. Jeśli do tej pory myślałeś o kodzie jak o jednej dużej maszynie, to teraz nauczysz się budować ekosystem niezależnych serwisów.

Go jest idealnym wyborem do mikrousług ze względu na kilka kluczowych cech:
- **Szybki start aplikacji** - 20-50ms cold start
- **Małe zużycie pamięci** - 20-100MB RAM na serwis  
- **Kompaktowe binarki** - 5-20MB na serwis
- **Wbudowana współbieżność** - obsługa tysięcy requestów
- **Silne typowanie** - kontrakty API wymuszane przez kompilator

## Wzorce projektowe dla mikrousług

### Service per Business Capability

Każda mikrousługa powinna obsługiwać jedną domenę biznesową. To nie jest kwestia rozmiaru kodu, ale odpowiedzialności.

```go
// Błędne podejście - jeden serwis robi wszystko
type UserService struct {
    db *sql.DB
}

func (s *UserService) CreateUser(user *User) error { /* ... */ }
func (s *UserService) SendNotification(msg string) error { /* ... */ }
func (s *UserService) ProcessPayment(amount float64) error { /* ... */ }
func (s *UserService) GenerateReport() error { /* ... */ }
// 30 innych metod...

// Poprawne podejście - każdy serwis ma swoją domenę
// User Service - zarządzanie użytkownikami
type UserService struct {
    repo UserRepository
}

func (s *UserService) Create(ctx context.Context, user *User) (*User, error) {
    if err := s.validateUser(user); err != nil {
        return nil, fmt.Errorf("user validation failed: %w", err)
    }
    return s.repo.Save(ctx, user)
}

func (s *UserService) GetByID(ctx context.Context, id string) (*User, error) {
    return s.repo.FindByID(ctx, id)
}

// Notification Service - tylko powiadomienia
type NotificationService struct {
    emailProvider EmailProvider
    smsProvider   SMSProvider
}

func (s *NotificationService) SendEmail(ctx context.Context, recipient, subject, body string) error {
    return s.emailProvider.Send(ctx, recipient, subject, body)
}

// Payment Service - tylko płatności
type PaymentService struct {
    gateway PaymentGateway
}

func (s *PaymentService) ProcessPayment(ctx context.Context, payment *Payment) (*PaymentResult, error) {
    return s.gateway.Process(ctx, payment)
}
```

**Jak podzielić domenę:**
1. **Sprawdź rzeczowniki** - User, Order, Payment = różne serwisy
2. **Sprawdź czasowniki** - jeśli serwis "robi wszystko", to za duży
3. **Sprawdź zespoły** - jeden serwis = jeden zespół może go rozwijać
4. **Sprawdź deployment** - czy można deployować niezależnie?

### Database per Service

W mikrousługach każdy serwis ma swoją bazę danych. Nie ma foreign key między serwisami.

```go
// User Service - PostgreSQL
type UserRepository struct {
    db *sql.DB // postgres://users-db:5432/users
}

func (r *UserRepository) Save(ctx context.Context, user *User) error {
    query := `INSERT INTO users (id, email, name) VALUES ($1, $2, $3)`
    _, err := r.db.ExecContext(ctx, query, user.ID, user.Email, user.Name)
    return err
}

// Order Service - może być inna baza!
type OrderRepository struct {
    client *mongo.Client // mongodb://orders-db:27017/orders
}

func (r *OrderRepository) Save(ctx context.Context, order *Order) error {
    collection := r.client.Database("orders").Collection("orders")
    _, err := collection.InsertOne(ctx, order)
    return err
}

// Jak pobrać dane z wielu serwisów?
type OrderService struct {
    orderRepo  OrderRepository
    userClient UserClient // HTTP client!
}

func (s *OrderService) GetOrderWithUser(ctx context.Context, orderID string) (*OrderWithUser, error) {
    // 1. Pobierz zamówienie lokalnie
    order, err := s.orderRepo.FindByID(ctx, orderID)
    if err != nil {
        return nil, fmt.Errorf("order not found: %w", err)
    }
    
    // 2. Pobierz użytkownika przez HTTP
    user, err := s.userClient.GetUser(ctx, order.UserID)
    if err != nil {
        return nil, fmt.Errorf("user service failed: %w", err)
    }
    
    return &OrderWithUser{
        Order: *order,
        User:  *user,
    }, nil
}

// UserClient - HTTP client dla User Service
type UserClient struct {
    baseURL string
    client  *http.Client
}

func (c *UserClient) GetUser(ctx context.Context, userID string) (*User, error) {
    url := fmt.Sprintf("%s/users/%s", c.baseURL, userID)
    
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    
    resp, err := c.client.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("HTTP %d", resp.StatusCode)
    }
    
    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, err
    }
    
    return &user, nil
}
```

### API Gateway Pattern

Gdy masz kilkanaście mikrousług, frontend nie może wywoływać każdej bezpośrednio. API Gateway agreguje wywołania.

```go
type APIGateway struct {
    userService    UserClient
    orderService   OrderClient
    paymentService PaymentClient
}

// Endpoint agregujący - jeden request, wiele serwisów
func (gw *APIGateway) GetUserDashboard(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    userID := getUserIDFromToken(r)
    
    // Równoległe wywołania
    type result struct {
        user     *User
        orders   []Order
        payments []Payment
        err      error
        source   string
    }
    
    results := make(chan result, 3)
    
    // Goroutine 1: pobierz użytkownika
    go func() {
        user, err := gw.userService.GetUser(ctx, userID)
        results <- result{user: user, err: err, source: "user"}
    }()
    
    // Goroutine 2: pobierz zamówienia
    go func() {
        orders, err := gw.orderService.GetUserOrders(ctx, userID)
        results <- result{orders: orders, err: err, source: "orders"}
    }()
    
    // Goroutine 3: pobierz płatności
    go func() {
        payments, err := gw.paymentService.GetUserPayments(ctx, userID)
        results <- result{payments: payments, err: err, source: "payments"}
    }()
    
    // Zbierz wyniki - graceful degradation
    var user *User
    var orders []Order
    var payments []Payment
    
    for i := 0; i < 3; i++ {
        res := <-results
        
        if res.err != nil {
            // Loguj błąd, ale nie przerywaj całego requestu
            log.Printf("%s service failed: %v", res.source, res.err)
            continue
        }
        
        if res.user != nil {
            user = res.user
        }
        if res.orders != nil {
            orders = res.orders
        }
        if res.payments != nil {
            payments = res.payments
        }
    }
    
    // Zwróć dane - nawet jeśli niektóre serwisy zawiodły
    dashboard := UserDashboard{
        User:     user,
        Orders:   orders,
        Payments: payments,
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(dashboard)
}
```

### Circuit Breaker - ochrona przed kaskadowymi awariami

Gdy jeden serwis ma problemy, nie powinien zabić całego systemu.

```go
type CircuitState int

const (
    Closed   CircuitState = iota // normalny stan
    Open                        // błędy - blokuj requesty
    HalfOpen                    // test recovery
)

type CircuitBreaker struct {
    maxFailures     int
    resetTimeout    time.Duration
    failureCount    int
    lastFailureTime time.Time
    state          CircuitState
    mu             sync.RWMutex
}

func NewCircuitBreaker(maxFailures int, resetTimeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        maxFailures:  maxFailures,
        resetTimeout: resetTimeout,
        state:        Closed,
    }
}

func (cb *CircuitBreaker) Call(operation func() error) error {
    cb.mu.RLock()
    state := cb.state
    failureCount := cb.failureCount
    lastFailure := cb.lastFailureTime
    cb.mu.RUnlock()
    
    // Sprawdź stan circuit breaker
    switch state {
    case Open:
        if time.Since(lastFailure) < cb.resetTimeout {
            return fmt.Errorf("circuit breaker is open")
        }
        // Przejdź do half-open
        cb.mu.Lock()
        cb.state = HalfOpen
        cb.mu.Unlock()
    }
    
    // Wykonaj operację
    err := operation()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failureCount++
        cb.lastFailureTime = time.Now()
        
        if cb.failureCount >= cb.maxFailures {
            cb.state = Open
        }
        return err
    }
    
    // Sukces - reset circuit breaker
    if cb.state == HalfOpen {
        cb.state = Closed
    }
    cb.failureCount = 0
    
    return nil
}

// Użycie w HTTP client
type ServiceClient struct {
    client  *http.Client
    baseURL string
    cb      *CircuitBreaker
}

func (c *ServiceClient) GetData(ctx context.Context, id string) (*Data, error) {
    var data *Data
    
    err := c.cb.Call(func() error {
        req, err := http.NewRequestWithContext(ctx, "GET", 
            fmt.Sprintf("%s/data/%s", c.baseURL, id), nil)
        if err != nil {
            return err
        }
        
        resp, err := c.client.Do(req)
        if err != nil {
            return err
        }
        defer resp.Body.Close()
        
        if resp.StatusCode >= 500 {
            return fmt.Errorf("server error: %d", resp.StatusCode)
        }
        
        return json.NewDecoder(resp.Body).Decode(&data)
    })
    
    if err != nil {
        // Fallback - zwróć cached data lub default
        return c.getFallbackData(id), nil
    }
    
    return data, nil
}
```

## Domain-Driven Design w praktyce

DDD to podejście do projektowania oprogramowania, które stawia domenę biznesową w centrum uwagi. W kontekście mikrousług DDD pomaga odpowiedzieć na kluczowe pytanie: **gdzie podzielić system?**

### Bounded Context - granice mikrousług

Bounded Context to kluczowy koncept DDD - to jasno zdefiniowany obszar, w którym konkretne modele mają jednoznaczne znaczenie. W mikrousługach każdy Bounded Context to potencjalny kandydat na osobny serwis.

**Jak znaleźć Bounded Context:**

1. **Ubiquitous Language** - każdy context ma swój język biznesowy
   - W contexcie "Order Management": "Customer" to kupujący
   - W contexcie "Support": "Customer" to osoba zgłaszająca ticket
   - To są różne modele, mimo tej samej nazwy

2. **Domain Events** - naturalne granice między contextami
   - "Order Placed" - przechodzi z Order Context do Inventory Context
   - "Payment Processed" - z Payment Context do Order Context

3. **Data ownership** - kto jest właścicielem danych
   - User Service - dane profilu, uwierzytelnianie
   - Order Service - historia zamówień, status
   - Payment Service - transakcje, billingi

```go
// Błąd - jeden model dla wszystkich kontekstów
type Customer struct {
    ID              string
    Name            string
    Email           string
    ShippingAddress Address    // Order Context
    BillingAddress  Address    // Payment Context  
    SupportTickets  []Ticket   // Support Context
    Preferences     Settings   // User Context
}

// Poprawnie - różne modele dla różnych kontekstów
// User Context
type User struct {
    ID          string
    Email       string
    Profile     Profile
    Preferences UserSettings
}

// Order Context
type Customer struct {
    ID              string
    Name            string
    ShippingAddress Address
    OrderHistory    []Order
}

// Support Context
type Contact struct {
    ID            string
    Name          string
    Email         string
    TicketHistory []Ticket
    Priority      SupportPriority
}
```

### Strategic Design - architektura systemów

**Context Map** - jak konteksty się komunikują:

```go
// Upstream/Downstream relationship
type OrderService struct {
    // Downstream od User Service - konsumuje user data
    userClient UserClient
    
    // Upstream dla Shipping Service - dostarcza order data  
    eventBus EventBus
}

// Customer-Supplier pattern
func (s *OrderService) CreateOrder(ctx context.Context, userID string) error {
    // Wywołanie do Upstream service (User)
    user, err := s.userClient.GetUser(ctx, userID)
    if err != nil {
        return fmt.Errorf("cannot create order without user: %w", err)
    }
    
    order := NewOrder(userID, user.ShippingAddress)
    
    // Publikacja dla Downstream services
    return s.eventBus.Publish(ctx, OrderCreatedEvent{
        OrderID: order.ID,
        UserID:  userID,
    })
}

// Anti-Corruption Layer - ochrona przed external APIs
type ExternalPaymentAdapter struct {
    stripeClient *stripe.Client
}

func (a *ExternalPaymentAdapter) ProcessPayment(ctx context.Context, payment DomainPayment) (*DomainPaymentResult, error) {
    // Tłumaczenie z domain model na external API
    stripePayment := &stripe.PaymentIntentParams{
        Amount:   stripe.Int64(payment.AmountCents()),
        Currency: stripe.String(string(payment.Currency())),
    }
    
    result, err := a.stripeClient.PaymentIntents.New(stripePayment)
    if err != nil {
        return nil, err
    }
    
    // Tłumaczenie z external API na domain model
    return &DomainPaymentResult{
        TransactionID: result.ID,
        Status:        mapStripeStatus(result.Status),
        Amount:        payment.Amount(),
    }, nil
}
```

### Tactical Design - wzorce wewnątrz kontekstu

DDD pomaga wydzielić granice mikrousług i modelować logikę biznesową.

### Value Objects - niezmienne wartości

```go
// Email - value object z walidacją
type Email struct {
    value string
}

func NewEmail(email string) (Email, error) {
    if !isValidEmail(email) {
        return Email{}, errors.New("invalid email format")
    }
    return Email{value: strings.ToLower(email)}, nil
}

func (e Email) String() string {
    return e.value
}

func (e Email) Domain() string {
    parts := strings.Split(e.value, "@")
    if len(parts) == 2 {
        return parts[1]
    }
    return ""
}

// Money - value object z walutą
type Money struct {
    amount   int64  // w groszach
    currency string
}

func NewMoney(amount float64, currency string) Money {
    return Money{
        amount:   int64(amount * 100),
        currency: strings.ToUpper(currency),
    }
}

func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, errors.New("cannot add different currencies")
    }
    
    return Money{
        amount:   m.amount + other.amount,
        currency: m.currency,
    }, nil
}

func (m Money) Amount() float64 {
    return float64(m.amount) / 100
}
```

### Aggregates - spójna logika biznesowa

```go
// Order - aggregate root
type Order struct {
    id       string
    userID   string
    items    []OrderItem
    total    Money
    status   OrderStatus
    events   []DomainEvent
}

func NewOrder(userID string, items []OrderItem) (*Order, error) {
    if len(items) == 0 {
        return nil, errors.New("order must have items")
    }
    
    order := &Order{
        id:     generateID(),
        userID: userID,
        items:  items,
        status: OrderStatusDraft,
        events: []DomainEvent{},
    }
    
    // Przelicz total
    if err := order.calculateTotal(); err != nil {
        return nil, err
    }
    
    // Dodaj domain event
    order.addEvent(OrderCreatedEvent{
        OrderID: order.id,
        UserID:  order.userID,
        Items:   order.items,
        Total:   order.total,
    })
    
    return order, nil
}

func (o *Order) AddItem(item OrderItem) error {
    if o.status != OrderStatusDraft {
        return errors.New("cannot modify confirmed order")
    }
    
    o.items = append(o.items, item)
    return o.calculateTotal()
}

func (o *Order) Confirm() error {
    if o.status != OrderStatusDraft {
        return errors.New("order already confirmed")
    }
    
    o.status = OrderStatusConfirmed
    o.addEvent(OrderConfirmedEvent{
        OrderID: o.id,
        Total:   o.total,
    })
    
    return nil
}

func (o *Order) calculateTotal() error {
    total := NewMoney(0, "PLN")
    
    for _, item := range o.items {
        itemTotal := NewMoney(item.Price*float64(item.Quantity), "PLN")
        var err error
        total, err = total.Add(itemTotal)
        if err != nil {
            return err
        }
    }
    
    o.total = total
    return nil
}

func (o *Order) addEvent(event DomainEvent) {
    o.events = append(o.events, event)
}

func (o *Order) GetEvents() []DomainEvent {
    return o.events
}

func (o *Order) ClearEvents() {
    o.events = []DomainEvent{}
}
```

### Repository Pattern

```go
type OrderRepository interface {
    Save(ctx context.Context, order *Order) error
    FindByID(ctx context.Context, id string) (*Order, error)
    FindByUserID(ctx context.Context, userID string) ([]Order, error)
}

// PostgreSQL implementation
type PostgresOrderRepository struct {
    db       *sql.DB
    eventBus EventBus
}

func (r *PostgresOrderRepository) Save(ctx context.Context, order *Order) error {
    tx, err := r.db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    defer tx.Rollback()
    
    // Zapisz aggregate
    query := `
        INSERT INTO orders (id, user_id, status, total_amount, total_currency, items, created_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        ON CONFLICT (id) DO UPDATE SET
            status = EXCLUDED.status,
            total_amount = EXCLUDED.total_amount,
            items = EXCLUDED.items,
            updated_at = NOW()
    `
    
    itemsJSON, _ := json.Marshal(order.items)
    
    _, err = tx.ExecContext(ctx, query,
        order.id,
        order.userID,
        order.status,
        order.total.amount,
        order.total.currency,
        itemsJSON,
        time.Now(),
    )
    if err != nil {
        return fmt.Errorf("save order: %w", err)
    }
    
    // Publikuj domain events
    events := order.GetEvents()
    for _, event := range events {
        if err := r.eventBus.Publish(ctx, event); err != nil {
            return fmt.Errorf("publish event: %w", err)
        }
    }
    
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit transaction: %w", err)
    }
    
    order.ClearEvents()
    return nil
}
```

## CQRS - teoria i praktyka

Command Query Responsibility Segregation to wzorzec architektoniczny wprowadzony przez Grega Younga, oparty na zasadzie CQS (Command Query Separation) Bertanda Meyera.

### Teoria CQRS

**Podstawowa zasada:**
- **Commands** - zmieniają stan systemu, ale nic nie zwracają
- **Queries** - zwracają dane, ale nie zmieniają stanu

**Dlaczego CQRS ma sens w mikrousługach:**

1. **Różne wymagania read vs write**
   - Write: konsystencja, walidacja, transaction
   - Read: wydajność, denormalizacja, caching

2. **Różne modele danych**
   - Write model: normalized, domain-driven
   - Read model: denormalized, query-optimized

3. **Różne skalowanie**
   - Write: mało operacji, wysokie wymagania spójności
   - Read: dużo operacji, eventual consistency OK

```go
// Tradycyjny CRUD - jeden model dla wszystkiego
type OrderService struct {
    repo OrderRepository
}

type Order struct {
    ID       string
    UserID   string
    Items    []OrderItem
    Status   OrderStatus
    // ... 20 innych pól dla różnych use cases
}

func (s *OrderService) CreateOrder(order *Order) error {
    // Skomplikowany model dla write
    return s.repo.Save(order)
}

func (s *OrderService) GetOrdersForDashboard(userID string) ([]Order, error) {
    // Ten sam skomplikowany model dla read
    // Musi załadować wszystkie pola nawet jeśli dashboard potrzebuje tylko 3
    return s.repo.FindByUserID(userID)
}

// CQRS - różne modele i serwisy
// Write Side - Command Model
type CreateOrderCommand struct {
    UserID string
    Items  []OrderItem
}

type Order struct {
    id       string
    userID   string
    items    []OrderItem
    status   OrderStatus
    events   []DomainEvent
    // Tylko pola potrzebne do business logic
}

// Read Side - Query Model
type OrderDashboardQuery struct {
    UserID string
    Limit  int
}

type OrderDashboardView struct {
    ID          string
    Status      string
    TotalAmount float64
    ItemCount   int
    CreatedAt   time.Time
    // Tylko pola potrzebne do wyświetlenia
}
```

### Event Sourcing - teoria i implementacja

Event Sourcing to wzorzec gdzie zamiast przechowywać aktualny stan obiektu, przechowujesz sekwencję zdarzeń które doprowadziły do tego stanu.

**Kluczowe koncepty:**

1. **Event** - niezmienne zdarzenie które miało miejsce
2. **Event Store** - baza danych eventów
3. **Aggregate** - obiekt odbudowywany z eventów
4. **Projection** - materialized view zbudowany z eventów

```go
// Tradycyjne CRUD - state-based
type Order struct {
    ID     string
    Status OrderStatus
    Items  []OrderItem
    Total  Money
}

func (s *OrderService) UpdateStatus(orderID string, status OrderStatus) error {
    order := s.repo.FindByID(orderID)
    order.Status = status  // Tracisz historię - dlaczego się zmieniło?
    return s.repo.Save(order)
}

// Event Sourcing - event-based
type OrderCreatedEvent struct {
    OrderID   string
    UserID    string
    Items     []OrderItem
    Timestamp time.Time
}

type OrderStatusChangedEvent struct {
    OrderID   string
    OldStatus OrderStatus
    NewStatus OrderStatus
    Reason    string
    Timestamp time.Time
    ChangedBy string
}

// Event Store
type EventStore interface {
    Append(ctx context.Context, aggregateID string, events []Event) error
    GetEvents(ctx context.Context, aggregateID string, fromVersion int) ([]Event, error)
}

// Aggregate odbudowywany z eventów
func (o *Order) Apply(event Event) error {
    switch e := event.(type) {
    case OrderCreatedEvent:
        o.id = e.OrderID
        o.userID = e.UserID
        o.items = e.Items
        o.status = OrderStatusDraft
        o.createdAt = e.Timestamp
        
    case OrderStatusChangedEvent:
        o.status = e.NewStatus
        o.lastModified = e.Timestamp
        
    default:
        return fmt.Errorf("unknown event: %T", event)
    }
    
    o.version++
    return nil
}

// Projekcje - read models zbudowane z eventów
type OrderProjection struct {
    db *sql.DB
}

func (p *OrderProjection) Handle(ctx context.Context, event Event) error {
    switch e := event.(type) {
    case OrderCreatedEvent:
        // Wstaw do denormalized table
        query := `
            INSERT INTO order_projections 
            (id, user_id, status, total_amount, created_at)
            VALUES ($1, $2, $3, $4, $5)`
        
        total := calculateTotal(e.Items)
        _, err := p.db.ExecContext(ctx, query,
            e.OrderID, e.UserID, "draft", total, e.Timestamp)
        return err
        
    case OrderStatusChangedEvent:
        query := `UPDATE order_projections SET status = $1 WHERE id = $2`
        _, err := p.db.ExecContext(ctx, query, e.NewStatus, e.OrderID)
        return err
    }
    
    return nil
}
```

**Zalety Event Sourcing:**
- **Audit log** - pełna historia zmian
- **Time travel** - stan systemu w dowolnym momencie
- **Debugging** - replay eventów do odtworzenia błędu
- **Analytics** - rich data dla business intelligence

**Wady Event Sourcing:**
- **Complexity** - więcej kodu niż CRUD
- **Schema evolution** - jak zmieniać format eventów
- **Storage** - eventy nigdy nie znikają
- **Performance** - odbudowa z eventów może być wolna

CQRS oddziela operacje zmieniające stan (Commands) od operacji odczytujących (Queries).

### Commands - zmieniają stan

```go
// Command
type CreateOrderCommand struct {
    UserID string
    Items  []OrderItem
}

// Command Handler
type CreateOrderHandler struct {
    orderRepo OrderRepository
}

func (h *CreateOrderHandler) Handle(ctx context.Context, cmd CreateOrderCommand) error {
    order, err := NewOrder(cmd.UserID, cmd.Items)
    if err != nil {
        return fmt.Errorf("create order: %w", err)
    }
    
    if err := h.orderRepo.Save(ctx, order); err != nil {
        return fmt.Errorf("save order: %w", err)
    }
    
    return nil
}
```

### Queries - tylko czytają

```go
// Query
type GetUserOrdersQuery struct {
    UserID string
    Limit  int
    Offset int
}

// Query Handler - może używać zoptymalizowanej bazy read-only
type GetUserOrdersHandler struct {
    readDB *sql.DB
}

func (h *GetUserOrdersHandler) Handle(ctx context.Context, query GetUserOrdersQuery) ([]OrderView, error) {
    // Denormalized query - szybkie odczyty
    querySQL := `
        SELECT 
            o.id,
            o.user_id,
            u.name as user_name,
            o.status,
            o.total_amount,
            o.total_currency,
            o.created_at
        FROM orders_view o
        JOIN users_view u ON o.user_id = u.id
        WHERE o.user_id = $1
        ORDER BY o.created_at DESC
        LIMIT $2 OFFSET $3
    `
    
    rows, err := h.readDB.QueryContext(ctx, querySQL, query.UserID, query.Limit, query.Offset)
    if err != nil {
        return nil, fmt.Errorf("query orders: %w", err)
    }
    defer rows.Close()
    
    var orders []OrderView
    for rows.Next() {
        var order OrderView
        err := rows.Scan(
            &order.ID,
            &order.UserID,
            &order.UserName,
            &order.Status,
            &order.TotalAmount,
            &order.TotalCurrency,
            &order.CreatedAt,
        )
        if err != nil {
            return nil, fmt.Errorf("scan order: %w", err)
        }
        orders = append(orders, order)
    }
    
    return orders, nil
}
```

### Mediator Pattern - routing requests

```go
type Mediator struct {
    commandHandlers map[string]CommandHandler
    queryHandlers   map[string]QueryHandler
}

func (m *Mediator) Send(ctx context.Context, command interface{}) error {
    cmdType := reflect.TypeOf(command).Name()
    handler, exists := m.commandHandlers[cmdType]
    if !exists {
        return fmt.Errorf("no handler for command %s", cmdType)
    }
    
    return handler.Handle(ctx, command)
}

func (m *Mediator) Query(ctx context.Context, query interface{}) (interface{}, error) {
    queryType := reflect.TypeOf(query).Name()
    handler, exists := m.queryHandlers[queryType]
    if !exists {
        return nil, fmt.Errorf("no handler for query %s", queryType)
    }
    
    return handler.Handle(ctx, query)
}

// HTTP Controller używa mediatora
type OrderController struct {
    mediator *Mediator
}

func (c *OrderController) CreateOrder(w http.ResponseWriter, r *http.Request) {
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }
    
    command := CreateOrderCommand{
        UserID: getUserIDFromContext(r.Context()),
        Items:  req.Items,
    }
    
    if err := c.mediator.Send(r.Context(), command); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    w.WriteHeader(http.StatusCreated)
}
```

## Hexagonal Architecture - teoria i implementacja

Hexagonal Architecture (znana też jako Ports & Adapters) została wprowadzona przez Alistair Cockburna w 2005 roku. Głównym celem jest izolacja logiki biznesowej od szczegółów technologicznych.

### Teoria Hexagonal Architecture

**Kluczowe koncepty:**

1. **Hexagon (Core)** - logika biznesowa aplikacji
2. **Ports** - interfejsy definiujące co potrzebuje aplikacja
3. **Adapters** - implementacje portów dla konkretnych technologii
4. **Primary Adapters** - inicjują interakcję (HTTP controllers, CLI)
5. **Secondary Adapters** - są wywoływane przez aplikację (databases, external APIs)

**Filozofia:**
- Aplikacja nie wie nic o tym czy jest wywoływana przez HTTP, CLI czy testy
- Aplikacja nie wie nic o tym czy dane są w PostgreSQL, MongoDB czy in-memory
- Logika biznesowa jest w centrum, otoczona interfejsami

```go
// Tradycyjna architektura - tight coupling
type OrderService struct {
    db           *sql.DB           // Bezpośrednia zależność od Postgres
    httpClient   *http.Client      // Bezpośrednia zależność od HTTP
    logger       *logrus.Logger    // Bezpośrednia zależność od Logrus
}

func (s *OrderService) CreateOrder(w http.ResponseWriter, r *http.Request) {
    // Logika biznesowa zmieszana z HTTP handling
    var request CreateOrderRequest
    json.NewDecoder(r.Body).Decode(&request)
    
    // Bezpośrednie SQL query
    _, err := s.db.Exec("INSERT INTO orders...")
    if err != nil {
        // Bezpośrednie HTTP response
        http.Error(w, "Database error", 500)
        return
    }
    
    // Bezpośredni HTTP call
    resp, err := s.httpClient.Post("http://payment-service/charge", ...)
    
    w.WriteHeader(201)
}

// Hexagonal Architecture - loose coupling
// 1. Core Domain (center of hexagon)
type OrderService struct {
    repo         OrderRepository    // Port
    paymentGW    PaymentGateway    // Port
    eventBus     EventPublisher    // Port
    logger       Logger            // Port
}

// 2. Ports (interfaces)
type OrderRepository interface {
    Save(ctx context.Context, order *Order) error
    FindByID(ctx context.Context, id string) (*Order, error)
}

type PaymentGateway interface {
    ProcessPayment(ctx context.Context, payment Payment) (*PaymentResult, error)
}

// 3. Use Cases (application services)
func (s *OrderService) CreateOrder(ctx context.Context, cmd CreateOrderCommand) (*Order, error) {
    // Czysta logika biznesowa - bez szczegółów technicznych
    order, err := NewOrder(cmd.UserID, cmd.Items)
    if err != nil {
        return nil, fmt.Errorf("invalid order: %w", err)
    }
    
    // Wywołanie przez port
    payment := Payment{Amount: order.Total(), OrderID: order.ID()}
    _, err = s.paymentGW.ProcessPayment(ctx, payment)
    if err != nil {
        return nil, fmt.Errorf("payment failed: %w", err)
    }
    
    // Wywołanie przez port
    if err := s.repo.Save(ctx, order); err != nil {
        return nil, fmt.Errorf("save failed: %w", err)
    }
    
    return order, nil
}

// 4. Primary Adapter (HTTP)
type OrderController struct {
    orderService *OrderService
}

func (c *OrderController) CreateOrder(w http.ResponseWriter, r *http.Request) {
    // Tylko HTTP handling - logika biznesowa w service
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }
    
    // Konwersja HTTP request -> domain command
    cmd := CreateOrderCommand{
        UserID: getUserIDFromToken(r),
        Items:  req.Items,
    }
    
    // Wywołanie use case
    order, err := c.orderService.CreateOrder(r.Context(), cmd)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    // Konwersja domain -> HTTP response
    response := CreateOrderResponse{
        OrderID: order.ID(),
        Status:  string(order.Status()),
    }
    
    json.NewEncoder(w).Encode(response)
}

// 5. Secondary Adapters
type PostgresOrderRepository struct {
    db *sql.DB
}

func (r *PostgresOrderRepository) Save(ctx context.Context, order *Order) error {
    // Implementacja dla konkretnej bazy
    query := `INSERT INTO orders (id, user_id, status) VALUES ($1, $2, $3)`
    _, err := r.db.ExecContext(ctx, query, order.ID(), order.UserID(), order.Status())
    return err
}

type StripePaymentGateway struct {
    apiKey string
    client *http.Client
}

func (g *StripePaymentGateway) ProcessPayment(ctx context.Context, payment Payment) (*PaymentResult, error) {
    // Implementacja dla konkretnego payment gateway
    // ...
}
```

**Zalety Hexagonal Architecture:**

1. **Testability** - można mockować wszystkie porty
2. **Technology agnostic** - łatwo zmienić database/framework
3. **Clear boundaries** - jasny podział odpowiedzialności  
4. **Screaming architecture** - use cases są widoczne na pierwszy rzut oka

**Dependency Inversion w praktyce:**

```go
// Dependency Injection Container
type Container struct {
    orderRepo    OrderRepository
    paymentGW    PaymentGateway  
    eventBus     EventPublisher
    orderService *OrderService
}

func NewContainer() *Container {
    // Konfiguracja adapters
    db := connectToDatabase()
    orderRepo := &PostgresOrderRepository{db: db}
    paymentGW := &StripePaymentGateway{apiKey: "sk_test_..."}
    eventBus := &KafkaEventBus{brokers: []string{"localhost:9092"}}
    
    // Injection dependencies do core
    orderService := &OrderService{
        repo:      orderRepo,
        paymentGW: paymentGW,
        eventBus:  eventBus,
    }
    
    return &Container{
        orderRepo:    orderRepo,
        paymentGW:    paymentGW,
        eventBus:     eventBus,
        orderService: orderService,
    }
}

// Test configuration - in-memory adapters
func NewTestContainer() *Container {
    return &Container{
        orderRepo:    &InMemoryOrderRepository{},
        paymentGW:    &MockPaymentGateway{},
        eventBus:     &MockEventBus{},
        orderService: orderService, // Ta sama logika biznesowa!
    }
}
```

Hexagonal Architecture oddziela logikę biznesową od szczegółów technicznych.

```go
// Core Domain - nie zależy od niczego zewnętrznego
type OrderService struct {
    repo     OrderRepository    // port
    payment  PaymentGateway     // port
    events   EventPublisher     // port
}

// Ports - interfejsy definiujące co potrzebuje domain
type OrderRepository interface {
    Save(ctx context.Context, order *Order) error
    FindByID(ctx context.Context, id string) (*Order, error)
}

type PaymentGateway interface {
    ProcessPayment(ctx context.Context, payment Payment) (*PaymentResult, error)
}

// Use Case
func (s *OrderService) CreateOrder(ctx context.Context, userID string, items []OrderItem) (*Order, error) {
    // Logika biznesowa
    order, err := NewOrder(userID, items)
    if err != nil {
        return nil, fmt.Errorf("invalid order: %w", err)
    }
    
    // Przetwórz płatność
    payment := Payment{
        Amount:  order.total.Amount(),
        OrderID: order.id,
    }
    
    paymentResult, err := s.payment.ProcessPayment(ctx, payment)
    if err != nil {
        return nil, fmt.Errorf("payment failed: %w", err)
    }
    
    // Potwierdź zamówienie
    order.status = OrderStatusPaid
    
    // Zapisz
    if err := s.repo.Save(ctx, order); err != nil {
        return nil, fmt.Errorf("save failed: %w", err)
    }
    
    // Publikuj event
    s.events.Publish(ctx, OrderPaidEvent{
        OrderID:       order.id,
        PaymentID:     paymentResult.ID,
        Amount:        order.total,
    })
    
    return order, nil
}

// Adaptery - implementacje portów (outer ring)

// HTTP Adapter (controller)
type OrderController struct {
    orderService *OrderService
}

func (c *OrderController) CreateOrder(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    userID := getUserIDFromContext(ctx)
    
    var req CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }
    
    order, err := c.orderService.CreateOrder(ctx, userID, req.Items)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    response := CreateOrderResponse{
        OrderID:   order.id,
        Status:    string(order.status),
        Total:     order.total.Amount(),
        CreatedAt: time.Now(),
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

// External Service Adapter
type StripePaymentGateway struct {
    client *http.Client
    apiKey string
}

func (g *StripePaymentGateway) ProcessPayment(ctx context.Context, payment Payment) (*PaymentResult, error) {
    payload := map[string]interface{}{
        "amount":   int(payment.Amount * 100), // cents
        "currency": "pln",
        "metadata": map[string]string{
            "order_id": payment.OrderID,
        },
    }
    
    payloadBytes, _ := json.Marshal(payload)
    
    req, err := http.NewRequestWithContext(ctx, "POST", 
        "https://api.stripe.com/v1/payment_intents", 
        bytes.NewReader(payloadBytes))
    if err != nil {
        return nil, err
    }
    
    req.Header.Set("Authorization", "Bearer "+g.apiKey)
    req.Header.Set("Content-Type", "application/json")
    
    resp, err := g.client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("stripe request failed: %w", err)
    }
    defer resp.Body.Close()
    
    var result struct {
        ID     string `json:"id"`
        Status string `json:"status"`
    }
    
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, err
    }
    
    return &PaymentResult{
        ID:     result.ID,
        Status: result.Status,
    }, nil
}
```

## Service mesh - teoria i implementacja

Service mesh to dedykowana warstwa infrastruktury do obsługi komunikacji między serwisami. Składa się z network proxies (data plane) i control plane zarządzającego konfiguracją.

### Architektura Service Mesh

**Data Plane:**
- **Sidecar proxy** (np. Envoy) przy każdym serwisie
- Przechwytuje cały network traffic
- Implementuje polityki security, routing, load balancing

**Control Plane:**
- Zarządza konfiguracją proxy
- Implementuje service discovery
- Zbiera telemetrię i metryki
- Zarządza certificatami (mTLS)

```go
// Problem: Cross-cutting concerns w kodzie aplikacji
type OrderService struct {
    userClient UserClient
}

func (s *OrderService) GetUser(ctx context.Context, userID string) (*User, error) {
    // 1. Service Discovery - gdzie jest User Service?
    userServiceURL, err := s.discoveryClient.Resolve("user-service")
    if err != nil {
        return nil, err
    }
    
    // 2. Load Balancing - który instance wybrać?
    instance := s.loadBalancer.PickInstance(userServiceURL)
    
    // 3. Circuit Breaking - czy service jest healthy?
    if s.circuitBreaker.IsOpen() {
        return nil, errors.New("circuit breaker open")
    }
    
    // 4. Retry Logic
    var lastErr error
    for attempt := 0; attempt < 3; attempt++ {
        // 5. Timeout
        ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
        defer cancel()
        
        // 6. Tracing - propagate trace context
        span, ctx := opentracing.StartSpanFromContext(ctx, "get_user")
        defer span.Finish()
        
        // 7. Security - mTLS certificates
        cert, err := s.certManager.GetCertificate()
        if err != nil {
            return nil, err
        }
        
        client := &http.Client{
            Transport: &http.Transport{
                TLSClientConfig: &tls.Config{
                    Certificates: []tls.Certificate{cert},
                },
            },
        }
        
        // 8. Metrics
        start := time.Now()
        defer func() {
            s.metrics.RecordLatency("user_service_call", time.Since(start))
        }()
        
        // 9. Actual business call (finally!)
        req, _ := http.NewRequestWithContext(ctx, "GET", 
            fmt.Sprintf("%s/users/%s", instance, userID), nil)
        
        resp, err := client.Do(req)
        if err != nil {
            lastErr = err
            if attempt < 2 {
                time.Sleep(time.Duration(attempt+1) * time.Second)
                continue
            }
            s.circuitBreaker.RecordFailure()
            return nil, err
        }
        
        s.circuitBreaker.RecordSuccess()
        
        var user User
        json.NewDecoder(resp.Body).Decode(&user)
        return &user, nil
    }
    
    return nil, lastErr
}

// Rozwiązanie: Service Mesh przenosi complexity do infrastruktury
type OrderService struct {
    httpClient *http.Client
}

func (s *OrderService) GetUser(ctx context.Context, userID string) (*User, error) {
    // Service mesh (Envoy sidecar) automatycznie obsługuje:
    // - Service discovery: user-service -> 10.0.1.123:8080
    // - Load balancing: round-robin/least-connection/random
    // - Circuit breaking: 5xx errors -> open circuit
    // - Retries: exponential backoff, max 3 attempts
    // - Timeouts: 30s request timeout
    // - Security: automatic mTLS between services
    // - Observability: metrics, logs, traces
    // - Rate limiting: 1000 RPS per service
    
    req, err := http.NewRequestWithContext(ctx, "GET", 
        "http://user-service/users/"+userID, nil)
    if err != nil {
        return nil, err
    }
    
    // Sidecar proxy intercepts this call
    resp, err := s.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var user User
    return &user, json.NewDecoder(resp.Body).Decode(&user)
}
```

### Service Mesh w Go - konfiguracja produkcyjna

**Istio Configuration:**
```yaml
# VirtualService - routing rules
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service
spec:
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: user-service
        subset: v2
      weight: 100
  - route:
    - destination:
        host: user-service
        subset: v1
      weight: 90
    - destination:
        host: user-service
        subset: v2
      weight: 10
  retries:
    attempts: 3
    perTryTimeout: 2s
  timeout: 10s

---
# DestinationRule - load balancing & circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: user-service
spec:
  host: user-service
  trafficPolicy:
    circuitBreaker:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 2
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**Go Service dla Service Mesh:**
```go
func main() {
    // Minimal setup - service mesh handles complexity
    service := &UserService{
        repo: &PostgresUserRepo{db: connectDB()},
    }
    
    router := chi.NewRouter()
    
    // Health checks dla Kubernetes + Istio
    router.Get("/health/live", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        fmt.Fprint(w, "OK")
    })
    
    router.Get("/health/ready", func(w http.ResponseWriter, r *http.Request) {
        if err := service.repo.Ping(); err != nil {
            http.Error(w, "Not Ready", http.StatusServiceUnavailable)
            return
        }
        w.WriteHeader(http.StatusOK)
        fmt.Fprint(w, "Ready")
    })
    
    // Business endpoints
    router.Get("/users/{id}", service.GetUser)
    router.Post("/users", service.CreateUser)
    
    server := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }
    
    // Graceful shutdown - ważne dla service mesh
    gracefulShutdown(server)
}

func gracefulShutdown(server *http.Server) {
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, os.Interrupt, syscall.SIGTERM)
    
    go func() {
        log.Println("Starting server on :8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()
    
    <-stop
    log.Println("Received shutdown signal")
    
    // Grace period - pozwól service mesh zaktualizować load balancer
    // Istio potrzebuje czasu na:
    // 1. Oznaczenie poda jako "not ready"
    // 2. Aktualizację konfiguracji we wszystkich proxy
    // 3. Drain istniejących połączeń
    time.Sleep(15 * time.Second)
    
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    log.Println("Shutting down server...")
    if err := server.Shutdown(ctx); err != nil {
        log.Fatal("Forced shutdown:", err)
    }
    
    log.Println("Server stopped gracefully")
}
```

**Observability w Service Mesh:**
```go
// Service mesh automatycznie zbiera metryki
// Ale możesz dodać custom business metrics

import "github.com/prometheus/client_golang/prometheus"

var (
    orderCreated = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "orders_created_total",
            Help: "Total number of orders created",
        },
        []string{"user_type", "payment_method"},
    )
    
    orderValue = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "order_value_dollars",
            Help:    "Order value distribution",
            Buckets: prometheus.LinearBuckets(0, 50, 20), // 0-1000$ in 50$ buckets
        },
        []string{"user_type"},
    )
)

func (s *OrderService) CreateOrder(ctx context.Context, cmd CreateOrderCommand) error {
    order, err := s.processOrder(cmd)
    if err != nil {
        return err
    }
    
    // Business metrics - service mesh zbiera technical metrics
    orderCreated.WithLabelValues(
        order.User.Type,
        string(order.PaymentMethod),
    ).Inc()
    
    orderValue.WithLabelValues(order.User.Type).Observe(order.Total.Amount())
    
    return nil
}
```

**Zalety Service Mesh:**
1. **Separation of concerns** - business logic oddzielony od network concerns
2. **Consistency** - te same polityki dla wszystkich serwisów
3. **Centralized configuration** - zmiana polityk bez redeployment
4. **Security by default** - automatic mTLS
5. **Rich observability** - out-of-the-box metrics, traces, logs

**Wady Service Mesh:**
1. **Complexity** - dodatkowa warstwa do zarządzania
2. **Performance overhead** - proxy adds latency (~1-3ms)
3. **Learning curve** - nowe koncepty do opanowania
4. **Debugging** - więcej moving parts

Service mesh obsługuje komunikację między serwisami na poziomie infrastruktury.

```go
// Bez service mesh - dużo boilerplate'u
type UserService struct {
    httpClient     *http.Client
    circuitBreaker *CircuitBreaker
    retryPolicy    RetryPolicy
    metrics        Metrics
    tracer         opentracing.Tracer
}

func (s *UserService) GetUser(ctx context.Context, id int) (*User, error) {
    // Manual circuit breaker
    return s.circuitBreaker.Call(func() error {
        // Manual retry
        for attempt := 0; attempt < s.retryPolicy.MaxAttempts; attempt++ {
            // Manual timeout
            reqCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
            defer cancel()
            
            // Manual tracing
            span, reqCtx := opentracing.StartSpanFromContext(reqCtx, "get_user")
            defer span.Finish()
            
            // Manual metrics
            start := time.Now()
            defer func() {
                s.metrics.RecordLatency("user_service.get_user", time.Since(start))
            }()
            
            // Actual request
            req, err := http.NewRequestWithContext(reqCtx, "GET", 
                fmt.Sprintf("http://user-service/users/%d", id), nil)
            if err != nil {
                return err
            }
            
            // Manual correlation IDs
            req.Header.Set("X-Trace-ID", getTraceID(ctx))
            req.Header.Set("X-Request-ID", getRequestID(ctx))
            
            resp, err := s.httpClient.Do(req)
            if err != nil {
                if attempt < s.retryPolicy.MaxAttempts-1 {
                    time.Sleep(s.retryPolicy.BackoffDuration(attempt))
                    continue
                }
                return err
            }
            defer resp.Body.Close()
            
            if resp.StatusCode >= 500 {
                if attempt < s.retryPolicy.MaxAttempts-1 {
                    time.Sleep(s.retryPolicy.BackoffDuration(attempt))
                    continue
                }
                return fmt.Errorf("server error: %d", resp.StatusCode)
            }
            
            var user User
            return json.NewDecoder(resp.Body).Decode(&user)
        }
        return nil
    })
}

// Z service mesh - czysty business logic
type UserService struct {
    httpClient *http.Client
}

func (s *UserService) GetUser(ctx context.Context, id int) (*User, error) {
    // Service mesh (Istio/Envoy) automatycznie obsługuje:
    // - Load balancing
    // - Circuit breaking
    // - Retries i timeouts  
    // - Metrics i tracing
    // - mTLS
    // - Rate limiting
    
    req, err := http.NewRequestWithContext(ctx, "GET", 
        fmt.Sprintf("http://user-service/users/%d", id), nil)
    if err != nil {
        return nil, err
    }
    
    resp, err := s.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    var user User
    return &user, json.NewDecoder(resp.Body).Decode(&user)
}

// Health check endpoints dla service mesh
func setupHealthChecks() {
    http.HandleFunc("/health/live", func(w http.ResponseWriter, r *http.Request) {
        // Liveness probe - czy aplikacja żyje?
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusOK)
        fmt.Fprint(w, `{"status":"ok"}`)
    })
    
    http.HandleFunc("/health/ready", func(w http.ResponseWriter, r *http.Request) {
        // Readiness probe - czy aplikacja gotowa obsługiwać traffic?
        ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
        defer cancel()
        
        // Sprawdź zależności
        if err := checkDatabase(ctx); err != nil {
            http.Error(w, `{"status":"not ready","reason":"database"}`, 
                http.StatusServiceUnavailable)
            return
        }
        
        if err := checkExternalServices(ctx); err != nil {
            http.Error(w, `{"status":"not ready","reason":"external_services"}`, 
                http.StatusServiceUnavailable)
            return
        }
        
        w.Header().Set("Content-Type", "application/json")
        fmt.Fprint(w, `{"status":"ready"}`)
    })
}

// Graceful shutdown dla service mesh
func gracefulShutdown(server *http.Server) {
    stop := make(chan os.Signal, 1)
    signal.Notify(stop, os.Interrupt, syscall.SIGTERM)
    
    go func() {
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()
    
    <-stop
    log.Println("Shutting down server...")
    
    // Grace period - pozwól service mesh update load balancer
    time.Sleep(15 * time.Second)
    
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := server.Shutdown(ctx); err != nil {
        log.Fatal("Server forced shutdown:", err)
    }
    
    log.Println("Server stopped")
}
```

## Weryfikacja kodu z AI

AI często generuje kod mikrousług, który wygląda dobrze, ale ma fundamentalne problemy. Oto czego unikać:

### 1. Tight coupling między serwisami

```go
// AI może wygenerować bezpośrednie zależności
type OrderService struct {
    userService *UserService  // BŁĄD - bezpośrednia zależność!
}

func (s *OrderService) CreateOrder(order *Order) error {
    user := s.userService.GetUser(order.UserID)  // synchroniczne wywołanie
    if !user.IsActive {
        return errors.New("user not active")
    }
    return s.save(order)
}

// Poprawnie - asynchroniczne z fallback
type OrderService struct {
    userClient UserClient
    cache      UserCache
}

func (s *OrderService) CreateOrder(ctx context.Context, order *Order) error {
    // Spróbuj cache
    user, err := s.cache.GetUser(order.UserID)
    if err != nil {
        // Fallback z timeoutem
        ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
        defer cancel()
        
        user, err = s.userClient.GetUser(ctx, order.UserID)
        if err != nil {
            // Nie blokuj zamówienia - sprawdź użytkownika później
            log.Printf("user service unavailable: %v", err)
            return s.createOrderWithoutUserCheck(ctx, order)
        }
    }
    
    if !user.IsActive {
        return errors.New("user not active")
    }
    
    return s.save(ctx, order)
}
```

### 2. Shared database antipattern

```go
// AI może wygenerować shared database
func (s *OrderService) GetOrdersWithUsers() ([]OrderWithUser, error) {
    // BŁĄD - JOIN między domenami
    query := `
        SELECT o.*, u.name, u.email
        FROM orders o
        JOIN users u ON o.user_id = u.id
    `
    // To łamie bounded context!
}

// Poprawnie - oddzielne bazy + API calls
func (s *OrderService) GetOrdersWithUsers(ctx context.Context, userID string) ([]OrderWithUser, error) {
    orders, err := s.orderRepo.FindByUserID(ctx, userID)
    if err != nil {
        return nil, err
    }
    
    user, err := s.userClient.GetUser(ctx, userID)
    if err != nil {
        return nil, err
    }
    
    var result []OrderWithUser
    for _, order := range orders {
        result = append(result, OrderWithUser{
            Order: order,
            User:  *user,
        })
    }
    
    return result, nil
}
```

### 3. Brak error boundaries

```go
// AI generuje kruchą implementację
func (gw *APIGateway) GetUserDashboard(userID string) (*Dashboard, error) {
    // Jeden błąd zabija cały dashboard
    user := gw.userService.GetUser(userID)
    orders := gw.orderService.GetOrders(userID)
    payments := gw.paymentService.GetPayments(userID)
    
    return &Dashboard{user, orders, payments}, nil
}

// Poprawnie - graceful degradation
func (gw *APIGateway) GetUserDashboard(ctx context.Context, userID string) (*Dashboard, error) {
    dashboard := &Dashboard{}
    
    var wg sync.WaitGroup
    
    // Parallel calls z error handling
    wg.Add(3)
    
    go func() {
        defer wg.Done()
        if user, err := gw.userService.GetUser(ctx, userID); err == nil {
            dashboard.User = user
        } else {
            log.Printf("user service failed: %v", err)
            dashboard.User = &User{ID: userID, Name: "Unknown"}
        }
    }()
    
    go func() {
        defer wg.Done()
        if orders, err := gw.orderService.GetOrders(ctx, userID); err == nil {
            dashboard.Orders = orders
        } else {
            log.Printf("order service failed: %v", err)
            dashboard.Orders = []Order{}
        }
    }()
    
    go func() {
        defer wg.Done()
        if payments, err := gw.paymentService.GetPayments(ctx, userID); err == nil {
            dashboard.Payments = payments
        } else {
            log.Printf("payment service failed: %v", err)
            dashboard.Payments = []Payment{}
        }
    }()
    
    wg.Wait()
    return dashboard, nil
}
```

## Podsumowanie

Mikrousługi w Go wymagają przemyślanego podejścia:

**Kluczowe zasady:**
- **Jeden serwis = jedna domena biznesowa**
- **Database per service** - nie ma shared database
- **API contracts** - komunikacja przez HTTP/gRPC
- **Fault tolerance** - circuit breakers, timeouts, fallbacks
- **Graceful degradation** - system działa nawet gdy niektóre serwisy są niedostępne
- **Independent deployment** - każdy serwis deployuje się osobno

**Wzorce do zapamiętania:**
- **API Gateway** - jeden entry point
- **Circuit Breaker** - ochrona przed kaskadowymi awariami  
- **Repository Pattern** - abstakcja nad bazą danych
- **Domain Events** - komunikacja asynchroniczna
- **CQRS** - rozdzielenie read/write operations
- **Hexagonal Architecture** - czysta logika biznesowa

**Najczęstsze błędy:**
- Tight coupling między serwisami
- Shared database
- Synchroniczne wywołania wszędzie
- Brak error boundaries
- Missing observability

W następnym rozdziale przejdziemy do HTTP i REST API - nauczysz się budować solidne API w Go, które będą podstawą komunikacji między mikrousługami. Zobaczymy middleware patterns, request validation, content negotiation i dokumentację OpenAPI.