# Rozdział 10: HTTP i REST API

Dotarliśmy do serca mikrousług - HTTP API. W PHP prawdopodobnie używałeś Laravel'a albo Symfony i miałeś wszystko podane na tacy: routing, middleware, walidację, serializację. Pamiętasz jak definiowałeś routes w `routes/web.php` albo konfiguracji YAML w Symfony? W Go jest inaczej.

Go daje ci standardową bibliotekę która jest potężna, ale minimalna. To jak dostać Lego Technic zamiast gotowego Millennium Falcona - możesz zbudować wszystko, ale musisz wiedzieć jak. Jako Engineering Manager który przez lata wspierał zespoły budujące na różnych frameworkach, powiem ci - Go nie ma czarnej magii. Widzisz każdy kawałek kodu, rozumiesz co się dzieje, i kiedy coś się psuje (a będzie się psuć), wiesz gdzie szukać.

## Pakiet net/http w szczegółach

Zacznijmy od podstaw. Go ma wbudowany HTTP server który jest production-ready. Nie potrzebujesz Apache, nginx jako serwera aplikacji, ani PHP-FPM. Po prostu uruchamiasz binary i masz działający serwer HTTP. To może być szokiem po latach stacka LAMP/LEMP, ale zobacz jak to wygląda:

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

// Handler function - podstawowa jednostka w Go HTTP
// To jest jak kontroler w Laravel, ale bardziej niskopoziomowy
func helloHandler(w http.ResponseWriter, r *http.Request) {
    // w - tu piszesz response (jak echo w PHP, ale z większą kontrolą)
    // r - tu masz request (jak $_GET, $_POST, $_SERVER w PHP, ale w jednej strukturze)
    
    fmt.Fprintf(w, "Hello from Go! Method: %s, URL: %s", r.Method, r.URL.Path)
}

// Handler jako metoda struktury - lepsze dla większych aplikacji
// Przypomina kontrolery w klasach PHP, ale bez magii konstruktora
type Server struct {
    startTime time.Time
    // Tu mogłyby być dependencje jak database connection, logger, cache
    // W Laravel wstrzykiwałbyś je przez constructor injection
}

func (s *Server) statusHandler(w http.ResponseWriter, r *http.Request) {
    uptime := time.Since(s.startTime)
    // W PHP użyłbyś json_encode() + header(), tu masz więcej kontroli
    fmt.Fprintf(w, "Server uptime: %v", uptime)
}

func main() {
    server := &Server{startTime: time.Now()}
    
    // Rejestracja handlerów - podobnie jak Route::get() w Laravel
    // Ale bez magii automatycznego odkrywania routes
    http.HandleFunc("/hello", helloHandler)
    http.HandleFunc("/status", server.statusHandler)
    
    // Konfiguracja serwera - w PHP to byłoby w php.ini, apache config lub .env
    // Tu masz wszystko w kodzie, jawnie skonfigurowane
    srv := &http.Server{
        Addr:         ":8080",
        ReadTimeout:  15 * time.Second,  // timeout na czytanie requesta - max_execution_time w PHP
        WriteTimeout: 15 * time.Second,  // timeout na pisanie response
        IdleTimeout:  60 * time.Second,  // timeout na keep-alive connections
    }
    
    log.Println("Server starting on :8080")
    // W PHP nigdy nie widziałeś tego kodu - serwer startował "magicznie"
    // Tu widzisz dokładnie co się dzieje i masz pełną kontrolę
    log.Fatal(srv.ListenAndServe())
}
```

**Kluczowe różnice z PHP które musisz zrozumieć:**

1. **`http.ResponseWriter`** - to jest interface do pisania odpowiedzi. W PHP po prostu `echo` albo `return response()` w Laravel. Tu masz pełną kontrolę nad headerami i body, ale musisz pamiętać o kolejności (najpierw headery, potem content).

2. **`http.Request`** - zawiera wszystko o żądaniu w jednej strukturze. W PHP to było rozproszone po `$_GET`, `$_POST`, `$_SERVER`, `$_FILES`. Tu masz wszystko w jednym miejscu, ale musisz wiedzieć jak to wyciągnąć.

3. **Explicit timeouts** - w PHP to Apache/nginx zarządzał timeoutami. Tu masz pełną kontrolę i **musisz** je ustawić, inaczej możesz dostać memory leaki lub hanging connections.

**GoLand super feature**: Po uruchomieniu serwera GoLand pokaże clickable link w konsoli. Kliknij i otworzy się przeglądarka z localhost:8080 - jak built-in browser preview.

### HTTP Request - anatomia żądania

W PHP request był "magicznie" sparsowany do superglobal arrays przez Apache/nginx + PHP. W Go widzisz i kontrolujesz każdy krok tego procesu:

```go
func analyzeRequest(w http.ResponseWriter, r *http.Request) {
    // Method - GET, POST, PUT, DELETE, etc.
    // W PHP: $_SERVER['REQUEST_METHOD']
    method := r.Method
    
    // URL path i query parameters
    // W PHP: $_SERVER['REQUEST_URI'] dla path, $_GET dla query params
    path := r.URL.Path
    query := r.URL.Query()  // map[string][]string - może być wiele wartości dla tego samego klucza!
    name := query.Get("name")  // pierwszy parametr "name", jak $_GET['name'] ?? '' w PHP
    
    // Headers - w PHP: $_SERVER z prefiksem HTTP_
    userAgent := r.Header.Get("User-Agent")  // odpowiednik $_SERVER['HTTP_USER_AGENT']
    contentType := r.Header.Get("Content-Type")
    
    // Body reading - w PHP to php://input albo $_POST
    // KRYTYCZNA różnica: body możesz przeczytać tylko raz! W PHP można było wielokrotnie
    body, err := io.ReadAll(r.Body)
    if err != nil {
        // W PHP dostałbyś false z file_get_contents(), tu masz explicit error handling
        http.Error(w, "Can't read body", http.StatusBadRequest)
        return
    }
    defer r.Body.Close()  // ZAWSZE zamknij body - w PHP garbage collector to robił za ciebie
    
    // Form data parsing (application/x-www-form-urlencoded)
    // Odpowiednik $_POST w PHP, ale musisz explicite wywołać parsing
    if err := r.ParseForm(); err == nil {
        formName := r.FormValue("name")  // jak $_POST['name'] ?? '' w PHP
        fmt.Printf("Form name: %s\n", formName)
    }
    
    // JSON parsing - w PHP: json_decode(file_get_contents('php://input'))
    if contentType == "application/json" {
        var data map[string]interface{}
        if err := json.Unmarshal(body, &data); err != nil {
            // W PHP sprawdzałbyś json_last_error()
            http.Error(w, "Invalid JSON", http.StatusBadRequest)
            return
        }
        // Pracuj z data - jak z associative array w PHP po json_decode($json, true)
    }
    
    // Response building - masz pełną kontrolę nad każdym elementem
    w.Header().Set("Content-Type", "application/json")  // header('Content-Type: application/json') w PHP
    w.WriteHeader(http.StatusOK)  // http_response_code(200) w PHP
    
    response := map[string]interface{}{
        "method":      method,
        "path":        path,
        "query_name":  name,
        "user_agent":  userAgent,
        "body_length": len(body),
    }
    
    // Odpowiednik echo json_encode($response) w PHP
    json.NewEncoder(w).Encode(response)
}
```

**Kluczowe różnice które jako Engineering Manager obserwujesz w kodzie zespołu:**

1. **Body tylko raz** - w PHP `php://input` można było czytać wielokrotnie (zależnie od konfiguracji). W Go stream się kończy po pierwszym odczycie. To częsta pułapka w kodzie AI.

