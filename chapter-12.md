# Rozdział 12: Bazy danych i persystencja

Cześć! Bazy danych to serce każdej aplikacji. W Go podejście do baz jest inne niż w PHP - nie ma tu Doctrine czy Eloquent out of the box. Za to masz pełną kontrolę nad SQL-em i świetne wsparcie dla różnych baz. Pokażę ci jak pracować z bazami w Go, od czystego SQL po ORM-y, i dlaczego DynamoDB to game changer dla serverless.

## PHP vs Go: Fundamentalne różnice

W PHP pewnie przywykłeś do:
```php
// PHP - Active Record pattern (Laravel Eloquent)
$user = User::find(1);  // magia - framework wie jak pobrać
$user->name = "Jan";     // magia - framework śledzi zmiany
$user->save();           // magia - framework wie co zaktualizować

// Albo PDO z automatycznym mapowaniem
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$id]);
$user = $stmt->fetch(PDO::FETCH_OBJ);  // automatycznie tworzy obiekt
```

W Go jest inaczej - musisz być bardziej explicit:
```go
// Go - manual scanning, wszystko jawne
var user User  // najpierw deklarujesz zmienną
err := db.QueryRow("SELECT id, name, email FROM users WHERE id = ?", id).
    Scan(&user.ID, &user.Name, &user.Email)  // musisz powiedzieć gdzie co wstawić
    
// Zwróć uwagę:
// - & przed polami - przekazujesz wskaźnik, żeby Scan mógł wpisać wartość
// - kolejność w Scan MUSI odpowiadać kolejności w SELECT
// - musisz obsłużyć błąd - w Go nie ma exceptions
```

**Dlaczego?** Bo Go nie ma refleksji w czasie wykonania jak PHP. Wszystko musi być jasne w czasie kompilacji. To trochę więcej pisania, ale zero magii = łatwiejsze debugowanie.

## database/sql - fundament wszystkiego

Package `database/sql` to standard w Go. To jak PDO w PHP, ale lepszy. Nie jest przywiązany do konkretnej bazy - używasz sterowników.

### Setup i connection pooling

```go
import (
    "database/sql"
    _ "github.com/lib/pq"  // PostgreSQL driver
    // Podkreślnik _ oznacza "importuj tylko dla efektów ubocznych"
    // Sterownik sam się rejestruje w database/sql podczas importu
    // Nie używasz go bezpośrednio, tylko przez interfejs database/sql
)

// Globalna zmienna dla connection pool
// W Go OK jest mieć globalną zmienną dla bazy - to bezpieczne dla współbieżności
var db *sql.DB

func initDB() error {
    var err error
    
    // Connection string - format zależy od sterownika
    // PostgreSQL używa formatu podobnego do URL
    dsn := "postgres://user:password@localhost/dbname?sslmode=disable"
    // MySQL ma inny format:
    // "user:password@tcp(localhost:3306)/dbname?parseTime=true"
    //                 ^^^                       ^^^^^^^^^^^
    //            protokół TCP                parseTime=true żeby 
    //                                        DATE/DATETIME były time.Time
    
    // sql.Open NIE otwiera połączenia! 
    // To tylko przygotowuje pulę połączeń
    db, err = sql.Open("postgres", dsn)
    if err != nil {
        return err  // błąd tylko jeśli źle skonfigurowany sterownik
    }
    
    // Konfiguracja puli - KLUCZOWE dla wydajności!
    db.SetMaxOpenConns(25)                 // max 25 połączeń naraz
    db.SetMaxIdleConns(5)                  // trzymaj 5 "ciepłych" połączeń
    db.SetConnMaxLifetime(5 * time.Minute) // zabij połączenie po 5 min
    
    // Dlaczego te limity?
    // - MaxOpenConns: za dużo = obciążasz bazę, za mało = wąskie gardło
    // - MaxIdleConns: za dużo = marnujesz zasoby, za mało = więcej uzgodnień połączenia
    // - ConnMaxLifetime: niektóre bazy (np. MySQL) zabijają stare połączenia
    
    // Ping() faktycznie otwiera połączenie i sprawdza czy działa
    if err = db.Ping(); err != nil {
        return err
    }
    
    return nil
}
```

**Czemu pula połączeń?** W PHP każdy request to nowy proces/wątek z nowym połączeniem do bazy. W Go masz długo żyjący proces który obsługuje tysiące requestów - musisz zarządzać pulą połączeń. Pula automatycznie:
- Otwiera nowe połączenia gdy potrzeba
- Używa ponownie istniejące połączenia
- Zamyka nieużywane połączenia
- Czeka gdy wszystkie są zajęte

### CRUD operations - czyste SQL

```go
// CREATE - dodawanie rekordów
func createUser(user *User) error {
    // Używamy backticks `` dla multi-line strings
    // $1, $2, $3 to placeholders dla PostgreSQL (zapobiega SQL injection)
    // MySQL używa ? zamiast $1
    query := `
        INSERT INTO users (name, email, created_at) 
        VALUES ($1, $2, $3) 
        RETURNING id`  // PostgreSQL feature - zwraca ID nowo utworzonego rekordu
    
    // QueryRow bo spodziewamy się jednego wyniku (RETURNING id)
    // Exec() byłoby dla INSERT bez RETURNING
    err := db.QueryRow(
        query, 
        user.Name,      // podstawi się za $1
        user.Email,     // podstawi się za $2
        time.Now(),     // podstawi się za $3
    ).Scan(&user.ID)    // wynik RETURNING id wpisz do user.ID
    
    // & przed user.ID bo Scan potrzebuje wskaźnika żeby móc zmienić wartość
    
    return err
}

// READ - pojedynczy rekord
func getUser(id int) (*User, error) {
    var user User  // tworzymy pustą strukturę
    
    // Kolejność w SELECT ma znaczenie!
    query := `SELECT id, name, email, created_at FROM users WHERE id = $1`
    
    // QueryRow dla pojedynczego wyniku
    // Query() byłoby dla wielu wyników
    err := db.QueryRow(query, id).Scan(
        &user.ID,        // pierwsze pole z SELECT
        &user.Name,      // drugie pole z SELECT
        &user.Email,     // trzecie pole z SELECT
        &user.CreatedAt, // czwarte pole z SELECT
    )
    
    // sql.ErrNoRows to specjalny błąd gdy nie ma wyników
    // To NIE jest "prawdziwy" błąd - to normalna sytuacja
    if err == sql.ErrNoRows {
        return nil, errors.New("user not found")
    }
    if err != nil {
        return nil, err  // inny błąd (np. connection error)
    }
    
    return &user, nil  // zwracamy wskaźnik do user
}

// READ - wiele rekordów
func getUsers(limit int) ([]User, error) {
    query := `SELECT id, name, email, created_at FROM users LIMIT $1`
    
    // Query() zwraca *Rows - iterator po wynikach
    rows, err := db.Query(query, limit)
    if err != nil {
        return nil, err
    }
    // KRYTYCZNE! Musisz zamknąć rows żeby zwolnić połączenie do puli
    // defer wykonuje się na końcu funkcji, nawet jeśli będzie panika
    defer rows.Close()
    
    var users []User  // slice na wyniki
    
    // rows.Next() przesuwa kursor na następny rekord
    // zwraca false gdy nie ma więcej rekordów
    for rows.Next() {
        var user User
        // Scan wczytuje aktualny rekord
        err := rows.Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt)
        if err != nil {
            return nil, err
        }
        // append dodaje element do slice'a
        users = append(users, user)
    }
    
    // rows.Err() sprawdza czy były błędy podczas iteracji
    // np. connection dropped podczas czytania
    if err = rows.Err(); err != nil {
        return nil, err
    }
    
    return users, nil
}

// UPDATE
func updateUser(user *User) error {
    query := `
        UPDATE users 
        SET name = $1, email = $2, updated_at = $3
        WHERE id = $4`
    
    // Exec() dla zapytań które nie zwracają rekordów (UPDATE, DELETE, INSERT bez RETURNING)
    result, err := db.Exec(query, user.Name, user.Email, time.Now(), user.ID)
    if err != nil {
        return err
    }
    
    // Result ma metody RowsAffected() i LastInsertId()
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err  // niektóre drivery nie wspierają RowsAffected
    }
    
    // Jeśli 0 rows affected = nie znaleziono rekordu do update
    if rowsAffected == 0 {
        return errors.New("user not found")
    }
    
    return nil
}

// DELETE
func deleteUser(id int) error {
    result, err := db.Exec("DELETE FROM users WHERE id = $1", id)
    if err != nil {
        return err
    }
    
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err
    }
    
    if rowsAffected == 0 {
        return errors.New("user not found")
    }
    
    return nil
}
```

