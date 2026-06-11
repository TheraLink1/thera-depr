# Rozdział 2 — Projektowanie nowej architektury

> **Status:** streszczenie wygenerowane przez skill `kod-do-pracy`, gotowe do rozbudowania w Coworku przez skill `praca-inzynierska`.
> **Mapowanie na zakres pracy:** punkt 2.
> **Data wygenerowania:** 2026-06-09
> **Uwagi dla Cowork:** rozdział jest podzielony na 8 bloków `## [ROZDZIAŁ:...]` o tej samej nazwie — każdy ma stanowić odrębny podrozdział (2.1, 2.2, …, 2.8). Notification-service został wycięty z zakresu pracy — nie wspominać.

---

## [ROZDZIAŁ: Projektowanie nowej architektury — wzorzec architektoniczny]

**Temat:** Wybór nadrzędnego wzorca architektonicznego dla nowej wersji systemu TheraLink — architektury mikroserwisowej, jej charakterystyki, korzyści i kosztów w porównaniu z alternatywami rozpatrywanymi w fazie projektowej.

**Technologie:** Architektura mikroserwisowa (ang. *microservices architecture*), monolit modułowy, *Service-Oriented Architecture*, *Domain-Driven Design*, *Event-Driven Architecture*.

**Co zostało zaimplementowane:** W fazie projektowej rozważone zostały trzy alternatywne wzorce architektoniczne. Pierwszym była refaktoryzacja istniejącego monolitu do postaci **monolitu modułowego** (ang. *modular monolith*) — pojedynczego procesu wykonawczego z jasno wytyczonymi granicami modułów i wewnętrznym podziałem na pakiety domenowe. Drugim wzorcem była architektura zorientowana na usługi (ang. *Service-Oriented Architecture*, SOA) z centralną szyną integracyjną typu ESB (ang. *Enterprise Service Bus*). Trzecim wzorcem — wybranym ostatecznie — była **architektura mikroserwisowa** z niezależnymi procesami wdrożeniowymi, własnymi bazami danych per serwis oraz komunikacją asynchroniczną przez magistralę zdarzeń. Wybór architektury mikroserwisowej podyktowany został pięcioma czynnikami pozwalającymi adresować ograniczenia monolitu zidentyfikowane w rozdziale 1: izolacją awarii (awaria jednego serwisu nie unieruchamia całego systemu), niezależnym skalowaniem (każdy serwis może być skalowany w odpowiedzi na własny profil obciążenia), niezależną technologią (każdy serwis może być rozwijany w optymalnym dla siebie języku i wersji JVM — w rzeczywistej implementacji thera-rest-service używa Java 25, a thera-payment-service Java 21), niezależnym wdrożeniem (modyfikacja serwisu płatności nie wymaga przebudowy serwisu użytkowników) oraz izolacją danych regulowanych (serwis płatności posiada własną bazę danych i własne repozytorium kodu zgodnie z wymaganiami standardu PCI-DSS). Architektura zaprojektowana została zgodnie z założeniami *Domain-Driven Design* — granice serwisów wytyczone zostały na podstawie granic kontekstów ograniczonych (ang. *bounded contexts*) w domenie biznesowej.

**Kluczowe decyzje techniczne:** Architektura mikroserwisowa nie jest rozwiązaniem uniwersalnym — wiąże się z istotnymi kosztami: złożonością operacyjną (orkiestracja kontenerów, monitoring rozproszony), koniecznością zarządzania spójnością danych w obrębie wielu baz oraz narzutem komunikacji sieciowej między serwisami. Wybór tego wzorca uzasadniony jest celem pracy dyplomowej (zdobycie praktycznej wiedzy z budowy systemu rozproszonego), a nie samodzielną korzyścią biznesową na obecnym etapie projektu — typowa platforma z porównywalnym wolumenem ruchu mogłaby z powodzeniem działać jako monolit modułowy. Wybór mikroserwisów zamiast SOA z ESB podyktowany był wycofaniem się rynku z koncepcji centralnej, monolitycznej szyny integracyjnej na rzecz lekkich, *brokerowych* mechanizmów komunikacji (Kafka, RabbitMQ).

**Pliki/komponenty:**
- `docs/architecture-diagram.md` §1 — diagram architektury logicznej nowego systemu
- Decyzje architektoniczne udokumentowane w niniejszym rozdziale stanowią uzasadnienie podziału na poszczególne repozytoria opisane w podrozdziale 2.2

---

## [ROZDZIAŁ: Projektowanie nowej architektury — granice serwisów]

**Temat:** Identyfikacja kontekstów ograniczonych (*bounded contexts*) w domenie biznesowej platformy TheraLink oraz odwzorowanie ich na cztery serwisy domenowe, jedną bramę API i jeden serwis tożsamości.

**Technologie:** Spring Boot 4.0.3, *Domain-Driven Design*, *bounded context*, *aggregate root*.

**Co zostało zaimplementowane:** Domena biznesowa platformy TheraLink została podzielona na cztery konteksty ograniczone, którym odpowiadają cztery serwisy domenowe:

**user-service** (`thera-rest-service`, port 8081) — odpowiada za zarządzanie profilami użytkowników w dwóch subdomenach: klientów (`Client`) oraz psychologów (`Psychologist`). Każda subdomena posiada dedykowaną encję, repozytorium, mapper MapStruct (`ClientMapper`, `PsychologistMapper`) oraz osobne kontrolery (`ClientController` w `controller/ClientController.java`, `PsychologistController` w `controller/PsychologistController.java`). Serwis publikuje zdarzenia `theralink.user.created` po utworzeniu nowego konta w obu subdomenach. Identyfikatorem domenowym użytkownika jest `keycloakId` — pole `String` korelujące encję z tożsamością zarządzaną w Keycloak.

**appointment-service** (planowany jako osobny mikroserwis) — odpowiada za cykl życia wizyt: rezerwację, akceptację, anulowanie, oznaczanie jako opłaconych po otrzymaniu zdarzenia płatności. Serwis konsumuje zdarzenia `theralink.payment.completed` z magistrali Kafka i publikuje `theralink.appointment.created`, `theralink.appointment.confirmed`. Integruje się z Zoom API (Server-to-Server OAuth) w celu automatycznego tworzenia spotkań wideokonferencyjnych dla wizyt zdalnych po zaksięgowaniu płatności.