2. **Manual parsing** - w PHP `$_POST` był automatycznie sparsowany przez serwer. Tu musisz wywołać `ParseForm()` lub `json.Unmarshal()`. Więcej kodu, ale widzisz co się dzieje.

3. **Error handling wszędzie** - w PHP często ignorowałeś błędy (warnings). Tu każda operacja może zwrócić error i musisz go obsłużyć. To dobra praktyka, ale wymaga zmiany myślenia.

**GoLand tip dla eksploracji**: Hover nad `http.Request` - zobaczysz wszystkie dostępne pola z dokumentacją. `Ctrl+B` na `http.Request` przeniesie cię do źródeł Go i zobaczysz pełną strukturę.

### HTTP Response - kontrola odpowiedzi

W PHP miałeś wiele sposobów na response: `echo`, `return`, `header()`, `http_response_code()`, plus różne helpery frameworków. W Go masz jeden spójny sposób, ale z większą kontrolą i odpowiedzialnością:

```go
// Różne sposoby odpowiadania - wszystkie patterns które znasz z PHP, ale bardziej explicit
func responseExamples(w http.ResponseWriter, r *http.Request) {
    switch r.URL.Path {
    case "/text":
        // Plain text response - jak echo w PHP
        w.Header().Set("Content-Type", "text/plain")
        fmt.Fprintf(w, "Hello, World!")
        
    case "/json":
        // JSON response - jak json_encode() + header() w PHP
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(map[string]string{
            "message": "Hello JSON!",
            "status":  "ok",
        })
        
    case "/error":
        // Error response - jak http_response_code(500) + echo w PHP
        http.Error(w, "Something went wrong", http.StatusInternalServerError)
        
    case "/custom-status":
        // Custom status - http_response_code() w PHP
        w.WriteHeader(http.StatusCreated)  // 201 Created - bardzo ważny dla REST
        fmt.Fprintf(w, "Resource created")
        
    case "/redirect":
        // Redirect - jak header('Location: ...') + http_response_code(302) w PHP
        http.Redirect(w, r, "/hello", http.StatusTemporaryRedirect)
        
    case "/headers":
        // Custom headers - jak header() w PHP, ale z większą kontrolą
        w.Header().Set("X-Custom-Header", "MyValue")
        w.Header().Add("Set-Cookie", "session=abc123; HttpOnly")  // Add vs Set - Add dodaje, Set zastępuje
        fmt.Fprintf(w, "Check headers!")
        
    default:
        http.NotFound(w, r)  // 404 - jak w PHP, ale musisz explicite wywołać
    }
}

// Structured response pattern - jak response helper w Laravel
// W PHP często używałeś return response()->json(), tu tworzysz własny helper
type APIResponse struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`  // omitempty = nie serializuj jeśli null/empty
    Error   string      `json:"error,omitempty"`
    Code    int         `json:"code"`
}

// Helper functions - podobnie jak response()->json() w Laravel
// Używasz ich zamiast powtarzania kodu w każdym handlerze
func sendJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    
    response := APIResponse{
        Success: status < 400,  // HTTP status codes < 400 to success
        Data:    data,
        Code:    status,
    }
    
    json.NewEncoder(w).Encode(response)
}

