# Rozdział 2: Podstawy składni

## Zmienne, stałe i typy danych (porównanie z PHP)

Pamiętasz jak w PHP mogłeś sobie żyć beztrosko? `$zmienna = 5;` i potem nagle `$zmienna = "teraz jestem stringiem";` i nikt nie narzekał? No to zapomnij. Go to jak surowy nauczyciel matematyki - raz ustaliłeś że x to liczba, to liczba zostanie do końca zadania.

**Deklarowanie zmiennych - masz trzy sposoby:**

```go
// Sposób 1: var z typem
var wiek int = 30
var imie string = "Krzysztof"

// Sposób 2: var bez typu (Go zgadnie)
var miasto = "Warszawa"  // Go wie że to string

// Sposób 3: krótsza deklaracja := (najczęściej używana)
wzrost := 180  // Go wywnioskuje że to int
waga := 75.5   // Go wywnioskuje że to float64
```

**GoLand ci pomoże:**
- Zacznij pisać `zmienna := ` i GoLand od razu pokaże jaki typ przypisze
- Najedź kursorem na zmienną - GoLand pokaże jej typ
- `Ctrl+Shift+P` (Win/Linux) / `Cmd+Shift+P` (Mac) - pokaże typ wyrażenia

**Zero values - genialna rzecz której nie ma w PHP:**

W PHP niezainicjalizowana zmienna to `null` albo notice/warning. W Go każdy typ ma swoją "zero value":

```go
var liczba int        // 0
var tekst string      // "" (pusty string)
var prawda bool       // false
var wskaznik *string  // nil
var tablica [5]int    // [0, 0, 0, 0, 0]
var mapa map[string]int // nil (uwaga!)
```

To znaczy że nigdy nie masz "undefined variable". Kod jest przewidywalny.

**Stałe - raz ustalone, na zawsze:**

```go
const pi = 3.14159
const (
    Monday = 1
    Tuesday = 2
    Wednesday = 3
)

// iota - magiczny licznik (zaczyna od 0)
const (
    Styczen = iota + 1  // 1
    Luty                // 2 (automatycznie iota + 1)
    Marzec              // 3
)
```

**GoLand trick:** Napisz `const` i naciśnij Tab - GoLand rozwinie template dla grupy stałych.

**Podstawowe typy danych:**

```go
// Liczby całkowite
var i8 int8 = 127               // -128 do 127
var i16 int16 = 32767           // -32768 do 32767  
var i32 int32 = 2147483647      // jak int w PHP na 32-bit
var i64 int64 = 9223372036854775807  // jak int w PHP na 64-bit
var i int = 42                  // rozmiar zależy od systemu (32 lub 64 bit)

// Liczby bez znaku (tylko dodatnie)
var u8 uint8 = 255              // 0 do 255 (byte to alias uint8)
var u16 uint16 = 65535
var u32 uint32 = 4294967295
var u64 uint64 = 18446744073709551615
var u uint = 42                 // jak int, zależy od systemu

// Zmiennoprzecinkowe
var f32 float32 = 3.14          // jak float w PHP
var f64 float64 = 3.14159265359 // większa precyzja (domyślny dla liczb z kropką)

// Boolean
var prawda bool = true
var falsz bool = false

// String (UTF-8 z pudełka!)
var tekst string = "Cześć! 你好! 🚀"  // wielojęzyczność bez problemów

// Byte i Rune
var bajt byte = 'A'             // byte to uint8, przechowuje ASCII
var znak rune = '世'            // rune to int32, przechowuje Unicode
```

**PHP developer pyta: "Po co tyle typów liczb?!"**

W PHP masz `int` i `float` i tyle. W Go masz kontrolę nad tym ile pamięci zużywasz. Robisz system dla IoT który ma 512KB RAM? Użyj `int8` gdzie się da. Procesujesz miliardy rekordów? `int64` żeby nie przekroczyć zakresu. To jak wybór między Smart a ciężarówką - czasem potrzebujesz jednego, czasem drugiego.

**GoLand automatycznie podpowie problem z zakresem:**

```go
var maly int8 = 200  // GoLand podkreśli: "200 overflows int8"
```

## Silne typowanie vs dynamiczne typowanie PHP

Czas na prawdziwą jazdę bez trzymanki. W PHP żyłeś sobie w błogiej nieświadomości:

```php
// PHP - wszystko ujdzie
$x = 5;
$y = "10";
$suma = $x + $y;  // 15, PHP sobie poradził
echo "Wynik: " . $suma;  // "Wynik: 15", znowu konwersja
```

W Go tak nie ma:

```go
// Go - musisz być explicit
x := 5
y := "10"
// suma := x + y  // BŁĄD! GoLand podkreśli na czerwono
// invalid operation: x + y (mismatched types int and string)

// Musisz jawnie konwertować
yInt, err := strconv.Atoi(y)  // konwertuj string na int
if err != nil {
    log.Fatal("Nie mogę skonwertować:", err)
}
suma := x + yInt  // Teraz OK: 15
```

**GoLand ci pomoże z konwersjami:**
- Napisz kod który nie działa (np. `x + y` gdzie typy się nie zgadzają)
- `Alt+Enter` na błędzie
- GoLand często zaproponuje "Convert to..." lub "Add type conversion"

**Konwersje typów - musisz je znać:**

```go
// Między liczbami - prosta składnia
var i int = 42
var f float64 = float64(i)      // int na float64
var i2 int = int(f)              // float64 na int (obcina część dziesiętną!)

// String na liczby - pakiet strconv
s := "123"
i, err := strconv.Atoi(s)       // string na int
if err != nil {
    // obsługa błędu
}

// Liczby na string
s2 := strconv.Itoa(42)          // int na string
s3 := fmt.Sprintf("%d", 42)     // jak sprintf w PHP

// Float na string z precyzją
f := 3.14159
s4 := strconv.FormatFloat(f, 'f', 2, 64)  // "3.14"
s5 := fmt.Sprintf("%.2f", f)    // prostsze, jak w PHP
```

**Type inference - Go nie jest głupi:**

```go
// Go sam wywnioskuje typy
x := 5           // int
y := 5.5         // float64
z := "hello"     // string
w := true        // bool

// W złożonych wyrażeniach też
suma := x + 10   // suma jest int
iloczyn := y * 2.0  // iloczyn jest float64

// Ale uważaj!
result := 10 / 3    // 3 (int / int = int)
result2 := 10.0 / 3 // 3.333... (float / int→float = float)
```

**GoLand feature - Type Hints:**
Settings → Editor → Inlay Hints → Go → zaznacz "Type hints" - GoLand będzie pokazywał typy zmiennych inline w kodzie. Świetne na początku nauki!

## Operatory i wyrażenia

Większość operatorów działa jak w PHP, ale są subtelne różnice które cię ugryzą jeśli nie uważasz.

**Operatory arytmetyczne:**

```go
a := 10
b := 3

suma := a + b        // 13
roznica := a - b     // 7
iloczyn := a * b     // 30
iloraz := a / b      // 3 (UWAGA! int/int = int, obcina!)
reszta := a % b      // 1

// Chcesz dzielenie zmiennoprzecinkowe?
ilorazFloat := float64(a) / float64(b)  // 3.333...

// Increment/decrement - tylko postfix, tylko jako statement!
i := 0
i++  // OK
// x := i++  // BŁĄD! To nie jest wyrażenie jak w PHP
// ++i       // BŁĄD! Nie ma prefix
```

**GoLand warning:** Jeśli napiszesz `10 / 3` w kontekście gdzie spodziewasz się float, GoLand pokaże warning "Integer division in floating point context".

**Operatory porównania:**

```go
x := 5
y := 10

rowne := x == y          // false
nierowne := x != y        // true
mniejsze := x < y         // true
wieksze := x > y          // false
mniejszeRowne := x <= y   // true
wiekszeRowne := x >= y    // false

// UWAGA na porównywanie floatów!
f1 := 0.1 + 0.2
f2 := 0.3
fmt.Println(f1 == f2)  // false! (0.30000000000000004 != 0.3)

// Lepiej używać math.Abs dla floatów
roznica := math.Abs(f1 - f2)
bliskoSiebie := roznica < 0.0001  // true
```

**Operatory logiczne:**

```go
prawda := true
falsz := false

and := prawda && falsz    // false
or := prawda || falsz     // true  
not := !prawda            // false

// Short-circuit evaluation - jak w PHP
func expensive() bool {
    fmt.Println("Droga operacja!")
    return true
}

// expensive() nie zostanie wywołane bo pierwsze jest false
result := false && expensive()  

// Częsty pattern w Go
if file != nil && file.Size() > 0 {
    // bezpieczne - file.Size() wywołane tylko gdy file != nil
}
```

