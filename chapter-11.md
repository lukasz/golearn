# Rozdział 11: gRPC i komunikacja między usługami

Pamiętasz jak w PHP robiłeś REST API? JSON tam, JSON z powrotem, ręczne walidacje, brak typów... gRPC to jest jak przesiadka z malucha na tesle. Protocol Buffers dają ci strongly-typed kontrakty, gRPC automatyczną generację klientów i serwerów, a do tego streaming w obie strony. Brzmi skomplikowanie? Zobaczysz, że to prostsze niż kolejny framework PHP do REST-ów.

## Protocol Buffers - typowany kontrakt między serwisami

Protocol Buffers (protobuf) to język definicji interfejsów. Zamiast dokumentować w README jakie pola ma twój JSON, piszesz `.proto` file i generujesz kod dla dowolnego języka.

### Podstawy protobuf

```protobuf
// user.proto - to jest plik definicji, nie kod Go!
syntax = "proto3";  // wersja składni protobuf

// package definiuje namespace w protobuf (nie mylić z Go package)
package user.v1;

// Ta opcja mówi gdzie wygenerować kod Go i jak nazwać package
// github.com/mycompany/api/user/v1 - ścieżka w projekcie
// userv1 - nazwa package w Go (bez kropek!)
option go_package = "github.com/mycompany/api/user/v1;userv1";

// Message to jak struct w Go lub klasa w PHP
message User {
    // Każde pole ma typ, nazwę i UNIKALNY numer
    // Numery są używane w binarnej serializacji (nie zmieniaj ich!)
    int64 id = 1;           // 64-bitowy int, pole numer 1
    string email = 2;       // string, pole numer 2
    string name = 3;        // pole numer 3
    
    // Enum - jak stałe w PHP ale strongly typed
    enum Status {
        // ZAWSZE zacznij od 0 z UNSPECIFIED
        // To ratuje przed błędami gdy ktoś nie ustawi wartości
        STATUS_UNSPECIFIED = 0;
        STATUS_ACTIVE = 1;
        STATUS_INACTIVE = 2;
        STATUS_SUSPENDED = 3;
    }
    
    Status status = 4;  // używamy enum jako typu
    
    // google.protobuf.Timestamp - "well-known type"
    // To nie jest zwykły int64, protobuf wie że to timestamp
    google.protobuf.Timestamp created_at = 5;
    google.protobuf.Timestamp updated_at = 6;
    
    // optional - pole może być null (w Go będzie *string)
    // Bez optional string zawsze ma wartość (pusty string "")
    optional string phone = 7;
    
    // repeated = array/slice w Go
    repeated string roles = 8;  // W Go: []string
    
    // map = mapa/dictionary
    // klucz musi być prosty typ (string, int), wartość dowolna
    map<string, string> metadata = 9;  // W Go: map[string]string
    
    // Nested message - jak nested class w PHP
    message Address {
        string street = 1;
        string city = 2;
        string country = 3;
    }
    
    Address address = 10;  // pole używające nested message
}

// Service to definicja API - jak interface w PHP
service UserService {
    // Unary RPC - zwykły request/response jak REST
    // GetUserRequest wchodzi, GetUserResponse wychodzi
    rpc GetUser(GetUserRequest) returns (GetUserResponse);
    
    // Server streaming - klient wysyła 1 request, 
    // server odpowiada WIELOMA message'ami
    rpc ListUsers(ListUsersRequest) returns (stream User);
    
    // Client streaming - klient wysyła WIELE requestów,
    // server odpowiada 1 response
    rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);
    
    // Bidirectional streaming - obie strony mogą wysyłać
    // wiele message'y jednocześnie (jak WebSocket)
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

// Messages dla request/response - konwencja nazewnictwa
message GetUserRequest {
    int64 id = 1;
}

message GetUserResponse {
    User user = 1;  // zwracamy cały obiekt User
}
```

### Generowanie kodu Go

```bash
# Najpierw instalujemy narzędzia do generowania
# protoc-gen-go - generuje struktury (typy danych)
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# protoc-gen-go-grpc - generuje kod dla serwisów (server/client)
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Generowanie kodu z pliku .proto
# --go_out=. - wygeneruj typy w current directory
# --go-grpc_out=. - wygeneruj service code w current directory
protoc --go_out=. --go-grpc_out=. user.proto

# To stworzy dwa pliki:
# user.pb.go - struktury dla wszystkich message'y
# user_grpc.pb.go - interfejsy i klienty dla service
```