func sendError(w http.ResponseWriter, status int, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    
    response := APIResponse{
        Success: false,
        Error:   message,
        Code:    status,
    }
    
    json.NewEncoder(w).Encode(response)
}
```

**Analogie PHP → Go dla łatwiejszego zrozumienia:**

- `w.Header().Set()` = `header()` w PHP
- `w.WriteHeader()` = `http_response_code()` w PHP
- `fmt.Fprintf(w, ...)` = `echo` w PHP
- `json.NewEncoder(w).Encode()` = `echo json_encode()` w PHP
- `http.Error()` = `http_response_code(500); echo "error"` w PHP

**Ważna różnica z PHP**: Kolejność ma znaczenie. W PHP mogłeś czasem wywołać `header()` po `echo` (z warningiem). W Go po `WriteHeader()` lub pierwszym `Write()` nie możesz już zmieniać headerów. GoLand cię ostrzeże jeśli spróbujesz.

## Routing - gorilla/mux i chi

Domyślny router Go (`http.ServeMux`) jest podstawowy. Jak vanilla PHP bez frameworka - możliwe, ale bolesne. Nie ma parametrów w URL (`/users/{id}`), nie ma middleware per route, nie ma HTTP method constraints. W Laravel masz `Route::get('/users/{id}', ...)`, w Symfony `@Route("/users/{id}", methods={"GET"})`. W Go domyślnie masz tylko proste ścieżki.

Dla prawdziwego API potrzebujesz czegoś lepszego. To jak przejście z vanilla PHP na Laravel - nie ma powrotu.

### Chi Router - lekki i szybki

Chi to popularny router w ekosystemie Go. Lekki (jak Slim w PHP), szybki, kompatybilny z `net/http`, a do tego ma świetne middleware. Składnia przypomina Express.js jeśli kiedyś go używałeś:

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "time"
    
    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
)

// Model - jak Eloquent model, ale bez ORM magii
// W Laravel miałbyś User extends Model, tu masz zwykłą strukturę
type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// Mock database - w prawdziwej aplikacji to byłaby baza danych
// W PHP mógłbyś użyć static property w klasie
var users = []User{
    {ID: 1, Name: "Jan", Email: "jan@example.com"},
    {ID: 2, Name: "Anna", Email: "anna@example.com"},
}

func main() {
    r := chi.NewRouter()
    
    // Global middleware - jak w Laravel app/Http/Kernel.php
    r.Use(middleware.Logger)      // loguj requesty (jak access.log Apache)
    r.Use(middleware.Recoverer)   // łap panic'i (jak try-catch na najwyższym poziomie)
    r.Use(middleware.Timeout(60 * time.Second))  // timeout (max_execution_time w php.ini)
    r.Use(middleware.Compress(5)) // gzip compression (mod_deflate w Apache)
    
    // CORS dla frontend'u - w PHP często robione w .htaccess lub middleware
    r.Use(func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            w.Header().Set("Access-Control-Allow-Origin", "*")
            w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
            w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
            
            // Preflight request handling - browser wysyła OPTIONS przed "real" request
            // W PHP często zapominanty aspekt CORS
            if r.Method == "OPTIONS" {
                w.WriteHeader(http.StatusOK)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    })
    
    // Routes - jak w Laravel routes/api.php
    r.Get("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "API is running!")
    })
    
    // REST endpoints - grupa routes jak Route::group() w Laravel
    r.Route("/api/v1", func(r chi.Router) {
        r.Get("/health", healthCheck)
        
        // Resource routes - jak Route::resource('users', UserController::class) w Laravel
        // Ale tu musisz każdy endpoint zdefiniować explicite
        r.Route("/users", func(r chi.Router) {
            r.Get("/", listUsers)           // GET /api/v1/users
            r.Post("/", createUser)         // POST /api/v1/users
            r.Get("/{id}", getUser)         // GET /api/v1/users/123 (jak {id} w Laravel)
            r.Put("/{id}", updateUser)      // PUT /api/v1/users/123
            r.Delete("/{id}", deleteUser)   // DELETE /api/v1/users/123
        })
    })
    
    // Static files - jak public folder w Laravel
    workDir, _ := os.Getwd()
    filesDir := http.Dir(filepath.Join(workDir, "static"))
    r.Handle("/static/*", http.StripPrefix("/static", http.FileServer(filesDir)))
    
    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", r)
}

// Health check endpoint - standard w mikrousługach
// W PHP często zapominany, ale kluczowy dla load balancerów
func healthCheck(w http.ResponseWriter, r *http.Request) {
    sendJSON(w, http.StatusOK, map[string]string{
        "status":    "ok",
        "timestamp": time.Now().Format(time.RFC3339),
        "service":   "user-api",
    })
}

func listUsers(w http.ResponseWriter, r *http.Request) {
    // Query parameters - jak $_GET w PHP
    // W Laravel: $request->input('limit', 10)
    limit := 10
    if l := r.URL.Query().Get("limit"); l != "" {
        if parsed, err := strconv.Atoi(l); err == nil {
            limit = parsed
        }
    }
    
    // Pagination logic (mock) - w Laravel użyłbyś User::paginate()
    result := users
    if limit < len(result) {
        result = result[:limit]
    }
    
    sendJSON(w, http.StatusOK, result)
}

func getUser(w http.ResponseWriter, r *http.Request) {
    // URL parameter - jak $request->route('id') w Laravel
    // chi używa {id} składni podobnej do Laravel
    idStr := chi.URLParam(r, "id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        sendError(w, http.StatusBadRequest, "Invalid user ID")
        return
    }
    
    // Find user (mock) - w Laravel: User::find($id)
    for _, user := range users {
        if user.ID == id {
            sendJSON(w, http.StatusOK, user)
            return
        }
    }
    
    sendError(w, http.StatusNotFound, "User not found")
}

func createUser(w http.ResponseWriter, r *http.Request) {
    var newUser User
    
    // Parse JSON body - w Laravel: $request->json()->all() lub $request->validate([...])
    if err := json.NewDecoder(r.Body).Decode(&newUser); err != nil {
        sendError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()  // Zawsze pamiętaj - w PHP nie trzeba było
    
    // Basic validation - w Laravel masz Request classes z rules()
    if newUser.Name == "" || newUser.Email == "" {
        sendError(w, http.StatusBadRequest, "Name and email are required")
        return
    }
    
    // Create (mock) - w Laravel: User::create($data)
    newUser.ID = len(users) + 1
    users = append(users, newUser)
    
    // 201 Created - bardzo ważny status dla REST API
    // W PHP często zapominany - używano 200 OK
    sendJSON(w, http.StatusCreated, newUser)
}

func updateUser(w http.ResponseWriter, r *http.Request) {
    idStr := chi.URLParam(r, "id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        sendError(w, http.StatusBadRequest, "Invalid user ID")
        return
    }
    
    var updatedUser User
    if err := json.NewDecoder(r.Body).Decode(&updatedUser); err != nil {
        sendError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    defer r.Body.Close()
    
    // Find and update (mock) - w Laravel: User::find($id)->update($data)
    for i, user := range users {
        if user.ID == id {
            updatedUser.ID = id  // preserve ID - ważne dla REST semantyki
            users[i] = updatedUser
            sendJSON(w, http.StatusOK, updatedUser)
            return
        }
    }
    
    sendError(w, http.StatusNotFound, "User not found")
}

func deleteUser(w http.ResponseWriter, r *http.Request) {
    idStr := chi.URLParam(r, "id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        sendError(w, http.StatusBadRequest, "Invalid user ID")
        return
    }
    
    // Find and delete (mock) - w Laravel: User::destroy($id)
    for i, user := range users {
        if user.ID == id {
            // Slice removing element - w PHP: array_splice() albo unset()
            users = append(users[:i], users[i+1:]...)
            // 204 No Content - standard dla successful DELETE
            w.WriteHeader(http.StatusNoContent)
            return
        }
    }
    
    sendError(w, http.StatusNotFound, "User not found")
}
```

**Kluczowe różnice z Laravel routing które dostrzegasz jako EM:**

1. **Route parameters**: `{id}` w chi vs `{id}` w Laravel - podobna składnia, ale pobierasz przez `chi.URLParam()` zamiast `$request->route()`

2. **Route groups**: `r.Route("/api/v1", func(r chi.Router) {...})` vs `Route::group(['prefix' => 'api/v1'], ...)` - podobna koncepcja, inna składnia

3. **Resource routes**: Musisz je zdefiniować ręcznie, nie ma `Route::resource()`. Więcej kodu, ale większa kontrola.

4. **HTTP status codes**: Musisz pamiętać o właściwych kodach (201 dla POST, 204 dla DELETE). W Laravel frameworki często to robiły za ciebie.

**GoLand integration dla chi:**

1. **Dodawanie dependency**: Terminal → `go get github.com/go-chi/chi/v5`
2. **Auto-import**: Wpisz `chi.` i GoLand zasugeruje import
3. **Method completion**: `Ctrl+Space` na `chi.` pokaże wszystkie dostępne metody z dokumentacją
4. **Quick documentation**: `Ctrl+Q` na metodzie pokaże dokumentację

### Gorilla Mux - funkcjonalność kosztem prostoty

Jeśli chi to Slim Framework, to Gorilla Mux to Laravel - więcej funkcji, więcej kodu, większe możliwości:

```go
import "github.com/gorilla/mux"

func gorillaMuxExample() {
    r := mux.NewRouter()
    
    // Subroutery - jak w Symfony z prefiksami
    api := r.PathPrefix("/api/v1").Subrouter()
    
    // Route constraints z regex - potężniejsze niż w Laravel
    api.HandleFunc("/users/{id:[0-9]+}", getUser).Methods("GET")  // tylko cyfry
    api.HandleFunc("/users/{name:[a-zA-Z]+}", getUserByName).Methods("GET")  // tylko litery
    
    // Host constraints - multi-tenant applications
    api.Host("api.example.com").HandleFunc("/users", listUsers)
    
    // Query constraints - wymusza obecność parametrów
    r.HandleFunc("/search", searchHandler).Queries("q", "")
    
    // Headers constraints - API versioning przez headery
    r.HandleFunc("/api", apiHandler).Headers("X-API-Version", "1.0")
    
    // Custom matchers - własna logika routingu
    r.MatcherFunc(func(r *http.Request, rm *mux.RouteMatch) bool {
        return r.Header.Get("Authorization") != ""
    }).HandlerFunc(protectedHandler)
    
    http.ListenAndServe(":8080", r)
}
```

**Chi vs Gorilla Mux - decision matrix dla EM:**