### Prepared statements

Kiedy wykonujesz to samo zapytanie wielokrotnie, użyj prepared statements:

```go
// Prepared statement to prekompilowane zapytanie w bazie
// Baza parsuje je raz, potem tylko podstawia parametry
// To szybsze i bezpieczniejsze

// Prepare kompiluje zapytanie w bazie
stmt, err := db.Prepare("INSERT INTO users (name, email) VALUES ($1, $2)")
if err != nil {
    return err
}
defer stmt.Close()  // WAŻNE! Prepared statement zajmuje zasoby w bazie

// Teraz możesz używać stmt wielokrotnie
for _, user := range users {
    // Exec na stmt, nie na db
    _, err = stmt.Exec(user.Name, user.Email)
    if err != nil {
        return err
    }
}

// Co się dzieje pod spodem:
// 1. Prepare wysyła zapytanie do bazy, baza je parsuje i zachowuje w pamięci podręcznej
// 2. Każde Exec wysyła tylko parametry, nie całe zapytanie
// 3. To oszczędza czas parsowania i przepustowość
```

## SQL builders vs ORMs

### sqlx - database/sql na sterydach

`sqlx` to biblioteka która rozszerza standardowe `database/sql`. Nie zmienia API, tylko dodaje wygodne funkcje. To jak jQuery dla DOM - ten sam DOM, ale wygodniejszy.

```go
import "github.com/jmoiron/sqlx"

// sqlx.DB osadza sql.DB, więc wszystkie metody sql.DB działają
var db *sqlx.DB

// Tagi struktury mówią sqlx jak mapować kolumny na pola
type User struct {
    ID        int       `db:"id"`         // kolumna "id" -> pole ID
    Name      string    `db:"name"`       // kolumna "name" -> pole Name
    Email     string    `db:"email"`      
    CreatedAt time.Time `db:"created_at"` // kolumna "created_at" -> pole CreatedAt
}

// Get - pobiera JEDEN rekord i automatycznie mapuje na strukturę
func getUser(id int) (*User, error) {
    var user User
    // Get robi za ciebie:
    // 1. QueryRow
    // 2. Scan do wszystkich pól według tagów `db`
    // 3. Obsługę sql.ErrNoRows
    err := db.Get(&user, "SELECT * FROM users WHERE id = $1", id)
    // Zwróć uwagę: SELECT * działa bo sqlx wie jak zmapować kolumny dzięki tagom
    
    return &user, err
}

// Select - pobiera WIELE rekordów
func getUsers() ([]User, error) {
    var users []User
    // Select robi za ciebie:
    // 1. Query
    // 2. Pętlę for rows.Next()
    // 3. Scan każdego rekordu
    // 4. append do slice'a
    // 5. rows.Close()
    err := db.Select(&users, "SELECT * FROM users ORDER BY created_at DESC")
    
    // To wszystko w jednej linijce!
    return users, err
}

// Named queries - używaj struct jako źródła parametrów
func createUser(user *User) error {
    // :name, :email, :created_at to named parameters
    // sqlx zamieni je na $1, $2, $3 dla PostgreSQL
    query := `
        INSERT INTO users (name, email, created_at)
        VALUES (:name, :email, :created_at)
        RETURNING id`
    
    // NamedQuery używa pól struktury według tagów
    rows, err := db.NamedQuery(query, user)
    // user.Name -> :name
    // user.Email -> :email
    // user.CreatedAt -> :created_at
    
    if err != nil {
        return err
    }
    
    if rows.Next() {
        rows.Scan(&user.ID)  // wczytaj returned ID
    }
    
    return rows.Close()
}

// sqlx ma też NamedExec dla UPDATE/DELETE
func updateUserNamed(user *User) error {
    query := `
        UPDATE users 
        SET name = :name, email = :email
        WHERE id = :id`
    
    _, err := db.NamedExec(query, user)
    return err
}
```

**Mocne strony sqlx:**
- Zero magii - to nadal czysty SQL
- Kompatybilne z database/sql (możesz migrować stopniowo)
- Skanowanie struktur oszczędza dużo kodu szablonowego
- Nazwane zapytania są czytelniejsze niż $1, $2, $3

**Słabe strony:**
- Nadal piszesz SQL ręcznie
- Nie ma migracji, relacji, ładowania z wyprzedzeniem
- SELECT * może być problem gdy zmienisz schemat

### Squirrel - konstruktor SQL

Squirrel to biblioteka do programowego budowania zapytań SQL. Zamiast kleić stringi, budujesz zapytanie jak LEGO z klocków.

```go
import sq "github.com/Masterminds/squirrel"

// Squirrel używa płynnego interfejsu (łańcuchowanie metod)
func searchUsers(filters UserFilters) ([]User, error) {
    // Rozpocznij od SELECT
    query := sq.Select("id", "name", "email").  // wybierz kolumny
        From("users").                           // z tabeli
        Where(sq.Eq{"active": true})            // gdzie active = true
    
    // sq.Eq to helper który generuje "column = value"
    // Jest też sq.NotEq, sq.Gt, sq.Lt, sq.GtOrEq, sq.LtOrEq
    
    // Dodawaj warunki dynamicznie
    if filters.Name != "" {
        // sq.Like generuje "column LIKE value"
        query = query.Where(sq.Like{"name": "%" + filters.Name + "%"})
        // Zwróć uwagę: query = query.Where
        // Squirrel jest niezmienny - każda metoda zwraca nowy konstruktor
    }
    
    if filters.Email != "" {
        query = query.Where(sq.Eq{"email": filters.Email})
    }
    
    if filters.CreatedAfter != nil {
        // sq.Gt generuje "column > value"
        query = query.Where(sq.Gt{"created_at": filters.CreatedAfter})
    }
    
    // Dodaj sortowanie i limit
    query = query.OrderBy("created_at DESC").Limit(10)
    
    // ToSql() generuje finalne SQL i slice parametrów
    sql, args, err := query.ToSql()
    // sql = "SELECT id, name, email FROM users WHERE active = ? AND name LIKE ? ORDER BY created_at DESC LIMIT 10"
    // args = []interface{}{true, "%john%"}
    
    if err != nil {
        return nil, err
    }
    
    // Wykonaj wygenerowane zapytanie
    var users []User
    err = db.Select(&users, sql, args...)  // ... rozpakuje slice na argumenty
    return users, err
}

// Bardziej złożone zapytania
func getOrdersWithUsers() ([]OrderWithUser, error) {
    query := sq.Select(
        "o.id", "o.total", "o.created_at",
        "u.id as user_id", "u.name as user_name",  // aliasy dla kolumn
    ).
        From("orders o").  // alias dla tabeli
        Join("users u ON o.user_id = u.id").  // INNER JOIN
        // Jest też LeftJoin, RightJoin, FullJoin
        Where(sq.And{  // sq.And łączy warunki przez AND
            sq.Eq{"o.status": "completed"},
            sq.Gt{"o.total": 100},
            // Możesz zagnieżdżać: sq.Or{sq.Eq{...}, sq.And{...}}
        }).
        OrderBy("o.created_at DESC")
    
    sql, args, err := query.ToSql()
    // sql = "SELECT o.id, o.total, o.created_at, u.id as user_id, u.name as user_name 
    //        FROM orders o 
    //        JOIN users u ON o.user_id = u.id 
    //        WHERE o.status = ? AND o.total > ? 
    //        ORDER BY o.created_at DESC"
    
    if err != nil {
        return nil, err
    }
    
    // ... wykonaj zapytanie
}

// Insert builder
func insertWithSquirrel() {
    insert := sq.Insert("users").
        Columns("name", "email", "created_at").
        Values("John", "john@example.com", time.Now()).
        Values("Jane", "jane@example.com", time.Now())  // wiele rekordów!
        
    sql, args, _ := insert.ToSql()
    // INSERT INTO users (name, email, created_at) VALUES (?, ?, ?), (?, ?, ?)
}

// Update builder
func updateWithSquirrel() {
    update := sq.Update("users").
        Set("name", "John Updated").
        Set("updated_at", time.Now()).
        Where(sq.Eq{"id": 1})
        
    sql, args, _ := update.ToSql()
    // UPDATE users SET name = ?, updated_at = ? WHERE id = ?
}
```

