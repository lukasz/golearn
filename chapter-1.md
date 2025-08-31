# Rozdział 1: Wprowadzenie do Go

## Historia i filozofia języka

Wyobraź sobie, że jest rok 2007. W Google trzech ziomków - Rob Pike, Ken Thompson i Robert Griesemer - siedzi przy kawie i narzeka na C++. Kompilacja trwa wieki, kod jest skomplikowany jak instrukcja IKEA napisana po chińsku, a zarządzanie zależnościami to horror gorszy niż "Piła" wszystkie części razem wzięte.

"A gdyby tak..." - pomyśleli - "...stworzyć język, który będzie prosty jak młotek, szybki jak Ferrari i tak łatwy w czytaniu, że nawet twój manager zrozumie, co robi kod?"

I tak narodził się Go (albo Golang, jak kto woli). Oficjalnie wypuszczony w 2009 roku, Go został zaprojektowany z jednym celem: uprościć życie programistom pracującym nad dużymi, rozproszonymi systemami. Google miał dość języków, które wymagały PhD żeby zrozumieć wszystkie ich zawiłości.

Filozofia Go jest prosta jak budowa cepa:
- **Less is more** - mniej ficzerów oznacza mniej rzeczy do zepsucia
- **Jeden sposób na zrobienie czegoś** - nie ma 15 różnych pętli jak w niektórych językach
- **Czytelność ponad spryt** - kod ma być nudny i przewidywalny, nie pokazywać jaki jesteś mądry
- **Kompilacja ma być szybka** - żeby nie zdążyłeś nawet sięgnąć po kawę
- **Współbieżność wbudowana w język** - bo w 2007 już było wiadomo, że przyszłość to wielordzeniowość

Rob Pike kiedyś powiedział: "Go jest nudny, i to jest jego największa zaleta". I miał rację - w Go nie ma magii, nie ma ukrytych zachowań, nie ma "a może tak, a może siak". Jest kod, który robi dokładnie to, co widać.

## Dlaczego Go dla mikrousług i serverless

Jeśli mikrousługi to jak budowanie z klocków LEGO, to Go jest jak te klocki LEGO Technic - proste, solidne i dokładnie do tego zaprojektowane.

Oto dlaczego Go jest świetny do mikrousług:

**Mały binary, szybki start**. Go kompiluje się do jednego pliku wykonywalnego. Żadnych runtime'ów, interpreterów, maszyn wirtualnych. Twoja aplikacja w Go to może być plik 10MB, który odpala się w milisekundy. Dla porównania - minimalna aplikacja Node.js to jakieś 50MB z node_modules, a Java... no, nawet nie zaczynajmy o JVM.

**Współbieżność z pudełka**. W PHP robienie rzeczy równolegle to jak próba żonglowania piłkami tenisowymi jedną ręką - da się, ale po co się męczyć? W Go masz goroutines - możesz odpalić tysiące "lekkich wątków" i system sobie poradzi. Jedna linijka `go doSomething()` i już masz asynchroniczne wykonanie.

**Prostota deployu**. Skopiuj binary na serwer. Koniec. Nie ma "ale czy mają odpowiednią wersję PHP?", "czy composer zainstalował wszystko?", "czy opcache jest włączony?". Po prostu kopiujesz i uruchamiasz.

**Świetne wsparcie dla HTTP**. Biblioteka standardowa Go ma wbudowany solidny serwer HTTP. Nie potrzebujesz Apache, nginx, czy innych wynalazków. `http.ListenAndServe(":8080", handler)` i masz działający serwer.

A serverless? Tu Go błyszczy jeszcze bardziej:

**Cold start jak marzenie**. AWS Lambda z Go startuje w 50-100ms. PHP z całym frameworkiem? 500ms-1s. Java? Czasem nawet 10 sekund przy pierwszym starcie.

**Niskie zużycie pamięci**. Lambda rozlicza cię za pamięć × czas. Go używa pamięci jak skąpiec - oszczędnie i tylko tyle, ile trzeba. Twoja funkcja w Go może działać na 128MB RAM, podczas gdy ta sama w Node.js potrzebuje 512MB.