**psychologist-service** (planowany jako osobny mikroserwis) — odpowiada za zarządzanie slotami dostępności psychologów (kalendarz). Wyodrębnienie tej funkcjonalności do osobnego kontekstu motywowane jest odmiennym profilem operacji bazodanowych (intensywne odczyty kalendarza) oraz potencjałem osobnego skalowania.

**payment-service** (`thera-payment-service`, port 8085) — odpowiada za obsługę płatności kartą przez platformę Stripe. Serwis jest jedynym posiadającym możliwość komunikacji z `api.stripe.com`, jako jedyny odbiera webhooki Stripe (`POST /payments/webhook`) i jako jedyny publikuje zdarzenia płatności (`theralink.payment.completed`, `theralink.payment.failed`). Wyodrębnienie serwisu płatności do osobnego repozytorium kodu (`thera-payment-service`) z ograniczonym dostępem jest wymogiem standardu PCI-DSS (ang. *Payment Card Industry Data Security Standard*).

**API Gateway** (`theralink-api-gateway`, planowany na bazie Spring Cloud Gateway, port 8090) — pojedynczy punkt wejścia dla klientów HTTP. Realizuje weryfikację tokenów JWT, routing do serwisów backendowych, ograniczanie częstotliwości żądań (ang. *rate limiting*) oraz konsolidację dokumentacji OpenAPI z wielu serwisów. Klient frontendowy nigdy nie komunikuje się bezpośrednio z serwisami backendowymi — wyłącznie przez bramę.

**Keycloak** (`thera-keycloak`, port 8080) — centralny serwis tożsamości oparty na otwartym oprogramowaniu. Realm `theralink` definiuje trzy role realm'owe (`CLIENT`, `PSYCHOLOGIST`, `ADMIN`) oraz dwóch klientów OIDC: `theralink-angular` (Public Client z PKCE) oraz `theralink-backend` (Confidential Client z włączonymi Service Accounts dla komunikacji *service-to-service*).

**Kluczowe decyzje techniczne:** Granice serwisów wytyczone zostały zgodnie z zasadami DDD — każdy serwis odpowiada za pojedynczy *bounded context* z własnym językiem domenowym, własnymi agregatami i własnymi regułami biznesowymi. Decyzja o pozostawieniu subdomen *Client* i *Psychologist* wewnątrz jednego serwisu (`user-service`) podyktowana była ich silnym podobieństwem strukturalnym (oba reprezentują profil osoby fizycznej w systemie) oraz wspólnym przepływem operacji (rejestracja, edycja profilu) — wydzielenie ich do osobnych serwisów wprowadziłoby narzut komunikacyjny bez wymiernej korzyści.

**Pliki/komponenty:**
- `thera-rest-service/src/main/java/.../controller/ClientController.java` — kontroler subdomeny klientów
- `thera-rest-service/src/main/java/.../controller/PsychologistController.java` — kontroler subdomeny psychologów
- `thera-payment-service/src/main/java/.../controller/PaymentController.java` — kontroler serwisu płatności (3 endpointy: `/payments/intent`, `/payments/webhook`, `/payments/me`)
- `thera-keycloak/realm-export.json` — definicja realm'u Keycloak z rolami i klientami

---

## [ROZDZIAŁ: Projektowanie nowej architektury — topologia komunikacji]

**Temat:** Projekt warstwy komunikacyjnej systemu obejmujący dwa wzorce: synchroniczną komunikację REST przez bramę API oraz asynchroniczną komunikację zdarzeniową przez magistralę Apache Kafka.

**Technologie:** REST (HTTP/JSON), OAuth2 (JWT Bearer), Apache Kafka 7.6 (Confluent Platform), Spring Cloud Gateway, Spring Kafka, JsonSerializer/JsonDeserializer.

**Co zostało zaimplementowane:** Topologia komunikacji w systemie zaprojektowana została jako hybryda dwóch wzorców komplementarnych. **Komunikacja synchroniczna** realizowana jest protokołem HTTP/REST przez Spring Cloud Gateway — klient frontendowy wysyła żądanie na adres bramy (port 8090 lokalnie, `https://api.theralink.com` na produkcji), brama weryfikuje token JWT, a następnie przekazuje żądanie do odpowiedniego serwisu backendowego. Serializacja danych odbywa się w formacie JSON. **Komunikacja asynchroniczna** realizowana jest przez magistralę Apache Kafka — serwisy publikują zdarzenia domenowe (ang. *domain events*) na ściśle określonych topikach, a inne serwisy konsumują je w ramach własnych grup konsumenckich. Konwencja nazewnictwa topików: `theralink.{domena}.{zdarzenie}` (np. `theralink.payment.completed`).

**Zaprojektowane topiki Kafka** (w zakresie pracy):
- `theralink.user.created` — publikowane przez user-service po utworzeniu nowego konta klienta lub psychologa
- `theralink.appointment.created` — publikowane przez appointment-service po utworzeniu nowej rezerwacji
- `theralink.appointment.confirmed` — publikowane przez appointment-service po opłaceniu i (dla wizyt zdalnych) utworzeniu spotkania Zoom
- `theralink.payment.completed` — publikowane przez payment-service po otrzymaniu webhooka `payment_intent.succeeded` od Stripe
- `theralink.payment.failed` — publikowane przez payment-service po otrzymaniu webhooka `payment_intent.payment_failed`

**Zaprojektowane przepływy konsumpcji:**
- appointment-service konsumuje `theralink.payment.completed` w celu zaktualizowania statusu wizyty z `PENDING` na `PAID` oraz wywołania integracji z Zoom dla wizyt zdalnych
- payment-service konsumuje `theralink.appointment.created` w celu przygotowania kontekstu płatności

Klucz wiadomości Kafka wybierany jest jako `appointmentId` (lub odpowiednik identyfikatora agregatu) — zapewnia to gwarancję porządku wewnątrz partycji dla wszystkich zdarzeń dotyczących tej samej wizyty. Serializacja payloadu odbywa się w formacie JSON poprzez `JsonSerializer` (producent) i `JsonDeserializer` (konsument); deserialer skonfigurowany jest z parametrem `spring.json.trusted.packages: "*"` w środowisku deweloperskim (na produkcji wartość jest zawężana do konkretnych pakietów). Każda grupa konsumencka posiada własny identyfikator (`spring.kafka.consumer.group-id`) zgodny ze wzorcem `theralink-{serwis}-group`.

