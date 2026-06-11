# Rozdział 3 — Migracja backendu (Express.js → Spring Boot)

> **Status:** streszczenie wygenerowane przez skill `kod-do-pracy`, gotowe do rozbudowania w Coworku przez skill `praca-inzynierska`.
> **Mapowanie na zakres pracy:** punkt 3.
> **Data wygenerowania:** 2026-06-10
> **Uwagi dla Cowork:** rozdział jest podzielony na 8 bloków `## [ROZDZIAŁ:...]` o tej samej nazwie — każdy ma stanowić odrębny podrozdział (3.1, 3.2, …, 3.8). Pole "Porównanie przed/po" jest obowiązkowe w tym rozdziale (migracja). Notification-service wycięty z zakresu — nie wspominać.

---

## [ROZDZIAŁ: Migracja backendu — struktura projektu i system budowania]

**Temat:** Porównanie struktury projektu, deklaracji zależności oraz systemu budowania pierwotnego backendu Node.js z architekturą nowych mikroserwisów Spring Boot.

**Technologie:** Node.js + npm + TypeScript + tsc, Spring Boot 4.0.3 + Maven + Java 25/21.

**Co zostało zaimplementowane:** Nowy backend zorganizowany został w dwóch niezależnych repozytoriach Git, z których każde zawiera odrębny projekt Maven z manifestem `pom.xml` oraz strukturą katalogów zgodną z konwencją Spring Boot: `src/main/java/.../` dla kodu produkcyjnego, `src/main/resources/` dla zasobów konfiguracyjnych (`application.yml`), `src/test/java/.../` dla testów. Każdy z dwóch zaimplementowanych serwisów (`thera-rest-service` na porcie 8081 z Java 25, `thera-payment-service` na porcie 8085 z Java 21) ma własny cykl życia budowania, własną historię kontroli wersji oraz własną konfigurację zależności. Plik `pom.xml` deklaruje zależności wprost (`spring-boot-starter-web`, `spring-boot-starter-data-mongodb`, `spring-boot-starter-security`, `spring-boot-starter-oauth2-resource-server`, `spring-boot-starter-validation`, `spring-boot-starter-actuator`, `spring-boot-starter-kafka`, `mapstruct 1.5.5`, `lombok`, oraz w przypadku serwisu płatności — `stripe-java 25.3.0`), natomiast wersje są zarządzane przez `spring-boot-starter-parent` zapewniający spójną kompatybilność między bibliotekami. Wbudowany serwer (Tomcat dla aplikacji blokujących) uruchamiany jest jako standalone JAR poleceniem `java -jar`. Każda klasa kompilowana jest do bajtkodu JVM przez `mvn compile`; testy uruchamiane są przez `mvn test`; obraz Docker budowany jest przez `mvn spring-boot:build-image` lub `az acr build`.

**Kluczowe decyzje techniczne:** Wybór paradygmatu *one repo per service* podyktowany był koniecznością izolacji serwisu płatności zgodnie z wymaganiami standardu PCI-DSS — repozytorium `thera-payment-service` posiada osobne uprawnienia dostępu do kodu. Konsekwentnie pozostałe serwisy domenowe również utrzymywane są w odrębnych repozytoriach, co umożliwia osobne strategie ewolucji wersji bibliotek. Wybór Maven zamiast Gradle podyktowany został dojrzałością ekosystemu i prostotą deklaratywnej składni `pom.xml`, ułatwiającej audyt zależności.

**Pliki/komponenty:**
- `thera-rest-service/pom.xml`, `thera-payment-service/pom.xml` — manifesty Maven
- `thera-rest-service/src/main/resources/application.yml`, `thera-payment-service/src/main/resources/application.yml` — konfiguracja
- Główne klasy aplikacji: `TheraRestServiceApplication.java`, `TheraPaymentServiceApplication.java` z adnotacją `@SpringBootApplication`

**Porównanie przed/po:**

| Wymiar | Monolit Express.js | Mikroserwisy Spring Boot |
|---|---|---|
| **Język** | TypeScript 5.8 (kompilowany do JavaScript przez `tsc`) | Java 25 (rest-service) / Java 21 (payment-service) |
| **System budowania** | npm + skrypty w `package.json` | Maven + `pom.xml` |
| **Liczba projektów** | 2 (`backend/`, `frontend/`) w jednym repozytorium | 2 niezależne repozytoria Git (`thera-rest-service`, `thera-payment-service`); każdy odrębny projekt Maven |
| **Liczba plików źródłowych** | 13 plików .ts w backendzie (729 linii) | ~30–50 klas Java per serwis, z jasnym podziałem na warstwy |
| **Serwer HTTP** | Express.js 4.21 (samodzielny) | Wbudowany Apache Tomcat (zintegrowany ze starterem `spring-boot-starter-web`) |
| **Uruchomienie** | `npm run dev` (nodemon + tsc -w) | `java -jar target/*.jar` lub `mvn spring-boot:run` |
| **Dystrybucja produkcyjna** | Cały folder `backend/` wraz z `node_modules` | Pojedynczy plik wykonywalny JAR z wbudowanymi zależnościami (*fat JAR*) |
| **Format konfiguracji** | Plik `.env` z parami klucz=wartość (płaska struktura) | Plik `application.yml` z hierarchiczną strukturą YAML + profile (`application-local.yml`, `application-prod.yml`) |

---

## [ROZDZIAŁ: Migracja backendu — warstwowość architektury aplikacji]