## gRPC Server - implementacja

Teraz napiszemy server który implementuje nasz UserService:

```go
package main

import (
    "context"    // do przekazywania metadanych między calls
    "errors"     // standardowa biblioteka błędów
    "fmt"
    "io"         // do sprawdzania EOF w streamingu
    "log"
    "net"        // do nasłuchiwania na porcie TCP
    "sync"       // do synchronizacji (mutex)
    "time"
    
    "google.golang.org/grpc"        // główna biblioteka gRPC
    "google.golang.org/grpc/codes"  // kody błędów (jak HTTP status codes)
    "google.golang.org/grpc/status" // do tworzenia błędów gRPC
    "google.golang.org/protobuf/types/known/timestamppb" // dla Timestamp
    
    // Import wygenerowanego kodu - zwróć uwagę na alias "userv1"
    userv1 "github.com/mycompany/api/user/v1"
)

// userServer to nasza implementacja service'u
type userServer struct {
    // WAŻNE: zawsze embeduj Unimplemented server!
    // To zapewnia forward compatibility - gdy dodasz nową metodę
    // do .proto, stary kod nadal się skompiluje
    userv1.UnimplementedUserServiceServer
    
    mu    sync.RWMutex               // mutex do synchronizacji
    users map[int64]*userv1.User     // nasza "baza danych" w pamięci
}

// GetUser - implementacja unary RPC (jak zwykły REST endpoint)
// ctx context.Context - zawiera deadline, cancellation, metadata
// req *userv1.GetUserRequest - wygenerowany typ z protobuf
// returns: response i error (konwencja Go)
func (s *userServer) GetUser(
    ctx context.Context, 
    req *userv1.GetUserRequest,
) (*userv1.GetUserResponse, error) {
    // ZAWSZE waliduj input!
    if req.Id <= 0 {
        // status.Error tworzy gRPC error z kodem (jak HTTP 400)
        // codes.InvalidArgument = HTTP 400 Bad Request
        return nil, status.Error(codes.InvalidArgument, "invalid user id")
    }
    
    // RLock = Read Lock - wiele goroutines może czytać jednocześnie
    s.mu.RLock()
    user, exists := s.users[req.Id]  // szukamy w mapie
    s.mu.RUnlock()  // ZAWSZE unlock! defer jest bezpieczniejsze
    
    if !exists {
        // codes.NotFound = HTTP 404
        // status.Errorf pozwala formatować message
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.Id)
    }
    
    // WAŻNE: robimy kopię! Inaczej klient mógłby zmodyfikować
    // nasz wewnętrzny stan przez wskaźnik
    userCopy := &userv1.User{
        Id:        user.Id,
        Email:     user.Email,
        Name:      user.Name,
        Status:    user.Status,
        CreatedAt: user.CreatedAt,
        UpdatedAt: user.UpdatedAt,
    }
    
    // Zwracamy response z zagnieżdżonym user object
    return &userv1.GetUserResponse{
        User: userCopy,
    }, nil
}

// ListUsers - Server streaming RPC
// req - normalny request
// stream - specjalny obiekt do wysyłania wielu responses
func (s *userServer) ListUsers(
    req *userv1.ListUsersRequest, 
    stream userv1.UserService_ListUsersServer,  // to nie jest zwykły response!
) error {
    // Domyślny page size z walidacją
    pageSize := req.PageSize
    if pageSize <= 0 || pageSize > 100 {
        pageSize = 10
    }
    
    // Lock na cały czas streamingu (może to być problem w produkcji!)
    s.mu.RLock()
    defer s.mu.RUnlock()  // defer = wykonaj gdy funkcja się kończy
    
    count := 0
    // Iterujemy po wszystkich userach
    for _, user := range s.users {
        // KLUCZOWE: sprawdzamy czy klient się nie rozłączył!
        // Context.Err() zwraca non-nil gdy context jest cancelled
        if stream.Context().Err() != nil {
            return status.Error(codes.Canceled, "client disconnected")
        }
        
        // Send() wysyła jednego usera do klienta
        // To NIE kończy funkcji - możemy wysłać wiele
        if err := stream.Send(user); err != nil {
            return err  // gRPC samo zamieni na proper status
        }
        
        count++
        if count >= int(pageSize) {
            break
        }
        
        // Symulacja opóźnienia - w prawdziwej aplikacji
        // to może być czas pobierania z bazy
        time.Sleep(100 * time.Millisecond)
    }
    
    // Gdy return nil, stream się kończy
    return nil
}

// CreateUsers - Client streaming RPC
// stream pozwala odbierać wiele requestów
func (s *userServer) CreateUsers(
    stream userv1.UserService_CreateUsersServer,
) error {
    var createdIDs []int64  // zbieramy ID stworzonych userów
    createdCount := 0
    
    // Pętla nieskończona - czytamy aż klient skończy
    for {
        // Recv() blokuje i czeka na następny message od klienta
        req, err := stream.Recv()
        if err != nil {
            // io.EOF = End Of File = klient skończył wysyłać
            if errors.Is(err, io.EOF) {
                // SendAndClose wysyła ostateczny response i zamyka stream
                return stream.SendAndClose(&userv1.CreateUsersResponse{
                    CreatedCount: int32(createdCount),
                    CreatedIds:   createdIDs,
                })
            }
            // Inny błąd - przerwij
            return err
        }
        
        // Walidacja każdego requesta
        if req.User == nil {
            return status.Error(codes.InvalidArgument, "user is required")
        }
        
        // Lock do zapisu (exclusive - tylko 1 goroutine)
        s.mu.Lock()
        
        // Generujemy ID (w prawdziwej app użyj UUID!)
        newID := int64(len(s.users) + 1)
        
        // Tworzymy nowego usera
        user := &userv1.User{
            Id:        newID,
            Email:     req.User.Email,
            Name:      req.User.Name,
            Status:    userv1.User_STATUS_ACTIVE,  // enum value
            CreatedAt: timestamppb.Now(),  // konwersja time.Time na protobuf
            UpdatedAt: timestamppb.Now(),
        }
        
        s.users[newID] = user
        s.mu.Unlock()  // WAŻNE: unlock jak najszybciej!
        
        createdIDs = append(createdIDs, newID)
        createdCount++
    }
}

// Chat - Bidirectional streaming
// Możemy jednocześnie odbierać i wysyłać
func (s *userServer) Chat(
    stream userv1.UserService_ChatServer,
) error {
    // W prawdziwej aplikacji broadcast do wszystkich klientów
    // Tu tylko echo dla demonstracji
    
    for {
        // Recv() czeka na message od klienta
        msg, err := stream.Recv()
        if err != nil {
            if errors.Is(err, io.EOF) {
                return nil  // klient się rozłączył
            }
            return err
        }
        
        // Tworzymy response
        response := &userv1.ChatMessage{
            UserId:    msg.UserId,
            Message:   fmt.Sprintf("Echo: %s", msg.Message),
            Timestamp: timestamppb.Now(),
        }
        
        // Send() wysyła do klienta (może być w tym samym czasie co Recv!)
        if err := stream.Send(response); err != nil {
            return err
        }
    }
}

func main() {
    // Tworzymy instancję naszego serwera
    server := &userServer{
        users: make(map[int64]*userv1.User),  // inicjalizacja mapy
    }
    
    // Dodajemy przykładowe dane
    server.users[1] = &userv1.User{
        Id:        1,
        Email:     "jan@example.com",
        Name:      "Jan Kowalski",
        Status:    userv1.User_STATUS_ACTIVE,
        CreatedAt: timestamppb.Now(),
        UpdatedAt: timestamppb.Now(),
    }
    
    // Tworzymy gRPC server (jak http.Server)
    grpcServer := grpc.NewServer()
    
    // KLUCZOWE: rejestrujemy naszą implementację
    // To łączy wygenerowany kod z naszą implementacją
    userv1.RegisterUserServiceServer(grpcServer, server)
    
    // Nasłuchujemy na porcie TCP (jak w HTTP server)
    lis, err := net.Listen("tcp", ":50051")  // port 50051 to konwencja dla gRPC
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    
    log.Println("gRPC server listening on :50051")
    
    // Serve() blokuje i obsługuje connections (jak http.Serve)
    if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}
```