**Konfiguracja listenerów Kafka w środowisku deweloperskim** (`docker-compose.yml` w `thera-docker-compose/`): broker udostępnia dwa równoległe listenery — `INTERNAL://kafka:29092` dla komunikacji wewnątrz sieci Docker (między kontenerami) oraz `EXTERNAL://localhost:9092` dla aplikacji uruchamianych na hoście (Spring Boot uruchamiany z IDE w trakcie pracy deweloperskiej). Bez podwójnego listenera serwisy uruchamiane na hoście nie mogłyby się połączyć — `localhost` z perspektywy kontenera wskazywałby na jego własny kontener, nie na broker. Konfiguracja `KAFKA_LISTENERS` i `KAFKA_ADVERTISED_LISTENERS` rozdziela te dwa kontekstne sieciowe.

**Kluczowe decyzje techniczne:** Wybór Kafka zamiast RabbitMQ podyktowany był trzema czynnikami: retencją wiadomości (Kafka przechowuje zdarzenia przez skonfigurowany okres niezależnie od ich konsumpcji, umożliwiając *event replay*), wsparciem dla zarządzanej usługi *cloud-native* (Azure Event Hubs udostępnia kompatybilny protokół Kafka, eliminując potrzebę zmiany kodu Spring Boot przy przejściu z dev do produkcji), oraz konwencją *one consumer group per service*. Wybór JSON jako formatu serializacji (zamiast Apache Avro lub Protocol Buffers) podyktowany był prostotą debugowania w fazie projektowej — narzędzia takie jak Kafka UI pozwalają na podgląd zawartości wiadomości bez schematu.

**Pliki/komponenty:**
- `thera-payment-service/src/main/java/.../kafka/PaymentEventProducer.java` — producent zdarzeń płatności
- `thera-payment-service/src/main/java/.../kafka/AppointmentEventConsumer.java` — konsument zdarzeń wizyt
- `thera-rest-service/src/main/java/.../kafka/UserEventProducer.java` — producent zdarzeń użytkowników
- `thera-rest-service/src/main/java/.../config/KafkaProducerConfig.java` — konfiguracja `KafkaTemplate<String, Object>`
- `thera-docker-compose/docker-compose.yml` — definicja brokera Kafka z dwoma listenerami

---

## [ROZDZIAŁ: Projektowanie nowej architektury — architektura danych]

**Temat:** Projekt warstwy danych systemu oparty o wzorzec *database per service*, wykorzystujący MongoDB jako podstawową bazę dokumentową dla serwisów domenowych oraz PostgreSQL jako bazę pomocniczą dla Keycloak.

**Technologie:** MongoDB 7.0, Spring Data MongoDB, Azure Cosmos DB (API MongoDB), PostgreSQL 16 (dla Keycloak), wzorzec *database per service*, *aggregate root*.

**Co zostało zaimplementowane:** Architektura danych zaprojektowana została w oparciu o wzorzec *database per service* — każdy serwis domenowy posiada własną, izolowaną bazę danych, do której nie ma dostępu z innych serwisów. Wymiana danych między serwisami odbywa się wyłącznie przez wysokopoziomowe interfejsy (REST, Kafka), nigdy przez bezpośrednie zapytania do cudzej bazy. Zaprojektowano cztery odrębne bazy MongoDB: `theralink-users` (dla user-service, obejmuje kolekcje `clients` i `psychologists`), `theralink-appointments` (dla appointment-service), `theralink-psychologist-schedules` (dla psychologist-service) oraz `theralink-payments` (dla payment-service). Każda baza posiada własną pulę połączeń, własne uprawnienia użytkownika oraz własną politykę kopii zapasowych. W środowisku deweloperskim wszystkie cztery bazy współdzielą jeden kontener MongoDB 7.0, lecz pozostają odizolowane na poziomie logicznym (osobne bazy). W środowisku produkcyjnym każda baza będzie odrębnym kontem Azure Cosmos DB z API zgodnym z MongoDB, eliminując konieczność samodzielnego zarządzania replikacją, kopiami i skalowaniem.

**Wybór dokumentowej bazy danych** podyktowany był trzema przesłankami: brakiem wymagania transakcji wielodokumentowych w domenie biznesowej (każda operacja biznesowa dotyczy pojedynczego agregatu — utworzenie wizyty, zaksięgowanie płatności), naturalnym mapowaniem 1:1 między agregatem DDD a dokumentem MongoDB (jeden dokument = jeden agregat), oraz dostępnością zarządzanej usługi Azure Cosmos DB z elastyczną skalowalnością i wbudowaną redundancją wielodatacentrową. W projekcie świadomie pomięto ryzyko związane z brakiem natywnego wsparcia dla relacji obcych — w warstwie aplikacyjnej referencje między agregatami realizowane są przez identyfikatory (np. `appointmentId` w dokumencie `Payment` referuje wizytę w innym serwisie), bez egzekwowania integralności na poziomie bazy.

**Zaprojektowane dokumenty (kluczowe pola):**

`Client` (kolekcja `clients` w bazie `theralink-users`):
- `_id` (MongoDB ObjectId), `keycloakId` (String, unique index), `name`, `email`, `phoneNumber`, `history`

`Psychologist` (kolekcja `psychologists` w bazie `theralink-users`):
- Rozszerzenie modelu `Client` o pola: `location`, `hourlyRate`, `yearsOfExperience`, `description`, `specialization`, `sessionFormat` (enum: `REMOTE`, `IN_PERSON`, `BOTH`)

`Payment` (kolekcja `payments` w bazie `theralink-payments`):
- `_id`, `appointmentId` (unique index), `clientKeycloakId` (indexed), `stripePaymentIntentId` (unique index), `amount` (Long, w groszach — `Stripe.Amount` używa najmniejszej jednostki waluty), `currency`, `status` (enum: `PENDING`, `COMPLETED`, `FAILED`, `REFUNDED`), `createdAt`, `paidAt`