**Temat:** Porównanie organizacji kodu pierwotnego monolitu, w którym kontrolery operowały bezpośrednio na warstwie dostępu do danych, z trzywarstwową architekturą nowych serwisów Spring Boot (kontroler → serwis → repozytorium).

**Technologie:** Spring Boot 4.0.3, Spring Data MongoDB, Dependency Injection przez konstruktor.

**Co zostało zaimplementowane:** Nowe serwisy Spring Boot stosują klasyczną architekturę warstwową (ang. *layered architecture*) z czterema warstwami i wyraźnym podziałem odpowiedzialności. Warstwa **kontrolera** (`@RestController`) odpowiada wyłącznie za mapowanie żądań HTTP, walidację wejścia, ekstrakcję parametrów z tokenu JWT oraz delegację do warstwy serwisowej; nie zawiera logiki biznesowej. Warstwa **serwisowa** (`@Service`) zawiera logikę domenową: walidację reguł biznesowych, orkiestrację wywołań do repozytorium, publikację zdarzeń Kafka, integrację z usługami zewnętrznymi (Stripe). Warstwa **repozytorium** (`extends MongoRepository<T, String>`) zapewnia dostęp do bazy danych przez metody pochodne nazw (`findByKeycloakId`, `existsByEmail`) lub adnotacje `@Query` dla bardziej złożonych zapytań. Warstwa **modelu** zawiera dokumenty MongoDB (`@Document`) oraz obiekty transferu danych DTO (osobne dla żądań i odpowiedzi). Komunikacja między warstwami realizowana jest przez **wstrzykiwanie zależności konstruktorem** (`@RequiredArgsConstructor` z biblioteki Lombok generuje konstruktor dla pól `private final`, dzięki czemu zależności są niezmienne i wymuszane na poziomie kompilatora).

**Kluczowe decyzje techniczne:** Wybór architektury warstwowej (zamiast hexagonalnej lub *clean architecture*) podyktowany został złożonością projektu — dla serwisów obsługujących pojedynczy *bounded context* z umiarkowaną liczbą operacji warstwowość jest wystarczająca, a bardziej rozbudowane wzorce wprowadzałyby narzut kodu bez wymiernej korzyści. Wybór wstrzykiwania konstruktorem (zamiast `@Autowired` na polach) podyktowany jest dwoma czynnikami: niezmiennością zależności (pole `final`) oraz łatwością testowania jednostkowego (mocki przekazywane bezpośrednio do konstruktora w teście).

**Pliki/komponenty:**
- `thera-rest-service/src/main/java/.../controller/ClientController.java`, `PsychologistController.java`
- `thera-rest-service/src/main/java/.../service/ClientService.java`, `PsychologistService.java`
- `thera-rest-service/src/main/java/.../repository/ClientRepository.java`, `PsychologistRepository.java`
- `thera-payment-service/src/main/java/.../service/PaymentService.java` (255 linii — najbardziej rozbudowany serwis ze względu na orkiestrację Stripe + Kafka)

**Porównanie przed/po:**

| Wymiar | Monolit Express.js | Mikroserwisy Spring Boot |
|---|---|---|
| **Liczba warstw** | 2 (kontroler + Prisma) | 4 (kontroler + serwis + repozytorium + model) |
| **Warstwa serwisowa** | Brak — logika biznesowa w kontrolerze (`appointmentController.ts:6-28`) | `@Service` z jednoznaczną odpowiedzialnością |
| **Dostęp do bazy** | Bezpośrednie wywołania `prisma.client.create()` z kontrolera | Repozytoria Spring Data z metodami pochodnymi |
| **Inicjalizacja klienta bazy** | `new PrismaClient()` w każdym kontrolerze (5 razy w monolicie) | Pojedynczy `MongoTemplate` zarządzany przez Spring DI |
| **Wstrzykiwanie zależności** | Brak — wymagane przez `require()` ad-hoc | Wstrzykiwanie konstruktorem z `@RequiredArgsConstructor` |
| **Testowalność jednostkowa** | Niska — kontroler i dostęp do bazy ściśle powiązane | Wysoka — każdą warstwę można testować z zamockowanymi zależnościami |
| **Reużycie logiki biznesowej** | Niskie — duplikaty kodu między kontrolerami (`availabilityController.ts` i `psychologistController.ts` zawierały te same funkcje) | Wysokie — wspólna logika trafia do serwisu współdzielonego |

---

## [ROZDZIAŁ: Migracja backendu — kontrolery REST i konwencje API]

**Temat:** Porównanie sposobu definiowania endpointów REST w pierwotnym monolicie Express.js oraz w nowych serwisach Spring Boot, obejmujące adnotacje, mapowanie ścieżek, typy zwracane oraz konwencje odpowiedzi HTTP.

**Technologie:** Spring Web MVC, `@RestController`, `@RequestMapping`, `ResponseEntity<T>`, Spring HATEOAS (`ProblemDetail` zgodny z RFC 7807).