**Operatory bitowe (rzadko używane, ale warto znać):**

```go
a := 60  // 0011 1100
b := 13  // 0000 1101

and := a & b       // 12 (0000 1100)
or := a | b        // 61 (0011 1101)
xor := a ^ b       // 49 (0011 0001)
leftShift := a << 2  // 240 (1111 0000)
rightShift := a >> 2 // 15 (0000 1111)
```

**Operatory przypisania:**

```go
x := 10
x += 5   // x = x + 5
x -= 3   // x = x - 3
x *= 2   // x = x * 2
x /= 4   // x = x / 4
x %= 3   // x = x % 3

// Bitowe też mają
y := 8
y <<= 1  // y = y << 1 (16)
y >>= 2  // y = y >> 2 (4)
```

**Brak operatora ternary! (największy szok dla PHP developera)**

```php
// PHP
$result = $condition ? $valueTrue : $valueFalse;
```

```go
// Go - musisz użyć if
var result string
if condition {
    result = valueTrue
} else {
    result = valueFalse
}

// Albo funkcja (dla prostych przypadków)
func ternary(condition bool, valueTrue, valueFalse string) string {
    if condition {
        return valueTrue
    }
    return valueFalse
}
result := ternary(x > 5, "duże", "małe")
```

**GoLand Live Template dla "ternary":**
1. Settings → Editor → Live Templates
2. Go → + → Live Template
3. Abbreviation: `tern`
4. Template text:
```go
var $VAR$ $TYPE$
if $CONDITION$ {
    $VAR$ = $TRUE$
} else {
    $VAR$ = $FALSE$
}
```

## Struktury kontrolne

### If - prosty ale ma swoje smaczki

```go
// Podstawowy if
if x > 0 {
    fmt.Println("Dodatnie")
} else if x < 0 {
    fmt.Println("Ujemne")
} else {
    fmt.Println("Zero")
}

// If z krótkim statementem (nie ma tego w PHP!)
if err := doSomething(); err != nil {
    // err istnieje tylko w tym bloku
    return err
}
// tu err już nie istnieje

// Praktyczny przykład
if file, err := os.Open("plik.txt"); err != nil {
    log.Fatal(err)
} else {
    defer file.Close()  // defer - wykonaj przy wyjściu z funkcji
    // pracuj z plikiem
}

// Częsty pattern - early return
func processUser(id int) error {
    if id <= 0 {
        return errors.New("invalid id")
    }
    
    user, err := getUser(id)
    if err != nil {
        return fmt.Errorf("getting user: %w", err)
    }
    
    if !user.Active {
        return errors.New("user not active")
    }
    
    // właściwa logika
    return nil
}
```

**GoLand shortcuts dla if:**
- Napisz `.if` po zmiennej bool i naciśnij Tab
- `Ctrl+Shift+Enter` dokończy if z klamrami
- Zaznacz kod → `Ctrl+Alt+T` → "Surround with..." → if

### Switch - potężniejszy niż w PHP

```go
// Podstawowy switch
switch day {
case "Monday":
    fmt.Println("Poniedziałek")
case "Tuesday", "Wednesday":  // można multiple values!
    fmt.Println("Wtorek lub środa")
default:
    fmt.Println("Inny dzień")
}
// Nie ma break! Automatyczne po każdym case

// Switch bez wyrażenia (jak if-else ladder)
score := 85
switch {  // brak wyrażenia!
case score >= 90:
    fmt.Println("A")
case score >= 80:
    fmt.Println("B")
case score >= 70:
    fmt.Println("C")
default:
    fmt.Println("F")
}

// Type switch - sprawdzanie typu (genialnie!)
func describe(i interface{}) {
    switch v := i.(type) {  // type assertion
    case int:
        fmt.Printf("Integer: %d\n", v)
    case string:
        fmt.Printf("String: %s\n", v)
    case bool:
        fmt.Printf("Boolean: %v\n", v)
    default:
        fmt.Printf("Nieznany typ: %T\n", v)
    }
}

// Fallthrough - jeśli naprawdę chcesz jak w PHP
switch num {
case 1:
    fmt.Println("Jeden")
    fallthrough  // przejdź do następnego case
case 2:
    fmt.Println("Dwa (wykonane też dla 1)")
}
```