`Appointment` (planowana kolekcja `appointments` w bazie `theralink-appointments`):
- `_id`, `clientKeycloakId`, `psychologistKeycloakId`, `date`, `status` (enum: `PENDING`, `PAID`, `CONFIRMED`, `CANCELLED`), `sessionFormat`, `meetingLink` (dla wizyt zdalnych — link Zoom utworzony po opłaceniu)

`AvailabilitySlot` (planowana kolekcja w bazie `theralink-psychologist-schedules`):
- `_id`, `psychologistKeycloakId`, `date`, `startHour`, `endHour`, `isBooked`

**Kluczowe decyzje techniczne:** Identyfikatorem domenowym użytkownika we wszystkich serwisach jest `keycloakId` — pole typu `String` zgodne z formatem `sub` w tokenie JWT z Keycloak. Eliminuje to potrzebę synchronizacji liczbowych identyfikatorów między serwisami i przekierowuje *single source of truth* dla tożsamości użytkownika do Keycloak. Decyzja o nietworzeniu kolekcji *write-through cache* na poziomie aplikacji — Spring Data MongoDB i Azure Cosmos DB zapewniają wystarczającą wydajność odczytu dla rozmiaru bazy spodziewanego w okresie projektu dyplomowego.

**Pliki/komponenty:**
- `thera-rest-service/src/main/java/.../model/Client.java`, `Psychologist.java` — dokumenty MongoDB
- `thera-rest-service/src/main/java/.../repository/ClientRepository.java`, `PsychologistRepository.java` — interfejsy `MongoRepository`
- `thera-payment-service/src/main/java/.../model/Payment.java` — dokument płatności
- `thera-payment-service/src/main/java/.../repository/PaymentRepository.java` — z czterema customowymi zapytaniami (`findByAppointmentId`, `findByClientKeycloakId`, `findByStripePaymentIntentId`, `findByClientKeycloakIdAndStatus`)
- `thera-docker-compose/configs/mongodb/init.js` — inicjalizacja czterech baz

---

## [ROZDZIAŁ: Projektowanie nowej architektury — architektura bezpieczeństwa]

**Temat:** Projekt warstwy bezpieczeństwa oparty o standard OAuth 2.0 z rozszerzeniem PKCE oraz centralizację tożsamości w serwerze Keycloak, eliminujący krytyczne luki bezpieczeństwa zidentyfikowane w architekturze monolitycznej.

**Technologie:** Keycloak 25.0.4, OAuth 2.0, OpenID Connect (OIDC), PKCE (Proof Key for Code Exchange) z metodą S256, Spring Security 6 + OAuth2 Resource Server, JWT (RS256), JWKS endpoint, JwtAuthenticationConverter.

**Co zostało zaimplementowane:** Architektura bezpieczeństwa zaprojektowana została zgodnie z trzema zasadami: centralizacją tożsamości w jednym dedykowanym serwerze (Keycloak), automatyczną weryfikacją kryptograficzną tokenów po stronie każdego serwisu (Spring Security OAuth2 Resource Server) oraz separacją autoryzacji (sprawdzenie, czy użytkownik może wykonać operację) od uwierzytelnienia (sprawdzenie, kim użytkownik jest). Realm Keycloak `theralink` (`thera-keycloak/realm-export.json`) definiuje trzy role realm'owe (`CLIENT`, `PSYCHOLOGIST`, `ADMIN`) oraz dwóch klientów OIDC: `theralink-angular` (Public Client wykorzystywany przez aplikację Angular, z aktywowanym przepływem Authorization Code i rozszerzeniem PKCE metodą S256) oraz `theralink-backend` (Confidential Client z włączonymi Service Accounts, przeznaczony do komunikacji między serwisami).

**Przepływ uwierzytelnienia użytkownika końcowego:**
1. Aplikacja Angular generuje parę `code_verifier` i `code_challenge` (PKCE) i przekierowuje przeglądarkę do `https://keycloak/realms/theralink/protocol/openid-connect/auth`
2. Keycloak prezentuje stronę logowania z customowym motywem `theralink` (szablony FTL w katalogu `themes/theralink/login/`)
3. Po uwierzytelnieniu Keycloak przekierowuje z powrotem do Angular z parametrem `code`
4. Angular wymienia `code` (wraz z `code_verifier`) na parę tokenów: `access_token` (JWT podpisany RS256, ważny 15 minut) i `refresh_token` (ważny 7 dni)
5. Każde żądanie HTTP z aplikacji Angular zawiera nagłówek `Authorization: Bearer <access_token>` dodawany automatycznie przez interceptor (`core/interceptors/jwt.interceptor.ts`)
6. Brama API (Spring Cloud Gateway) oraz każdy serwis backendowy weryfikują podpis tokenu przez pobranie kluczy publicznych z punktu JWKS Keycloak (`/realms/theralink/protocol/openid-connect/certs`)
7. Spring Security automatycznie cache'uje pobrane klucze publiczne; rotacja kluczy w Keycloak jest obsługiwana bez modyfikacji aplikacji

**Konfiguracja Spring Security Resource Server** (`SecurityConfig.java` w każdym serwisie backendowym):
- Sesje wyłączone (`SessionCreationPolicy.STATELESS`) — każde żądanie wymaga pełnego tokenu
- Ochrona CSRF wyłączona (tokeny JWT w nagłówku, nie w ciasteczku — atak CSRF niemożliwy)
- `oauth2ResourceServer().jwt()` aktywuje automatyczne pobieranie kluczy z JWKS określonym przez `spring.security.oauth2.resourceserver.jwt.issuer-uri`
- `JwtAuthenticationConverter` ekstrahuje role z claim'u `realm_access.roles` w tokenie i konwertuje je na obiekty `SimpleGrantedAuthority` z prefiksem `ROLE_`
- Identyfikator użytkownika pobierany jest w kontrolerach przez `@AuthenticationPrincipal Jwt jwt`, a następnie `jwt.getSubject()` zwraca `keycloakId`
- Autoryzacja na poziomie metod aktywowana przez `@EnableMethodSecurity` umożliwia użycie adnotacji `@PreAuthorize("hasRole('CLIENT')")` w serwisie płatności

**Sekrety i konfiguracja:** Wszystkie sekrety (klucz Stripe, hasła do baz, sekrety webhooków) zarządzane są przez zmienne środowiskowe (`${VAR_NAME}` w `application.yml`). W środowisku deweloperskim wartości pobierane są z lokalnego pliku `.env` (gitignored). W środowisku produkcyjnym sekrety przechowywane są w Azure Key Vault, a do podów Kubernetes dostarczane przez sterownik CSI Secret Store.