## gRPC Client - konsumowanie serwisu

Teraz napiszemy klienta który woła nasz serwis:

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "io"
    "log"
    "strings"
    "time"
    
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/protobuf/types/known/timestamppb"
    
    userv1 "github.com/mycompany/api/user/v1"
)

func main() {
    // Łączymy się z serwerem
    // grpc.Dial tworzy connection pool (nie pojedyncze połączenie!)
    conn, err := grpc.Dial(
        "localhost:50051",  // adres serwera
        // W produkcji użyj TLS! insecure tylko do developmentu
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatalf("failed to connect: %v", err)
    }
    defer conn.Close()  // ZAWSZE zamknij connection
    
    // Tworzymy klienta z wygenerowanego kodu
    // To tylko wrapper na connection - lekki obiekt
    client := userv1.NewUserServiceClient(conn)
    
    // Przykłady różnych typów RPC
    unaryCall(client)
    serverStreaming(client)
    clientStreaming(client)
    bidirectionalStreaming(client)
}

// Unary call - jak zwykły REST request
func unaryCall(client userv1.UserServiceClient) {
    // Context z timeout - ZAWSZE dawaj timeout!
    // Po 5 sekundach request zostanie anulowany
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()  // cancel zwalnia zasoby contextu
    
    // Wywołujemy metodę - wygląda jak lokalne wywołanie!
    resp, err := client.GetUser(ctx, &userv1.GetUserRequest{
        Id: 1,
    })
    if err != nil {
        // Tu możesz sprawdzić typ błędu
        // st, ok := status.FromError(err)
        // if st.Code() == codes.NotFound { ... }
        log.Printf("GetUser failed: %v", err)
        return
    }
    
    // resp.User to *userv1.User - strongly typed!
    log.Printf("Got user: %+v", resp.User)
}