**Chi wybierz gdy:**
- Performance jest priorytetem (chi jest zauważalnie szybszy w benchmarkach)
- Team preferuje minimalizm (less is more philosophy)
- Middleware ecosystem jest ważny (chi ma świetne builtin middleware)
- Kompatybilność z stdlib (łatwiejsze testowanie, mniej surprises)

**Gorilla Mux gdy:**
- Potrzebujesz advanced routing features (regex constraints, host-based routing)
- Zespół pracuje z legacy code używającym Gorilla ecosystem
- Złożone routing requirements (enterprise applications)
- "Batteries included" approach

**GoLand tip**: Obie biblioteki mają świetne auto-completion. Napisz `r.` i zobaczysz dostępne metody z inline documentation.

## Middleware pattern - łańcuch odpowiedzialności

Middleware to functions które wykonują się przed/po handlerze. To jak onion layers - request przechodzi przez każdą warstwę. W Laravel masz middleware w `app/Http/Middleware/`, w Symfony to event listeners. W Go to pattern, nie framework feature. Ale ten pattern jest potężniejszy niż mogłoby się wydawać.

```go
// Middleware signature w Go - każdy middleware implementuje ten sam typ
// To jest kluczowe dla kompozycji - wszystkie middleware są interchangeable
type Middleware func(http.Handler) http.Handler

// Logging middleware - jak Laravel's LogRequests::class
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Przed handlerem - wykonuje się "w drodze w dół"
        log.Printf("Started %s %s", r.Method, r.URL.Path)
        
        // Wywołaj następny handler w chain - tu jest "przełącznik"
        next.ServeHTTP(w, r)
        
        // Po handlerze - wykonuje się "w drodze w górę"
        log.Printf("Completed %s %s in %v", r.Method, r.URL.Path, time.Since(start))
    })
}

// Authentication middleware - jak auth middleware w Laravel
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        
        if token == "" {
            http.Error(w, "Authorization header required", http.StatusUnauthorized)
            return  // Przerwij chain - jak early return w Laravel middleware
        }
        
        // Validate token (mock) - w Laravel masz Auth::guard()->check()
        if !isValidToken(token) {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }
        
        // Add user info to context - jak Auth::user() w Laravel
        // Context to sposób przekazywania danych między middleware w Go
        userID := getUserIDFromToken(token)
        ctx := context.WithValue(r.Context(), "userID", userID)
        r = r.WithContext(ctx)  // Request jest immutable, musisz stworzyć nowy
        
        next.ServeHTTP(w, r)
    })
}

// CORS middleware - często potrzebne dla SPA applications
func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // W PHP często robione w .htaccess albo na poziomie nginx
        // Tu masz pełną kontrolę nad logiką CORS
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        w.Header().Set("Access-Control-Max-Age", "3600")
        
        // Preflight request - browser wysyła OPTIONS przed "prawdziwym" requestem
        // W PHP middleware często tego nie obsługiwał poprawnie
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}

// Rate limiting middleware - ochrona przed abuse
import "golang.org/x/time/rate"

// Global rate limiter - w production byłby per-IP lub per-user
var limiter = rate.NewLimiter(10, 20) // 10 req/sec, burst capacity 20

func RateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !limiter.Allow() {
            // 429 Too Many Requests - standard HTTP status dla rate limiting
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}

// Request ID middleware - dla tracking i debugging
func RequestIDMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = generateRequestID() // UUID albo random string
        }
        
        // Add to response headers - frontend może użyć w error reporting
        w.Header().Set("X-Request-ID", requestID)
        
        // Add to context for logging - dostępne w handlerach i innych middleware
        ctx := context.WithValue(r.Context(), "requestID", requestID)
        r = r.WithContext(ctx)
        
        next.ServeHTTP(w, r)
    })
}

// Stosowanie middleware z chi - jak w Laravel app/Http/Kernel.php
func setupRoutes() chi.Router {
    r := chi.NewRouter()
    
    // Global middleware - aplikowane do wszystkich routes
    // Kolejność ma znaczenie! Wykonują się w kolejności dodawania
    r.Use(RequestIDMiddleware)
    r.Use(LoggingMiddleware)
    r.Use(CORSMiddleware)
    r.Use(RateLimitMiddleware)
    
    // Route-specific middleware - jak ->middleware('auth') w Laravel
    r.Route("/api", func(r chi.Router) {
        r.Use(AuthMiddleware)  // tylko dla /api/* routes
        r.Get("/profile", getProfile)
        r.Get("/dashboard", getDashboard)
    })
    
    // Public routes (bez auth) - jak routes z ->middleware('guest') w Laravel
    r.Get("/health", healthCheck)
    r.Post("/login", login)
    
    return r
}

// Helper functions dla middleware
func isValidToken(token string) bool {
    // W production: JWT validation, database lookup, cache check
    return token == "Bearer valid-token"
}

func getUserIDFromToken(token string) int {
    // W production: parsowanie JWT, query do bazy
    return 123
}

func generateRequestID() string {
    // W production: UUID, ulid, albo custom format
    return fmt.Sprintf("req-%d", time.Now().UnixNano())
}
```

**Kluczowe różnice z PHP middleware pattern:**

1. **Function wrapping**: W Go middleware "owija" handler funkcją. W Laravel middleware ma `handle()` method który dostaje request i next closure.

2. **Context passing**: W Go używasz `context.Context` do przekazywania danych między middleware. W Laravel modyfikujesz properties na Request object.

3. **Chain breaking**: W Go `return` bez wywoływania `next.ServeHTTP()`. W Laravel return response object w `handle()`.

4. **Immutable Request**: W Go Request jest read-only (prawie), musisz stworzyć nowy z `WithContext()`. W Laravel modyfikujesz istniejący Request.

### Advanced middleware patterns

Te patterns przydają się w skomplikowanych aplikacjach enterprise:

```go
// Middleware z konfiguracją - parametrized middleware
func TimeoutMiddleware(timeout time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.TimeoutHandler(next, timeout, "Request timeout")
    }
}

// Usage: różne timeouts dla różnych endpoints
r.Use(TimeoutMiddleware(30 * time.Second))

// Conditional middleware - wykonuje się tylko gdy warunek spełniony
func ConditionalMiddleware(condition func(*http.Request) bool, mw func(http.Handler) http.Handler) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if condition(r) {
                mw(next).ServeHTTP(w, r)
            } else {
                next.ServeHTTP(w, r)
            }
        })
    }
}

// Usage - auth tylko dla admin routes
r.Use(ConditionalMiddleware(
    func(r *http.Request) bool {
        return strings.HasPrefix(r.URL.Path, "/admin")
    },
    AdminAuthMiddleware,
))

// Recovery middleware z structured logging - production-ready panic handling
func RecoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                requestID, _ := r.Context().Value("requestID").(string)
                
                // Structured logging - w Laravel masz exception handler
                log.Printf("PANIC [%s] %s %s: %v\n%s", 
                    requestID, r.Method, r.URL.Path, err, debug.Stack())
                
                // Nie leak'uj internal errors do klienta
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}

// Middleware chain builder - dla convenience i reusability
type MiddlewareChain []func(http.Handler) http.Handler

func (chain MiddlewareChain) Then(h http.Handler) http.Handler {
    // Apply middleware w reverse order - ostatni wrap jako pierwszy
    for i := len(chain) - 1; i >= 0; i-- {
        h = chain[i](h)
    }
    return h
}

// Usage - jak middleware groups w Laravel
authChain := MiddlewareChain{
    LoggingMiddleware,
    AuthMiddleware,
    RateLimitMiddleware,
}

// Aplikuj chain do handlera
handler := authChain.Then(http.HandlerFunc(protectedHandler))
```