**Kluczowe decyzje techniczne:** Wybór PKCE z metodą S256 (zamiast prostszego Authorization Code Flow z client_secret) jest standardem branżowym dla aplikacji jednostronicowych (ang. *Single Page Application*) — eliminuje konieczność przechowywania client secret w przeglądarce. Wybór czasu życia access tokenu na 15 minut to kompromis między bezpieczeństwem (krótki czas życia ogranicza okno wykorzystania skradzionego tokenu) a wygodą użytkowania (rzadkie odświeżanie refresh tokenem). Decyzja o stateless API (brak sesji HTTP) wymuszona została wymaganiem horyzontalnego skalowania serwisów — każda instancja serwisu musi być w stanie obsłużyć dowolne żądanie bez stanu współdzielonego.

**Pliki/komponenty:**
- `thera-rest-service/src/main/java/.../config/SecurityConfig.java` — konfiguracja Resource Server (linie 1-70)
- `thera-payment-service/src/main/java/.../config/SecurityConfig.java` — analogiczna konfiguracja z `@EnableMethodSecurity` (linie 1-101)
- `thera-keycloak/realm-export.json` — definicja realm'u, klientów, ról
- `thera-keycloak/themes/theralink/login/` — szablony FTL customowego motywu logowania
- `thera-ui/src/app/core/interceptors/jwt.interceptor.ts` — interceptor HTTP dołączający token
- `thera-ui/src/app/core/auth/guards/auth.guard.ts` — guard tras chronionych

---

## [ROZDZIAŁ: Projektowanie nowej architektury — infrastruktura logiczna]

**Temat:** Projekt warstwy infrastruktury obejmujący konteneryzację Docker, orkiestrację Kubernetes, dwa odrębne środowiska uruchomieniowe (LOCAL, PROD) oraz mapowanie zasobów na usługi zarządzane chmury Microsoft Azure.

**Technologie:** Docker, Docker Compose, Kubernetes, Helm, Azure Kubernetes Service (AKS), Azure Container Registry (ACR), Azure Cosmos DB (API MongoDB), Azure Event Hubs (protokół Kafka), Azure Key Vault, CSI Secret Store Driver.

**Co zostało zaimplementowane:** Architektura infrastruktury zaprojektowana została jako *separation of concerns* między dwoma środowiskami: deweloperskim (LOCAL) i produkcyjnym (PROD). Środowisko LOCAL oparte jest na Docker Compose — pojedynczy plik `docker-compose.yml` w repozytorium `thera-docker-compose` uruchamia kontenery infrastrukturalne (Keycloak z bazą PostgreSQL 16, MongoDB 7.0, Kafka 7.6 z Zookeeper, Kafka UI), podczas gdy serwisy aplikacyjne (Spring Boot, Angular) uruchamiane są lokalnie z poziomu zintegrowanego środowiska programistycznego. Środowisko PROD oparte jest na klastrze Azure Kubernetes Service — każdy serwis aplikacyjny pakowany jest w obraz Docker (zbudowany przez `az acr build` bez konieczności instalacji Docker Engine lokalnie), przechowywany w Azure Container Registry `acrtheralink.azurecr.io`, a następnie wdrażany na klastrze AKS w regionie `polandcentral` (grupa zasobów `rg-theralink`).

**Mapowanie usług między środowiskami:**

| Komponent | LOCAL | PROD |
|---|---|---|
| Baza danych dokumentowa | MongoDB 7.0 (kontener Docker) | Azure Cosmos DB (API MongoDB) |
| Magistrala zdarzeń | Apache Kafka 7.6 (kontener Docker) | Azure Event Hubs (protokół Kafka) |
| Serwis tożsamości | Keycloak 25.0.4 + PostgreSQL 16 (kontenery) | Keycloak 25.0.4 + Azure Database for PostgreSQL |
| Sekrety | Plik `.env` (gitignored) | Azure Key Vault + CSI Secret Store Driver |
| Bilans portów wystawionych na hosta | 8080 (Keycloak), 8081 (user), 8085 (payment), 4200 (Angular), 27017 (MongoDB), 9092 (Kafka), 9090 (Kafka UI) | Wyłącznie 443 (HTTPS) przez Ingress Controller |
| Konfiguracja Spring Boot | Profile `local` w `application-local.yml` | Profile `prod` w `application.yml` + zmienne środowiskowe |
| Kod aplikacyjny | **Identyczny** w obu środowiskach |

**Kluczowa zasada:** kod aplikacyjny Spring Boot jest dokładnie taki sam w obu środowiskach — różnice ograniczają się do konfiguracji (URI baz, adresów brokerów, źródła sekretów). Dzięki wsparciu Azure Cosmos DB dla protokołu MongoDB oraz Azure Event Hubs dla protokołu Kafka, przejście z dev do produkcji nie wymaga zmiany kodu Spring Boot — wystarczy zmiana zmiennych środowiskowych `MONGODB_URI` oraz `KAFKA_BOOTSTRAP_SERVERS`.

**Obecny stan implementacji infrastruktury PROD:** w repozytorium `thera-infrastructure` zaprojektowany został wykres Helm dla bazy MongoDB (chart `bitnami/mongodb`) z konfiguracją *standalone* (bez *replica set* — system nie wymaga transakcji), persystencji 2 GiB oraz limitów zasobów (request CPU 100m, RAM 256Mi). Pełne manifesty Kubernetes (Deployment, Service, Ingress, ConfigMap, Secret) dla serwisów aplikacyjnych zostały zaprojektowane konceptualnie i będą zrealizowane w rozdziale 8.

**Kluczowe decyzje techniczne:** Wybór paradygmatu jednolitego kodu aplikacyjnego z różną konfiguracją (zamiast osobnych gałęzi lub buildów per środowisko) jest zgodny z wzorcem *12-factor app*. Wybór Azure (zamiast AWS lub Google Cloud) podyktowany został dostępnością regionu `polandcentral` (niskie opóźnienia dla docelowej grupy użytkowników), dojrzałością usługi AKS oraz natywną kompatybilnością Azure Cosmos DB z MongoDB i Azure Event Hubs z Kafka — eliminuje to potrzebę modyfikacji kodu Spring Boot przy migracji do chmury.