**Mocne strony Squirrel:**
- Bezpieczne typowo budowanie zapytań (kompilator sprawdza składnię Go)
- Łatwe budowanie dynamicznych zapytań
- Czytelny kod (lepszy niż konkatenacja stringów)
- Wspiera wszystkie funkcje SQL

**Słabe strony:**
- To nadal "ręczny" SQL, tylko w innej formie
- Nie ma funkcji ORM (relacje, migracje, walidacja)
- Możesz zbudować niepoprawne SQL (Squirrel nie zna twojego schematu)

### GORM - najpopularniejszy ORM

GORM to pełnoprawny ORM jak Eloquent czy Doctrine. Ma wszystko - migracje, relacje, hooki, validację.

```go
import "gorm.io/gorm"

// gorm.Model to osadzona struktura która dodaje standardowe pola
type User struct {
    gorm.Model  // dodaje: ID uint, CreatedAt, UpdatedAt, DeletedAt time.Time
    Name  string `gorm:"size:100;not null"`   // ograniczenia w tagach
    Email string `gorm:"uniqueIndex"`         // tworzy unikalny indeks
    Posts []Post // relacja jeden-do-wielu (użytkownik ma wiele postów)
}
}

type Post struct {
    gorm.Model
    Title  string
    Body   string
    UserID uint  // klucz obcy - GORM rozpoznaje po nazwie
    User   User  // relacja należy do
}

// Initialize GORM
func initGORM() (*gorm.DB, error) {
    dsn := "postgres://user:pass@localhost/db"
    
    // gorm.Open zamiast sql.Open
    // Drugi parametr to konfiguracja
    return gorm.Open(postgres.Open(dsn), &gorm.Config{
        // Opcje konfiguracji
        SkipDefaultTransaction: true,  // wyłącz transakcje dla pojedynczych operacji
        PrepareStmt: true,             // używaj przygotowanych instrukcji
    })
}

// CRUD z GORM - zobacz jak różni się od czystego SQL
func gormExample(db *gorm.DB) {
    // CREATE - automatycznie wypełnia CreatedAt, UpdatedAt
    user := User{Name: "Jan", Email: "jan@example.com"}
    result := db.Create(&user)  // INSERT INTO users (created_at, updated_at, name, email) VALUES (...)
    
    // result ma pomocne metody
    if result.Error != nil {
        log.Fatal(result.Error)
    }
    fmt.Printf("Inserted %d rows\n", result.RowsAffected)
    // user.ID jest teraz wypełnione!
    
    // READ - różne sposoby
    var found User
    
    // First - pobiera pierwszy rekord
    db.First(&found, 1)  // SELECT * FROM users WHERE id = 1 ORDER BY id LIMIT 1
    
    // First z warunkiem
    db.First(&found, "email = ?", "jan@example.com")
    
    // Where chains
    db.Where("name = ?", "Jan").First(&found)
    db.Where("name = ? AND email = ?", "Jan", "jan@example.com").First(&found)
    
    // Where ze strukturą
    db.Where(&User{Name: "Jan", Email: "jan@example.com"}).First(&found)
    // generuje: WHERE name = "Jan" AND email = "jan@example.com"
    
    // Find - pobiera wiele
    var users []User
    db.Find(&users)  // SELECT * FROM users
    db.Where("name LIKE ?", "%Jan%").Find(&users)
    
    // UPDATE - różne sposoby
    // Update pojedyncze pole
    db.Model(&user).Update("name", "Janusz")
    // UPDATE users SET name = "Janusz", updated_at = "..." WHERE id = 1
    
    // Updates - wiele pól ze struktury
    db.Model(&user).Updates(User{Name: "Jan", Email: "new@example.com"})
    // Zero values (0, "", false) są ignorowane!
    
    // Updates z mapą - nie ignoruje zero values
    db.Model(&user).Updates(map[string]interface{}{
        "name": "Jan",
        "age": 0,  // to się zapisze
    })
    
    // DELETE - soft delete domyślnie gdy używasz gorm.Model
    db.Delete(&user, 1)  
    // UPDATE users SET deleted_at = "..." WHERE id = 1
    
    // Hard delete
    db.Unscoped().Delete(&user, 1)
    // DELETE FROM users WHERE id = 1
    
    // Wstępne ładowanie - chętne ładowanie relacji
    var userWithPosts User
    db.Preload("Posts").First(&userWithPosts, 1)
    // Wykonuje 2 zapytania:
    // SELECT * FROM users WHERE id = 1
    // SELECT * FROM posts WHERE user_id = 1
    
    // Zagnieżdżone wstępne ładowanie
    db.Preload("Posts.Comments").First(&userWithPosts, 1)
    
    // Joins - gdy chcesz jedno zapytanie zamiast N+1
    var result []struct {
        UserName string
        PostTitle string
    }
    db.Table("users").
        Select("users.name as user_name, posts.title as post_title").
        Joins("JOIN posts ON posts.user_id = users.id").
        Where("posts.published = ?", true).
        Scan(&result)
    
    // Transactions
    err := db.Transaction(func(tx *gorm.DB) error {
        // Wszystkie operacje w transakcji
        if err := tx.Create(&user).Error; err != nil {
            return err  // automatyczny rollback
        }
        
        if err := tx.Create(&post).Error; err != nil {
            return err  // automatyczny rollback
        }
        
        return nil  // automatyczny commit
    })
    
    // Haki - metody wywoływane automatycznie
    // BeforeSave, BeforeCreate, AfterCreate, AfterSave
    // BeforeUpdate, AfterUpdate, BeforeDelete, AfterDelete
}

// Przykład haka
func (u *User) BeforeCreate(tx *gorm.DB) error {
    // Wykonuje się przed INSERT
    u.Email = strings.ToLower(u.Email)
    return nil
}

// Migracje - GORM automatycznie tworzy/modyfikuje tabele
func runMigrations(db *gorm.DB) {
    // AutoMigrate analizuje struktury i:
    // - Tworzy tabele jeśli nie istnieją
    // - Dodaje brakujące kolumny
    // - Dodaje brakujące indeksy
    // NIE usuwa kolumn ani nie zmienia typów!
    db.AutoMigrate(&User{}, &Post{})
}

// Zakresy - fragmenty zapytań do ponownego użycia
func Active(db *gorm.DB) *gorm.DB {
    return db.Where("active = ?", true)
}

func Recent(db *gorm.DB) *gorm.DB {
    return db.Where("created_at > ?", time.Now().AddDate(0, -1, 0))
}

// Użycie zakresów
db.Scopes(Active, Recent).Find(&users)
// WHERE active = true AND created_at > ...
```

**Mocne strony GORM:**
- Pełny ORM - wszystko w jednym
- Automatyczne migracje
- Relacje i chętne ładowanie
- Haki dla logiki biznesowej
- Miękkie usuwanie od ręki

**Słabe strony:**
- "Magia" - dużo dzieje się pod spodem
- Narzut wydajnościowy vs czysty SQL
- Może generować nieoptymalne zapytania
- Krzywa uczenia dla zaawansowanych funkcji
- Czasem walczysz z ORM zamiast pisać prosty SQL

## NoSQL w Go

### MongoDB

MongoDB to dokumentowa baza NoSQL. Zamiast tabel i rekordów masz kolekcje i dokumenty (jak JSON).