**GoLand pomaga ze switchem:**
- Stań na zmiennej → `Alt+Enter` → "Create switch statement"
- GoLand automatycznie doda wszystkie możliwe case dla enum/const
- Warning gdy brakuje default lub nie wszystkie przypadki obsłużone

### For - jedna pętla, wiele zastosowań

W Go jest tylko `for`. Nie ma `while`, `do-while`, `foreach`. Ale `for` może wszystko!

```go
// Klasyczny for (jak w PHP/C)
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// For jako while
count := 0
for count < 10 {
    fmt.Println(count)
    count++
}

// Nieskończona pętla
for {
    fmt.Println("Na zawsze!")
    if someCondition {
        break  // wyjście z pętli
    }
}

// Range - iteracja po kolekcjach (jak foreach w PHP)
slice := []string{"a", "b", "c"}
for index, value := range slice {
    fmt.Printf("%d: %s\n", index, value)
}

// Tylko wartość (ignoruj index)
for _, value := range slice {
    fmt.Println(value)
}

// Tylko index
for index := range slice {
    fmt.Println(index)
}

// Range po mapie
m := map[string]int{"a": 1, "b": 2}
for key, value := range m {
    fmt.Printf("%s: %d\n", key, value)
}

// Range po stringu (zwraca runy, nie bajty!)
for index, runeValue := range "Hello, 世界" {
    fmt.Printf("%d: %c\n", index, runeValue)
}

// Continue i break z labelkami (rzadko używane)
outer:
for i := 0; i < 3; i++ {
    for j := 0; j < 3; j++ {
        if i == 1 && j == 1 {
            break outer  // wyjdź z obu pętli
        }
        fmt.Printf("(%d,%d) ", i, j)
    }
}
```

**GoLand Live Templates dla pętli:**
- `for` + Tab → klasyczna pętla
- `forr` + Tab → for range
- `.range` po slice/map + Tab → for range

**GoLand refactoring:** Zaznacz pętlę → `Ctrl+Alt+M` → "Extract Method"

## Funkcje i zwracanie wielu wartości

Tu Go błyszczy! Zapomnij o `return array($result, $error)` z PHP.

```go
// Podstawowa funkcja
func add(x int, y int) int {
    return x + y
}

// Skrócony zapis parametrów tego samego typu
func add2(x, y int) int {
    return x + y
}

// Zwracanie wielu wartości (killer feature!)
func divide(x, y float64) (float64, error) {
    if y == 0 {
        return 0, errors.New("division by zero")
    }
    return x / y, nil
}

// Użycie
result, err := divide(10, 2)
if err != nil {
    log.Fatal(err)
}
fmt.Println(result)

// Named return values (czytelniejsze)
func getCoordinates() (x, y int) {
    x = 10  // przypisanie do named return
    y = 20
    return  // "naked return" - zwraca x, y
}

// Variadic functions (jak ...$args w PHP)
func sum(nums ...int) int {
    total := 0
    for _, num := range nums {
        total += num
    }
    return total
}

// Wywołanie
fmt.Println(sum(1, 2, 3, 4, 5))  // 15

// Rozpakowanie slice'a
numbers := []int{1, 2, 3}
fmt.Println(sum(numbers...))  // spread operator

// Funkcje jako wartości (first-class functions)
var operation func(int, int) int
operation = add
fmt.Println(operation(5, 3))  // 8

// Anonymous functions (closures)
increment := func(x int) int {
    return x + 1
}
fmt.Println(increment(5))  // 6

// Closure z captured variables
counter := 0
incrementCounter := func() int {
    counter++  // modyfikuje zewnętrzną zmienną
    return counter
}
fmt.Println(incrementCounter())  // 1
fmt.Println(incrementCounter())  // 2

// Higher-order functions
func apply(nums []int, f func(int) int) []int {
    result := make([]int, len(nums))
    for i, num := range nums {
        result[i] = f(num)
    }
    return result
}

// Użycie
doubled := apply([]int{1, 2, 3}, func(x int) int {
    return x * 2
})
fmt.Println(doubled)  // [2, 4, 6]
```