**Pliki/komponenty:**
- `thera-docker-compose/docker-compose.yml` — definicja środowiska LOCAL (Keycloak, MongoDB, Kafka, Kafka UI)
- `thera-docker-compose/configs/mongodb/init.js` — skrypt inicjalizacji czterech baz MongoDB
- `thera-infrastructure/helm/mongodb/values.yaml` — wykres Helm dla MongoDB
- `thera-ui/Dockerfile` — wieloetapowy build aplikacji Angular (builder + nginx)
- Planowane: `thera-infrastructure/k8s/` (manifesty), `thera-infrastructure/helm/charts/` (wykresy dla serwisów)

---

## [ROZDZIAŁ: Projektowanie nowej architektury — porównanie z architekturą monolityczną]

**Temat:** Zestawienie zaprojektowanej architektury mikroserwisowej z architekturą monolityczną opisaną w rozdziale 1, obejmujące porównanie warstw, komunikacji, danych, bezpieczeństwa, skalowania i charakterystyk operacyjnych.

**Technologie:** Wszystkie wymienione w poprzednich podrozdziałach.

**Co zostało zaimplementowane:** Porównanie obu architektur prowadzone jest w siedmiu wymiarach charakteryzujących każdą warstwę systemu.

**Wymiar 1 — liczba procesów wykonawczych.** W architekturze monolitycznej system działa jako dwa procesy (Node.js + Next.js) wdrażane wspólnie. W architekturze mikroserwisowej system docelowo składa się z ośmiu procesów: cztery serwisy domenowe (user, appointment, psychologist, payment), brama API, Keycloak, serwer Kafka, serwer MongoDB — każdy w osobnym kontenerze. Konsekwencja: izolacja awarii (jedna usterka nie powoduje paraliżu systemu) kosztem złożoności operacyjnej (osiem komponentów do monitorowania zamiast dwóch).

**Wymiar 2 — komunikacja między modułami.** W monolicie wszystkie operacje realizowane są synchronicznie w obrębie pojedynczego procesu (bezpośrednie wywołania funkcji Prisma). W architekturze mikroserwisowej komunikacja synchroniczna (REST przez bramę API) współistnieje z asynchroniczną (zdarzenia Kafka). Konsekwencja: możliwość realizacji przepływów zdarzeniowych (potwierdzenie płatności → aktualizacja statusu wizyty → utworzenie spotkania Zoom), niemożliwych w monolicie bez blokowania wątku obsługującego żądanie HTTP.

**Wymiar 3 — model danych i baza.** Monolit korzysta z jednej bazy PostgreSQL z rozszerzeniem PostGIS, jednego schematu Prisma z pięcioma modelami i kluczami obcymi opartymi na `cognitoId`. Architektura mikroserwisowa wprowadza wzorzec *database per service* — cztery odrębne bazy MongoDB, każda obsługiwana przez jeden serwis, bez fizycznych kluczy obcych między bazami. Konsekwencja: niezależność ewolucji schematu każdego serwisu kosztem rezygnacji z transakcji wielodokumentowych i konieczności obsługi spójności ostatecznej (ang. *eventual consistency*) na poziomie aplikacji.

**Wymiar 4 — bezpieczeństwo.** Monolit weryfikuje tokeny JWT funkcją `jwt.decode()` bez sprawdzania podpisu kryptograficznego — luka krytyczna umożliwiająca podszycie się pod dowolną rolę. Architektura mikroserwisowa wykorzystuje Spring Security OAuth2 Resource Server, który automatycznie pobiera klucze publiczne z punktu JWKS Keycloak i weryfikuje podpis RS256 każdego tokenu. Konsekwencja: eliminacja krytycznej luki bezpieczeństwa bez konieczności samodzielnego implementowania weryfikacji.

**Wymiar 5 — tożsamość użytkownika.** Monolit korzysta z AWS Cognito jako zewnętrznego dostawcy tożsamości, z silnym sprzężeniem (biblioteka AWS Amplify używana w 12 plikach źródłowych frontendu, pole `cognitoId` jako klucz obcy w 4 z 5 modeli bazy). Architektura mikroserwisowa korzysta z otwartego Keycloak hostowanego we własnej infrastrukturze, z pełną kontrolą nad rolami, motywami logowania i przepływami autoryzacyjnymi.

**Wymiar 6 — skalowanie.** Monolit skaluje się wyłącznie horyzontalnie jako całość — moduł czatu AI o intensywnym ruchu sieciowym oraz moduł rezerwacji o niskim ruchu skalują się wspólnie. Architektura mikroserwisowa pozwala na niezależne skalowanie każdego serwisu w odpowiedzi na jego indywidualny profil obciążenia — serwis płatności może działać na jednej instancji w okresach niskiego ruchu, podczas gdy serwis użytkowników może mieć trzy instancje.

**Wymiar 7 — niezależność wdrożeniowa.** Modyfikacja dowolnego modułu w monolicie wymaga ponownego zbudowania i wdrożenia całego procesu Node.js, w tym krytycznych endpointów rezerwacji wizyt. W architekturze mikroserwisowej każdy serwis jest wdrażany niezależnie — modyfikacja serwisu płatności nie wpływa na dostępność pozostałych komponentów. Wdrożenie nowej wersji serwisu realizowane jest jako strategia *rolling update* w Kubernetes z zero-downtime.

**Kluczowe decyzje techniczne:** Porównanie ilustruje, że architektura mikroserwisowa nie jest po prostu "lepszą" alternatywą — wprowadza nowe klasy problemów (rozproszone debugowanie, *eventual consistency*, narzut komunikacji sieciowej, złożoność operacyjna), które w monolicie nie istnieją. Wybór tej architektury w niniejszej pracy uzasadniony jest celem edukacyjnym (poznanie wzorców systemów rozproszonych) oraz konkretnymi ograniczeniami monolitu zidentyfikowanymi w rozdziale 1 (krytyczne luki bezpieczeństwa, brak izolacji awarii, vendor lock-in). Dla typowej platformy z porównywalnym wolumenem ruchu monolit modułowy mógłby być rozwiązaniem równie efektywnym przy znacząco niższych kosztach operacyjnych.