```go
import (
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
)

// Struktury z tagami BSON (Binary JSON)
type Product struct {
    // primitive.ObjectID to typ MongoDB dla _id
    // omitempty oznacza "nie wysyłaj jeśli puste" - MongoDB sam wygeneruje ID
    ID          primitive.ObjectID `bson:"_id,omitempty"`
    Name        string             `bson:"name"`
    Price       float64            `bson:"price"`
    Categories  []string           `bson:"categories"`      // tablica w MongoDB
    Attributes  map[string]string  `bson:"attributes"`      // zagnieżdżony obiekt
    CreatedAt   time.Time          `bson:"created_at"`
}

func mongoExample(client *mongo.Client) {
    // Wybierz bazę i kolekcję
    db := client.Database("shop")
    products := db.Collection("products")
    
    // INSERT
    product := Product{
        Name:  "Laptop",
        Price: 2999.99,
        Categories: []string{"electronics", "computers"},
        Attributes: map[string]string{
            "brand": "Dell",
            "ram":   "16GB",
            "cpu":   "Intel i7",
        },
        CreatedAt: time.Now(),
    }
    
    // InsertOne zwraca result z InsertedID
    result, err := products.InsertOne(context.TODO(), product)
    if err != nil {
        log.Fatal(err)
    }
    
    // Konwersja interface{} na ObjectID
    insertedID := result.InsertedID.(primitive.ObjectID)
    fmt.Printf("Inserted document with ID: %s\n", insertedID.Hex())
    
    // ZNAJDŹ JEDEN
    var found Product
    
    // bson.M to mapa dla zapytań MongoDB (M = Map)
    // Jest też bson.D (D = Document, uporządkowany) i bson.A (A = Array)
    filter := bson.M{"_id": insertedID}
    
    // FindOne zwraca SingleResult, nie błąd
    // Błąd sprawdzasz przy Decode
    err = products.FindOne(context.TODO(), filter).Decode(&found)
    if err == mongo.ErrNoDocuments {
        log.Println("Nie znaleziono dokumentu")
    }
    
    // ZNAJDŹ WIELE - bardziej złożone zapytanie
    filter = bson.M{
        // $gte = większe lub równe (operator MongoDB)
        "price": bson.M{"$gte": 1000},
        // $in = wartość jest w tablicy
        "categories": bson.M{"$in": []string{"electronics"}},
    }
    
    // Find zwraca Cursor - iterator po wynikach (jak rows w SQL)
    cursor, err := products.Find(context.TODO(), filter)
    if err != nil {
        log.Fatal(err)
    }
    defer cursor.Close(context.TODO())  // WAŻNE! Zamknij kursor
    
    // Iteruj po wynikach
    var results []Product
    for cursor.Next(context.TODO()) {
        var product Product
        if err := cursor.Decode(&product); err != nil {
            log.Fatal(err)
        }
        results = append(results, product)
    }
    
    // Lub wszystko naraz
    cursor, _ = products.Find(context.TODO(), filter)
    if err = cursor.All(context.TODO(), &results); err != nil {
        log.Fatal(err)
    }
    
    // AKTUALIZACJA
    filter = bson.M{"_id": insertedID}
    update := bson.M{
        // $set = ustaw wartości
        "$set": bson.M{"price": 2799.99},
        // $push = dodaj do tablicy
        "$push": bson.M{"categories": "sale"},
        // Inne: $pull (usuń z tablicy), $inc (zwiększ), $unset (usuń pole)
    }
    
    updateResult, err := products.UpdateOne(context.TODO(), filter, update)
    fmt.Printf("Zmodyfikowano %d dokumentów\n", updateResult.ModifiedCount)
    
    // POTOK AGREGACJI - potężne zapytania
    // Potok to sekwencja etapów przetwarzania
    pipeline := mongo.Pipeline{
        // Etap 1: Filtruj dokumenty
        {{"$match", bson.M{"price": bson.M{"$gte": 1000}}}},
        
        // Etap 2: Grupuj i agreguj
        {{"$group", bson.M{
            "_id": "$categories",  // grupuj po categories
            "avgPrice": bson.M{"$avg": "$price"},  // średnia cena
            "count": bson.M{"$sum": 1},  // liczba produktów
        }}},
        
        // Etap 3: Sortuj wyniki
        {{"$sort", bson.M{"avgPrice": -1}}},  // -1 = malejąco
    }
    
    cursor, err = products.Aggregate(context.TODO(), pipeline)
    // ... przetwórz wyniki
    
    // USUWANIE
    deleteResult, err := products.DeleteOne(context.TODO(), bson.M{"_id": insertedID})
    fmt.Printf("Usunięto %d dokumentów\n", deleteResult.DeletedCount)
}
```

**Mocne strony MongoDB:**
- Elastyczny schemat (każdy dokument może być inny)
- Świetny dla danych hierarchicznych
- Potężny framework agregacji
- Skalowanie horyzontalne (sharding)

**Słabe strony:**
- Brak transakcji między kolekcjami (w starszych wersjach)
- Może używać dużo RAM
- Brak JOIN-ów (musisz denormalizować lub robić wiele zapytań)

### DynamoDB - król serverless

DynamoDB to NoSQL od AWS. Idealny dla Lambda bo w pełni zarządzany i płacisz tylko za użycie.