**Natywne wsparcie AWS**. AWS SDK dla Go jest pierwszej klasy. Amazon sam używa Go do wielu swoich usług. To nie jest "port" czy "wrapper" - to natywne SDK pisane z myślą o Go.

## Go vs PHP - kluczowe różnice w podejściu

Okej, czas na prawdziwą rozmowę. Jeśli całe życie pisałeś w PHP, Go będzie jak przesiadka z automatycznej skrzyni biegów na manualną. Na początku będziesz gasić silnik na każdym skrzyżowaniu, ale jak się nauczysz, to już nie będziesz chciał wracać.

**Kompilowany vs interpretowany**. PHP to skrypt - piszesz, zapisujesz, odświeżasz przeglądarkę i widzisz efekt (albo białą stronę śmierci). Go musisz skompilować. Ale to nie jest wada! Kompilator złapie 90% twoich błędów zanim w ogóle uruchomisz program. Literówka w nazwie zmiennej? Kompilator krzyknie. Zapomniałeś return? Kompilator krzyknie. Używasz zmiennej której nie zadeklarowałeś? Zgadnij co zrobi kompilator.

**Typowanie statyczne vs dynamiczne**. W PHP:
```php
$x = 5;
$x = "teraz jestem stringiem";
$x = ["a teraz", "jestem", "tablicą"];
```
W Go tak nie możesz. Jak `x` jest intem, to zostanie intem do końca życia. To jak małżeństwo - raz się zdecydowałeś i tyle. Ale dzięki temu IDE może ci podpowiadać, co możesz zrobić ze zmienną, kompilator łapie błędy, a kod jest przewidywalny.

**Struktura projektu**. PHP: wrzucasz pliki gdzie chcesz, `include` i `require` wedle uznania, autoloader sobie poradzi. Go: jest struktura i się jej trzymasz. Pakiety, moduły, ścieżki importów - wszystko ma swoje miejsce. To jak różnica między "gdzieś tu rzuciłem klucze" a "klucze zawsze wiszą na haczyku przy drzwiach".

**Obsługa błędów**. PHP: exceptions latają jak konfetti na sylwestra. Go: błędy to wartości, które zwracasz z funkcji. Zamiast:
```php
try {
    $result = doSomething();
} catch (Exception $e) {
    // obsługa błędu
}
```
Masz:
```go
result, err := doSomething()
if err != nil {
    // obsługa błędu
}
```
Tak, będziesz pisał `if err != nil` milion razy. Tak, to jest denerwujące. Tak, to jest celowe - masz widzieć gdzie mogą być błędy i je obsługiwać.

**Współbieżność**. PHP: "Co to jest współbieżność? Ah, masz na myśli kolejkę w RabbitMQ?" Go: "Hold my beer, odpalam 10000 goroutines na jednym rdzeniu."

**Filozofia "batteries included"**. PHP: potrzebujesz framework (Laravel, Symfony), ORM (Eloquent, Doctrine), router, template engine... Go: standardowa biblioteka ma 90% tego co potrzebujesz. HTTP server? Jest. JSON? Jest. Cryptografia? Jest. Testing? Jest.

## Instalacja i konfiguracja środowiska z GoLand

Skoro używasz GoLand, to znaczy że lubisz wygodę i nie żałujesz kasy na dobre narzędzia. Szacunek! GoLand to Ferrari wśród IDE dla Go - ma wszystko czego potrzebujesz i jeszcze trochę.

**Instalacja Go:**

GoLand potrafi sam pobrać i zainstalować Go, ale lepiej mieć kontrolę nad wersją.

**macOS** (bo wiem, że pewnie masz MacBooka):
```bash
brew install go
```