// Server streaming - odbieramy wiele responses
func serverStreaming(client userv1.UserServiceClient) {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    // ListUsers zwraca stream object, nie response!
    stream, err := client.ListUsers(ctx, &userv1.ListUsersRequest{
        PageSize: 5,
    })
    if err != nil {
        log.Printf("ListUsers failed: %v", err)
        return
    }
    
    // Czytamy w pętli aż do końca streamu
    for {
        // Recv() blokuje i czeka na następny User
        user, err := stream.Recv()
        if err != nil {
            // io.EOF = koniec streamu (to normalne!)
            if errors.Is(err, io.EOF) {
                break
            }
            // Prawdziwy błąd
            log.Printf("stream recv failed: %v", err)
            return
        }
        
        // user to *userv1.User - pełny obiekt
        log.Printf("Received user: %+v", user)
    }
}

// Client streaming - wysyłamy wiele requestów
func clientStreaming(client userv1.UserServiceClient) {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    // CreateUsers zwraca stream do wysyłania
    stream, err := client.CreateUsers(ctx)
    if err != nil {
        log.Printf("CreateUsers failed: %v", err)
        return
    }
    
    // Dane do wysłania
    users := []string{"Alice", "Bob", "Charlie"}
    
    // Wysyłamy każdego usera osobno
    for _, name := range users {
        req := &userv1.CreateUserRequest{
            User: &userv1.User{
                Email: fmt.Sprintf("%s@example.com", strings.ToLower(name)),
                Name:  name,
            },
        }
        
        // Send() wysyła request do serwera
        if err := stream.Send(req); err != nil {
            log.Printf("send failed: %v", err)
            return
        }
        
        time.Sleep(500 * time.Millisecond)  // symulacja opóźnienia
    }
    
    // CloseAndRecv kończy wysyłanie i czeka na response
    // To mówi serwerowi "skończyłem, daj mi wynik"
    resp, err := stream.CloseAndRecv()
    if err != nil {
        log.Printf("CloseAndRecv failed: %v", err)
        return
    }
    
    log.Printf("Created %d users: %v", resp.CreatedCount, resp.CreatedIds)
}