```go
import (
    "github.com/aws/aws-sdk-go-v2/service/dynamodb"
    "github.com/aws/aws-sdk-go-v2/feature/dynamodb/attributevalue"
    "github.com/aws/aws-sdk-go-v2/service/dynamodb/types"
)

// Struktury z tagami dynamodbav
type Order struct {
    OrderID    string    `dynamodbav:"order_id"`    // Klucz partycji
    UserID     string    `dynamodbav:"user_id"`     // Klucz sortowania lub GSI
    Total      float64   `dynamodbav:"total"`
    Status     string    `dynamodbav:"status"`
    Items      []Item    `dynamodbav:"items"`       // DynamoDB wspiera zagnieżdżone
    CreatedAt  time.Time `dynamodbav:"created_at"`
}

type Item struct {
    ProductID string  `dynamodbav:"product_id"`
    Quantity  int     `dynamodbav:"quantity"`
    Price     float64 `dynamodbav:"price"`
}

func dynamoExample(client *dynamodb.Client) {
    tableName := "orders"
    
    // WSTAW ELEMENT - wstawienie lub pełne zastąpienie
    order := Order{
        OrderID:   "ORD-12345",
        UserID:    "USER-789",
        Total:     99.99,
        Status:    "pending",
        Items: []Item{
            {ProductID: "PROD-1", Quantity: 2, Price: 49.99},
        },
        CreatedAt: time.Now(),
    }
    
    // MarshalMap konwertuje strukturę Go na AttributeValue DynamoDB
    // To jak json.Marshal ale dla DynamoDB
    item, err := attributevalue.MarshalMap(order)
    if err != nil {
        log.Fatal(err)
    }
    
    // PutItem zastępuje cały element jeśli istnieje
    _, err = client.PutItem(context.TODO(), &dynamodb.PutItemInput{
        TableName: &tableName,
        Item:      item,
        // Opcjonalnie: ConditionExpression żeby nie nadpisać istniejącego
        // ConditionExpression: aws.String("attribute_not_exists(order_id)"),
    })
    
    // POBIERZ ELEMENT - pobierz po kluczu
    response, err := client.GetItem(context.TODO(), &dynamodb.GetItemInput{
        TableName: &tableName,
        Key: map[string]types.AttributeValue{
            // Musisz podać klucz partycji (i klucz sortowania jeśli masz)
            "order_id": &types.AttributeValueMemberS{Value: "ORD-12345"},
            // AttributeValueMemberS = String
            // Jest też MemberN (Number), MemberB (Binary), MemberBOOL, itd.
        },
    })
    
    var found Order
    // UnmarshalMap konwertuje z DynamoDB do struktury Go
    err = attributevalue.UnmarshalMap(response.Item, &found)
    
    // ZAPYTANIE - szukaj po kluczu partycji lub GSI
    // Zapytanie jest efektywne bo używa indeksu
    queryInput := &dynamodb.QueryInput{
        TableName:              &tableName,
        IndexName:              aws.String("user-index"),  // Globalny indeks wtórny
        KeyConditionExpression: aws.String("user_id = :userid"),
        // Symbol zastępczy :userid zamiast wstawiać wartość bezpośrednio
        ExpressionAttributeValues: map[string]types.AttributeValue{
            ":userid": &types.AttributeValueMemberS{Value: "USER-789"},
        },
        // Opcje:
        // Limit: aws.Int32(10),  // max wyników
        // ScanIndexForward: aws.Bool(false),  // sortowanie malejąco
    }
    
    result, err := client.Query(context.TODO(), queryInput)
    
    var orders []Order
    // UnmarshalListOfMaps dla wielu elementów
    err = attributevalue.UnmarshalListOfMaps(result.Items, &orders)
    
    // SKANOWANIE - przeszukaj całą tabelę (DROGIE!)
    // Używaj tylko gdy musisz, zapytanie jest lepsze
    scanInput := &dynamodb.ScanInput{
        TableName: &tableName,
        FilterExpression: aws.String("total > :min"),
        ExpressionAttributeValues: map[string]types.AttributeValue{
            ":min": &types.AttributeValueMemberN{Value: "50"},
        },
    }
    
    scanResult, err := client.Scan(context.TODO(), scanInput)
    // Skanowanie czyta CAŁĄ tabelę i filtruje - bardzo drogie!
    
    // AKTUALIZUJ ELEMENT - częściowa aktualizacja
    updateInput := &dynamodb.UpdateItemInput{
        TableName: &tableName,
        Key: map[string]types.AttributeValue{
            "order_id": &types.AttributeValueMemberS{Value: "ORD-12345"},
        },
        // UpdateExpression definiuje co zmienić
        UpdateExpression: aws.String("SET #status = :status, total = :total"),
        // # przed status bo "status" to zastrzeżone słowo kluczowe
        ExpressionAttributeNames: map[string]string{
            "#status": "status",
        },
        ExpressionAttributeValues: map[string]types.AttributeValue{
            ":status": &types.AttributeValueMemberS{Value: "completed"},
            ":total": &types.AttributeValueMemberN{Value: "109.99"},
        },
    }
    
    _, err = client.UpdateItem(context.TODO(), updateInput)
    
    // OPERACJE WSADOWE - wiele operacji naraz
    writeRequests := []types.WriteRequest{
        {
            PutRequest: &types.PutRequest{
                Item: item1,
            },
        },
        {
            PutRequest: &types.PutRequest{
                Item: item2,
            },
        },
        // Max 25 elementów na partię
    }
    
    _, err = client.BatchWriteItem(context.TODO(), &dynamodb.BatchWriteItemInput{
        RequestItems: map[string][]types.WriteRequest{
            tableName: writeRequests,
        },
    })
}

// DynamoDB Streams - reaguj na zmiany w czasie rzeczywistym
// To wyzwalacz Lambda - funkcja wywołuje się automatycznie gdy coś się zmieni
func processDynamoStream(record events.DynamoDBEventRecord) {
    // EventName mówi co się stało
    switch record.EventName {
    case "INSERT":
        // Nowy rekord dodany
        var newOrder Order
        err := attributevalue.UnmarshalMap(
            record.Change.NewImage,  // nowy stan
            &newOrder,
        )
        // np. wyślij email powitalny
        
    case "MODIFY":
        // Rekord zmieniony
        var oldOrder, newOrder Order
        attributevalue.UnmarshalMap(record.Change.OldImage, &oldOrder)  // stary stan
        attributevalue.UnmarshalMap(record.Change.NewImage, &newOrder)  // nowy stan
        
        if oldOrder.Status != newOrder.Status {
            // Status się zmienił - wyślij powiadomienie
            fmt.Printf("Zamówienie %s zmieniło się z %s na %s\n", 
                newOrder.OrderID, oldOrder.Status, newOrder.Status)
        }
        
    case "REMOVE":
        // Rekord usunięty
        var deletedOrder Order
        attributevalue.UnmarshalMap(record.Change.OldImage, &deletedOrder)
        // np. archiwizuj
    }
}
```

**Mocne strony DynamoDB:**
- W pełni zarządzany - zero administracji
- Automatyczne skalowanie - sam się skaluje
- Płać za żądanie - płacisz tylko za użycie
- DynamoDB Streams - wyzwalacze w czasie rzeczywistym
- Świetna integracja z Lambda

**Słabe strony:**
- Tylko zapytania klucz-wartość (brak SQL)
- Musisz projektować dla wzorców dostępu
- Trudne do zmiany schematu później
- Uzależnienie od dostawcy (tylko AWS)

## Transactions

### SQL Transactions

Transakcje zapewniają że wszystkie operacje wykonają się lub żadna (ACID).

```go
// Przykład: przelew pieniędzy między kontami
func transferMoney(fromID, toID int, amount float64) error {
    // Begin() rozpoczyna transakcję
    // Od teraz wszystkie operacje są w "trybie draft"
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    
    // defer wykonuje się na końcu funkcji
    // Rollback anuluje transakcję jeśli nie było Commit
    // Jeśli już był Commit, Rollback nic nie robi (operacja pusta)
    defer tx.Rollback()
    
    // Operacja 1: Sprawdź saldo i pobierz pieniądze
    // Używamy tx, nie db!
    result, err := tx.Exec(
        `UPDATE accounts 
         SET balance = balance - $1 
         WHERE id = $2 AND balance >= $1`,  // warunek: saldo >= amount
        amount, fromID,
    )
    if err != nil {
        return err  // defer rollback się wykona
    }
    
    // Sprawdź czy update się wykonał
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return err
    }
    
    if rowsAffected == 0 {
        // Nie zaktualizowano = brak środków lub nie ma konta
        return errors.New("insufficient funds or account not found")
    }
    
    // Operacja 2: Dodaj pieniądze do drugiego konta
    result, err = tx.Exec(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount, toID,
    )
    if err != nil {
        return err  // defer rollback się wykona
    }
    
    rowsAffected, err = result.RowsAffected()
    if err != nil {
        return err
    }
    
    if rowsAffected == 0 {
        return errors.New("recipient account not found")
        // defer rollback anuluje też pierwszą operację!
    }
    
    // Wszystko OK - zatwierdź transakcję
    return tx.Commit()
    // Po Commit, defer Rollback nic nie zrobi
}

// Transakcje z context (dla timeout/cancel)
func txWithContext(ctx context.Context) error {
    // BeginTx przyjmuje kontekst
    // Jeśli kontekst zostanie anulowany, transakcja też
    tx, err := db.BeginTx(ctx, nil)  // nil = domyślny poziom izolacji
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // ExecContext respektuje limit czasu kontekstu
    _, err = tx.ExecContext(ctx, "UPDATE ...")
    if err != nil {
        // Może być context.Canceled jeśli przekroczono limit czasu
        return err
    }
    
    return tx.Commit()
}

// Helper dla transakcji - ułatwia życie
func withTransaction(fn func(*sql.Tx) error) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    
    // Skomplikowany defer który obsługuje wszystko
    defer func() {
        if p := recover(); p != nil {
            // Była panika - wycofaj i ponownie rzuć
            tx.Rollback()
            panic(p)
        } else if err != nil {
            // Był błąd - wycofaj
            tx.Rollback()
        } else {
            // Wszystko OK - zatwierdź
            err = tx.Commit()
        }
    }()
    
    // Wywołaj funkcję użytkownika
    err = fn(tx)
    return err
}

// Użycie helpera
err := withTransaction(func(tx *sql.Tx) error {
    // Twoje operacje - nie musisz myśleć o rollback/commit
    _, err := tx.Exec("INSERT INTO logs ...")
    if err != nil {
        return err  // automatyczny rollback
    }
    
    _, err = tx.Exec("UPDATE users ...")
    return err  // automatyczny commit jeśli nil
})
```

### Transakcje rozproszone (wzorzec Saga)

W mikrousługach każdy serwis ma swoją bazę. Nie możesz zrobić transakcji SQL między bazami. Używasz wzorca Saga - sekwencja lokalnych transakcji z kompensacją.