**Co zostało zaimplementowane:** Kontrolery Spring Boot definiowane są jako klasy z adnotacją `@RestController` (kombinacja `@Controller` i `@ResponseBody`, automatycznie serializująca zwracane obiekty do formatu JSON). Adnotacja `@RequestMapping("/clients")` na poziomie klasy definiuje ścieżkę bazową, a kolejne adnotacje na metodach (`@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`) doprecyzowują ścieżki i metody HTTP. Parametry ścieżki ekstrahowane są przez `@PathVariable`, ciało żądania przez `@RequestBody`, parametry zapytania przez `@RequestParam`, a tożsamość uwierzytelnionego użytkownika przez `@AuthenticationPrincipal Jwt jwt`. Każda metoda kontrolera zwraca `ResponseEntity<T>` zawierający kod HTTP, nagłówki oraz ciało odpowiedzi — daje to pełną kontrolę nad odpowiedzią serwera. Typowe statusy: `201 CREATED` dla utworzenia zasobu, `200 OK` dla pobrania lub aktualizacji, `204 NO CONTENT` dla operacji bez ciała odpowiedzi. Walidacja wejścia odbywa się przez adnotację `@Valid` na parametrze `@RequestBody`, która aktywuje Bean Validation (Jakarta Validation) sprawdzający adnotacje walidacyjne na polach DTO. Każdy serwis backendowy udostępnia również punkty Actuator (`/actuator/health`, `/actuator/info`) wykorzystywane przez sondy Kubernetes (*liveness*, *readiness*).

**Kluczowe decyzje techniczne:** Wybór jawnego `ResponseEntity<T>` zamiast bezpośredniego zwracania DTO (które domyślnie skutkuje statusem 200) wymusza świadome określanie kodu HTTP dla każdej operacji, zwiększając czytelność API. Zastosowanie `ProblemDetail` (standard RFC 7807) dla błędów ujednolica format odpowiedzi błędu z polami `type`, `title`, `status`, `detail`, `instance`, ułatwiając obsługę po stronie frontendu.

**Pliki/komponenty:**
- `thera-rest-service/src/main/java/.../controller/ClientController.java` (linie 1-51): `POST /clients`, `GET /clients/{id}`, `GET /clients/me`, `PUT /clients/{id}`
- `thera-rest-service/src/main/java/.../controller/PsychologistController.java` (linie 1-52): analogiczne endpointy dla subdomeny psychologów
- `thera-payment-service/src/main/java/.../controller/PaymentController.java` (linie 39-142): trzy endpointy z odmienną polityką autoryzacji

**Porównanie przed/po:**

| Wymiar | Monolit Express.js | Mikroserwisy Spring Boot |
|---|---|---|
| **Definicja routingu** | `router.post('/path', authMiddleware([...]), controllerFn)` w plikach `routes/*.ts` | `@PostMapping("/path")` na metodzie kontrolera |
| **Ekstrakcja parametrów** | `req.params.id`, `req.body.field`, `req.query.foo` | `@PathVariable String id`, `@RequestBody @Valid DTO dto`, `@RequestParam String foo` |
| **Identyfikacja użytkownika** | `req.user.id` (wstawiana ręcznie przez middleware) | `@AuthenticationPrincipal Jwt jwt`, `jwt.getSubject()` |
| **Typ zwracany** | `res.status(201).json(payload)` (efekt uboczny) | `return ResponseEntity.status(CREATED).body(payload)` (wartość zwracana) |
| **Status HTTP** | Brak konwencji — odpowiedzi 500 dla wszystkich błędów | Standardowe kody: 201, 200, 204, 400, 401, 403, 404, 409 |
| **Endpointy DELETE** | Brak (0 endpointów w monolicie) | Implementowane gdzie biznesowo zasadne |
| **Sondy diagnostyczne** | Brak | `/actuator/health`, `/actuator/info` wykorzystywane przez Kubernetes |
| **Łączna liczba endpointów** | 18 w monolicie | Docelowo ~30 we wszystkich serwisach |

---

## [ROZDZIAŁ: Migracja backendu — warstwa serwisowa, DTO i mapowanie obiektów]

**Temat:** Wprowadzenie warstwy serwisowej z izolacją logiki biznesowej oraz obiektów transferu danych (DTO) z automatycznym mapowaniem MapStruct, których pierwotny monolit całkowicie nie posiadał.

**Technologie:** Spring `@Service`, MapStruct 1.5.5, Lombok `@Data`, `@Builder`, `@RequiredArgsConstructor`, Jakarta Bean Validation.

**Co zostało zaimplementowane:** Każdy mikroserwis Spring Boot wprowadza pełną warstwę serwisową oraz oddzielne obiekty DTO dla żądań (`CreateClientRequest`, `UpdateClientRequest`, `CreatePaymentIntentRequest`) i odpowiedzi (`ClientResponse`, `PsychologistResponse`, `PaymentResponse`). Encje MongoDB (`@Document Client`, `Psychologist`, `Payment`) nigdy nie są bezpośrednio serializowane do JSON i nie opuszczają warstwy serwisowej — w ten sposób unikana jest *over-exposure* pól wewnętrznych (np. `_id` typu MongoDB ObjectId, daty audytu, klucze techniczne). Mapowanie między encją a DTO realizowane jest przez **MapStruct** — preprocesor adnotacji, który podczas kompilacji generuje implementację interfejsu mapującego (`ClientMapper`, `PsychologistMapper`, `PaymentMapper`) bez użycia refleksji w czasie wykonania. Mapper deklaruje metody `toResponse(Entity)`, `toEntity(CreateRequest)`, `updateEntityFromRequest(UpdateRequest, @MappingTarget Entity)`, a MapStruct sam wnioskuje mapowanie pól na podstawie ich nazw i typów. Adnotacja `@Mapping(target = "id", ignore = true)` na metodzie `toEntity` zabezpiecza przed nadpisaniem identyfikatora generowanego przez MongoDB; adnotacja `@BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)` na metodzie `updateEntityFromRequest` zapewnia, że pola `null` w żądaniu aktualizacji nie nadpisują istniejących wartości w bazie. Komplementarna biblioteka **Lombok** eliminuje szablonowy kod (gettery, settery, konstruktor dla pól `final`, `equals`, `hashCode`, wzorzec Builder) przez adnotacje na klasie (`@Data`, `@Builder`, `@RequiredArgsConstructor`). W zestawie z MapStruct wymagany jest dodatkowy procesor `lombok-mapstruct-binding`, który zapewnia widoczność getterów generowanych przez Lombok dla MapStruct podczas kompilacji.

