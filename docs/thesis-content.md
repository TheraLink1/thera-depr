# TheraLink — Materiały do pracy dyplomowej

---

## 1. Wprowadzenie — cel pracy

Celem pracy inżynierskiej jest zaprojektowanie i implementacja platformy do rezerwacji wizyt psychologicznych **TheraLink**, ze szczególnym uwzględnieniem migracji z architektury monolitycznej do architektury mikroserwisowej.

Pierwotna wersja systemu oparta była na stosie Node.js/Express (backend) oraz Next.js (frontend) z bazą danych PostgreSQL. Architektura monolityczna, choć prosta w początkowej fazie projektu, ujawniła szereg ograniczeń: trudności w niezależnym skalowaniu poszczególnych modułów, brak izolacji awarii, skomplikowany proces wdrożeń oraz krytyczne luki bezpieczeństwa (m.in. brak weryfikacji podpisów tokenów JWT).

Nowa architektura opiera się na zestawie niezależnych mikroserwisów zbudowanych w oparciu o Spring Boot, komunikujących się asynchronicznie przez Apache Kafka. Każdy serwis posiada własną bazę danych MongoDB, co zapewnia pełną izolację danych i niezależność deploymentu. Warstwa uwierzytelniania została oparta na Keycloak — open-source'owym serwerze tożsamości obsługującym standard OAuth2/OIDC, który zastąpił zewnętrzną usługę AWS Cognito.

System umożliwia:
- rejestrację i zarządzanie profilami klientów oraz psychologów,
- rezerwację wizyt w formie stacjonarnej lub zdalnej,
- bezpieczne przetwarzanie płatności kartą przez Stripe.

Docelowe środowisko produkcyjne oparte jest na Azure Kubernetes Service (AKS) z wykorzystaniem Azure Container Registry, Azure Cosmos DB (MongoDB API) oraz Azure Event Hubs (protokół Kafka).

---

## 2. Lista użytych technologii

### Frontend

| Technologia | Wersja | Opis |
|---|---|---|
| **Angular** | 21 | Framework SPA (Single Page Application) do budowy interfejsu użytkownika. Architektura komponentowa, lazy loading modułów, typowany TypeScript. |
| **TypeScript** | 5.x | Typowany nadzbiór JavaScript. Zapewnia bezpieczeństwo typów w czasie kompilacji i lepsze wsparcie IDE. |
| **NGXS** | — | Biblioteka do zarządzania stanem aplikacji Angular. Używana do stanu współdzielonego między niezwiązanymi komponentami. |
| **Keycloak Angular** | — | Oficjalna biblioteka integrująca Angular z Keycloak. Obsługuje logowanie PKCE, odświeżanie tokenów i interceptory HTTP. |
| **Stripe.js** | — | Biblioteka JavaScript do obsługi formularza płatności po stronie klienta. Tokenizuje dane karty — backend nigdy nie widzi numerów kart. |

### Backend — mikroserwisy

| Technologia | Wersja | Opis |
|---|---|---|
| **Spring Boot** | 4.0.3 | Framework Java do budowy mikroserwisów. Dostarcza auto-konfigurację, wbudowany serwer HTTP i ekosystem gotowych bibliotek (Spring Data, Spring Security, Spring Kafka). |
| **Spring Cloud Gateway** | — | Bramka API (API Gateway) — centralny punkt wejścia do systemu. Weryfikuje tokeny JWT, routuje ruch do odpowiednich serwisów, obsługuje rate limiting. |
| **Spring Security + OAuth2 Resource Server** | — | Automatyczna weryfikacja podpisów tokenów JWT przez pobieranie kluczy publicznych z Keycloak (JWKS endpoint). Zastępuje ręczne `jwt.decode()` z pierwotnej implementacji. |
| **Spring Data MongoDB** | — | Warstwa dostępu do danych dla MongoDB. Automatycznie generuje zapytania z nazw metod (np. `findByKeycloakId`). |
| **Spring Kafka** | — | Integracja Spring Boot z Apache Kafka. Udostępnia `KafkaTemplate` (producer) i `@KafkaListener` (consumer). |
| **Lombok** | — | Biblioteka Java generująca boilerplate code (gettery, settery, konstruktory, builder) za pomocą adnotacji (`@Data`, `@Builder`, `@RequiredArgsConstructor`). |
| **MapStruct** | 1.5.5 | Procesor adnotacji generujący kod mapowania między obiektami (Entity ↔ DTO). Działa w czasie kompilacji — zero narzutu w runtime. |
| **Java** | 25 / 21 | Język programowania. User/Appointment Service: Java 25. Payment Service: Java 21. |