**GoLand generowanie funkcji:**
1. Napisz wywołanie nieistniejącej funkcji
2. `Alt+Enter` → "Create function"
3. GoLand wygeneruje sygnaturę na podstawie użycia

**GoLand refactoring funkcji:**
- `Ctrl+F6` - Change Signature (dodaj/usuń parametry)
- `Shift+F6` - Rename
- `Ctrl+Alt+M` - Extract Method
- `Ctrl+Alt+P` - Extract Parameter

## Defer, panic i recover

To jest coś czego w PHP nie ma, a jest genialne.

### Defer - wykonaj na końcu, nieważne co

```go
// Defer - wykonuje się przy wyjściu z funkcji
func readFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()  // wykonaj Close() przy wyjściu
    
    // Nieważne jak wyjdziesz z funkcji (return, panic, koniec)
    // file.Close() ZAWSZE się wykona
    
    // ... kod pracujący z plikiem ...
    return nil
}

// Multiple defers - LIFO (stack)
func multiple() {
    defer fmt.Println("3")  // wykona się jako trzeci
    defer fmt.Println("2")  // wykona się jako drugi
    defer fmt.Println("1")  // wykona się jako pierwszy
    fmt.Println("Start")
}
// Output: Start, 1, 2, 3

// Defer z argumentami - evaluated immediately
func deferArgs() {
    x := 10
    defer fmt.Println("Defer x:", x)  // x=10 zapamietane teraz
    x = 20
    fmt.Println("Current x:", x)
}
// Output: Current x: 20, Defer x: 10

// Pattern: cleanup
func doWork() error {
    mu.Lock()
    defer mu.Unlock()  // zawsze odblokuj mutex
    
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()  // rollback jeśli nie było commit
    
    // ... operacje na tx ...
    
    return tx.Commit()  // jeśli commit OK, rollback nic nie zrobi
}

// Pattern: timing
func timeFunction() {
    start := time.Now()
    defer func() {
        fmt.Printf("Took: %v\n", time.Since(start))
    }()
    
    // ... kod do zmierzenia ...
}
```

**GoLand snippet:** napisz `defer` po `os.Open` - GoLand zasugeruje `defer file.Close()`

### Panic - gdy coś poszło bardzo źle

```go
// Panic - zatrzymuje normalny flow, wykonuje defer'y, crashuje program
func mustPositive(x int) {
    if x <= 0 {
        panic("x must be positive")  // jak throw Exception w PHP
    }
}

// Panic z własnym typem
type ValidationError struct {
    Field string
    Value interface{}
}

func validate(age int) {
    if age < 0 {
        panic(ValidationError{
            Field: "age",
            Value: age,
        })
    }
}

// Kiedy używać panic:
// 1. Inicjalizacja - gdy aplikacja nie może wystartować
func init() {
    if os.Getenv("API_KEY") == "" {
        panic("API_KEY not set")
    }
}

// 2. Programmer error - niemożliwa sytuacja
func process(data []byte) {
    if data == nil {
        panic("data cannot be nil")  // to nie powinno się zdarzyć
    }
}

// NIE używaj panic dla normalnych błędów!
// Źle:
func divide(a, b int) int {
    if b == 0 {
        panic("division by zero")  // NIE!
    }
    return a / b
}

// Dobrze:
func divideGood(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}
```

### Recover - łapanie panic (jak catch w PHP)

```go
// Recover - łapie panic i pozwala kontynuować
func safeCall(f func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            // r zawiera wartość przekazaną do panic
            err = fmt.Errorf("panic recovered: %v", r)
        }
    }()
    
    f()  // może spanikować
    return nil
}

// Użycie
err := safeCall(func() {
    panic("oh no!")
})
fmt.Println(err)  // panic recovered: oh no!

// Pattern: goroutine protection
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("Goroutine panicked: %v", r)
            // log stack trace
            debug.PrintStack()
        }
    }()
    
    // kod który może spanikować
}()

// Pattern: middleware w HTTP
func recoveryMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("Panic: %v", err)
                http.Error(w, "Internal Server Error", 500)
            }
        }()
        
        next(w, r)
    }
}
```

**GoLand debugging panic:**
1. Ustaw breakpoint w defer z recover
2. Run → View Breakpoints → zaznacz "Catch panic"
3. GoLand zatrzyma się gdy wystąpi panic

## Wskaźniki w Go (czym różnią się od referencji w PHP)