**Kluczowe decyzje techniczne:** Decyzja o oddzieleniu DTO dla żądań i odpowiedzi (zamiast wspólnej klasy `ClientDTO`) podyktowana jest różną odpowiedzialnością obu obiektów: DTO żądania zawiera tylko pola modyfikowalne przez klienta i wymusza walidację (`@NotBlank`, `@Email`, `@Positive`); DTO odpowiedzi zawiera pola obliczone lub pochodne niedostępne w żądaniu. Wybór MapStruct (zamiast biblioteki ModelMapper opartej na refleksji) podyktowany jest wydajnością — kod generowany przez MapStruct podczas kompilacji wykonuje się z prędkością ręcznie napisanego kodu, podczas gdy ModelMapper introspekcjonuje typy w czasie wykonania.

**Pliki/komponenty:**
- `thera-rest-service/src/main/java/.../dto/CreateClientRequest.java`, `ClientResponse.java`, `UpdateClientRequest.java`
- `thera-rest-service/src/main/java/.../mapper/ClientMapper.java`, `PsychologistMapper.java` (interfejsy z `@Mapper(componentModel = "spring")`)
- `thera-payment-service/src/main/java/.../dto/CreatePaymentIntentRequest.java`, `PaymentResponse.java`
- `thera-rest-service/src/main/java/.../service/ClientService.java`, `PsychologistService.java` (każdy 70+ linii logiki biznesowej)

**Porównanie przed/po:**

| Wymiar | Monolit Express.js | Mikroserwisy Spring Boot |
|---|---|---|
| **Warstwa DTO** | Brak — `req.body` przekazywane bezpośrednio do Prisma | Oddzielne DTO dla żądań i odpowiedzi |
| **Eksponowanie pól bazy** | Pełna encja Prisma serializowana do JSON (`include: { client: true, psychologist: true }`) | Wyłącznie pola wybrane do DTO odpowiedzi |
| **Mapowanie encja ↔ DTO** | Brak — bezpośrednia serializacja Prisma | MapStruct generujący kod podczas kompilacji |
| **Walidacja typów wejścia** | Brak — wartości z `req.body` używane bez sprawdzenia | Bean Validation: `@NotBlank`, `@Email`, `@Positive`, `@Size(min, max)` |
| **Walidacja reguł biznesowych** | Inline w kontrolerze | W warstwie serwisowej, oddzielona od warstwy prezentacji |
| **Boilerplate code** | Wysoki — ręczne definiowanie typów, brak konstruktorów | Niski — Lombok eliminuje gettery/settery/konstruktory |

---

## [ROZDZIAŁ: Migracja backendu — walidacja danych wejściowych]

**Temat:** Wprowadzenie deklaratywnej walidacji danych wejściowych przez standard Jakarta Bean Validation, eliminującej możliwość zapisania nieprawidłowych danych do bazy obserwowaną w pierwotnym monolicie.

**Technologie:** Jakarta Bean Validation 3.0, `spring-boot-starter-validation` (Hibernate Validator), adnotacje `@NotBlank`, `@Email`, `@Size`, `@Positive`, `@Valid`, `MethodArgumentNotValidException`.

**Co zostało zaimplementowane:** Walidacja danych wejściowych w nowych serwisach Spring Boot realizowana jest deklaratywnie przez adnotacje umieszczone na polach klas DTO. Adnotacja `@Valid` umieszczona przy parametrze `@RequestBody` w metodzie kontrolera aktywuje walidator Bean Validation przed przekazaniem obiektu do warstwy serwisowej. Naruszenie reguły walidacji powoduje wyrzucenie wyjątku `MethodArgumentNotValidException`, który przechwytywany jest przez globalny obsługiwacz wyjątków (`@RestControllerAdvice` opisany w podrozdziale 3.6) i konwertowany na odpowiedź HTTP 400 Bad Request z listą błędnych pól wraz z komunikatami. Stosowane adnotacje walidacyjne w projekcie obejmują: `@NotBlank` (pole tekstowe nie może być puste ani złożone wyłącznie z białych znaków), `@NotNull` (pole nie może być `null`), `@Email` (poprawny format adresu e-mail wg RFC 5322), `@Size(min, max)` (zakres długości), `@Positive` (wartość liczbowa większa od zera, używana dla `hourlyRate`, `amount`), `@Pattern` (zgodność z wyrażeniem regularnym). W kontrolerze płatności dodatkowo zastosowano `@Min(value = 100)` dla pola `amount` w celu odrzucenia płatności mniejszych niż minimalna kwota Stripe (100 groszy = 1 PLN). Adnotacje walidacyjne mogą zostać wzbogacone o komunikat błędu w języku polskim (`@NotBlank(message = "Pole imienia jest wymagane")`).