// Bidirectional streaming - ping-pong z serwerem
func bidirectionalStreaming(client userv1.UserServiceClient) {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    // Chat zwraca stream do wysyłania I odbierania
    stream, err := client.Chat(ctx)
    if err != nil {
        log.Printf("Chat failed: %v", err)
        return
    }
    
    // Goroutine do odbierania (działa równolegle!)
    go func() {
        for {
            // Recv w osobnym wątku
            msg, err := stream.Recv()
            if err != nil {
                if errors.Is(err, io.EOF) {
                    return
                }
                log.Printf("recv failed: %v", err)
                return
            }
            
            log.Printf("Received: %s", msg.Message)
        }
    }()
    
    // Główny wątek wysyła
    messages := []string{"Hello", "How are you?", "Goodbye"}
    
    for _, text := range messages {
        msg := &userv1.ChatMessage{
            UserId:    1,
            Message:   text,
            Timestamp: timestamppb.Now(),
        }
        
        if err := stream.Send(msg); err != nil {
            log.Printf("send failed: %v", err)
            return
        }
        
        time.Sleep(2 * time.Second)
    }
    
    // CloseSend mówi serwerowi że skończyliśmy wysyłać
    // ale nadal możemy odbierać!
    stream.CloseSend()
}
```

## Interceptors i middleware

Interceptory w gRPC to jak middleware w HTTP - mogą przechwytywać i modyfikować requesty/response. To jak przed każdym kontrolerem w PHP miałbyś funkcję która może zmodyfikować request.

```go
import (
    "google.golang.org/grpc/metadata"  // do przesyłania headers
    "google.golang.org/grpc/peer"      // info o kliencie
)

// Unary interceptor - dla pojedynczych calls (nie streaming)
// To jak middleware w HTTP - dostaje request, może go zmodyfikować,
// wywołać handler (lub nie!) i zmodyfikować response
func loggingUnaryInterceptor(
    ctx context.Context,              // context z metadanymi
    req interface{},                  // request (any type)
    info *grpc.UnaryServerInfo,       // info o wywoływanej metodzie
    handler grpc.UnaryHandler,        // właściwa funkcja do wywołania
) (interface{}, error) {
    start := time.Now()
    
    // Log PRZED wywołaniem handlera
    // info.FullMethod to np. "/user.v1.UserService/GetUser"
    log.Printf("Starting %s", info.FullMethod)
    
    // Wywołujemy właściwy handler (to jak next() w PHP middleware)
    resp, err := handler(ctx, req)
    
    // Log PO wywołaniu
    duration := time.Since(start)
    if err != nil {
        log.Printf("Failed %s: %v (took %v)", info.FullMethod, err, duration)
    } else {
        log.Printf("Completed %s (took %v)", info.FullMethod, duration)
    }
    
    // Zwracamy response (możemy go zmodyfikować!)
    return resp, err
}

// Authentication interceptor - sprawdza token przed wywołaniem
func authUnaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    // Niektóre endpointy nie wymagają auth
    if info.FullMethod == "/user.v1.UserService/Login" {
        return handler(ctx, req)  // przepuść bez sprawdzania
    }
    
    // metadata to jak HTTP headers
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        // Brak metadata = brak headers
        return nil, status.Error(codes.Unauthenticated, "no metadata")
    }
    
    // Get zwraca []string bo header może mieć wiele wartości
    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Error(codes.Unauthenticated, "no token")
    }
    
    // Sprawdzamy format tokena
    token := tokens[0]  // bierzemy pierwszy
    if !strings.HasPrefix(token, "Bearer ") {
        return nil, status.Error(codes.Unauthenticated, "invalid token format")
    }
    
    // W prawdziwej aplikacji - weryfikacja JWT
    tokenValue := strings.TrimPrefix(token, "Bearer ")
    userID := validateToken(tokenValue)  // twoja funkcja
    if userID == 0 {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }
    
    // Dodajemy user ID do contextu - handler może to pobrać
    // context.WithValue tworzy NOWY context z dodaną wartością
    ctx = context.WithValue(ctx, "userID", userID)
    
    // Wywołujemy handler z NOWYM contextem
    return handler(ctx, req)
}