**GoLand middleware development tips:**

- **Extract Method** (`Ctrl+Alt+M`) na powtarzający się kod middleware
- **Quick fix** (`Alt+Enter`) na błędzie: "Convert to middleware pattern"
- **Live Templates** (`Ctrl+J`) → Go → `middleware` - szkielet nowego middleware
- **Debugging**: Postaw breakpoint w middleware i obserwuj flow przez chain

## Request validation i serialization

W PHP miałeś request validation w frameworkach - Laravel Form Requests z `rules()`, Symfony Validator z annotations. W Go musisz to zrobić sam, ale masz pełną kontrolę i type safety. To może wyglądać na więcej pracy, ale w practice daje ci większą elastyczność i lepszy performance.

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "regexp"
    "strings"
    "time"
)

// Validation errors structure - jak ValidationException w Laravel
type ValidationError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

type ValidationErrors []ValidationError

func (ve ValidationErrors) Error() string {
    var messages []string
    for _, err := range ve {
        messages = append(messages, fmt.Sprintf("%s: %s", err.Field, err.Message))
    }
    return strings.Join(messages, "; ")
}

// Request model z validation metadata - jak Laravel Form Request
// W Laravel byś miał rules() method, tu używasz struct tags jako hints
type CreateUserRequest struct {
    Name     string `json:"name" validate:"required,min=2,max=50"`
    Email    string `json:"email" validate:"required,email"`
    Age      int    `json:"age" validate:"min=18,max=120"`
    Password string `json:"password" validate:"required,min=8"`
}

// Custom validator - jak w Laravel custom validation rules
type Validator struct {
    emailRegex *regexp.Regexp
}