**Kluczowe decyzje techniczne:** Wybór deklaratywnej walidacji adnotacjami (zamiast walidacji imperatywnej w kodzie serwisu) podyktowany został trzema czynnikami: zwięzłością (jedna linia adnotacji zamiast bloku `if/throw`), separacją odpowiedzialności (definicja kontraktu DTO zawiera walidację jako część kontraktu) oraz spójnym formatem błędu walidacji (wszystkie naruszenia agregowane są w pojedynczej odpowiedzi 400, podczas gdy walidacja imperatywna typowo zatrzymywałaby się na pierwszym błędzie).

**Pliki/komponenty:**
- `thera-rest-service/src/main/java/.../dto/CreateClientRequest.java` z adnotacjami `@NotBlank`, `@Email`
- `thera-payment-service/src/main/java/.../dto/CreatePaymentIntentRequest.java` z `@NotNull`, `@Positive`, `@Min(100)`
- `thera-rest-service/src/main/java/.../exception/GlobalExceptionHandler.java` — przechwytuje `MethodArgumentNotValidException`

**Porównanie przed/po:**

| Wymiar | Monolit Express.js | Mikroserwisy Spring Boot |
|---|---|---|
| **Biblioteka walidacyjna** | Brak (`package.json` nie zawiera Joi, Zod, ani express-validator) | Jakarta Bean Validation 3.0 (Hibernate Validator) |
| **Sposób walidacji** | Imperatywne czeki `if (!email) { res.status(400)... }` (występują w 2 z 5 kontrolerów) | Deklaratywne adnotacje na DTO |
| **Pokrycie walidacji** | Pojedyncze pola sprawdzane *ad hoc*; większość pól nie ma żadnej walidacji | Wszystkie pola DTO opisane adnotacjami |
| **Format odpowiedzi błędu** | `{ message: "Missing required fields" }` — pojedyncza wiadomość | Lista błędów pól z komunikatami |
| **Lokalizacja komunikatów** | Brak | Możliwa przez `@NotBlank(message = "...")` lub `messages.properties` |
| **Zatrzymanie na pierwszym błędzie** | Tak — pierwszy `if` zwraca błąd, kolejne nie są sprawdzane | Wszystkie naruszenia są agregowane w jednej odpowiedzi |

---

## [ROZDZIAŁ: Migracja backendu — obsługa błędów i wyjątków]

**Temat:** Wprowadzenie centralnego obsługiwacza wyjątków zgodnego ze standardem RFC 7807 (Problem Details for HTTP APIs), zastępującego rozproszone bloki `try/catch` w pierwotnym monolicie i ujednolicającego format błędów.

**Technologie:** Spring `@RestControllerAdvice`, `@ExceptionHandler`, `ProblemDetail` (RFC 7807), Spring Boot 4 ulepszona obsługa wyjątków.

**Co zostało zaimplementowane:** W każdym serwisie Spring Boot zdefiniowano centralną klasę `GlobalExceptionHandler` z adnotacją `@RestControllerAdvice`, której metody opatrzone adnotacjami `@ExceptionHandler(WyjątekDomenowy.class)` przechwytują wyjątki rzucane z dowolnego kontrolera w aplikacji. Każda metoda obsługi zwraca obiekt `ProblemDetail` zgodny ze standardem RFC 7807 (Problem Details for HTTP APIs) zawierający pola: `type` (URI identyfikujący typ błędu), `title` (krótki opis), `status` (kod HTTP), `detail` (szczegółowy komunikat), `instance` (URI konkretnego wystąpienia). Dzięki temu wszystkie błędy w systemie posiadają jednolitą strukturę, co znacząco upraszcza obsługę po stronie frontendu. Wyjątki domenowe (`ClientNotFoundException`, `ClientAlreadyExistsException`, `PsychologistNotFoundException`, `PaymentNotFoundException`, `InvalidWebhookSignatureException`) są klasami rozszerzającymi `RuntimeException`, definiowanymi w pakiecie `exception/`. Rzucane są one z warstwy serwisowej w odpowiedzi na naruszenia reguł biznesowych (np. próba pobrania nieistniejącego zasobu, próba utworzenia duplikatu) i mapowane na odpowiednie kody HTTP: 404 Not Found, 409 Conflict, 400 Bad Request, 401 Unauthorized, 403 Forbidden. Globalna obsługa wyjątków obejmuje również `MethodArgumentNotValidException` (błędy walidacji wejścia, status 400 z listą błędnych pól), `AccessDeniedException` z Spring Security (status 403), `HttpMessageNotReadableException` (błąd parsowania JSON, status 400) oraz `Exception` (fallback dla nieoczekiwanych błędów, status 500 z generycznym komunikatem — wewnętrzny stack trace nie jest eksponowany klientowi).

**Kluczowe decyzje techniczne:** Wybór `ProblemDetail` (standard RFC 7807) zamiast własnego DTO `ErrorResponse` podyktowany jest ustandaryzowaną semantyką pól — klienci HTTP (w tym Angular HttpInterceptor) mogą oczekiwać tej samej struktury niezależnie od konkretnego serwisu. Decyzja o niewysyłaniu szczegółów wyjątku 500 do klienta zapobiega ujawnianiu wewnętrznej struktury kodu (m.in. nazw klas, ścieżek plików), co stanowiło problem w pierwotnym monolicie wyrzucającym `error.message` bezpośrednio do klienta.