// Rate limiting - ograniczanie liczby requestów
type rateLimiter struct {
    requests map[string]int  // licznik per IP
    mu       sync.Mutex      // synchronizacja dostępu do mapy
}

func (rl *rateLimiter) unaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    // peer.FromContext daje info o kliencie (IP, etc)
    p, ok := peer.FromContext(ctx)
    if !ok {
        return nil, status.Error(codes.Internal, "no peer info")
    }
    
    clientIP := p.Addr.String()  // np. "192.168.1.1:54321"
    
    // Sprawdzamy limit (mutex bo mapa nie jest thread-safe!)
    rl.mu.Lock()
    count := rl.requests[clientIP]
    if count >= 100 {  // 100 req per minute
        rl.mu.Unlock()  // WAŻNE: unlock przed return!
        return nil, status.Error(codes.ResourceExhausted, "rate limit exceeded")
    }
    rl.requests[clientIP]++
    rl.mu.Unlock()
    
    // Reset counter po minucie (uproszczone - użyj proper rate limiter)
    go func() {
        time.Sleep(time.Minute)
        rl.mu.Lock()
        rl.requests[clientIP]--
        if rl.requests[clientIP] <= 0 {
            delete(rl.requests, clientIP)  // cleanup
        }
        rl.mu.Unlock()
    }()
    
    return handler(ctx, req)
}

// Rejestracja interceptorów przy tworzeniu serwera
func createServerWithInterceptors() *grpc.Server {
    rl := &rateLimiter{
        requests: make(map[string]int),
    }
    
    // ChainUnaryInterceptor łączy wiele interceptorów
    // Wykonują się w kolejności: logging -> auth -> rate limit -> handler
    opts := []grpc.ServerOption{
        grpc.ChainUnaryInterceptor(
            loggingUnaryInterceptor,
            authUnaryInterceptor,
            rl.unaryInterceptor,
        ),
    }
    
    return grpc.NewServer(opts...)
}
```

## Client interceptors

Klient też może mieć interceptory - do retry, timeout, dodawania headers:

```go
// Retry interceptor - powtarza failed requests
func retryUnaryInterceptor(
    ctx context.Context,
    method string,              // np. "/user.v1.UserService/GetUser"  
    req, reply interface{},     // request i response (any type)
    cc *grpc.ClientConn,        // connection do serwera
    invoker grpc.UnaryInvoker,  // funkcja do wywołania RPC
    opts ...grpc.CallOption,    // dodatkowe opcje
) error {
    var lastErr error
    
    // Próbujemy 3 razy
    for i := 0; i < 3; i++ {
        // invoker to właściwe wywołanie RPC
        err := invoker(ctx, method, req, reply, cc, opts...)
        if err == nil {
            return nil  // sukces!
        }
        
        lastErr = err
        
        // Sprawdzamy czy błąd jest "retryable"
        st, ok := status.FromError(err)
        if !ok {
            return err  // nie gRPC error - nie retry
        }
        
        // Retry tylko dla niektórych kodów
        switch st.Code() {
        case codes.Unavailable:      // serwer niedostępny
        case codes.DeadlineExceeded:  // timeout
            log.Printf("Retrying %s (attempt %d)", method, i+1)
            // Exponential backoff - czekamy coraz dłużej
            time.Sleep(time.Duration(i) * time.Second)
        default:
            return err  // inny błąd - nie retry
        }
    }
    
    return lastErr
}

// Metadata interceptor - dodaje headers do każdego requesta
func metadataUnaryInterceptor(
    ctx context.Context,
    method string,
    req, reply interface{},
    cc *grpc.ClientConn,
    invoker grpc.UnaryInvoker,
    opts ...grpc.CallOption,
) error {
    // Tworzymy metadata (jak HTTP headers)
    md := metadata.New(map[string]string{
        "x-request-id":     generateRequestID(),  // tracking
        "x-client-version": "1.0.0",              // wersja klienta
    })
    
    // NewOutgoingContext dodaje metadata do contextu
    // To będzie wysłane jako headers
    ctx = metadata.NewOutgoingContext(ctx, md)
    
    // Wywołujemy z nowym contextem
    return invoker(ctx, method, req, reply, cc, opts...)
}
```

## Service discovery i load balancing

W mikrousługach nie znasz adresu serwisu z góry - musisz go znaleźć (service discovery) i rozłożyć ruch między wiele instancji (load balancing).

```go
// Client-side load balancing z DNS
func connectWithLoadBalancing() (*grpc.ClientConn, error) {
    // dns:/// mówi gRPC żeby użył DNS resolver
    // user-service może mieć wiele IP (A records)
    conn, err := grpc.Dial(
        "dns:///user-service:50051",
        grpc.WithDefaultServiceConfig(`{
            "loadBalancingPolicy": "round_robin",
            "healthCheckConfig": {
                "serviceName": "user.v1.UserService"
            }
        }`),
    )
    return conn, err
}