**Linux** (Ubuntu/Debian):
```bash
# Pobierz najnowszą wersję z https://go.dev/dl/
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

**Windows**:
Idź na https://go.dev/dl/, pobierz MSI installer, klikaj "Next" aż się zainstaluje.

Sprawdź czy działa:
```bash
go version
# powinno wyświetlić: go version go1.21.5 darwin/arm64 (lub coś podobnego)
```

**Konfiguracja GoLand - pierwsze uruchomienie:**

1. **Otwórz GoLand** - przy pierwszym uruchomieniu zapyta o licencję (mam nadzieję że masz, bo warto)

2. **New Project** → **Go** 
   - Location: wybierz gdzie chcesz trzymać projekt
   - GOROOT: GoLand powinien automatycznie znaleźć twoją instalację Go
   - Zaznacz "Enable vendoring support automatically"

3. **Ustawienia które warto zmienić od razu** (GoLand → Settings na Mac, File → Settings na Windows/Linux):
   - **Editor → Code Style → Go**: zostaw domyślne, GoLand używa `gofmt` automatycznie
   - **Editor → Inspections → Go**: włącz wszystkie - GoLand ma genialne inspekcje
   - **Tools → File Watchers**: dodaj `go fmt` i `goimports` jeśli nie ma
   - **Editor → General → Auto Import**: zaznacz "Add unambiguous imports on the fly"
   - **Build, Execution, Deployment → Go Modules**: włącz "Enable Go modules integration"

**Skróty klawiszowe w GoLand które musisz znać:**

- `Cmd+Shift+Enter` (Mac) / `Ctrl+Shift+Enter` (Win/Linux) - inteligentne dokończenie linii
- `Alt+Enter` - quick fix (napraw błąd/warning)
- `Cmd+Alt+L` (Mac) / `Ctrl+Alt+L` (Win/Linux) - formatuj kod
- `Cmd+B` (Mac) / `Ctrl+B` (Win/Linux) - idź do definicji
- `Cmd+Alt+B` (Mac) / `Ctrl+Alt+B` (Win/Linux) - idź do implementacji
- `Shift+F6` - zmień nazwę (refactor)
- `Cmd+Shift+T` (Mac) / `Ctrl+Shift+T` (Win/Linux) - idź do testu / stwórz test
- `Cmd+/` (Mac) / `Ctrl+/` (Win/Linux) - zakomentuj linię
- `Cmd+D` (Mac) / `Ctrl+D` (Win/Linux) - duplikuj linię
- `Cmd+N` (Mac) / `Alt+Insert` (Win/Linux) - generuj kod (konstruktory, gettery, etc.)

**Live Templates które przyśpieszą ci pracę:**

GoLand ma wbudowane templates. Napisz i naciśnij Tab:
- `for` → pętla for
- `forr` → pętla for z range
- `if` → if statement
- `iferr` → `if err != nil { return err }`
- `func` → deklaracja funkcji
- `method` → deklaracja metody

Możesz dodać własne: Settings → Editor → Live Templates → Go.

## Struktura projektu Go i Go modules

Go modules to system zarządzania zależnościami w Go. To jak Composer w PHP, tylko że działa (żartuję, Composer też działa... czasami).

**Stwórzmy pierwszy projekt w GoLand:**

1. **File → New → Project**
2. Wybierz **Go**
3. Nazwij projekt `moj-pierwszy-serwis`
4. W polu "Module name" wpisz: `github.com/twoj-username/moj-pierwszy-serwis`
5. Kliknij **Create**

GoLand automatycznie utworzy `go.mod`:
```go
module github.com/twoj-username/moj-pierwszy-serwis