**Pliki/komponenty:**
- `thera-rest-service/src/main/java/.../exception/GlobalExceptionHandler.java` (linie 1-65)
- `thera-rest-service/src/main/java/.../exception/ClientNotFoundException.java`, `ClientAlreadyExistsException.java`, `PsychologistNotFoundException.java`
- `thera-payment-service/src/main/java/.../service/PaymentService.java` (linie 237-254 — wyjątki zagnieżdżone jako klasy wewnętrzne dla zwięzłości)

**Porównanie przed/po:**

| Wymiar | Monolit Express.js | Mikroserwisy Spring Boot |
|---|---|---|
| **Strategia obsługi błędów** | Bloki `try/catch` w każdej funkcji kontrolera | Centralna klasa `@RestControllerAdvice` + wyjątki rzucane z warstwy serwisowej |
| **Liczba kodów HTTP** | Praktycznie 500 dla każdego błędu (11 z 18 kontrolerów); pojedyncze 201, 400, 409 | Pełen zakres: 201, 200, 204, 400, 401, 403, 404, 409, 500 |
| **Format odpowiedzi błędu** | `{ message: "Error: ${error.message}" }` — niespójny, czasem ujawnia stack trace | `ProblemDetail` zgodny z RFC 7807 — jednolity dla wszystkich błędów |
| **Bezpieczeństwo informacji** | Wycieki — `error.message` może ujawnić strukturę bazy lub kodu | Ochrona — szczegóły wyjątku logowane, lecz nie eksponowane klientowi |
| **Centralizacja** | Brak — każdy kontroler obsługuje błędy niezależnie | Jeden punkt prawdy dla obsługi błędów w serwisie |
| **Mapowanie wyjątków domenowych** | Brak — błędy biznesowe rzucane jako standardowy `Error` | Hierarchia własnych wyjątków (`*NotFoundException`, `*AlreadyExistsException`) |

---

## [ROZDZIAŁ: Migracja backendu — konfiguracja i zarządzanie sekretami]

**Temat:** Zastąpienie pojedynczego pliku `.env` z płaską strukturą zmiennych hierarchiczną konfiguracją Spring Boot opartą na pliku `application.yml` z systemem profili środowiskowych oraz integracją z Azure Key Vault na produkcji.

**Technologie:** Spring Boot `application.yml`, profile (`@Profile`, `spring.profiles.active`), `@Value`, `@ConfigurationProperties`, zmienne środowiskowe, Azure Key Vault, CSI Secret Store Driver.

**Co zostało zaimplementowane:** Konfiguracja każdego serwisu Spring Boot zdefiniowana jest w pliku `application.yml` w katalogu `src/main/resources/`, z hierarchiczną strukturą sekcji (`server`, `spring.data.mongodb`, `spring.kafka`, `spring.security.oauth2.resourceserver.jwt`, `stripe`). Każdy parametr może być odczytany ze zmiennej środowiskowej przez składnię `${VAR_NAME:default-value}` — domyślne wartości używane są w środowisku deweloperskim, podczas gdy w środowisku produkcyjnym wszystkie krytyczne parametry (URI MongoDB, sekret webhooka Stripe, klucz API Stripe) wymuszone są przez zmienne środowiskowe. **Profile Spring** (`spring.profiles.active=local|prod`) umożliwiają definiowanie odrębnych plików konfiguracyjnych: `application-local.yml` (środowisko deweloperskie z domyślnymi URI lokalnymi), `application-prod.yml` (środowisko produkcyjne ze ścisłymi limitami zasobów i włączonym TLS). Klasy konfiguracyjne typu `StripeConfig` używają `@ConfigurationProperties(prefix = "stripe")` do bezpiecznego wstrzykiwania całych grup parametrów jako jednego obiektu. W środowisku produkcyjnym sekrety przechowywane są w **Azure Key Vault** i udostępniane podom Kubernetes przez **CSI Secret Store Driver** — sterownik montuje sekrety jako pliki w volumenie poda lub udostępnia je jako zmienne środowiskowe, bez zapisywania ich w manifestach Kubernetes (które byłyby widoczne w `kubectl describe`). Konfiguracja CSI Secret Store dla każdego serwisu znajduje się w manifestach `SecretProviderClass` w repozytorium `thera-infrastructure`.

**Kluczowe decyzje techniczne:** Wybór YAML zamiast tradycyjnego `application.properties` podyktowany jest czytelnością hierarchicznej struktury (zwłaszcza dla zagnieżdżonych sekcji Spring Security i Kafka). Wybór Azure Key Vault zamiast natywnych `Kubernetes Secret` podyktowany jest większymi możliwościami audytu (Azure rejestruje każde odczytanie sekretu), automatyczną rotacją kluczy oraz wspólnym repozytorium sekretów dla wszystkich serwisów w klastrze. Decyzja o nieumieszczaniu domyślnych wartości produkcyjnych w pliku `application.yml` (zamiast tego wymuszenie ustawienia zmiennej środowiskowej) eliminuje ryzyko przypadkowego uruchomienia serwisu z domyślnymi sekretami w produkcji.

**Pliki/komponenty:**
- `thera-rest-service/src/main/resources/application.yml` (linie 1-45)
- `thera-payment-service/src/main/resources/application.yml` (linie 1-67 — dodatkowe sekcje `stripe.*`)
- `thera-rest-service/src/main/resources/application-local.yml` — profil dla środowiska LOCAL
- `thera-payment-service/src/main/java/.../config/StripeConfig.java` — klasa z `@ConfigurationProperties(prefix = "stripe")`
- Planowane: `thera-infrastructure/k8s/secret-provider/*.yaml` — manifesty CSI Secret Store