// Service discovery z Consul (popularne w mikrousługach)
type ConsulResolver struct {
    client  *api.Client  // Consul client
    service string       // nazwa serwisu do znalezienia
}

func (r *ConsulResolver) ResolveNow(o resolver.ResolveNowOptions) {
    // Pytamy Consul o zdrowe instancje serwisu
    services, _, err := r.client.Health().Service(
        r.service,  // np. "user-service"
        "",         // tag filter
        true,       // tylko healthy
        nil,        // query options
    )
    if err != nil {
        return
    }
    
    // Konwertujemy na adresy dla gRPC
    var addrs []resolver.Address
    for _, s := range services {
        addr := fmt.Sprintf("%s:%d", 
            s.Service.Address,  // IP
            s.Service.Port,      // port
        )
        addrs = append(addrs, resolver.Address{
            Addr: addr,
        })
    }
    
    // Update gRPC z nowymi adresami
    // gRPC automatycznie rozłoży ruch między nie
    r.cc.UpdateState(resolver.State{
        Addresses: addrs,
    })
}
```

## Best practices i pułapki

### 1. Versioning API - jak nie zepsuć backward compatibility

```protobuf
// ZAWSZE używaj wersji w package name
package user.v1;  // nie: package user

message User {
    int64 id = 1;
    string email = 2;
    
    // Dodajesz nowe pole? Użyj NOWEGO numeru!
    string phone = 10;  // OK - nowy numer
    
    // NIGDY nie zmieniaj numerów istniejących pól!
    // string email = 3;  // ŹLE - złamie deserializację!
    
    // NIGDY nie usuwaj pól - oznacz jako deprecated
    string old_field = 5 [deprecated = true];
    
    // Dla usuniętych pól użyj reserved
    // To zapobiega przypadkowemu reużyciu numerów
    reserved 3, 4;  // numery
    reserved "old_name", "legacy_field";  // nazwy
}

// Gdy potrzebujesz breaking changes - nowa wersja!
package user.v2;  // nowy package
```

### 2. Error handling - używaj proper status codes

```go
// ZAWSZE używaj status package, nie fmt.Errorf!
import "google.golang.org/grpc/status"

func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    // Proste błędy
    if req.Id <= 0 {
        // codes.InvalidArgument = HTTP 400
        return nil, status.Error(codes.InvalidArgument, "invalid id")
    }
    
    // Błędy z dodatkowymi szczegółami
    st := status.New(codes.NotFound, "user not found")
    st, _ = st.WithDetails(&errdetails.ResourceInfo{
        ResourceType: "user",
        ResourceName: fmt.Sprintf("users/%d", req.Id),
        Description:  "User does not exist",
    })
    
    return nil, st.Err()
}

// Klient może wyciągnąć szczegóły
if err != nil {
    st := status.Convert(err)  // konwersja na Status
    
    // Sprawdzenie kodu
    if st.Code() == codes.NotFound {
        log.Printf("Not found: %s", st.Message())
    }
    
    // Wyciągnięcie details
    for _, detail := range st.Details() {
        switch t := detail.(type) {
        case *errdetails.ResourceInfo:
            log.Printf("Resource error: %s", t.Description)
        }
    }
}
```

### 3. Context i deadlines - ZAWSZE używaj!

```go
// Server - respektuj deadline klienta
func (s *userServer) LongOperation(ctx context.Context, req *Request) (*Response, error) {
    // Sprawdź czy klient ustawił deadline
    deadline, ok := ctx.Deadline()
    if ok {
        remaining := time.Until(deadline)
        log.Printf("Mam %v czasu na odpowiedź", remaining)
    }
    
    // Długa operacja z sprawdzaniem czy anulowano
    for i := 0; i < 100; i++ {
        select {
        case <-ctx.Done():
            // Context anulowany - klient się rozłączył lub timeout
            return nil, status.Error(codes.DeadlineExceeded, "cancelled")
        default:
            // Robimy robotę
            time.Sleep(100 * time.Millisecond)
        }
    }
    
    return &Response{}, nil
}