```go
// Orkiestrator Sagi - koordynuje transakcję rozproszoną
type OrderSaga struct {
    orderService     OrderService
    paymentService   PaymentService
    inventoryService InventoryService
    compensations    []func() error  // stos funkcji wycofania
}

func (s *OrderSaga) Execute(order Order) error {
    // Saga to sekwencja kroków
    // Każdy krok ma swoją kompensację (undo)
    
    // Krok 1: Stwórz zamówienie
    orderID, err := s.orderService.CreateOrder(order)
    if err != nil {
        return err  // nic do rollback
    }
    
    // Dodaj kompensację - funkcję która cofnie ten krok
    s.compensations = append(s.compensations, func() error {
        log.Printf("Kompensacja: anulowanie zamówienia %s", orderID)
        return s.orderService.CancelOrder(orderID)
    })
    
    // Krok 2: Rezerwuj towary
    reservationID, err := s.inventoryService.ReserveItems(order.Items)
    if err != nil {
        // Błąd - wykonaj wszystkie kompensacje
        return s.compensate()
    }
    
    // Dodaj kompensację dla tego kroku
    s.compensations = append(s.compensations, func() error {
        log.Printf("Kompensacja: zwalnianie rezerwacji %s", reservationID)
        return s.inventoryService.ReleaseReservation(reservationID)
    })
    
    // Krok 3: Pobierz płatność
    paymentID, err := s.paymentService.ProcessPayment(order.Payment)
    if err != nil {
        // Błąd - cofnij kroki 2 i 1
        return s.compensate()
    }
    
    s.compensations = append(s.compensations, func() error {
        log.Printf("Kompensacja: zwrot płatności %s", paymentID)
        return s.paymentService.RefundPayment(paymentID)
    })
    
    // Krok 4: Potwierdź zamówienie
    err = s.orderService.ConfirmOrder(orderID)
    if err != nil {
        // Błąd - cofnij kroki 3, 2, 1
        return s.compensate()
    }
    
    return nil  // Sukces! Saga complete
}

func (s *OrderSaga) compensate() error {
    // Wykonaj kompensacje w odwrotnej kolejności (LIFO)
    // Jak undo w edytorze tekstu
    
    for i := len(s.compensations) - 1; i >= 0; i-- {
        if err := s.compensations[i](); err != nil {
            // Kompensacja failed - to źle!
            // Log ale kontynuuj - best effort
            log.Printf("CRITICAL: Compensation failed: %v", err)
            // W produkcji: alert dla ops team
        }
    }
    
    return errors.New("saga failed and was compensated")
}

// Użycie
saga := &OrderSaga{
    orderService:     orderService,
    paymentService:   paymentService, 
    inventoryService: inventoryService,
}

if err := saga.Execute(order); err != nil {
    // Saga failed ale wszystko zostało cofnięte
    return fmt.Errorf("order processing failed: %w", err)
}
```

**Saga vs Transaction:**
- Transaction: wszystko albo nic, strong consistency
- Saga: eventual consistency, może być stan przejściowy
- Saga jest bardziej skomplikowana ale skaluje się lepiej

## Database migrations

Migracje to sposób na wersjonowanie schematu bazy. W Go używamy `golang-migrate`.

```go
// Struktura projektu:
// migrations/
//   001_create_users.up.sql      -- migracja "do przodu"
//   001_create_users.down.sql    -- migracja "do tyłu" (rollback)
//   002_add_email_index.up.sql
//   002_add_email_index.down.sql

// 001_create_users.up.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

// 001_create_users.down.sql
DROP TABLE IF EXISTS users;

// 002_add_email_index.up.sql
CREATE INDEX idx_users_created_at ON users(created_at);

// 002_add_email_index.down.sql  
DROP INDEX IF EXISTS idx_users_created_at;

// W kodzie Go
import "github.com/golang-migrate/migrate/v4"
import _ "github.com/golang-migrate/migrate/v4/source/file"
import _ "github.com/golang-migrate/migrate/v4/database/postgres"

func runMigrations(dbURL string) error {
    // Stwórz migrate instance
    m, err := migrate.New(
        "file://migrations",  // skąd brać pliki migracji
        dbURL,                // connection string do bazy
    )
    if err != nil {
        return err
    }
    
    // Wykonaj wszystkie migracje "up"
    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        // ErrNoChange = wszystkie migracje już wykonane
        return err
    }
    
    // Inne metody:
    // m.Down() - cofnij wszystkie migracje
    // m.Steps(2) - wykonaj 2 migracje do przodu
    // m.Steps(-1) - cofnij 1 migrację
    // m.Force(3) - ustaw wersję na 3 (recovery)
    
    return nil
}

// Jak to działa?
// 1. Migrate tworzy tabelę schema_migrations
// 2. Zapisuje tam którą migrację ostatnio wykonał
// 3. Przy następnym Up() wykonuje tylko nowe migracje
// 4. Migracje są wykonywane w transakcji (jeśli baza wspiera)

// CLI usage (osobny binary)
// migrate -path migrations -database postgres://... up
// migrate -path migrations -database postgres://... down 1
// migrate -path migrations -database postgres://... version
// migrate -path migrations -database postgres://... force 3
```

## Caching strategies

Cache przyspiesza aplikację i zmniejsza obciążenie bazy. Ale musisz pilnować spójności!

### Redis jako cache

Redis to in-memory key-value store. Superszybki, popularny jako cache.

```go
import "github.com/go-redis/redis/v8"

var rdb *redis.Client

func initRedis() {
    rdb = redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",  // Redis address
        Password: "",                 // no password
        DB:       0,                  // default DB
        
        // Connection pool (tak, Redis też ma pool!)
        PoolSize: 10,
        MinIdleConns: 3,
    })
    
    // Test connection
    ctx := context.Background()
    if err := rdb.Ping(ctx).Err(); err != nil {
        panic(err)
    }
}

// CACHE-ASIDE PATTERN (najpopularniejszy)
// Aplikacja sama zarządza cache
func getUser(ctx context.Context, id int) (*User, error) {
    key := fmt.Sprintf("user:%d", id)
    
    // 1. Sprawdź cache
    val, err := rdb.Get(ctx, key).Result()
    if err == nil {
        // Cache HIT - mamy w cache
        var user User
        // Deserializuj z JSON
        if err := json.Unmarshal([]byte(val), &user); err == nil {
            log.Printf("Cache hit for user %d", id)
            return &user, nil
        }
        // Jeśli deserializacja failed, ignoruj i idź do bazy
    } else if err != redis.Nil {
        // redis.Nil = klucz nie istnieje (to OK)
        // Inny błąd = problem z Redis
        log.Printf("Redis error: %v", err)
        // Nie failuj - idź do bazy (Redis to tylko cache)
    }
    
    // 2. Cache MISS - pobierz z bazy
    log.Printf("Cache miss for user %d", id)
    user, err := getUserFromDB(id)
    if err != nil {
        return nil, err
    }
    
    // 3. Zapisz w cache na przyszłość
    data, _ := json.Marshal(user)
    // Set z TTL (Time To Live) - automatycznie wygaśnie
    err = rdb.Set(ctx, key, data, 5*time.Minute).Err()
    if err != nil {
        // Log ale nie failuj - cache to optimization
        log.Printf("Failed to cache: %v", err)
    }
    
    return user, nil
}

// WRITE-THROUGH PATTERN
// Przy zapisie aktualizuj bazę I cache
func updateUser(ctx context.Context, user *User) error {
    // 1. Update w bazie (source of truth)
    if err := updateUserInDB(user); err != nil {
        return err
    }
    
    // 2. Update w cache (żeby był świeży)
    key := fmt.Sprintf("user:%d", user.ID)
    data, _ := json.Marshal(user)
    
    // Pipeline - wyślij wiele komend naraz
    pipe := rdb.Pipeline()
    pipe.Set(ctx, key, data, 5*time.Minute)
    pipe.Set(ctx, "user:email:"+user.Email, user.ID, 5*time.Minute)  // secondary index
    _, err := pipe.Exec(ctx)
    
    if err != nil {
        log.Printf("Cache update failed: %v", err)
        // Nie failuj - cache będzie stale przy miss
    }
    
    return nil
}

// CACHE INVALIDATION - najtrudniejszy problem w CS ;)
func deleteUser(ctx context.Context, id int) error {
    // 1. Pobierz user (potrzebujemy email do invalidacji)
    user, err := getUserFromDB(id)
    if err != nil {
        return err
    }
    
    // 2. Delete z bazy
    if err := deleteUserFromDB(id); err != nil {
        return err
    }
    
    // 3. Invalidate wszystkie powiązane klucze
    keys := []string{
        fmt.Sprintf("user:%d", id),
        fmt.Sprintf("user:email:%s", user.Email),
        "users:all",  // lista wszystkich userów
    }
    
    // Del przyjmuje wiele kluczy
    if err := rdb.Del(ctx, keys...).Err(); err != nil {
        log.Printf("Cache invalidation failed: %v", err)
    }
    
    return nil
}

// CACHE WARMING - wypełnij cache przed ruchem
func warmCache(ctx context.Context) error {
    users, err := getAllUsersFromDB()
    if err != nil {
        return err
    }
    
    pipe := rdb.Pipeline()
    for _, user := range users {
        key := fmt.Sprintf("user:%d", user.ID)
        data, _ := json.Marshal(user)
        pipe.Set(ctx, key, data, 5*time.Minute)
    }
    
    _, err = pipe.Exec(ctx)
    return err
}

// Bardziej zaawansowane: Cache z lock (prevent thundering herd)
func getUserWithLock(ctx context.Context, id int) (*User, error) {
    key := fmt.Sprintf("user:%d", id)
    lockKey := key + ":lock"
    
    // Sprawdź cache
    val, err := rdb.Get(ctx, key).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(val), &user)
        return &user, nil
    }
    
    // Cache miss - spróbuj zdobyć lock
    // SetNX = Set if Not eXists (atomic)
    locked, err := rdb.SetNX(ctx, lockKey, 1, 10*time.Second).Result()
    if err != nil || !locked {
        // Ktoś inny już fetchuje - poczekaj i sprawdź cache
        time.Sleep(100 * time.Millisecond)
        return getUserWithLock(ctx, id)  // rekurencja
    }
    
    // Mamy lock - fetch z bazy
    defer rdb.Del(ctx, lockKey)  // zwolnij lock
    
    user, err := getUserFromDB(id)
    if err != nil {
        return nil, err
    }
    
    // Update cache
    data, _ := json.Marshal(user)
    rdb.Set(ctx, key, data, 5*time.Minute)
    
    return user, nil
}
```