**Porównanie przed/po:**

| Wymiar | Monolit Express.js | Mikroserwisy Spring Boot |
|---|---|---|
| **Format konfiguracji** | Plik `.env` z parami `KLUCZ=wartość` (płaska struktura) | `application.yml` z hierarchiczną strukturą YAML |
| **Wsparcie dla wielu środowisk** | Pojedynczy `.env` — bez wsparcia natywnego dla wielu profili | Profile Spring (`local`, `prod`) — aktywne przez `spring.profiles.active` |
| **Składnia zmiennych z fallbackiem** | Brak — `process.env.X` zwraca `undefined` jeśli brak | `${X:default-value}` — wartość domyślna jeśli zmienna nieustawiona |
| **Walidacja konfiguracji przy starcie** | Brak — błędy odkrywane dopiero w trakcie wywołania | `@ConfigurationProperties` z `@Validated` — walidacja przy starcie aplikacji |
| **Zarządzanie sekretami w produkcji** | Manualne przekazywanie zmiennych przy uruchomieniu | Azure Key Vault + CSI Secret Store Driver — automatyczne pobieranie sekretów |
| **Hardcoded klucze API** | Klucz `OPENROUTER_API_KEY` w kodzie (`routes/chat.ts:7`) | Wszystkie sekrety przez zmienne środowiskowe; klucz Stripe w Azure Key Vault |
| **Eksponowanie sekretów w repo** | `frontend/public/.env` serwowany publicznie | Niemożliwe — `application.yml` zawiera wyłącznie referencje `${VAR_NAME}`, nie wartości |

---

## [ROZDZIAŁ: Migracja backendu — logowanie i obserwowalność]

**Temat:** Zastąpienie nieustrukturyzowanego logowania `console.log` i biblioteki `morgan` w pierwotnym monolicie strukturyzowanym logowaniem SLF4J/Logback z natywną integracją Spring Boot Actuator, otwierającą drogę do produkcyjnej obserwowalności.

**Technologie:** SLF4J (interfejs logowania), Logback (implementacja domyślna w Spring Boot), Spring Boot Actuator, format JSON dla logów produkcyjnych.

**Co zostało zaimplementowane:** Logowanie w nowych serwisach Spring Boot realizowane jest przez interfejs **SLF4J** (Simple Logging Facade for Java) z domyślną implementacją **Logback** dostarczaną przez Spring Boot. Każda klasa wymagająca logowania deklaruje pole `private static final Logger log = LoggerFactory.getLogger(KlasaName.class)` (lub stosuje adnotację `@Slf4j` z Lomboka generującą identyczny kod). Komunikaty logowane są na pięciu poziomach (`TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`) z możliwością niezależnej konfiguracji per pakiet w `application.yml` (`logging.level.com.theralink.payment.service: DEBUG`). Format logów konfigurowalny jest przez plik `logback-spring.xml`: w środowisku LOCAL używany jest format czytelny dla człowieka (z kolorowaniem poziomów), w środowisku PROD format JSON (jeden wiersz = jeden obiekt JSON z polami `timestamp`, `level`, `logger`, `message`, `thread`, `traceId`, `spanId`) umożliwiający automatyczne parsowanie przez systemy agregacji logów (Azure Monitor, Grafana Loki). Komunikaty logowane są przez parametryzowane wywołania (`log.info("Payment {} completed for client {}", paymentId, clientKeycloakId)`) — leniwa interpolacja oznacza, że budowanie wiadomości pomijane jest, jeśli poziom logu jest wyłączony. **Spring Boot Actuator** udostępnia endpointy `/actuator/health` (sprawdzanie stanu zdrowia z sondami komponentów: MongoDB, Kafka, Stripe), `/actuator/info` (metadane aplikacji), `/actuator/metrics` (metryki JVM, HTTP, Kafka), `/actuator/loggers` (dynamiczne zarządzanie poziomami logów bez restartu). W środowisku produkcyjnym dostęp do endpointów Actuator ograniczony jest do podsieci klastra Kubernetes.

**Kluczowe decyzje techniczne:** Wybór SLF4J jako interfejsu (zamiast bezpośredniego użycia Logback) podyktowany jest możliwością wymiany implementacji bez modyfikacji kodu aplikacyjnego — w razie potrzeby Logback może zostać zastąpiony przez Log4j2 wyłącznie przez zmianę zależności w `pom.xml`. Wybór formatu JSON w środowisku produkcyjnym podyktowany jest wymaganiem strukturyzowanej obserwowalności — narzędzia agregujące logi mogą filtrować i agregować po polach JSON bez analizy tekstu. Włączenie Actuator z dostępem ograniczonym do podsieci klastra zapewnia obserwowalność bez kompromisu bezpieczeństwa.

**Pliki/komponenty:**
- `thera-rest-service/src/main/resources/logback-spring.xml` — konfiguracja formatu logów
- `thera-rest-service/src/main/resources/application.yml` — sekcje `logging.level.*`, `management.endpoints.web.exposure.include`
- Klasy serwisowe używające `@Slf4j` z Lomboka

**Porównanie przed/po:**