go 1.21
```

**Struktura katalogów - stwórz ją w GoLand:**

Kliknij prawym na główny folder projektu → New → Directory. Twórz kolejno:

```
moj-pierwszy-serwis/
├── cmd/                    # Punkty wejścia aplikacji
│   └── api/
│       └── main.go        # Tu jest func main() dla API
├── internal/              # Kod prywatny aplikacji
│   ├── handlers/          # HTTP handlers (kontrolery w PHP)
│   ├── models/            # Struktury danych
│   └── services/          # Logika biznesowa
├── pkg/                   # Kod który może być użyty przez inne projekty
├── configs/               # Pliki konfiguracyjne
├── scripts/               # Skrypty pomocnicze
├── go.mod                 # Definicja modułu i zależności
└── go.sum                 # Checksums zależności (jak composer.lock)
```

**Pro-tip dla GoLand**: Możesz utworzyć własny template struktury:
1. Settings → Editor → File and Code Templates
2. Zakładka "Other"
3. Dodaj nowy template dla swojej struktury

**Dlaczego `internal`?** To genialne - wszystko w katalogu `internal` może być importowane TYLKO przez kod w tym samym module. GoLand to rozumie i nie pozwoli ci zaimportować tego z zewnątrz. Podpowiedzi będą ignorować `internal` packages z innych projektów.

**Importowanie pakietów - GoLand robi to za ciebie!**

Zacznij pisać kod który używa pakietu:
```go
func main() {
    fmt.Println("Hello") // GoLand automatycznie doda import
}
```

GoLand automatycznie doda na górze:
```go
import "fmt"
```

Możesz też użyć `Alt+Enter` na nierozpoznanym symbolu i wybrać "Import package".

**Dodawanie zewnętrznych zależności w GoLand:**

1. **Opcja 1**: Napisz kod używający pakietu, GoLand zapyta czy pobrać
2. **Opcja 2**: Tools → Go Tools → Go Modules → Download Go Modules
3. **Opcja 3**: Terminal w GoLand (View → Tool Windows → Terminal):
```bash
go get github.com/go-chi/chi/v5
```

GoLand automatycznie odświeży projekt i pokaże nowe zależności w oknie "Project".

## Pierwszy program i narzędzia

Czas na klasyka - "Hello, World!". Ale zrobimy to po Go'emu - od razu jako mikroserwis HTTP.

**W GoLand:**
1. Prawy klik na folder `cmd/api` → New → Go File
2. Nazwij go `main`
3. GoLand zapyta o typ - wybierz "Empty file"

Wpisz kod:
```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

// handler to funkcja obsługująca request HTTP
func helloHandler(w http.ResponseWriter, r *http.Request) {
    // w to ResponseWriter - tu piszemy odpowiedź
    // r to Request - tu mamy dane o zapytaniu
    
    name := r.URL.Query().Get("name")
    if name == "" {
        name = "World"
    }
    
    // fmt.Fprintf to jak sprintf w PHP, tylko od razu pisze do writera
    fmt.Fprintf(w, "Hello, %s! Witaj w świecie Go!", name)
}

func main() {
    // Rejestrujemy handler pod ścieżką /hello
    http.HandleFunc("/hello", helloHandler)
    
    // Informujemy że startujemy
    log.Println("Serwer startuje na porcie 8080...")
    
    // Startujemy serwer na porcie 8080
    // nil oznacza że używamy domyślnego ServeMux (routera)
    if err := http.ListenAndServe(":8080", nil); err != nil {
        log.Fatal("Serwer nie mógł wystartować:", err)
    }
}
```

**Uruchomienie w GoLand - masz kilka opcji:**

1. **Zielona strzałka** obok `func main()` - kliknij ją
2. **Prawy klik** na pliku → Run 'go build main.go'
3. **Skrót** `Ctrl+Shift+F10` (Win/Linux) / `Cmd+Shift+R` (Mac)

GoLand automatycznie:
- Skompiluje kod
- Uruchomi aplikację
- Pokaże output w konsoli na dole
- Podświetli clickable link: `http://localhost:8080`

**Debugowanie w GoLand (to jest magia!):**

1. Kliknij w lewy margines obok linii kodu - pojawi się czerwona kropka (breakpoint)
2. Kliknij prawym na zieloną strzałkę → Debug
3. Otwórz `http://localhost:8080/hello?name=Debug`
4. GoLand zatrzyma się na breakpoincie

W oknie debuggera możesz:
- Podejrzeć wartości zmiennych
- Evaluate expressions (`Alt+F8`)
- Step over (`F8`), Step into (`F7`), Step out (`Shift+F8`)
- Resume (`F9`)

**Run Configurations w GoLand:**

GoLand automatycznie tworzy konfiguracje uruchomieniowe. Możesz je edytować:

1. Góra okna → rozwińka obok zielonej strzałki → "Edit Configurations"
2. Możesz dodać:
   - Environment variables (np. `PORT=3000`)
   - Program arguments
   - Working directory

**Wbudowane narzędzia Go w GoLand:**

GoLand integruje wszystkie narzędzia Go w menu **Tools → Go Tools**:

- **go fmt** - formatuje automatycznie przy zapisie (możesz to zmienić w Settings → Tools → Actions on Save)
- **go vet** - uruchamia się automatycznie, błędy pokazuje w edytorze
- **goimports** - organizuje importy automatycznie
- **go test** - możesz uruchomić klikając zieloną strzałkę obok testu
- **go mod tidy** - Tools → Go Tools → Go Mod Tidy

**Terminal w GoLand** (View → Tool Windows → Terminal):

Czasem łatwiej użyć terminala:
```bash
go run cmd/api/main.go
go build -o moj-serwis cmd/api/main.go
go test ./...
go mod tidy
```

**Database tools w GoLand** (przyda się później):

GoLand ma wbudowane narzędzia do baz danych:
1. View → Tool Windows → Database
2. + → Data Source → wybierz swoją bazę
3. Możesz przeglądać tabele, pisać zapytania, debugować SQL

## Jak czytać dokumentację Go i weryfikować kod z AI

**Quick Documentation w GoLand** - to twoja pierwsza linia obrony:

- Najedź kursorem na funkcję/typ → `Ctrl+Q` (Win/Linux) / `F1` (Mac)
- Pokaże dokumentację inline
- Kliknij link żeby przejść do pkg.go.dev

**External Documentation:**
- Kliknij prawym na symbol → Go to → Declaration or Usages
- Shift+F1 - otwiera dokumentację w przeglądarce

**GoLand i przykłady z dokumentacji:**

Kiedy piszesz kod, GoLand pokazuje przykłady użycia:
1. Zacznij pisać nazwę funkcji
2. W podpowiedzi zobaczysz "Example"
3. `Ctrl+Shift+I` pokaże quick definition z przykładami

**Go to Declaration (`Cmd+B` / `Ctrl+B`)** - nieocenione przy weryfikacji:
- Kliknij na dowolny symbol i zobacz jego implementację
- Działa też dla bibliotek standardowych - możesz zobaczyć jak naprawdę działa `fmt.Println`

## AI checkpoint: Weryfikacja kodu z ChatGPT/Claude/Copilot w GoLand

GoLand ma genialną integrację z AI, ale musisz wiedzieć jak jej używać mądrze.

**GitHub Copilot w GoLand:**

1. Settings → Plugins → szukaj "GitHub Copilot"
2. Zainstaluj i zaloguj się
3. Copilot będzie podpowiadał kod inline

**Ale uwaga! GoLand pomoże ci wyłapać błędy Copilota:**

**Inspections są twoim przyjacielem:**

GoLand ma setki wbudowanych inspekcji. Kiedy AI generuje kod, zobacz co podkreśla IDE:

- **Żółte podkreślenie** - warning, kod działa ale nie jest idiomatyczny
- **Czerwone podkreślenie** - błąd, kod się nie skompiluje
- **Szare tekst** - nieużywany kod
- **Żółta żarówka** - GoLand ma sugestię poprawy

**Czerwone flagi - GoLand je podkreśli:**

1. **Ignorowanie błędów** - GoLand pokaże "Unhandled error":
```go
// GoLand podkreśli _ na żółto
result, _ := someFunction()
```

`Alt+Enter` → "Handle error" → GoLand doda sprawdzenie błędu

2. **Niepotrzebne wskaźniki** - GoLand pokaże "Unnecessary pointer":
```go
func getName() *string {
    name := "Jan"
    return &name  // GoLand podkreśli &
}
```

3. **Złe nazewnictwo** - GoLand automatycznie podpowie poprawną nazwę:
```go
func GetUserById(userId int) {}  // GoLand podkreśli Id i userId
// Alt+Enter → "Rename to GetUserByID"
```

4. **Brak defer** - GoLand pokaże warning "Resource not closed":
```go
file, err := os.Open("plik.txt")
// GoLand podkreśli że brak defer file.Close()
```

**Code → Inspect Code** - uruchom pełną inspekcję:
1. Code → Inspect Code
2. Wybierz scope (cały projekt, plik, etc.)
3. GoLand pokaże wszystkie problemy w oknie "Inspection Results"

**Reformatting po AI:**
- `Cmd+Alt+L` (Mac) / `Ctrl+Alt+L` (Win/Linux) - formatuje kod
- `Cmd+Alt+O` (Mac) / `Ctrl+Alt+O` (Win/Linux) - optymalizuje importy