// Client - ZAWSZE dawaj timeout!
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()  // zwalnia zasoby

resp, err := client.GetUser(ctx, req)
```

## AI checkpoint - na co uważać

Kiedy AI generuje ci kod gRPC, sprawdź te rzeczy:

### 1. Numeracja pól w protobuf
```protobuf
// AI często duplikuje numery
message User {
    string name = 1;
    string email = 1;  // ŹLE! Ten sam numer!
}
```

### 2. Brak error handling
```go
// AI generuje:
resp, _ := client.GetUser(ctx, req)  // ignoruje błąd!

// Popraw na:
resp, err := client.GetUser(ctx, req)
if err != nil {
    return fmt.Errorf("failed to get user: %w", err)
}
```

### 3. Brak context timeout
```go
// AI generuje:
resp, err := client.GetUser(context.Background(), req)  // brak timeout!

// Popraw na:
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
resp, err := client.GetUser(ctx, req)
```

### 4. Synchronizacja w streaming
```go
// AI zapomina o mutex w streaming handlers
func (s *Server) StreamData(req *Request, stream Service_StreamDataServer) error {
    // AI nie daje mutex!
    for id, data := range s.data {  // concurrent map access - PANIC!
        stream.Send(data)
    }
}

// Popraw na:
func (s *Server) StreamData(req *Request, stream Service_StreamDataServer) error {
    s.mu.RLock()  // Read lock
    defer s.mu.RUnlock()
    
    for id, data := range s.data {
        if err := stream.Send(data); err != nil {  // sprawdź błąd!
            return err
        }
    }
    return nil
}
```

### 5. Resource cleanup
```go
// AI zapomina o cleanup
conn, _ := grpc.Dial("localhost:50051")
client := NewClient(conn)
// używa client...
// ALE NIE ZAMYKA CONNECTION!

// Popraw na:
conn, err := grpc.Dial("localhost:50051")
if err != nil {
    return err
}
defer conn.Close()  // ZAWSZE zamknij!
```

## Podsumowanie

gRPC to potężne narzędzie dla komunikacji między mikrousługami. W porównaniu do REST API które znasz z PHP:

**Zalety gRPC:**
- **Strongly typed** - błędy typów łapiesz podczas kompilacji, nie w runtime
- **Automatyczna generacja** - nie piszesz boilerplate dla klientów
- **Performance** - binarna serializacja szybsza niż JSON
- **Streaming** - natywne wsparcie dla real-time komunikacji
- **Built-in features** - health checking, load balancing, retry

**Wady gRPC:**
- **Trudniejsze debugowanie** - binary format, nie zobaczysz w curl
- **Browser support** - nie działa natywnie (potrzebny grpc-web proxy)
- **Learning curve** - więcej konceptów niż prosty REST

**Kiedy używać:**
- **Backend-to-backend** - między mikrousługami gRPC świetnie się sprawdza
- **Real-time** - gdy potrzebujesz streaming (chat, live updates)
- **Performance critical** - gdy liczy się każda milisekunda

**Kiedy NIE używać:**
- **Public API** - REST jest bardziej uniwersalny
- **Browser clients** - lepszy REST/GraphQL/WebSocket
- **Simple CRUD** - REST może być prostszy dla podstawowych operacji

Pamiętaj: jako Engineering Manager wiesz, że najlepsze narzędzie to takie które zespół zna i umie używać. gRPC ma stromą krzywą uczenia, ale gdy zespół je opanuje, produktywność wzrasta znacząco dzięki automatycznej generacji kodu i type safety.

W następnym rozdziale zajmiemy się bazami danych - jak Go radzi sobie z SQL, NoSQL i jak zarządzać transakcjami w rozproszonej architekturze mikrousług.