Uff, wskaźniki. Dla PHP developera to jak nauka jazdy na rowerze od nowa. W PHP masz referencje (`&$variable`), ale to zabawki w porównaniu ze wskaźnikami w Go.

```go
// Wskaźnik to adres w pamięci
x := 42
p := &x  // p to wskaźnik do x (adres x w pamięci)
fmt.Println(p)   // 0xc0000180a8 (jakiś adres)
fmt.Println(*p)  // 42 (wartość pod adresem)

// & - pobierz adres (reference)
// * - pobierz wartość (dereference)

// Zmiana przez wskaźnik
*p = 100
fmt.Println(x)  // 100 (x też się zmieniło!)

// Wskaźniki jako parametry funkcji
func addOne(x int) {
    x = x + 1  // zmienia lokalną kopię
}

func addOnePointer(x *int) {
    *x = *x + 1  // zmienia oryginał
}

value := 5
addOne(value)
fmt.Println(value)  // 5 (bez zmian)

addOnePointer(&value)
fmt.Println(value)  // 6 (zmienione!)

// Kiedy używać wskaźników:

// 1. Duże struktury - unikaj kopiowania
type BigStruct struct {
    Data [1000000]int
}

func processCopy(b BigStruct) {}      // kopiuje 4MB danych!
func processPointer(b *BigStruct) {}  // przekazuje 8 bajtów

// 2. Modyfikacja - gdy chcesz zmienić oryginał
type User struct {
    Name string
    Age  int
}

func (u User) Birthday() {
    u.Age++  // zmienia kopię!
}

func (u *User) BirthdayCorrect() {
    u.Age++  // zmienia oryginał
}

// 3. Opcjonalne wartości (jak null w PHP)
type Config struct {
    Timeout *int  // może być nil
}

config := Config{}
if config.Timeout != nil {
    fmt.Printf("Timeout: %d\n", *config.Timeout)
} else {
    fmt.Println("Timeout not set")
}

// Zero value dla wskaźnika to nil
var p2 *int
fmt.Println(p2)  // <nil>
// fmt.Println(*p2)  // PANIC! nil pointer dereference

// Sprawdzaj przed użyciem!
if p2 != nil {
    fmt.Println(*p2)
}

// new() - alokuje pamięć i zwraca wskaźnik
p3 := new(int)  // *int, wartość 0
fmt.Println(*p3)  // 0

// Struktury i wskaźniki
user := &User{Name: "Jan", Age: 30}
// Go automatycznie dereferencuje przy dostępie do pól
fmt.Println(user.Name)  // nie musisz pisać (*user).Name

// Slice'y, mapy, channels - to już są "reference types"
// Działają jak wskaźniki, nie musisz używać &
slice := []int{1, 2, 3}
modifySlice(slice)
fmt.Println(slice)  // zmodyfikowane!

func modifySlice(s []int) {
    s[0] = 100  // modyfikuje oryginał
}
```

**GoLand pomoże ze wskaźnikami:**
- Najedź na zmienną - pokaże czy to wskaźnik
- `Ctrl+Click` - przejdź do definicji/użyć
- Warning: "Possible nil pointer dereference"
- Quick fix: `Alt+Enter` → "Add nil check"

**Różnice PHP referencje vs Go wskaźniki:**

```php
// PHP
$a = 5;
$b = &$a;  // referencja
$b = 10;
echo $a;  // 10

function modify(&$x) {  // pass by reference
    $x = 100;
}
```

```go
// Go
a := 5
b := &a  // wskaźnik
*b = 10  // musisz dereferencować
fmt.Println(a)  // 10

func modify(x *int) {  // przyjmuje wskaźnik
    *x = 100
}
modify(&a)  // przekaż adres
```

**Kiedy NIE używać wskaźników:**
- Małe struktury (< 64 bytes) - kopiowanie może być szybsze
- Immutable data - gdy nie chcesz modyfikować
- Współbieżność - wskaźniki = potencjalne race conditions

## AI checkpoint

Czas sprawdzić czy AI generuje poprawny kod Go. Oto czego GoLand będzie szukał:

**1. Unused variables - Go nie toleruje**
```go
// AI często generuje
func process() {
    result := calculate()  // GoLand: "result declared but not used"
    // zapomniał użyć result
}

// Fix: użyj lub _ 
result := calculate()
_ = result  // explicit ignore
```