### Uwierzytelnianie

| Technologia | Wersja | Opis |
|---|---|---|
| **Keycloak** | 25.0.4 | Open-source'owy serwer tożsamości (Identity Provider). Obsługuje OAuth2, OpenID Connect, PKCE, zarządzanie użytkownikami, role, własne motywy logowania (FTL). Zastępuje AWS Cognito. |

### Bazy danych

| Technologia | Wersja | Opis |
|---|---|---|
| **MongoDB** | 7.0 | Dokumentowa baza danych NoSQL. Każdy mikroserwis posiada własną, izolowaną bazę (osobne kolekcje, brak współdzielenia). |
| **Azure Cosmos DB** | — | Zarządzana usługa bazodanowa Microsoft Azure z API zgodnym z MongoDB. Używana na produkcji — eliminuje potrzebę samodzielnego zarządzania replikacją i backupami. |
| **PostgreSQL** | 16 | Relacyjna baza danych używana wyłącznie przez Keycloak do przechowywania konfiguracji realm, użytkowników i sesji. |

### Komunikacja asynchroniczna

| Technologia | Wersja | Opis |
|---|---|---|
| **Apache Kafka** | 7.6 (Confluent) | Rozproszony system kolejkowania wiadomości. Zapewnia luźne powiązanie (loose coupling) między serwisami — payment-service publikuje zdarzenia płatności niezależnie od ich konsumentów. Używany lokalnie. |
| **Azure Event Hubs** | — | Zarządzana usługa Microsoft Azure z wbudowaną kompatybilnością protokołu Kafka. Używana na produkcji — kod Spring Boot działa bez zmian. |

### Integracje zewnętrzne

| Technologia | Opis |
|---|---|
| **Stripe** | Platforma płatnicza. Obsługuje tworzenie PaymentIntent, webhooks (potwierdzenie płatności), tokenizację kart. Klucze testowe (`sk_test_`) pozwalają testować bez rzeczywistych transakcji. |

### Infrastruktura

| Technologia | Opis |
|---|---|
| **Docker + Docker Compose** | Konteneryzacja aplikacji. Docker Compose używany w środowisku deweloperskim do uruchamiania całego stacku infrastrukturalnego (Keycloak, MongoDB, Kafka) jedną komendą. |
| **Kubernetes (Azure AKS)** | Orkiestracja kontenerów w środowisku produkcyjnym. Azure Kubernetes Service to zarządzany klaster K8s — Microsoft zarządza warstwą control plane. |
| **Helm** | Package manager dla Kubernetes. Używany do instalacji MongoDB (bitnami/mongodb) na klastrze staging i zarządzania konfiguracją deploymentów. |
| **Azure Container Registry (ACR)** | Prywatny rejestr obrazów Docker w Azure. Obrazy budowane są zdalnie przez `az acr build` — bez potrzeby instalacji Dockera lokalnie. |
| **Azure Key Vault** | Bezpieczne przechowywanie sekretów (klucze API, connection stringi) w środowisku produkcyjnym. Zintegrowany z AKS przez CSI Secret Store Driver. |

---

## 3. Napotkane problemy

### Problem 1 — Brak weryfikacji podpisów JWT (krytyczna luka bezpieczeństwa)

**Opis:** Pierwotna implementacja middleware autentykacji (`backend/src/middleware/authMiddleware.ts`) używała funkcji `jwt.decode()` zamiast `jwt.verify()`. Funkcja `decode()` jedynie dekoduje payload tokenu Base64 — nie sprawdza, czy token jest podpisany ważnym kluczem. Oznaczało to, że każda osoba mogła sfabrykować token JWT z dowolnymi uprawnieniami i uzyskać nieautoryzowany dostęp do API.

**Rozwiązanie:** W nowej architekturze Spring Boot automatycznie weryfikuje tokeny za pomocą Spring Security OAuth2 Resource Server. Serwis pobiera klucze publiczne z Keycloak JWKS endpoint i kryptograficznie weryfikuje podpis każdego tokenu. Deweloper nie musi pisać żadnego kodu weryfikacji.