func NewValidator() *Validator {
    return &Validator{
        // Email regex - w Laravel masz to built-in
        emailRegex: regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`),
    }
}

func (v *Validator) Validate(req CreateUserRequest) ValidationErrors {
    var errors ValidationErrors
    
    // Name validation - Laravel equivalent: 'name' => 'required|min:2|max:50'
    if req.Name == "" {
        errors = append(errors, ValidationError{"name", "Name is required"})
    } else if len(req.Name) < 2 {
        errors = append(errors, ValidationError{"name", "Name must be at least 2 characters"})
    } else if len(req.Name) > 50 {
        errors = append(errors, ValidationError{"name", "Name must be less than 50 characters"})
    }
    
    // Email validation - Laravel: 'email' => 'required|email'
    if req.Email == "" {
        errors = append(errors, ValidationError{"email", "Email is required"})
    } else if !v.emailRegex.MatchString(req.Email) {
        errors = append(errors, ValidationError{"email", "Invalid email format"})
    }
    
    // Age validation - Laravel: 'age' => 'min:18|max:120'
    if req.Age < 18 {
        errors = append(errors, ValidationError{"age", "Age must be at least 18"})
    } else if req.Age > 120 {
        errors = append(errors, ValidationError{"age", "Age must be less than 120"})
    }
    
    // Password validation - Laravel: 'password' => 'required|min:8'
    if req.Password == "" {
        errors = append(errors, ValidationError{"password", "Password is required"})
    } else if len(req.Password) < 8 {
        errors = append(errors, ValidationError{"password", "Password must be at least 8 characters"})
    }
    
    return errors
}

// JSON binding z validation - jak $request->validate() w Laravel
func (v *Validator) BindJSON(r *http.Request, target interface{}) error {
    if r.Header.Get("Content-Type") != "application/json" {
        return fmt.Errorf("content-Type must be application/json")
    }
    
    decoder := json.NewDecoder(r.Body)
    decoder.DisallowUnknownFields()  // strict JSON parsing - extra fields = error
    
    return decoder.Decode(target)
}

// Handler z validation - jak Laravel Controller z Form Request injection
func (v *Validator) createUserHandler(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    
    // Bind JSON - jak $request->json() w Laravel
    if err := v.BindJSON(r, &req); err != nil {
        sendError(w, http.StatusBadRequest, "Invalid JSON: "+err.Error())
        return
    }
    
    // Validate - jak $request->validate() w Laravel
    if errors := v.Validate(req); len(errors) > 0 {
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]interface{}{
            "error":   "Validation failed",
            "details": errors,  // Format podobny do Laravel validation errors
        })
        return
    }
    
    // Business logic - tu tworzysz usera
    user := User{
        ID:    len(users) + 1,
        Name:  req.Name,
        Email: req.Email,
    }
    
    users = append(users, user)
    sendJSON(w, http.StatusCreated, user)
}
```

**Zewnętrzna biblioteka validation - production approach**

Dla większych projektów polecam go-playground/validator - jak używanie Laravel validation zamiast pisania własnego:

```go
import "github.com/go-playground/validator/v10"

// Struct z validation tags - jak Laravel rules ale w struct
type CreateUserRequestV2 struct {
    Name     string `json:"name" validate:"required,min=2,max=50"`
    Email    string `json:"email" validate:"required,email"`
    Age      int    `json:"age" validate:"min=18,max=120"`
    Password string `json:"password" validate:"required,min=8"`
    Website  string `json:"website" validate:"omitempty,url"`  // optional, ale jeśli podane to musi być URL
}

var validate = validator.New()

func validateStruct(s interface{}) ValidationErrors {
    var errors ValidationErrors
    
    err := validate.Struct(s)
    if err != nil {
        // Parse validation errors - w Laravel to automatyczne
        for _, err := range err.(validator.ValidationErrors) {
            field := strings.ToLower(err.Field())
            message := getValidationMessage(err)
            errors = append(errors, ValidationError{field, message})
        }
    }
    
    return errors
}

func getValidationMessage(err validator.FieldError) string {
    switch err.Tag() {
    case "required":
        return "This field is required"
    case "email":
        return "Invalid email format"
    case "min":
        return fmt.Sprintf("Must be at least %s", err.Param())
    case "max":
        return fmt.Sprintf("Must be at most %s", err.Param())
    case "url":
        return "Must be a valid URL"
    default:
        return "Invalid value"
    }
}

// Custom validation rules - jak Laravel custom rules
func init() {
    // Custom validation dla Polish phone numbers
    validate.RegisterValidation("polish_phone", func(fl validator.FieldLevel) bool {
        phone := fl.Field().String()
        matched, _ := regexp.MatchString(`^\+48[0-9]{9}$`, phone)
        return matched
    })
}

// Usage w struct
type ContactRequest struct {
    Phone string `json:"phone" validate:"omitempty,polish_phone"`
}
```

### Response serialization patterns

W PHP często używałeś `json_encode()` i koniec. W Go masz więcej kontroli nad serialization process:

```go
// Content negotiation - różne formaty odpowiedzi
func multiFormatHandler(w http.ResponseWriter, r *http.Request) {
    data := map[string]interface{}{
        "id":    1,
        "name":  "Jan",
        "email": "jan@example.com",
    }
    
    // Content negotiation based on Accept header
    accept := r.Header.Get("Accept")
    
    switch {
    case strings.Contains(accept, "application/json"):
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(data)
        
    case strings.Contains(accept, "application/xml"):
        w.Header().Set("Content-Type", "application/xml")
        // XML encoding (uproszczone) - w PHP masz SimpleXML
        fmt.Fprintf(w, "<user><id>%v</id><name>%v</name></user>", 
            data["id"], data["name"])
        
    case strings.Contains(accept, "text/csv"):
        w.Header().Set("Content-Type", "text/csv")
        fmt.Fprintf(w, "id,name,email\n%v,%v,%v", 
            data["id"], data["name"], data["email"])
        
    default:
        // Default to JSON - najbezpieczniejszy wybór
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(data)
    }
}

// Pagination response - jak Laravel paginate()
type PaginatedResponse struct {
    Data     interface{} `json:"data"`
    Page     int         `json:"page"`
    PageSize int         `json:"page_size"`
    Total    int         `json:"total"`
    HasNext  bool        `json:"has_next"`
}

func paginatedUsersHandler(w http.ResponseWriter, r *http.Request) {
    page := 1
    pageSize := 10
    
    // Parse query params - Laravel: $request->input('page', 1)
    if p := r.URL.Query().Get("page"); p != "" {
        if parsed, err := strconv.Atoi(p); err == nil && parsed > 0 {
            page = parsed
        }
    }
    
    if ps := r.URL.Query().Get("page_size"); ps != "" {
        if parsed, err := strconv.Atoi(ps); err == nil && parsed > 0 && parsed <= 100 {
            pageSize = parsed
        }
    }
    
    // Calculate pagination - w Laravel paginate() robi to automatycznie
    total := len(users)
    start := (page - 1) * pageSize
    end := start + pageSize
    
    if start >= total {
        start = total
        end = total
    } else if end > total {
        end = total
    }
    
    paginatedUsers := users[start:end]
    hasNext := end < total
    
    response := PaginatedResponse{
        Data:     paginatedUsers,
        Page:     page,
        PageSize: pageSize,
        Total:    total,
        HasNext:  hasNext,
    }
    
    sendJSON(w, http.StatusOK, response)
}

// Response transformation - jak Laravel Resources
type UserResponse struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
    // Password field wykluczony dla bezpieczeństwa - ważne!
}

func transformUserResponse(user User) UserResponse {
    return UserResponse{
        ID:        user.ID,
        Name:      user.Name,
        Email:     user.Email,
        CreatedAt: user.CreatedAt,
        // Sensitive fields jak password NIE są included
    }
}
```

**GoLand validation tips które zaoszczędzą czas w developmencie:**

- **Generate JSON tags**: `Alt+Insert` → Generate → JSON tags
- **Regex testing**: Tools → Test Regular Expression
- **JSON validation**: Wklej JSON do string literal - GoLand zwaliduje format
- **Struct from JSON**: Code → Generate → JSON to Go Struct (plugin)

## OpenAPI/Swagger generation

Dokumentacja API to podstawa. Bez niej twój API jest bezwartościowy dla innych zespołów. W Laravel masz L5-Swagger z adnotacjami, w Symfony ApiPlatform. W Go możesz generować OpenAPI spec z kodu lub kod z spec.

### Manual OpenAPI annotations - code-first approach

```go
import "github.com/swaggo/swag"

// Global API info - jak w Laravel L5-Swagger config
// @title User API
// @version 1.0
// @description This is a sample user management API.
// @termsOfService http://swagger.io/terms/

// @contact.name API Support
// @contact.url http://www.swagger.io/support
// @contact.email support@swagger.io

// @license.name Apache 2.0
// @license.url http://www.apache.org/licenses/LICENSE-2.0.html

// @host localhost:8080
// @BasePath /api/v1

// @securityDefinitions.apikey ApiKeyAuth
// @in header
// @name Authorization

func main() {
    // Swagger generation command - dodaj do Makefile:
    // swag init -g main.go -o ./docs
}

// @Summary Get user by ID
// @Description Get a single user by their ID
// @Tags users
// @Accept json
// @Produce json
// @Param id path int true "User ID"
// @Success 200 {object} User
// @Failure 400 {object} APIResponse
// @Failure 404 {object} APIResponse
// @Router /users/{id} [get]
func getUserSwagger(w http.ResponseWriter, r *http.Request) {
    // Implementation identyczna jak wcześniej
    getUser(w, r)
}

// @Summary Create user
// @Description Create a new user
// @Tags users
// @Accept json
// @Produce json
// @Param user body CreateUserRequest true "User data"
// @Success 201 {object} User
// @Failure 400 {object} APIResponse
// @Router /users [post]
// @Security ApiKeyAuth
func createUserSwagger(w http.ResponseWriter, r *http.Request) {
    createUser(w, r)
}

// Swagger UI integration - endpoint dostępny pod /swagger/
import (
    _ "your-api/docs"  // import generated swagger docs
    httpSwagger "github.com/swaggo/http-swagger"
)

func setupSwagger(r chi.Router) {
    // Swagger UI endpoint - jak /docs w FastAPI
    r.Get("/swagger/*", httpSwagger.WrapHandler)
}
```

### Schema-first approach - lepsze dla większych projektów

Dla enterprise applications polecam oapi-codegen - generuje type-safe kod z OpenAPI spec:

```go
// Using oapi-codegen for contract-first development
//go:generate oapi-codegen -package api -generate types,chi-server openapi.yaml

// openapi.yaml - single source of truth
/*
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
paths:
  /users:
    get:
      operationId: listUsers
      parameters:
        - name: limit
          in: query
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 10
      responses:
        '200':
          description: List of users
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
    post:
      operationId: createUser
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          description: Validation error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ValidationErrorResponse'

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
          format: email
        created_at:
          type: string
          format: date-time
      required:
        - id
        - name
        - email
        
    CreateUserRequest:
      type: object
      properties:
        name:
          type: string
          minLength: 2
          maxLength: 50
        email:
          type: string
          format: email
        age:
          type: integer
          minimum: 18
          maximum: 120
      required:
        - name
        - email
        
    ValidationErrorResponse:
      type: object
      properties:
        error:
          type: string
        details:
          type: array
          items:
            type: object
            properties:
              field:
                type: string
              message:
                type: string
*/

// Generated types są automatycznie dostępne:
// - User struct
// - CreateUserRequest struct  
// - ServerInterface - implement this interface
// - HandlerFromMux - creates chi router from your implementation

type APIServer struct {
    users []User
}

// Implement generated interface - compiler wymusi implementację wszystkich methods
func (s *APIServer) ListUsers(w http.ResponseWriter, r *http.Request, params ListUsersParams) {
    limit := 10
    if params.Limit != nil {
        limit = *params.Limit
    }
    
    result := s.users
    if limit < len(result) {
        result = result[:limit]
    }
    
    sendJSON(w, http.StatusOK, result)
}

func (s *APIServer) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        sendError(w, http.StatusBadRequest, "Invalid JSON")
        return
    }
    
    user := User{
        Id:        len(s.users) + 1,  // Generated struct używa Id, nie ID
        Name:      req.Name,
        Email:     req.Email,
        CreatedAt: time.Now(),
    }
    
    s.users = append(s.users, user)
    sendJSON(w, http.StatusCreated, user)
}

func main() {
    server := &APIServer{}
    
    // Generated router - type-safe routing!
    r := HandlerFromMux(server, chi.NewRouter())
    
    http.ListenAndServe(":8080", r)
}
```

**Zalety schema-first approach dla zespołów:**

1. **Contract-first development** - frontend i backend mogą pracować równolegle
2. **Type safety** - kompilator złapie niezgodności z API contract
3. **Auto-generated clients** - możesz wygenerować SDK dla różnych języków
4. **Documentation zawsze aktualna** - generowana z kodu
5. **Validation automatyczna** - sprawdzanie zgodności z schema

**GoLand OpenAPI integration:**

1. **OpenAPI plugin** - syntax highlighting, validation
2. **Schema completion** - auto-completion w YAML files
3. **Reference navigation** - Ctrl+Click na `$ref` przechodzi do definicji
4. **Real-time validation** - błędy w schema natychmiast widoczne
5. **HTTP Client** - testuj API bezpośrednio z OpenAPI spec

## Rate limiting i circuit breakers

Żeby twoje API nie padło pod obciążeniem lub przez cascade failures. W PHP często robiłeś to na poziomie nginx/Apache (mod_evasive) lub używałeś Redis. W Go masz to w kodzie aplikacji - większa kontrola, łatwiejsze testowanie.

```go
import (
    "sync"
    "time"
    "golang.org/x/time/rate"
)

// Per-IP rate limiter - track każdego klienta osobno
type RateLimiter struct {
    clients map[string]*rate.Limiter  // mapa IP -> rate limiter
    mu      sync.RWMutex              // ochrona przed race conditions
    limit   rate.Limit                // requests per second
    burst   int                       // burst capacity
}

func NewRateLimiter(rps rate.Limit, burst int) *RateLimiter {
    return &RateLimiter{
        clients: make(map[string]*rate.Limiter),
        limit:   rps,   // np. 10 req/sec
        burst:   burst, // np. 20 req burst
    }
}

func (rl *RateLimiter) getLimiter(ip string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    limiter, exists := rl.clients[ip]
    if !exists {
        // Tworzymy nowy limiter dla tego IP
        limiter = rate.NewLimiter(rl.limit, rl.burst)
        rl.clients[ip] = limiter
        
        // TODO: cleanup old limiters (memory leak prevention)
    }
    
    return limiter
}

func (rl *RateLimiter) Allow(ip string) bool {
    return rl.getLimiter(ip).Allow()
}

// Rate limiting middleware - aplikuje limit per IP
func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ip := getClientIP(r)
        
        if !rl.Allow(ip) {
            // 429 Too Many Requests - standard HTTP status
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}

func getClientIP(r *http.Request) string {
    // Check X-Forwarded-For header first (load balancer, CDN)
    if xForwardedFor := r.Header.Get("X-Forwarded-For"); xForwardedFor != "" {
        // Take the first IP - może być lista "client, proxy1, proxy2"
        return strings.TrimSpace(strings.Split(xForwardedFor, ",")[0])
    }
    
    // Check X-Real-IP header (nginx proxy_pass)
    if xRealIP := r.Header.Get("X-Real-IP"); xRealIP != "" {
        return xRealIP
    }
    
    // Fall back to remote address
    ip, _, _ := net.SplitHostPort(r.RemoteAddr)
    return ip
}
```

### Circuit breaker pattern - ochrona przed cascade failures

Circuit breaker chroni przed cascade failures. Kiedy external service pada, circuit breaker "otwiera się" i nie wysyła requestów, tylko od razu zwraca error:

```go
// Circuit breaker states - jak w electrical circuit breaker
type CircuitState int

const (
    Closed   CircuitState = iota // Normal operation - requests przechodzą
    Open                         // Circuit otwarty - requests blokowane
    HalfOpen                     // Testing czy service się odzyskał
)

type CircuitBreaker struct {
    maxFailures int           // ile błędów przed otwarciem
    resetTime   time.Duration // jak długo czekać przed half-open
    
    mu           sync.RWMutex
    failures     int
    lastFailTime time.Time
    state        CircuitState
}

func NewCircuitBreaker(maxFailures int, resetTime time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        maxFailures: maxFailures, // np. 5 failures
        resetTime:   resetTime,   // np. 30 seconds
        state:       Closed,      // start w closed state
    }
}

func (cb *CircuitBreaker) Call(operation func() error) error {
    cb.mu.RLock()
    state := cb.state
    failures := cb.failures
    lastFailTime := cb.lastFailTime
    cb.mu.RUnlock()
    
    switch state {
    case Open:
        // Circuit otwarty - sprawdź czy czas na test
        if time.Since(lastFailTime) >= cb.resetTime {
            cb.mu.Lock()
            cb.state = HalfOpen
            cb.mu.Unlock()
        } else {
            return fmt.Errorf("circuit breaker is open")
        }
    case HalfOpen:
        // Half-open - pozwól jeden request na test
    case Closed:
        // Normal operation
    }
    
    // Execute operation (np. HTTP call do external service)
    err := operation()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failures++
        cb.lastFailTime = time.Now()
        
        if cb.failures >= cb.maxFailures {
            cb.state = Open  // Otwórz circuit
        }
        return err
    }
    
    // Success - reset circuit breaker
    cb.failures = 0
    cb.state = Closed
    return nil
}

// Circuit breaker dla external API calls
type ExternalAPIClient struct {
    cb     *CircuitBreaker
    client *http.Client
}

func NewExternalAPIClient() *ExternalAPIClient {
    return &ExternalAPIClient{
        cb: NewCircuitBreaker(5, 30*time.Second), // 5 failures, 30s reset
        client: &http.Client{Timeout: 5 * time.Second},
    }
}

func (api *ExternalAPIClient) CallExternalAPI(url string) (*http.Response, error) {
    var resp *http.Response
    
    err := api.cb.Call(func() error {
        var err error
        resp, err = api.client.Get(url)
        if err != nil {
            return err
        }
        
        // Treat 5xx jako failures - server errors
        if resp.StatusCode >= 500 {
            return fmt.Errorf("server error: %d", resp.StatusCode)
        }
        
        return nil
    })
    
    return resp, err
}

// Usage w handlerze - graceful degradation pattern
func (api *ExternalAPIClient) enrichUserHandler(w http.ResponseWriter, r *http.Request) {
    userID := chi.URLParam(r, "id")
    
    // Pobierz user z bazy (zawsze działa)
    user := getUserFromDB(userID)
    
    // Spróbuj wzbogacić danymi z external service (może się nie udać)
    resp, err := api.CallExternalAPI("https://external-api.com/users/" + userID)
    if err != nil {
        log.Printf("External API failed: %v", err)
        // Graceful degradation - kontynuuj bez external data
        sendJSON(w, http.StatusOK, user)
        return
    }
    defer resp.Body.Close()
    
    // Wzbogać dane usera
    var externalData map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&externalData)
    
    enrichedUser := map[string]interface{}{
        "user":     user,
        "external": externalData,
    }
    
    sendJSON(w, http.StatusOK, enrichedUser)
}
```

**Circuit breaker vs rate limiting - kiedy co używać:**

**Rate limiting**: Chroni twój serwis przed nadmiernym obciążeniem (incoming requests od klientów)

**Circuit breaker**: Chroni twój serwis przed cascading failures (outgoing requests do dependencies)

**Razem**: Defense in depth - ochrona z obu stron

## AI checkpoint - weryfikacja kodu HTTP

AI często generuje HTTP kod który wygląda poprawnie, ale ma subtelne błędy. Jako Engineering Manager powinieneś nauczyć zespół rozpoznawania tych problemów:

### 1. Nie zamyka request body

```go
// AI często generuje - BŁĄD!
func handler(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)  // brak defer r.Body.Close()!
    // ... rest of code
}

// GoLand warning: "Resource leak: 'r.Body' is not closed"
// Fix - zawsze dodawaj:
defer r.Body.Close()
```

### 2. Ignoruje Content-Type

```go
// AI może założyć że zawsze JSON - BŁĄD!
func handler(w http.ResponseWriter, r *http.Request) {
    var data map[string]interface{}
    json.NewDecoder(r.Body).Decode(&data)  // co jeśli to nie JSON?
}

// Poprawnie - sprawdź Content-Type:
if r.Header.Get("Content-Type") != "application/json" {
    http.Error(w, "Expected JSON", http.StatusBadRequest)
    return
}
```

### 3. Brak walidacji URL parametrów

```go
// AI często generuje - ignoruje error!
idStr := chi.URLParam(r, "id")
id, _ := strconv.Atoi(idStr)  // co jeśli to nie liczba?

// Poprawnie:
id, err := strconv.Atoi(idStr)
if err != nil {
    sendError(w, http.StatusBadRequest, "Invalid user ID")
    return
}
```

### 4. Niepoprawne HTTP status codes

```go
// AI często daje status 200 dla wszystkiego
sendJSON(w, http.StatusOK, user)  // dla POST powinno być 201!

// Poprawne status codes:
// GET successful - 200 OK
// POST successful - 201 Created
// PUT successful - 200 OK lub 204 No Content
// DELETE successful - 204 No Content
// Validation error - 400 Bad Request
// Not found - 404 Not Found
// Server error - 500 Internal Server Error
```

### 5. Race conditions w shared state

```go
// AI może wygenerować shared variables bez synchronizacji
var requestCount int  // RACE CONDITION!

func CounterMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestCount++  // multiple goroutines = race!
        next.ServeHTTP(w, r)
    })
}

// Fix - use atomic operations:
var requestCount int64
atomic.AddInt64(&requestCount, 1)
```

### 6. Brak timeout'ów dla HTTP clients

```go
// AI generuje bez timeout'ów - NIEBEZPIECZNE!
resp, err := http.Get("http://external-api.com/data")

// Poprawnie - zawsze z timeout:
client := &http.Client{Timeout: 10 * time.Second}
resp, err := client.Get("http://external-api.com/data")
```

**GoLand inspections dla HTTP które pomogą zespołowi:**

1. **Code → Inspect Code** na całym projekcie
2. Kategorie do sprawdzenia:
   - "Probable bugs" - resource leaks, nil pointers
   - "HTTP specific" - missing timeouts, wrong status codes
   - "Concurrency" - race conditions, shared state
3. **Alt+Enter** na każdym warning dla quick fixes

## Monitoring i observability

Production-ready HTTP API potrzebuje monitoring. W PHP często polegałeś na Apache/nginx logs. W Go musisz to zbudować w aplikacji:

```go
// Structured logging middleware
func StructuredLoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Wrapped ResponseWriter żeby capture status code
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}
        
        next.ServeHTTP(wrapped, r)
        
        // Log strukturalny - łatwy do parsowania przez ELK, Splunk, etc.
        log.Printf(`{"method":"%s","path":"%s","status":%d,"duration":"%v","ip":"%s","user_agent":"%s","request_id":"%s"}`,
            r.Method,
            r.URL.Path,
            wrapped.statusCode,
            time.Since(start),
            getClientIP(r),
            r.Header.Get("User-Agent"),
            r.Context().Value("requestID"),
        )
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

// Metrics collection - compatible z Prometheus
var (
    httpRequestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )
    
    httpRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )
)

func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}
        
        next.ServeHTTP(wrapped, r)
        
        duration := time.Since(start).Seconds()
        
        httpRequestsTotal.WithLabelValues(
            r.Method,
            r.URL.Path,
            fmt.Sprintf("%d", wrapped.statusCode),
        ).Inc()
        
        httpRequestDuration.WithLabelValues(
            r.Method,
            r.URL.Path,
        ).Observe(duration)
    })
}
```

## Podsumowanie

HTTP w Go to potęga połączona z odpowiedzialnością. Najważniejsze rzeczy dla zespołu:

**Fundamenty:**
1. **net/http to fundament** - poznaj go dobrze zanim sięgniesz po framework
2. **Router wybierz mądrze** - chi dla performance, gorilla/mux dla features
3. **Middleware to architektura** - buduj composable functionality
4. **Status codes mają znaczenie** - używaj właściwych dla REST semantyki

**Bezpieczeństwo i stabilność:**
5. **Zawsze zamykaj Body** - `defer r.Body.Close()` w każdym handlerze
6. **Walidacja PRZED logiką biznesową** - fail fast, fail explicit
7. **Rate limiting to must-have** - chroń przed abuse
8. **Circuit breaker dla dependencies** - unikaj cascade failures

**Profesjonalizm:**
9. **OpenAPI dokumentacja** - API bez docs nie istnieje
10. **Monitoring i metrics** - widzialność w production
11. **Graceful degradation** - system powinien degradować się stopniowo

**Development workflow:**
12. **GoLand integracje** - wykorzystuj IDE do maximum
13. **AI code review** - sprawdzaj wygenerowany kod pod kątem typowych błędów
14. **Testing strategy** - unit testy dla handlers, integration testy dla middleware chains

**Real-world considerations dla Engineering Managera:**

- **Team onboarding**: HTTP w Go wymaga zmiany myślenia z framework magic na explicit control
- **Code review focus**: Resource leaks, proper status codes, error handling
- **Architecture decisions**: Kiedy własny middleware, kiedy external library
- **Performance expectations**: Go HTTP server może handle dziesiątki tysięcy concurrent connections
- **Production readiness**: Health checks, graceful shutdown, proper logging

**Kolejny rozdział**: gRPC i komunikacja między usługami. Nauczysz się Protocol Buffers, różne typy streamingu (unary, server, client, bidirectional), interceptors (gRPC middleware), service discovery. To przyszłość komunikacji między mikrousługami - szybsza i bardziej efektywna niż REST, z built-in type safety.

GoLand ma świetne wsparcie dla .proto files i code generation - będziesz mógł debugować gRPC calls tak samo łatwo jak HTTP requests. Plus integracja z tools jak grpcurl do testowania API.