**Pliki/komponenty:** Tabela porównawcza zaprezentowana w postaci diagramu (Rys. 2.7).

---

## [ROZDZIAŁ: Projektowanie nowej architektury — przypadki użycia]

**Temat:** Identyfikacja aktorów systemu oraz katalog ich przypadków użycia (ang. *use cases*) realizowanych w nowej architekturze, stanowiących pełną listę wymagań funkcjonalnych adresowanych przez projekt.

**Technologie:** Notacja UML 2.5 (diagram przypadków użycia), Use Case Driven Design.

**Co zostało zaimplementowane:** System TheraLink obsługuje trzech aktorów: klienta (osobę poszukującą pomocy psychologicznej), psychologa (specjalistę świadczącego usługi) oraz administratora (rolę zarządzającą platformą). Aktorami zewnętrznymi są również trzy systemy: Keycloak (dostawca tożsamości), Stripe (procesor płatności) i Zoom (platforma wideokonferencji).

**Przypadki użycia aktora "Klient":**

1. **Rejestracja w systemie** — klient tworzy konto przez formularz Keycloak (polski lub angielski) i otrzymuje domyślną rolę `CLIENT`
2. **Logowanie** — klient uwierzytelnia się przez przepływ Authorization Code z PKCE; otrzymuje access token (15 min) i refresh token (7 dni)
3. **Wyszukiwanie psychologów** — klient przegląda listę psychologów; filtruje po słowie kluczowym i lokalizacji
4. **Przeglądanie profilu psychologa** — klient widzi szczegółowy profil (opis, specjalizacja, stawka, format wizyt: stacjonarna/zdalna)
5. **Rezerwacja wizyty stacjonarnej** — klient wybiera dostępny slot, system tworzy rezerwację w statusie `PENDING`
6. **Rezerwacja wizyty zdalnej** — analogicznie do rezerwacji stacjonarnej, dodatkowo po opłaceniu generowany jest link do spotkania Zoom
7. **Opłacenie wizyty kartą** — klient wypełnia formularz Stripe; backend tworzy `PaymentIntent`; po sukcesie Stripe wywołuje webhook, status wizyty zmienia się z `PENDING` na `PAID`, a następnie (dla wizyt zdalnych) na `CONFIRMED` po utworzeniu spotkania Zoom
8. **Dołączenie do spotkania Zoom** — klient otwiera link Zoom z e-maila lub panelu klienta (link wygenerowany automatycznie)
9. **Przeglądanie historii wizyt** — klient widzi listę swoich wizyt z filtrowaniem po statusie i dacie
10. **Edycja profilu** — klient aktualizuje dane kontaktowe (telefon, historię terapii)

**Przypadki użycia aktora "Psycholog":**

1. **Rejestracja jako psycholog** — psycholog tworzy konto z dodatkowymi polami (specjalizacja, stawka, opis); rola `PSYCHOLOGIST` przypisywana jest przez administratora po weryfikacji
2. **Logowanie** — analogicznie do klienta
3. **Edycja profilu zawodowego** — psycholog aktualizuje opis, stawkę godzinową, specjalizację, lokalizację, format wizyt
4. **Definiowanie dostępności** — psycholog dodaje sloty czasowe do kalendarza (data, godzina rozpoczęcia, czas trwania)
5. **Przeglądanie nadchodzących wizyt** — psycholog widzi rezerwacje pogrupowane wg statusu (`PENDING`, `CONFIRMED`)
6. **Akceptacja / odrzucenie wizyty** — psycholog ręcznie zatwierdza lub odrzuca rezerwacje (status `PENDING` → `ACCEPTED` lub `REJECTED`)
7. **Dostęp do linku Zoom spotkania zdalnego** — psycholog widzi link Zoom dla wizyt zdalnych w swoim panelu (ten sam link co klient)
8. **Przeglądanie historii płatności** — psycholog widzi listę zaksięgowanych płatności za swoje wizyty
9. **Anulowanie wizyty** — psycholog może anulować wizytę z odpowiednim powiadomieniem (status `CANCELLED`)

**Przypadki użycia aktora "Administrator"** (zakres ograniczony — pełny moduł administracyjny poza zakresem pracy):

1. **Logowanie z rolą `ADMIN`** w Keycloak
2. **Promocja klienta na psychologa** — weryfikacja danych i przypisanie roli `PSYCHOLOGIST` w Keycloak
3. **Przegląd dziennika zdarzeń systemu** przez panel Keycloak

**Relacje między przypadkami użycia:**
- `Opłacenie wizyty` **«include»** `Logowanie` (klient musi być uwierzytelniony)
- `Rezerwacja wizyty zdalnej` **«extend»** `Rezerwacja wizyty` (rozszerzenie o krok generowania linku Zoom)
- `Dostęp do linku Zoom` (zarówno dla klienta, jak i psychologa) **«include»** `Opłacenie wizyty` (precondition: wizyta opłacona)

**Aktorzy zewnętrzni i ich zaangażowanie:**
- **Keycloak** uczestniczy we wszystkich przypadkach użycia wymagających uwierzytelnienia (przepływ OIDC PKCE)
- **Stripe** uczestniczy w przypadku użycia `Opłacenie wizyty` (utworzenie `PaymentIntent`, webhook z potwierdzeniem)
- **Zoom** uczestniczy w przypadkach użycia `Rezerwacja wizyty zdalnej` i `Dołączenie do spotkania Zoom` (utworzenie spotkania przez API Server-to-Server OAuth)

**Kluczowe decyzje techniczne:** Świadome ograniczenie zakresu przypadków użycia dla aktora "Administrator" do niezbędnego minimum (promocja roli) podyktowane było potrzebą zachowania realistycznych ram czasowych pracy. Pełny moduł administracyjny (zarządzanie psychologami, statystyki, moderacja recenzji) został wyłączony z zakresu pracy. Świadomie wycięto również przypadki użycia związane z systemem powiadomień email (potwierdzenie wizyty, przypomnienia) oraz systemem rekomendacji psychologów opartym na uczeniu maszynowym — funkcjonalności te nie są częścią architektury docelowej opisanej w pracy.

**Pliki/komponenty:** Diagramy przypadków użycia w notacji UML zaprezentowane jako Rys. 2.8 (klient) i Rys. 2.9 (psycholog).

---