| Wymiar | Monolit Express.js | Mikroserwisy Spring Boot |
|---|---|---|
| **Biblioteka logowania** | `morgan` (HTTP access log) + `console.log` ad-hoc | SLF4J + Logback (zintegrowane z Spring Boot) |
| **Strukturyzacja logów** | Brak — format tekstowy `morgan("common")` | Format JSON w produkcji, czytelny dla człowieka w dev |
| **Poziomy logowania** | Brak — `console.log`/`console.error` bez poziomów | 5 poziomów: TRACE, DEBUG, INFO, WARN, ERROR |
| **Konfiguracja poziomu per pakiet** | Brak | Sekcja `logging.level.*` w `application.yml` |
| **Korelacja żądań** | Brak — niemożliwe powiązanie logów dotyczących tego samego żądania | Wsparcie dla `traceId`/`spanId` (przez Micrometer Tracing) |
| **Endpoint sprawdzania stanu** | Brak | `/actuator/health` używany przez Kubernetes liveness/readiness probes |
| **Metryki JVM/HTTP** | Brak | `/actuator/metrics` z bazą Micrometer (eksport do Prometheus) |
| **Dynamiczne zarządzanie logami** | Brak — wymagany restart procesu | `/actuator/loggers` umożliwia zmianę poziomu w runtime |

---

## Sugerowane zrzuty ekranu do tego rozdziału

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Zrzut struktury katalogu projektu Spring Boot w IntelliJ IDEA — drzewo `thera-rest-service/src/main/java/com/theralink/rest/` z rozwiniętym podziałem na pakiety: `controller`, `service`, `repository`, `model`, `dto`, `mapper`, `config`, `exception`, `kafka`. Obok pokazać drzewo `backend/src/` z monolitu (controllers, routes, middleware) dla porównania.
> **Sugerowany podpis:** Rys. 3.1. Porównanie struktury katalogów monolitu Express.js i mikroserwisu Spring Boot
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram warstwowej architektury Spring Boot — cztery prostokąty (Controller, Service, Repository, Model) ułożone pionowo, ze strzałkami w dół oznaczającymi kierunek wywołań. Po lewej stronie analogiczny diagram monolitu z tylko dwoma blokami (Controller, Prisma).
> **Sugerowany podpis:** Rys. 3.2. Porównanie architektury warstwowej monolitu i mikroserwisu Spring Boot
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Side-by-side fragment kodu kontrolera — z lewej `backend/src/routes/userRoutes.ts` z `router.post(...)` i wywołaniem kontrolera, z prawej `thera-rest-service/.../controller/ClientController.java` z `@PostMapping`, `@Valid @RequestBody`, `@AuthenticationPrincipal Jwt jwt`, `ResponseEntity<ClientResponse>`.
> **Sugerowany podpis:** Rys. 3.3. Porównanie definicji endpointu REST w Express.js i Spring Boot
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Side-by-side fragment kodu warstwy serwisowej — z lewej `backend/src/controllers/appointmentController.ts` (cała logika w jednej funkcji), z prawej `thera-rest-service/.../service/ClientService.java` (warstwa serwisowa z wstrzykiwaniem `@RequiredArgsConstructor`).
> **Sugerowany podpis:** Rys. 3.4. Porównanie organizacji logiki biznesowej — monolit (brak warstwy serwisowej) vs mikroserwis (warstwa `@Service`)
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Fragment kodu klasy DTO Spring Boot — `CreateClientRequest.java` z adnotacjami `@NotBlank`, `@Email`, `@Size`. Pokazać też wynik walidacji nieudanej w Postmanie — odpowiedź 400 z listą błędnych pól.
> **Sugerowany podpis:** Rys. 3.5. Deklaratywna walidacja danych wejściowych przez Jakarta Bean Validation
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Fragment kodu `GlobalExceptionHandler.java` z `@RestControllerAdvice` i metodą `@ExceptionHandler(ClientNotFoundException.class)` zwracającą `ProblemDetail`. Obok — przykład odpowiedzi błędu w formacie JSON wg RFC 7807 (pola: `type`, `title`, `status`, `detail`, `instance`).
> **Sugerowany podpis:** Rys. 3.6. Centralna obsługa wyjątków z odpowiedzią w formacie RFC 7807
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Side-by-side pliku `.env` z monolitu (płaska lista zmiennych) oraz `application.yml` ze Spring Boot (hierarchiczna struktura). Na prawym zrzucie wyróżnić sekcje `spring.data.mongodb`, `spring.kafka`, `spring.security.oauth2.resourceserver.jwt`, `stripe`.
> **Sugerowany podpis:** Rys. 3.7. Porównanie formatu konfiguracji — plik `.env` w Express.js i `application.yml` w Spring Boot
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Zrzut endpointów Spring Boot Actuator w przeglądarce lub Postmanie — `GET /actuator/health` zwracający status `UP` z komponentami (`mongo: UP`, `kafka: UP`, `diskSpace: UP`).
> **Sugerowany podpis:** Rys. 3.8. Endpoint Actuator `/health` z sondami komponentów infrastrukturalnych
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Przykład logu w formacie JSON z produkcyjnego serwisu Spring Boot — jeden wiersz JSON z polami `timestamp`, `level`, `logger`, `message`, `thread`, `traceId`. Obok przykład logu `morgan("common")` z monolitu (płaski tekst).
> **Sugerowany podpis:** Rys. 3.9. Porównanie formatu logów — nieustrukturyzowany format `morgan` (monolit) vs strukturyzowany JSON (mikroserwis Spring Boot)
> **Źródło:** opracowanie własne