### In-memory cache

Dla prostych przypadków możesz cache'ować w pamięci procesu.

```go
// Opcja 1: Biblioteka go-cache
import "github.com/patrickmn/go-cache"

// Initialize z default expiration i cleanup interval
c := cache.New(5*time.Minute, 10*time.Minute)
//              ^^^             ^^^
//         default TTL      cleanup co 10 min

// Set
c.Set("user:1", user, cache.DefaultExpiration)
c.Set("user:2", user2, 1*time.Hour)  // custom TTL

// Get
if val, found := c.Get("user:1"); found {
    user := val.(*User)  // type assertion
    // używaj user
}

// Delete
c.Delete("user:1")

// Flush all
c.Flush()

// Opcja 2: sync.Map - concurrent-safe map
var cache sync.Map

// Store
cache.Store("user:1", user)

// Load
if val, ok := cache.Load("user:1"); ok {
    user := val.(*User)
}

// Delete
cache.Delete("user:1")

// Iterate
cache.Range(func(key, value interface{}) bool {
    fmt.Printf("%v: %v\n", key, value)
    return true  // continue iteration
})

// Opcja 3: Własny cache z mutex
type Cache struct {
    mu    sync.RWMutex  // Read/Write mutex
    items map[string]*CacheItem
}

type CacheItem struct {
    Value     interface{}
    ExpiresAt time.Time
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()  // Read lock - wiele goroutines może czytać
    defer c.mu.RUnlock()
    
    item, found := c.items[key]
    if !found {
        return nil, false
    }
    
    // Sprawdź TTL
    if time.Now().After(item.ExpiresAt) {
        return nil, false  // expired
    }
    
    return item.Value, true
}

func (c *Cache) Set(key string, value interface{}, ttl time.Duration) {
    c.mu.Lock()  // Write lock - exclusive
    defer c.mu.Unlock()
    
    c.items[key] = &CacheItem{
        Value:     value,
        ExpiresAt: time.Now().Add(ttl),
    }
}

// Cleanup expired items
func (c *Cache) cleanup() {
    ticker := time.NewTicker(1 * time.Minute)
    for range ticker.C {
        c.mu.Lock()
        now := time.Now()
        for key, item := range c.items {
            if now.After(item.ExpiresAt) {
                delete(c.items, key)
            }
        }
        c.mu.Unlock()
    }
}
```

**Redis vs In-memory:**
- Redis: współdzielony między procesami, survives restart, więcej features
- In-memory: superszybki (no network), prosty, ale per-process i ginie przy restart

## AI checkpoint: Czego pilnować

AI często generuje kod z bazami który wygląda OK, ale ma subtelne błędy. Oto najczęstsze pułapki:

### 1. Brak zamykania rows

```go
// AI może zapomnieć o defer rows.Close()
rows, err := db.Query("SELECT * FROM users")
if err != nil {
    return err
}
// BRAK: defer rows.Close() - to krytyczny błąd!

for rows.Next() {
    // ... scan
}

// Co się stanie?
// - Połączenie zostanie "zawieszone" w pool
// - Po kilku takich błędach pool się wyczerpie
// - Aplikacja przestanie działać (deadlock na pool)

// ZAWSZE rób tak:
rows, err := db.Query("SELECT * FROM users")
if err != nil {
    return err
}
defer rows.Close()  // defer gwarantuje wykonanie nawet przy panic
```

### 2. Ignorowanie sql.ErrNoRows

```go
// AI często nie sprawdza ErrNoRows
var user User
err := db.QueryRow("SELECT name FROM users WHERE id = ?", id).Scan(&user.Name)
if err != nil {
    // To zwróci "sql: no rows in result set" - brzydki błąd dla usera
    return err  
}

// Poprawnie - special case dla "not found":
err := db.QueryRow("SELECT name FROM users WHERE id = ?", id).Scan(&user.Name)
if err == sql.ErrNoRows {
    // To normalna sytuacja, nie błąd
    return fmt.Errorf("user %d not found", id)
}
if err != nil {
    // To prawdziwy błąd (np. connection problem)
    return fmt.Errorf("database error: %w", err)
}
```

### 3. SQL Injection przez string concatenation

```go
// AI może wygenerować (NIEBEZPIECZNE!)
userName := "'; DROP TABLE users; --"
query := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", userName)
rows, err := db.Query(query)  // BOOM! Tabela users właśnie zniknęła

// AI też lubi:
query := "SELECT * FROM users WHERE name = '" + userName + "'"

// ZAWSZE używaj placeholders:
rows, err := db.Query("SELECT * FROM users WHERE name = ?", userName)
// Driver sam escapuje wartości - bezpieczne

// Dla dynamicznych nazw kolumn (rzadkie):
allowedColumns := map[string]bool{"name": true, "email": true}
if !allowedColumns[columnName] {
    return errors.New("invalid column")
}
// Teraz bezpiecznie możesz użyć w query
```

### 4. Błędna kolejność Scan

```go
// AI może pomylić kolejność
// SELECT zwraca: id, name, email
err := db.QueryRow("SELECT id, name, email FROM users WHERE id = ?", id).
    Scan(&user.Email, &user.Name, &user.ID)  // ZŁA kolejność!
    
// Co się stanie?
// - ID trafi do Email (type mismatch - error)
// - Albo gorzej: cichy błąd jeśli typy się zgadzają

// ZAWSZE pilnuj kolejności:
err := db.QueryRow("SELECT id, name, email FROM users WHERE id = ?", id).
    Scan(&user.ID, &user.Name, &user.Email)  // zgodne z SELECT

// Pro tip: używaj named columns
err := db.QueryRow(`
    SELECT 
        id,         -- 1
        name,       -- 2  
        email       -- 3
    FROM users 
    WHERE id = ?`, id).
    Scan(
        &user.ID,    // 1
        &user.Name,  // 2
        &user.Email, // 3
    )
```

### 5. Brak obsługi NULL values

```go
// Tabela: users
// bio VARCHAR(500) NULL  -- może być NULL!

// AI często ignoruje NULL:
var user User
err := db.QueryRow("SELECT name, bio FROM users WHERE id = ?", id).
    Scan(&user.Name, &user.Bio)  // PANIC jeśli bio IS NULL!

// Go nie wie jak wstawić NULL do string
// Musisz użyć sql.NullString:
var bio sql.NullString  // specjalny typ dla nullable
err := db.QueryRow("SELECT name, bio FROM users WHERE id = ?", id).
    Scan(&user.Name, &bio)
    
if bio.Valid {  // sprawdź czy nie NULL
    user.Bio = bio.String
} else {
    user.Bio = ""  // lub inna default value
}

// Jest też sql.NullInt64, sql.NullFloat64, sql.NullBool, sql.NullTime

// Alternatywa - użyj COALESCE w SQL:
err := db.QueryRow(`
    SELECT name, COALESCE(bio, '') as bio 
    FROM users 
    WHERE id = ?`, id).
    Scan(&user.Name, &user.Bio)  // teraz bezpieczne
```