**GoLand Plugins dla lepszej weryfikacji:**

1. **Go Linter** - integruje golangci-lint:
   - Settings → Plugins → Marketplace → "Go Linter"
   - Po instalacji: Tools → Go Linter → Run
   
2. **SonarLint** - dodatkowa analiza statyczna:
   - Settings → Plugins → Marketplace → "SonarLint"
   - Automatycznie analizuje kod podczas pisania

**Integracja z golangci-lint:**

```bash
# Zainstaluj
brew install golangci-lint

# Skonfiguruj w GoLand
Settings → Tools → External Tools → +
Name: golangci-lint
Program: golangci-lint
Arguments: run $FilePath$
Working directory: $ProjectFileDir$
```

Teraz możesz: prawy klik na pliku → External Tools → golangci-lint

**Quick-fix dla typowych błędów AI:**

GoLand ma quick-fixy dla większości problemów. Kiedy AI wygeneruje problematyczny kod:

1. Stań na błędzie/warningu
2. `Alt+Enter`
3. Wybierz fix z listy

Na przykład:
- "Add error handling" - doda if err != nil
- "Simplify" - uprości kod
- "Extract variable" - wyciągnie powtarzający się kod
- "Add comment" - doda komentarz w formacie godoc

**Testowanie kodu z AI w GoLand:**

1. Wygeneruj kod z AI
2. `Cmd+Shift+T` (Mac) / `Ctrl+Shift+T` (Win/Linux) - "Create New Test"
3. GoLand wygeneruje szkielet testu
4. Uruchom test (`Ctrl+Shift+F10`)
5. Jeśli test przechodzi - kod prawdopodobnie OK

## Podsumowanie

No i tyle na pierwszy rozdział! Co powinieneś zapamiętać:

1. **Go jest nudny i to dobrze** - prostota to feature, nie bug
2. **GoLand to potężne narzędzie** - wykorzystuj jego możliwości, nie pisz jak w Notepadzie
3. **Kompilator + GoLand = dream team** - złapią większość błędów zanim uruchomisz kod
4. **Inspections są święte** - jeśli GoLand coś podkreśla, sprawdź dlaczego
5. **AI generuje kod, GoLand go weryfikuje** - zawsze ufaj IDE bardziej niż AI

W następnym rozdziale zagłębimy się w składnię. Będzie o typach, zmiennych, funkcjach i tym całym statycznym typowaniu które na początku będzie cię wkurzać, a potem pokochasz. GoLand będzie ci pomagał na każdym kroku - podpowiadał typy, łapał błędy, generował kod.

A teraz zadanie domowe: 
1. Otwórz w GoLand jakiś projekt Go z GitHuba
2. Użyj "Code → Inspect Code" i zobacz ile znajduje problemów
3. Spróbuj poprawić kilka z nich używając Alt+Enter
4. Przepisz jakiś swój prosty skrypt PHP na Go używając Copilota
5. Zobacz ile razy GoLand podkreślił kod z Copilota

**Pro-tipy na koniec:**

- **Cmd+Shift+A** (Mac) / **Ctrl+Shift+A** (Win/Linux) - "Find Action" - jak nie wiesz gdzie coś jest
- **Podwójny Shift** - "Search Everywhere" - szukaj plików, klas, symboli, akcji
- **Cmd+E** (Mac) / **Ctrl+E** (Win/Linux) - ostatnio otwierane pliki
- **Cmd+Shift+F12** (Mac) / **Ctrl+Shift+F12** (Win/Linux) - maksymalizuj edytor

**Przydatne linki:**
- https://www.jetbrains.com/go/learn/ - oficjalne tutoriale GoLand
- https://go.dev/doc/ - oficjalna dokumentacja Go
- https://gobyexample.com/ - nauka przez przykłady
- https://pkg.go.dev/ - dokumentacja wszystkich pakietów

I pamiętaj - GoLand to nie tylko edytor, to twój pair programming partner. Wykorzystuj jego podpowiedzi, quick-fixy i refaktoringi. Zobaczysz, że pisanie Go z GoLand to przyjemność, nie męczarnia!