## Sugerowane zrzuty ekranu do tego rozdziału

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram architektury logicznej nowego systemu (na podstawie `docs/architecture-diagram.md` §1, po usunięciu notification-service). Bloki: Angular (4200) → Spring Cloud Gateway (8090) → cztery serwisy domenowe (user 8081, appointment 8082, psychologist 8083, payment 8085) → MongoDB (4 bazy) i Apache Kafka. Bramka i serwisy komunikują się z Keycloak (8080). Payment-service ma linię do Stripe API, appointment-service do Zoom API. Diagram można wygenerować z bloku Mermaid `graph TB` i wyeksportować przez mermaid.live.
> **Sugerowany podpis:** Rys. 2.1. Architektura logiczna systemu TheraLink w wersji mikroserwisowej
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram granic serwisów wg DDD — cztery prostokąty reprezentujące *bounded contexts* (User Context, Appointment Context, Psychologist Schedule Context, Payment Context) z wymienionymi agregatami wewnątrz każdego (Client, Psychologist / Appointment / AvailabilitySlot / Payment). Strzałki między kontekstami oznaczają zdarzenia Kafka.
> **Sugerowany podpis:** Rys. 2.2. Konteksty ograniczone (*bounded contexts*) w domenie biznesowej systemu TheraLink
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram topologii komunikacji — strzałki synchroniczne (REST z JWT) między Angular, Gateway, a serwisami; strzałki asynchroniczne (Kafka) między serwisami publikującymi a konsumującymi zdarzenia. Topiki Kafka wyróżnione jako prostokąty (`theralink.user.created`, `theralink.appointment.created`, `theralink.appointment.confirmed`, `theralink.payment.completed`, `theralink.payment.failed`).
> **Sugerowany podpis:** Rys. 2.3. Topologia komunikacji synchronicznej i asynchronicznej w systemie TheraLink
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram architektury danych wg wzorca *database per service* — cztery cylindry (`theralink-users`, `theralink-appointments`, `theralink-psychologist-schedules`, `theralink-payments`) z wymienionymi kolekcjami, każdy podpięty do swojego serwisu. Linie przerywane symbolizują logiczne referencje przez identyfikatory (bez fizycznych kluczy obcych).
> **Sugerowany podpis:** Rys. 2.4. Architektura danych w paradygmacie *database per service*
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram sekwencji przepływu uwierzytelnienia OIDC PKCE — Klient → Angular → Keycloak → Angular → Gateway → Service. Wymienione kroki: 1. Generacja `code_verifier` i `code_challenge`, 2. Redirect do Keycloak, 3. Strona logowania, 4. Authorization code, 5. Wymiana code+verifier na tokeny, 6. Bearer JWT w żądaniu, 7. Weryfikacja JWKS, 8. Odpowiedź. Można wygenerować z `docs/architecture-diagram.md` §4.
> **Sugerowany podpis:** Rys. 2.5. Przepływ uwierzytelnienia użytkownika w protokole OpenID Connect z rozszerzeniem PKCE
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram mapowania zasobów LOCAL ↔ PROD — dwie kolumny z parami: MongoDB ↔ Azure Cosmos DB, Apache Kafka ↔ Azure Event Hubs, Keycloak + PostgreSQL ↔ Keycloak + Azure DB for PostgreSQL, plik `.env` ↔ Azure Key Vault + CSI Driver, porty wystawione na host ↔ Ingress HTTPS. Diagram ilustruje zasadę identycznego kodu aplikacji w obu środowiskach.
> **Sugerowany podpis:** Rys. 2.6. Mapowanie komponentów infrastruktury między środowiskiem deweloperskim (LOCAL) a produkcyjnym (PROD)
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Tabela porównawcza side-by-side architektury monolitycznej i mikroserwisowej w siedmiu wymiarach (liczba procesów, komunikacja, model danych, bezpieczeństwo, tożsamość, skalowanie, niezależność wdrożeniowa). Kolumny: "Wymiar", "Architektura monolityczna", "Architektura mikroserwisowa". Tabelę można wykonać w Wordzie lub jako diagram.
> **Sugerowany podpis:** Rys. 2.7. Porównanie architektury monolitycznej i mikroserwisowej w siedmiu wymiarach
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram przypadków użycia UML dla aktora "Klient" — owalne use case'y (Rejestracja, Logowanie, Wyszukiwanie psychologów, Przeglądanie profilu, Rezerwacja wizyty stacjonarnej, Rezerwacja wizyty zdalnej, Opłacenie wizyty kartą, Dołączenie do spotkania Zoom, Przeglądanie historii wizyt, Edycja profilu) z aktorem-figurką "Klient" oraz aktorami zewnętrznymi (Keycloak, Stripe, Zoom). Relacje «include» i «extend» zgodnie z opisem w podrozdziale 2.8.
> **Sugerowany podpis:** Rys. 2.8. Diagram przypadków użycia dla aktora "Klient"
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram przypadków użycia UML dla aktora "Psycholog" — owalne use case'y (Rejestracja jako psycholog, Logowanie, Edycja profilu zawodowego, Definiowanie dostępności, Przeglądanie wizyt, Akceptacja/odrzucenie wizyty, Dostęp do linku Zoom, Przeglądanie płatności, Anulowanie wizyty) z aktorem-figurką "Psycholog" oraz aktorami zewnętrznymi (Keycloak, Zoom).
> **Sugerowany podpis:** Rys. 2.9. Diagram przypadków użycia dla aktora "Psycholog"
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram sekwencji UML dla głównego przepływu biznesowego — rezerwacja wizyty zdalnej z płatnością. Aktorzy: Klient, Angular, Gateway, appointment-service, payment-service, Kafka, Stripe, Zoom. Kroki: rezerwacja → utworzenie PaymentIntent → płatność w Stripe → webhook → publikacja `theralink.payment.completed` → konsumpcja w appointment-service → utworzenie spotkania Zoom → publikacja `theralink.appointment.confirmed`. Można wygenerować z `docs/architecture-diagram.md` §3 (po usunięciu kroków notification).
> **Sugerowany podpis:** Rys. 2.10. Diagram sekwencji procesu rezerwacji i opłacenia wizyty zdalnej z automatycznym utworzeniem spotkania Zoom
> **Źródło:** opracowanie własne