**2. Ignorowanie błędów**
```go
// AI lubi ignorować błędy
file, _ := os.Open("file.txt")  // GoLand: "Unhandled error"

// Poprawnie
file, err := os.Open("file.txt")
if err != nil {
    return err
}
defer file.Close()
```

**3. Niepotrzebne type assertions**
```go
// AI czasem przesadza
var i interface{} = "hello"
s := i.(string)  // OK, ale...
s2 := s.(string)  // GoLand: "Invalid type assertion"
```

**4. Defer w pętli**
```go
// AI może wygenerować
for _, file := range files {
    f, _ := os.Open(file)
    defer f.Close()  // GoLand: "Defer in loop"
}

// Lepiej
for _, file := range files {
    func() {
        f, _ := os.Open(file)
        defer f.Close()
        // process
    }()
}
```

**5. Nil pointer bez sprawdzenia**
```go
// AI zakłada że wszystko działa
func process(u *User) {
    fmt.Println(u.Name)  // GoLand: "Possible nil pointer"
}

// Dodaj sprawdzenie
if u != nil {
    fmt.Println(u.Name)
}
```

**GoLand Code Inspection dla AI code:**
1. Code → Inspect Code
2. Scope: "Uncommitted files" (jeśli wklejałeś kod AI)
3. Sprawdź kategorie:
   - "Probable bugs"
   - "Error handling"
   - "General"

## Podsumowanie

Dobra, przebrnęliśmy przez podstawy składni. Co powinieneś zapamiętać:

1. **Typy są święte** - raz int, zawsze int. Konwersje musisz robić jawnie
2. **Zero values są twoim przyjacielem** - nie ma undefined, wszystko ma wartość domyślną
3. **if err != nil będzie wszędzie** - przyzwyczaj się, to feature nie bug
4. **Defer to magia** - użyj do cleanup, zawsze się wykona
5. **Panic to ostateczność** - dla normalnych błędów zwracaj error
6. **Wskaźniki to nie rocket science** - & bierze adres, * bierze wartość
7. **GoLand łapie błędy** - czerwone podkreślenie = kod się nie skompiluje

**Ćwiczenia do przećwiczenia w GoLand:**

1. **Type Gymnastics:**
```go
// Napraw ten kod (GoLand pokaże błędy)
func calculate() {
    x := 10
    y := "20"
    z := 30.5
    result := x + y + z  // ???
    fmt.Println(result)
}
```

2. **Error Handling Practice:**
```go
// Dodaj proper error handling
func readConfig(filename string) Config {
    data, _ := os.ReadFile(filename)
    var config Config
    json.Unmarshal(data, &config)
    return config
}
```

3. **Defer Challenge:**
```go
// Przewidź output
func deferPuzzle() {
    for i := 0; i < 3; i++ {
        defer fmt.Println(i)
    }
    fmt.Println("Done")
}
```

4. **Pointer or Value:**
```go
// Kiedy użyć wskaźnika?
type Small struct { X, Y int }
type Large struct { Data [1000000]int }

func processSmall(s Small) {}   // OK czy *Small?
func processLarge(l Large) {}   // OK czy *Large?
```

**Następny rozdział:** Typy złożone - slice'y (dynamiczne tablice których nie ma w PHP), mapy (jak associative arrays ale strongly typed), struktury (jak klasy ale bez dziedziczenia) i interfejsy (najważniejsza rzecz w Go). GoLand będzie nam pomagał z type inference, auto-complete i refactoringiem struktur.

**Pro-tips GoLand na koniec:**
- `Ctrl+Shift+I` - quick definition, pokaż implementację bez przechodzenia
- `Ctrl+P` - parameter info, przypomnij parametry funkcji
- `Ctrl+Q` - quick documentation
- `Alt+F7` - find usages, gdzie używana jest zmienna/funkcja
- `Ctrl+Alt+V` - extract variable, wyciągnij wyrażenie do zmiennej
- Double `Shift` → wpisz "Generate" - generuj gettery, settery, konstruktory

I pamiętaj - w Go "nudny" kod to dobry kod. Nie kombinuj, pisz prosto i czytelnie. GoLand ci w tym pomoże, podkreślając wszystkie "sprytne" rozwiązania na żółto lub czerwono!