### 6. Connection pool exhaustion

```go
// AI może otworzyć za dużo połączeń naraz:
userIDs := []int{1, 2, 3, /* ... 1000 więcej */}

for _, id := range userIDs {
    go func(id int) {
        // To może otworzyć 1000 połączeń NARAZ!
        rows, _ := db.Query("SELECT * FROM users WHERE id = ?", id)
        // Process...
        rows.Close()
    }(id)
}

// Co się stanie?
// - Przekroczysz MaxOpenConns
// - Goroutines będą czekać na wolne połączenie
// - Albo gorzej: zabijesz bazę danych

// Rozwiązanie 1: Ogranicz współbieżność
sem := make(chan struct{}, 10)  // max 10 równoległych queries

for _, id := range userIDs {
    sem <- struct{}{}  // zablokuje gdy 10 aktywnych
    go func(id int) {
        defer func() { <-sem }()  // zwolnij slot
        
        rows, _ := db.Query("SELECT * FROM users WHERE id = ?", id)
        defer rows.Close()
        // Process...
    }(id)
}

// Rozwiązanie 2: Batch query
// Zamiast 1000 queries, zrób 1:
query := `SELECT * FROM users WHERE id = ANY($1)`  // PostgreSQL
rows, err := db.Query(query, pq.Array(userIDs))

// MySQL:
placeholders := strings.Repeat("?,", len(userIDs)-1) + "?"
query := fmt.Sprintf("SELECT * FROM users WHERE id IN (%s)", placeholders)
args := make([]interface{}, len(userIDs))
for i, id := range userIDs {
    args[i] = id
}
rows, err := db.Query(query, args...)
```

### 7. Transakcje bez defer Rollback

```go
// AI może zapomnieć o rollback:
tx, err := db.Begin()
if err != nil {
    return err
}

_, err = tx.Exec("UPDATE ...")
if err != nil {
    tx.Rollback()  // OK
    return err
}

_, err = tx.Exec("UPDATE ...")
if err != nil {
    // Ups, zapomniał rollback!
    return err  // transakcja wisi, connection leak
}

return tx.Commit()

// ZAWSZE używaj defer:
tx, err := db.Begin()
if err != nil {
    return err
}
defer tx.Rollback()  // no-op jeśli już committed

// ... wszystkie operacje ...

return tx.Commit()
```

### 8. GORM i N+1 problem

```go
// AI często generuje N+1 queries z GORM:
var users []User
db.Find(&users)  // SELECT * FROM users

for _, user := range users {
    db.Find(&user.Posts)  // SELECT * FROM posts WHERE user_id = ?
    // To wykona dodatkowe zapytanie dla KAŻDEGO usera!
}

// Jeśli masz 100 userów = 101 zapytań do bazy!

// Poprawnie - użyj Preload:
var users []User
db.Preload("Posts").Find(&users)
// Wykonuje tylko 2 zapytania:
// SELECT * FROM users
// SELECT * FROM posts WHERE user_id IN (1,2,3,...)
```

### 9. Ignorowanie context w nowych wersjach

```go
// AI może generować stary kod:
rows, err := db.Query("SELECT * FROM users")

// Nowoczesny Go używa context:
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

rows, err := db.QueryContext(ctx, "SELECT * FROM users")
// Jeśli query trwa > 5 sekund, zostanie anulowane

// To samo dla wszystkich operacji:
db.ExecContext(ctx, ...)
db.QueryRowContext(ctx, ...)
tx.ExecContext(ctx, ...)

// Context pozwala anulować długie operacje
// Kluczowe dla serverless gdzie masz timeout na Lambda
```

### 10. Błędne używanie DynamoDB

```go
// AI często myli AttributeValue types:
item := map[string]types.AttributeValue{
    "id": &types.AttributeValueMemberS{Value: 123},  // ŹLE! S = String
}

// Poprawnie:
item := map[string]types.AttributeValue{
    "id": &types.AttributeValueMemberN{Value: "123"},  // N = Number (jako string!)
    // DynamoDB przechowuje liczby jako stringi dla precyzji
}

// AI może też robić SCAN zamiast Query:
// SCAN czyta CAŁĄ tabelę - super drogie!
result, _ := dynamoClient.Scan(ctx, &dynamodb.ScanInput{
    TableName: aws.String("huge-table"),
})

// Użyj Query gdy możesz (wymaga index):
result, _ := dynamoClient.Query(ctx, &dynamodb.QueryInput{
    TableName: aws.String("huge-table"),
    IndexName: aws.String("user-index"),
    KeyConditionExpression: aws.String("user_id = :uid"),
    ExpressionAttributeValues: map[string]types.AttributeValue{
        ":uid": &types.AttributeValueMemberS{Value: userID},
    },
})
```

## Podsumowanie

Bazy danych w Go to sporo roboty manualnej w porównaniu do PHP ORM-ów, ale masz pełną kontrolę. Kluczowe rzeczy do zapamiętania:

1. **database/sql** - zawsze zamykaj rows (`defer rows.Close()`), sprawdzaj `sql.ErrNoRows`
2. **Connection pooling** - skonfiguruj `MaxOpenConns`, `MaxIdleConns`, `ConnMaxLifetime`
3. **Placeholders** - ZAWSZE używaj dla parametrów (zapobiega SQL injection)
4. **NULL handling** - użyj `sql.NullString` lub COALESCE w SQL
5. **sqlx** - wygodniejsze niż czysty database/sql, automatic struct scanning
6. **Squirrel** - builder dla dynamicznych zapytań, lepszy niż string concatenation
7. **GORM** - full ORM, świetny dla CRUD, słaby dla skomplikowanych queries
8. **MongoDB** - elastyczny schemat, dobre dla hierarchicznych danych
9. **DynamoDB** - król dla serverless na AWS, pay-per-request
10. **Transakcje** - zawsze `defer tx.Rollback()` przed `tx.Commit()`
11. **Saga pattern** - dla distributed transactions w mikrousługach
12. **Caching** - Redis dla shared cache, in-memory dla szybkości
13. **Migrations** - golang-migrate dla wersjonowania schematu
14. **Context** - używaj `QueryContext` dla timeouts
15. **AI bugs** - szczególnie pilnuj rows.Close(), NULL handling, SQL injection

**Zadania do przećwiczenia:**

1. **Repository pattern** - napisz interface dla User repository, implementację SQL i mock dla testów
2. **Connection monitoring** - stwórz middleware który loguje ile masz aktywnych połączeń
3. **Smart cache** - wrapper który automatycznie cachuje GET queries i invaliduje przy UPDATE/DELETE
4. **Migration system** - napisz prosty system który sprawdza i wykonuje migracje przy starcie
5. **Optimistic locking** - dodaj pole `version` i sprawdzaj przy UPDATE czy nikt nie zmienił rekordu
6. **Batch insert** - napisz funkcję która dzieli 10000 rekordów na batche i insertuje równolegle
7. **DynamoDB local** - postaw lokalnie DynamoDB i napisz CRUD
8. **Transaction helper** - ulepsz `withTransaction` helper żeby przyjmował context i options

**Debugging tips:**

- `db.Stats()` - pokazuje statystyki pool'a (ile open, idle, waiting)
- `EXPLAIN ANALYZE` przed zapytaniem - pokaże query plan
- Włącz query logging w driverze dla developmentu
- Użyj DataGrip lub DBeaver do podglądu bazy
- W testach używaj testcontainers dla prawdziwej bazy

**Następny rozdział:** AWS Lambda z Go - jak zoptymalizować cold starts, zarządzać secrets, i dlaczego Lambda + DynamoDB to match made in heaven dla mikrousług. Pokażę ci jak zbudować serverless API które skaluje się do milionów requestów bez żadnej administracji.