---

### Problem 2 — Sekrety w kodzie źródłowym

**Opis:** W pierwotnym projekcie klucz API OpenRouter był zakodowany bezpośrednio w pliku `backend/src/routes/chat.ts`, a klucze AWS znajdowały się w plikach `frontend/.env.local` i `frontend/public/.env` — oba potencjalnie trafiające do repozytorium.

**Rozwiązanie:** Wszystkie sekrety przeniesiono do zmiennych środowiskowych. Lokalnie przechowywane są w plikach `.env` i `application-local.yml` (gitignored). Na produkcji zarządzane przez Azure Key Vault z dostępem przez CSI Secret Store Driver.

---

### Problem 3 — Błędna konfiguracja listenerów Kafka w Docker

**Opis:** Pierwotny `docker-compose.yml` konfigurował Kafka z pojedynczym listenerem `PLAINTEXT://localhost:9092`. Gdy Spring Boot uruchamiany jest wewnątrz kontenera Docker i próbuje połączyć się z Kafka, `localhost` wskazuje na jego własny kontener, nie na kontener Kafka. Efekt: serwisy uruchomione w Docker nie mogły wysyłać ani odbierać wiadomości Kafka.

**Rozwiązanie:** Skonfigurowano dwa osobne listenery:
- `INTERNAL://kafka:29092` — dla komunikacji kontener-kontener (nazwa serwisu Docker)
- `EXTERNAL://localhost:9092` — dla aplikacji uruchomionych na hoście (IDE, terminal)

---

### Problem 4 — Błędne zmienne środowiskowe Keycloak

**Opis:** Konfiguracja `docker-compose.yml` używała zmiennych `KC_BOOTSTRAP_ADMIN_USERNAME` i `KC_BOOTSTRAP_ADMIN_PASSWORD`, które zostały wprowadzone dopiero w Keycloak 26. Używany obraz Keycloak 25.0.4 oczekiwał starszych zmiennych `KEYCLOAK_ADMIN` i `KEYCLOAK_ADMIN_PASSWORD`. Efekt: Keycloak startował bez konta administratora, wyświetlając komunikat _"Local access required"_.

**Rozwiązanie:** Zmiana nazw zmiennych środowiskowych w `docker-compose.yml` na `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD`.

---

### Problem 5 — Izolacja serwisu płatności (PCI-DSS)

**Opis:** Standard PCI-DSS (Payment Card Industry Data Security Standard) nakłada wymóg izolacji komponentów przetwarzających dane płatnicze. Trzymanie kodu płatności w tym samym repozytorium co reszta aplikacji naruszałoby wymagania audytowe.

**Rozwiązanie:** Serwis płatności (`thera-payment-service`) prowadzony jest jako oddzielne, prywatne repozytorium z ograniczonym dostępem. Pozostałe serwisy nie mają wiedzy o Stripe — komunikacja odbywa się wyłącznie przez zdarzenia Kafka (`theralink.payment.completed`).

---

### Problem 6 — MongoDB replica set a brak transakcji

**Opis:** MongoDB wymaga replica set (minimum 3 węzły) do obsługi transakcji ACID. Uruchamianie replica set w środowisku deweloperskim generuje znaczny narzut konfiguracyjny i zużycie zasobów.

**Rozwiązanie:** Po konsultacji zdecydowano, że mikroserwisy TheraLink nie używają transakcji wielodokumentowych — każda operacja dotyczy jednego dokumentu. Środowisko deweloperskie używa MongoDB standalone (jedna replika), środowisko produkcyjne używa Azure Cosmos DB z wbudowaną redundancją.

---

### Problem 7 — Vendor lock-in AWS Cognito

**Opis:** Pierwotna implementacja opierała się na AWS Cognito jako dostawcy tożsamości z biblioteką AWS Amplify po stronie frontendu. Takie podejście wiązało architekturę z konkretną chmurą i utrudniało migrację lub zmianę dostawcy.

**Rozwiązanie:** Zastąpienie Cognito przez Keycloak — open-source'owy serwer tożsamości oparty na standardach OAuth2/OIDC. Keycloak można uruchomić w dowolnej chmurze lub on-premise, a integracja opiera się na standardach, a nie na SDK konkretnego dostawcy.