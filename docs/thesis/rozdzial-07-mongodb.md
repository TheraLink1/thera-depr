# Rozdział 7. Migracja warstwy danych na bazę MongoDB

Rozdział opisuje migrację warstwy trwałości danych systemu TheraLink z relacyjnej bazy
PostgreSQL zarządzanej przez Prisma ORM na bazę dokumentową MongoDB z interfejsem
Spring Data MongoDB. Omówiono różnice między modelem relacyjnym a dokumentowym,
uzasadnienie przyjęcia wzorca *database per service*, modelowanie dokumentów
z uwzględnieniem zagadnień embeddingu i referencji, implementację repozytoriów
Spring Data, podejście do zastąpienia operacji JOIN zdarzeniami Kafka oraz strategię
migracji istniejących danych.

---

## 7.1 Bazy danych relacyjne a dokumentowe — podstawy teoretyczne

### Model relacyjny — PostgreSQL

PostgreSQL [X] jest systemem zarządzania relacyjną bazą danych (ang. *Relational Database
Management System*, RDBMS) opartym na modelu tabel, wierszy i kluczy obcych.
Dane są ściśle ustrukturyzowane: przed wstawieniem pierwszego rekordu konieczne
jest zdefiniowanie schematu tabeli — nazw i typów kolumn, ograniczeń i relacji.
Podejście to określa się mianem *schema-on-write* (schemat przy zapisie).

Relacyjne bazy danych gwarantują własności ACID (ang. *Atomicity, Consistency, Isolation,
Durability*) [X]: operacje są atomowe (albo całe się powiodą, albo cofną), spójne
(baza przechodzi między poprawnymi stanami), izolowane (współbieżne transakcje
nie widzą swoich częściowych efektów) i trwałe (zatwierdzone dane przeżywają restarty).
Gwarancje te mają szczególne znaczenie w systemach finansowych — właśnie dlatego
serwis płatności `theralink-payment-service` korzysta z MongoDB z zaostrzonym
modelowaniem, a nie z rezygnacji z integralności danych.

Dostęp do danych w modelu relacyjnym opiera się na języku SQL [X] z operacją
`JOIN`, która łączy wiersze z wielu tabel na podstawie kluczy obcych. W monolicie
TheraLink jedna baza PostgreSQL przechowywała wszystkie encje: `Client`, `Psychologist`,
`Appointment`, `Payment`, `CalendarAppointment`, a encja `Appointment` łączyła
się relacjami z `Client` i `Psychologist`.

### Model dokumentowy — MongoDB

MongoDB [X] jest bazą danych dokumentową (ang. *document database*), w której
jednostką przechowywania jest dokument BSON (ang. *Binary JSON*) — zagnieżdżona
struktura klucz-wartość analogiczna do obiektu JSON. Dokumenty grupowane są
w kolekcjach (ang. *collections*), które pełnią rolę analogiczną do tabel SQL,
lecz nie narzucają stałego schematu. Każdy dokument w kolekcji może mieć inny
zestaw pól — jest to podejście *schema-on-read* (schemat przy odczycie): struktura
interpretowana jest dopiero przy pobieraniu danych przez aplikację.

MongoDB projektowana jest z myślą o dostępności i skalowalności poziomej kosztem
pełnych gwarancji ACID — właściwości systemu określa się jako BASE (ang. *Basically
Available, Soft state, Eventually consistent*) [X]. Od wersji 4.0 MongoDB wspiera
transakcje wielodokumentowe, jednak ich użycie jest rzadkie w praktyce mikrousługowej,
gdzie granica transakcji wyznaczona jest przez jeden serwis i jeden dokument.

### Tabela porównawcza

**Tabela 7.1.** Porównanie baz danych PostgreSQL i MongoDB

| Kryterium | PostgreSQL (monolit) | MongoDB (mikrousługi) |
|---|---|---|
| Model danych | Tabele i wiersze (relacyjny) | Dokumenty BSON (dokumentowy) |
| Schemat | Schema-on-write (migracje DDL) | Schema-on-read (elastyczny) |
| Relacje | Klucze obce, JOIN | Brak JOIN — referencja przez String |
| Transakcje | ACID (wielotabelowe) | BASE; transakcje od v4.0 (jedno-serwisowe) |
| Skalowanie | Wertykalne + replikacja | Poziome (sharding) |
| Migracje schematu | Wymagane (Prisma Migrate, Flyway) | Brak — pola dodawane bez DDL |
| Zapytania | SQL (JOIN, GROUP BY, subquery) | MQL (aggregation pipeline) |
| Driver Spring | Spring Data JPA / Hibernate | Spring Data MongoDB |
| Prod w TheraLink | Zarządzany PostgreSQL (Heroku/AWS) | Azure Cosmos DB (MongoDB API) |

**Przed / Po — warstwa trwałości danych**

| Aspekt | Monolit (przed) | Mikrousługi (po) |
|---|---|---|
| Technologia | PostgreSQL 15 + Prisma ORM | MongoDB 7.0 + Spring Data MongoDB |
| Schemat | Migracje Prisma (`prisma migrate deploy`) | Brak migracji DDL — pola elastyczne |
| Identyfikator użytkownika | `cognitoId` (AWS Cognito) | `keycloakId` (Keycloak JWT `sub`) |
| Środowisko prod | Zarządzana instancja PostgreSQL | Azure Cosmos DB (MongoDB API) |

---

## 7.2 Wzorzec Database per Service

### Granice kontekstu w architekturze DDD

Wzorzec *database per service* [X] jest konsekwencją zasady ograniczonego kontekstu
(ang. *bounded context*) z dziedziny Domain-Driven Design (ang. *Domain-Driven Design*,
DDD) [X]: każdy serwis jest właścicielem wyłącznie własnych danych i nie może
bezpośrednio odczytywać ani modyfikować bazy innego serwisu. Zapewnia to:

- **Niezależne wdrożenia** — zmiana schematu danych jednego serwisu nie blokuje
  wdrożeń pozostałych serwisów,
- **Izolację awarii** — przeciążenie bazy jednego serwisu nie wpływa na pozostałe,
- **Swobodę doboru technologii** — każdy serwis może wybrać technologię bazy
  danych odpowiadającą charakterystyce swoich danych.

### Bazy danych w systemie TheraLink

W systemie TheraLink wydzielono cztery logiczne bazy danych. W środowisku deweloperskim
każda baza jest osobną bazą w jednym kontenerze MongoDB, identyfikowaną przez
ostatni segment adresu URI. W środowisku produkcyjnym każda baza mapuje się
na odrębną instancję Azure Cosmos DB, co zapewnia pełną izolację na poziomie
infrastruktury.

**Tabela 7.2.** Podział baz danych na serwisy w systemie TheraLink

| Serwis | Baza MongoDB | Kolekcje | Uzasadnienie izolacji |
|---|---|---|---|
| `theralink-user-service` | `theralink-users` | `clients`, `psychologists` | Dane profilowe użytkowników — niezbędne przy każdym żądaniu |
| `theralink-appointment-service` | `theralink-appointments` | `appointments` | Harmonogram wizyt — wysoka częstotliwość odczytów/zapisów |
| `theralink-psychologist-service` | `theralink-psychologist-schedules` | `schedules` | Dostępność terminów — niezależna od profili użytkowników |
| `theralink-payment-service` | `theralink-payments` | `payments` | **PCI-DSS** — izolacja danych finansowych jest wymogiem standardu |

Izolacja bazy `theralink-payments` ma szczególne znaczenie dla zgodności ze standardem
bezpieczeństwa płatności PCI-DSS (ang. *Payment Card Industry Data Security Standard*) [X]:
dane transakcyjne nie mogą być przechowywane w bazie wspólnej z danymi niefinansowymi,
co uzasadnia osobne repozytorium kodu i osobną bazę danych dla serwisu płatności.

**Przed / Po — organizacja bazy danych**

| Aspekt | Monolit (przed) | Mikrousługi (po) |
|---|---|---|
| Liczba baz | 1 instancja PostgreSQL, 5 tabel | 4 bazy MongoDB (logicznie oddzielone) |
| Dostęp między domenami | Bezpośrednie JOIN przez klucze obce | Tylko przez zdarzenia Kafka lub API |
| Zmiana schematu | Migracja blokuje wszystkie moduły | Zmiana schematu w jednej bazie bez wpływu na inne |
| Zgodność z PCI-DSS | Dane płatnicze w tej samej bazie co dane klientów | Baza `theralink-payments` całkowicie izolowana |

---

## 7.3 Modelowanie dokumentów MongoDB

### Embedding kontra referencing

W modelu dokumentowym MongoDB projektant musi zdecydować, w jaki sposób przechowywać
powiązane dane. Do dyspozycji są dwa podejścia:

- **Embedding** (zagnieżdżenie) — dane powiązane przechowywane bezpośrednio
  wewnątrz dokumentu nadrzędnego jako subdokument lub tablica. Optymalnie gdy
  dane zawsze odczytywane są razem, relacja jest trwała i nie przekracza granicy
  16 MB dokumentu MongoDB.
- **Referencing** (referencja) — dokument przechowuje jedynie identyfikator
  powiązanego dokumentu, podobnie do klucza obcego w SQL. Optymalnie gdy
  powiązane dane są dużych rozmiarów, często aktualizowane niezależnie lub
  należą do innego serwisu.

W systemie TheraLink stosowane są wyłącznie referencje między serwisami — dokument
`Payment` przechowuje `appointmentId` jako typ `String`, a nie zagnieżdżony dokument
`Appointment`. Jest to konsekwencja wzorca *database per service*: serwis płatności
nie ma dostępu do kolekcji `appointments` i nie może zagnieździć jej dokumentów.

### Stan przed migracją — schemat Prisma

Poniższy listing prezentuje schemat bazy danych monolitu zdefiniowany w pliku
konfiguracyjnym Prisma ORM [X] — punkt wyjścia do migracji:

**Listing 7.1.** Schemat relacyjny monolitu — encje `Client`, `Psychologist`, `Appointment` i `Payment`
(plik `TheraLink/backend/prisma/schema.prisma`, linie 25–71)

```prisma
model Client {
  id           Int           @id @default(autoincrement())
  cognitoId    String        @unique
  name         String
  email        String
  phoneNumber  String
  history      String?
  appointments Appointment[]
  payments     Payment[]
}

model Psychologist {
  id                  Int                   @id @default(autoincrement())
  cognitoId           String                @unique
  name                String
  email               String
  phoneNumber         String
  location            String                @default("")
  appointments        Appointment[]
  hourlyRate          Int
  Description         String
  Specialization      String
  CalendarAppointment CalendarAppointment[]
}

model Appointment {
  id              Int               @id @default(autoincrement())
  clientCognitoId String
  client          Client            @relation(fields: [clientCognitoId], references: [cognitoId])
  psychologistId  String
  psychologist    Psychologist      @relation(fields: [psychologistId], references: [cognitoId])
  meetingLink     String
  date            DateTime
  Status          ApplicationStatus
  payment         Payment?
}

model Payment {
  id              Int         @id @default(autoincrement())
  appointmentId   Int         @unique
  appointment     Appointment @relation(fields: [appointmentId], references: [id])
  clientCognitoId String?
  client          Client?     @relation(fields: [clientCognitoId], references: [cognitoId])
  paymentDate     DateTime
  isPaid          Boolean
  amount          Int
}
```

Schemat relacyjny cechuje się gęstą siecią relacji: encja `Appointment` łączy się
z `Client` i `Psychologist` przez klucze obce, a `Payment` łączy się z `Appointment`
relacją jeden-do-jednego. Taki model wymaga operacji `JOIN` przy każdym zapytaniu
o szczegóły wizyty lub historię płatności. Identyfikatorem użytkownika jest `cognitoId`
— numer konta AWS Cognito, który w wyniku migracji uwierzytelnienia (rozdział 5) musi
zostać zastąpiony `keycloakId` — wartością pola `sub` tokenu JWT wystawionego przez Keycloak.

### Dokument Client

**Listing 7.2.** Dokument MongoDB `Client` — adnotacje Spring Data
(plik `thera-rest-service/src/main/java/com/example/therarestservice/model/Client.java`, linie 1–28)

```java
package com.example.therarestservice.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "clients")
public class Client {

    @Id
    private String id;

    @Indexed(unique = true)
    private String keycloakId;

    private String name;
    private String email;
    private String phoneNumber;
    private String history;
}
```

Adnotacja `@Document(collection = "clients")` wskazuje Spring Data MongoDB,
że klasa mapuje się na kolekcję `clients` w bazie `theralink-users`. Jest to odpowiednik
adnotacji `@Entity` i `@Table` ze Spring Data JPA, jednak MongoDB nie tworzy schematu
tabeli — kolekcja powstaje automatycznie przy pierwszym wstawieniu dokumentu.

Pole `id` oznaczone `@Id` przechowuje identyfikator dokumentu MongoDB — domyślnie
generowany jako ObjectId (24-znakowy łańcuch szesnastkowy, np. `507f1f77bcf86cd799439011`).
Pole `keycloakId` opatrzone `@Indexed(unique = true)` posiada unikalny indeks MongoDB,
co gwarantuje, że dla każdego konta Keycloak istnieje co najwyżej jeden profil klienta.
Indeks umożliwia też efektywne wyszukiwanie po `keycloakId` bez skanowania całej kolekcji.

Kluczową różnicą w stosunku do schematu Prisma jest zastąpienie pola `cognitoId`
(typ `String @unique`) przez `keycloakId` (`@Indexed(unique = true)`) oraz usunięcie
relacji `appointments` i `payments` — w modelu dokumentowym serwis użytkowników
nie przechowuje referencji do wizyt ani płatności z innych serwisów.

### Dokument Psychologist

**Listing 7.3.** Dokument MongoDB `Psychologist` — wzbogacone pole `sessionFormat`
(plik `thera-rest-service/src/main/java/com/example/therarestservice/model/Psychologist.java`, linie 1–36)

```java
package com.example.therarestservice.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Document(collection = "psychologists")
public class Psychologist {

    @Id
    private String id;

    @Indexed(unique = true)
    private String keycloakId;

    private String name;
    private String email;
    private String phoneNumber;
    private String location;
    private Integer hourlyRate;
    private Integer yearsOfExperience;
    private String description;
    private String specialization;

    // Format prowadzenia wizyt — widoczny publicznie na profilu psychologa.
    // Appointment Service używa tego pola do walidacji przy rezerwacji.
    private SessionFormat sessionFormat;
}
```

Dokument `Psychologist` zawiera pole `sessionFormat` z wyliczeniem
`SessionFormat { REMOTE, IN_PERSON, BOTH }`, nieobecne w schemacie Prisma.
Jest to przykład wzbogacenia modelu danych możliwego dzięki *schema-on-read* —
dodanie nowego pola do dokumentu MongoDB nie wymaga migracji DDL ani zatrzymania
bazy danych. Serwis wizyt może odczytać wartość `sessionFormat` przez API serwisu
użytkowników przy rezerwacji wizyty, bez dostępu do kolekcji `psychologists`.

### Dokument Payment

**Listing 7.4.** Dokument MongoDB `Payment` — izolacja finansowa i model Stripe
(plik `thera-payment-service/src/main/java/com/theralink/paymentservice/model/Payment.java`, linie 27–67)

```java
@Document(collection = "payments")
@Data           // Lombok: generuje gettery, settery, equals, hashCode, toString
@Builder        // Lombok: wzorzec Builder — Payment.builder().appointmentId("x").build()
@NoArgsConstructor
@AllArgsConstructor
public class Payment {

    @Id // MongoDB _id (generowany automatycznie jako ObjectId, ale trzymamy jako String)
    private String id;

    // ID wizyty z appointment-service.
    // Referencja przez String (nie JOIN jak w SQL) — mikroserwisy nie dzielą bazy!
    @Indexed(unique = true)
    private String appointmentId;

    // Subject z tokenu JWT Keycloak — unikalny identyfikator użytkownika
    @Indexed
    private String clientKeycloakId;

    // ID obiektu PaymentIntent w Stripe (format: "pi_3QdXXXXXXXXXXXX")
    // Potrzebny do identyfikacji w webhooku i ewentualnych refundacji
    @Indexed(unique = true)
    private String stripePaymentIntentId;

    // Kwota w GROSZACH (najmniejsza jednostka waluty).
    // 150,00 PLN = 15000. To najczęstszy błąd przy integracji Stripe!
    private Long amount;

    // Kod waluty ISO 4217, małe litery: "pln", "eur", "usd"
    private String currency;

    private PaymentStatus status;

    // Kiedy backend stworzył PaymentIntent (nie kiedy klient zapłacił!)
    private Instant createdAt;

    // Kiedy Stripe przysłał webhook potwierdzający płatność (null do momentu sukcesu)
    private Instant paidAt;

    private String description;
}
```

Dokument `Payment` ilustruje kilka istotnych decyzji projektowych. Pole `appointmentId`
jest typem `String` — referencja do dokumentu z kolekcji `appointments` innego serwisu,
nie kluczem obcym SQL. Pole `stripePaymentIntentId` przechowuje identyfikator obiektu
PaymentIntent ze Stripe (format `pi_3Q...`), niezbędny do identyfikacji płatności
w webhooku i obsługi zwrotów. Kwota `amount` przechowywana jest w groszach
(ang. *smallest currency unit*) jako typ `Long` zgodnie z wymaganiem Stripe API —
podanie wartości `15000` oznacza 150,00 PLN.

W stosunku do encji `Payment` z schematu Prisma model MongoDB jest znacznie bogatszy:
zamiast pól `paymentDate` i `isPaid` (boolean) wprowadzono `createdAt` i `paidAt`
(znaczniki czasu Instant) oraz `status` (wyliczenie `PENDING/COMPLETED/FAILED/REFUNDED`),
co pozwala na śledzenie pełnego cyklu życia płatności.

---

## 7.4 Spring Data MongoDB — repozytoria i zapytania

### Interfejs MongoRepository

Spring Data MongoDB [X] dostarcza mechanizm repozytoriów analogiczny do Spring Data JPA:
wystarczy zadeklarować interfejs rozszerzający `MongoRepository<T, ID>`, a Spring
automatycznie wygeneruje w czasie wykonania implementację klas, przekształcając nazwy
metod na zapytania MongoDB. Nie jest pisany żaden kod dostępu do bazy danych.

**Listing 7.5.** Repozytorium kolekcji `clients` z zapytaniami po `keycloakId`
(plik `thera-rest-service/src/main/java/com/example/therarestservice/repository/ClientRepository.java`, linie 1–13)

```java
package com.example.therarestservice.repository;

import com.example.therarestservice.model.Client;
import org.springframework.data.mongodb.repository.MongoRepository;

import java.util.Optional;

public interface ClientRepository extends MongoRepository<Client, String> {

    Optional<Client> findByKeycloakId(String keycloakId);

    boolean existsByKeycloakId(String keycloakId);
}
```

Deklaracja `extends MongoRepository<Client, String>` dostarcza gotowych metod:
`save()`, `findById()`, `findAll()`, `deleteById()` i kilkunastu innych. Własne metody
`findByKeycloakId` i `existsByKeycloakId` są generowane przez Spring Data z nazwy:
prefiks `findBy`/`existsBy` wskazuje typ operacji, a `KeycloakId` — nazwę pola dokumentu
(`keycloakId`). Wygenerowane zapytanie to odpowiednik:

```
db.clients.findOne({ keycloakId: "<wartość>" })
```

**Przed / Po — dostęp do danych użytkownika**

| Operacja | Prisma ORM (przed) | Spring Data MongoDB (po) |
|---|---|---|
| Znalezienie klienta po id użytkownika | `prisma.client.findUnique({ where: { cognitoId } })` | `clientRepository.findByKeycloakId(keycloakId)` |
| Identyfikator użytkownika | `cognitoId` (AWS Cognito) | `keycloakId` (Keycloak `sub` z JWT) |
| Sprawdzenie istnienia profilu | `prisma.client.count({ where: { cognitoId } }) > 0` | `clientRepository.existsByKeycloakId(keycloakId)` |
| Zapis nowego profilu | `prisma.client.create({ data: { ... } })` | `clientRepository.save(client)` |

**Listing 7.6.** Repozytorium kolekcji `payments` — derived query methods z filtrowaniem
(plik `thera-payment-service/src/main/java/com/theralink/paymentservice/repository/PaymentRepository.java`, linie 25–38)

```java
public interface PaymentRepository extends MongoRepository<Payment, String> {

    // Znajdź płatność powiązaną z konkretną wizytą
    Optional<Payment> findByAppointmentId(String appointmentId);

    // Historia płatności klienta (np. do wyświetlenia w profilu)
    List<Payment> findByClientKeycloakId(String clientKeycloakId);

    // Używane w handleWebhook() — Stripe podaje nam swoje ID, nie nasze
    Optional<Payment> findByStripePaymentIntentId(String stripePaymentIntentId);

    // Filtrowanie płatności po statusie — przydatne do raportów i dashboardu admina
    List<Payment> findByClientKeycloakIdAndStatus(String clientKeycloakId, PaymentStatus status);
}
```

Metoda `findByClientKeycloakIdAndStatus` demonstruje łączenie wielu kryteriów filtrowania
przez Spring Data: słowo `And` w nazwie metody generuje zapytanie z operatorem `$and`:

```
db.payments.find({ clientKeycloakId: "<id>", status: "COMPLETED" })
```

Metoda `findByStripePaymentIntentId` jest wywoływana wewnątrz handlera webhooka Stripe:
gdy Stripe dostarcza powiadomienie `payment_intent.succeeded`, identyfikatorem
jest `stripePaymentIntentId` (format `pi_3Q...`), a nie wewnętrzny identyfikator MongoDB.

---

## 7.5 Brak joinów — denormalizacja i eventy Kafka

### Problem zastąpienia operacji JOIN

Jedną z największych różnic architektonicznych między modelem relacyjnym a dokumentowym
w kontekście mikrousług jest brak operacji JOIN. W schemacie Prisma zapytanie
o szczegóły wizyty wraz z danymi klienta i psychologa realizowane było jednym
poleceniem SQL z dwoma JOIN:

```sql
-- Zapytanie Prisma (TypeScript, monolit)
const appointment = await prisma.appointment.findUnique({
  where: { id: appointmentId },
  include: {
    client: true,       -- JOIN clients ON clientCognitoId = clients.cognitoId
    psychologist: true  -- JOIN psychologists ON psychologistId = psychologists.cognitoId
  }
});
```

W architekturze mikrousługowej nie ma możliwości wykonania JOIN między kolekcjami
różnych serwisów — bazy są fizycznie izolowane. Do dyspozycji pozostają trzy podejścia:

**1. Denormalizacja (embedding)**

Dane potrzebne razem przechowuje się w tym samym dokumencie, nawet jeśli duplikują
informacje z innej kolekcji. Dokument `Appointment` mógłby zawierać imię i e-mail
klienta jako pola `clientName` i `clientEmail`, kopiowane w momencie rezerwacji.
Wadą jest konieczność aktualizacji wszystkich kopii przy zmianie danych klienta.

**2. Client-side join (złączenie po stronie aplikacji)**

Serwis wykonuje dwa osobne zapytania do dwóch serwisów przez API i scala wyniki
w pamięci aplikacji. Podejście poprawne semantycznie, ale zwiększa opóźnienie (dwa
HTTP round-tripy) i tworzy synchroniczne sprzężenie między serwisami.

**3. Zdarzenia Kafka (podejście przyjęte w TheraLink)**

Serwisy komunikują się wyłącznie przez zdarzenia asynchroniczne. Gdy serwis wizyt
potrzebuje informacji o płatności, subskrybuje topik `theralink.payment.completed`
i aktualizuje własny dokument `Appointment` polem `isPaid` lub `paymentStatus`.
Każdy serwis posiada podzbiór danych wystarczający do obsługi własnych zapytań.

**Tabela 7.3.** Porównanie podejść do zastąpienia operacji JOIN

| Podejście | Spójność danych | Sprzężenie serwisów | Złożoność | Zastosowanie w TheraLink |
|---|---|---|---|---|
| Denormalizacja | Ewentualna (kopie danych) | Brak | Niska | Częściowo — `clientKeycloakId` w `Payment` |
| Client-side join | Natychmiastowa | Synchroniczne | Średnia | Nie stosowane |
| Zdarzenia Kafka | Ewentualna | Luźne (przez topik) | Wyższa | Tak — `payment.completed` → Appointment |

Pole `clientKeycloakId` w dokumencie `Payment` to przykład minimalnej denormalizacji:
serwis płatności przechowuje identyfikator klienta, który jest potrzebny do filtrowania
historii płatności bez odwoływania się do serwisu użytkowników. Jednocześnie serwis
nie przechowuje imienia klienta — ta informacja dostępna jest przez API serwisu
użytkowników wyłącznie na potrzeby wyświetlenia w interfejsie.

---

## 7.6 Konfiguracja środowiska MongoDB

### Docker Compose — środowisko deweloperskie

Konfiguracja MongoDB w środowisku deweloperskim opiera się na jednym kontenerze
z obrazem `mongo:7.0`. Wzorzec *database per service* realizowany jest przez
odrębne bazy danych w ramach tej samej instancji MongoDB — każdy serwis wskazuje
na własną bazę przez ostatni segment URI:

**Listing 7.7.** Konfiguracja URI MongoDB w pliku `application.yml` serwisu użytkowników
(plik `thera-rest-service/src/main/resources/application.yml`, linie 1–12)

```yaml
server:
  port: 8081

spring:
  application:
    name: theralink-user-service

  data:
    mongodb:
      # ${ZMIENNA:domyslna_wartosc} — czyta zmienną środowiskową,
      # jeśli nie istnieje, używa wartości po dwukropku (dla lokalnego dev)
      uri: ${MONGODB_URI:mongodb://localhost:27017/theralink-users}
```

Wzorzec `${MONGODB_URI:mongodb://localhost:27017/theralink-users}` oznacza: użyj wartości
zmiennej środowiskowej `MONGODB_URI`, a jeśli ta nie jest zdefiniowana, zastosuj wartość
domyślną `mongodb://localhost:27017/theralink-users`. Analogicznie serwis płatności
korzysta z bazy `theralink-payments` pod adresem `mongodb://localhost:27017/theralink-payments`.

**Listing 7.8.** Definicja kontenera MongoDB 7.0 w Docker Compose
(plik `thera-infrastructure/docker-compose/docker-compose.yml`, linie 73–85)

```yaml
  mongodb:
    image: mongo:7.0
    container_name: thera-mongodb
    networks: [theralink-network]
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
```

Wolumen `mongodb_data` zapewnia trwałość danych między restartami kontenera.
Healthcheck korzysta z `mongosh` — powłoki MongoDB Shell — do weryfikacji
dostępności bazy przed uruchomieniem zależnych serwisów Spring Boot.

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** MongoDB Compass (`mongodb://localhost:27017`) — rozwinięta lista baz danych po lewej stronie: widoczne `theralink-users` (kolekcje: `clients`, `psychologists`) i `theralink-payments` (kolekcja: `payments`); ewentualnie kilka dokumentów w kolekcji `clients` z polami `keycloakId`, `name`, `email`
> **Sugerowany podpis:** Rys. 7.1. Widok baz danych TheraLink w MongoDB Compass — podział na bazy per serwis
> **Źródło:** opracowanie własne

### Azure Cosmos DB — środowisko produkcyjne

W środowisku produkcyjnym Azure AKS zastosowano usługę Azure Cosmos DB (interfejs
MongoDB API) [X]. Cosmos DB jest zarządzaną, globalnie dystrybuowaną bazą danych
oferującą pełną kompatybilność z protokołem MongoDB w wersji 4.2+. Aplikacje Spring Boot
nie wymagają żadnych zmian w kodzie — podmiana środowiska produkcyjnego ogranicza
się do ustawienia zmiennej środowiskowej `MONGODB_URI` na connection string
dostarczony przez portal Azure:

```
MONGODB_URI=mongodb://<account>:<key>@<account>.mongo.cosmos.azure.com:10255/<dbname>?ssl=true&replicaSet=globaldb
```

Cosmos DB automatycznie zarządza replikacją, tworzeniem kopii zapasowych i skalowaniem
przepustowości, eliminując potrzebę ręcznego zarządzania klastrem MongoDB w środowisku
produkcyjnym. Każdy serwis TheraLink posiada własną instancję Cosmos DB, co zapewnia
izolację rozliczeniową i kontrolę dostępu na poziomie zasobu Azure.

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Portal Azure → Azure Cosmos DB → widok zasobu `cosmos-theralink-payments` (lub analogicznego) — zakładka „Data Explorer" z kolekcją `payments` i kilkoma dokumentami, lub widok „Connection String" z zamazanym kluczem
> **Sugerowany podpis:** Rys. 7.2. Zasób Azure Cosmos DB (MongoDB API) dla serwisu płatności TheraLink
> **Źródło:** opracowanie własne

---

## 7.7 Migracja danych z PostgreSQL

### Strategia ETL

Migracja istniejących danych z bazy PostgreSQL monolitu do baz MongoDB poszczególnych
serwisów realizowana jest przez skrypt ETL (ang. *Extract–Transform–Load*) [X]:

1. **Extract (ekstrakcja)** — pobranie danych z PostgreSQL przy użyciu klienta SQL
   (np. `pg` w Node.js lub `psycopg2` w Pythonie),
2. **Transform (transformacja)** — przekształcenie struktury: zmiana typów identyfikatorów
   (integer `id` → string ObjectId), zamiana `cognitoId` na `keycloakId` (wymaga
   mapowania przez API Keycloak lub tabelę migracyjną) oraz usunięcie relacji
   (encja `Appointment` trafia do serwisu wizyt bez pól `client.*` i `psychologist.*`),
3. **Load (ładowanie)** — wstawienie przekształconych dokumentów do odpowiednich
   kolekcji MongoDB za pomocą sterownika MongoDB.

**Tabela 7.4.** Mapowanie kolumn PostgreSQL na pola dokumentu MongoDB podczas migracji

| Encja Prisma | Kolumna PostgreSQL | Dokument MongoDB | Pole MongoDB | Transformacja |
|---|---|---|---|---|
| `Client` | `id` (INT) | `clients` | `id` (String) | `ObjectId.toHexString()` |
| `Client` | `cognitoId` (String) | `clients` | `keycloakId` (String) | lookup w Keycloak Admin API |
| `Client` | `name`, `email`, `phoneNumber`, `history` | `clients` | (bez zmian) | brak |
| `Psychologist` | `cognitoId` | `psychologists` | `keycloakId` | lookup w Keycloak Admin API |
| `Psychologist` | `Description`, `Specialization` | `psychologists` | `description`, `specialization` | zmiana wielkości liter (camelCase) |
| `Appointment` | `clientCognitoId` | `appointments` | `clientKeycloakId` | lookup w Keycloak Admin API |
| `Appointment` | `Status` | `appointments` | `status` | mapowanie enum: `Pending→PENDING` |
| `Payment` | `appointmentId` (INT) | `payments` | `appointmentId` (String) | `String.valueOf()` |
| `Payment` | `isPaid` (Boolean) | `payments` | `status` (Enum) | `true→COMPLETED`, `false→PENDING` |
| `Payment` | `amount` (INT, w złotych) | `payments` | `amount` (Long, w groszach) | `value * 100` |

Najtrudniejszą częścią migracji jest mapowanie `cognitoId` na `keycloakId`. Realizowane jest
przez Keycloak Admin REST API [X], które umożliwia wyszukanie użytkownika po atrybucie
niestandardowym (każdy zmigrowany użytkownik powinien mieć atrybut `cognitoId` dodany
w procesie tworzenia konta Keycloak). Alternatywnie, jeśli adres e-mail jest unikalny,
można wyszukać użytkownika Keycloak po adresie e-mail i pobrać jego `sub` (keycloakId).

Skrypt ETL uruchamiany jest jednorazowo przy przejściu produkcyjnym z monolitu na
mikrousługi. Zalecane podejście to migracja w trybie read-only (monolit wyłączony),
weryfikacja liczby rekordów po załadowaniu i ponowne uruchomienie z nowymi serwisami.

---

## 7.8 Podsumowanie — migracja warstwy danych

Zastąpienie centralnej bazy PostgreSQL czterema niezależnymi bazami MongoDB jest
fundamentalną zmianą architektoniczną w systemie TheraLink. Zmiana ta jest warunkiem
koniecznym dla realizacji wzorca *database per service* i wynikającego z niego
luźnego sprzężenia serwisów.

**Tabela 7.5.** Zbiorcze porównanie warstwy danych przed i po migracji

| Kryterium | Monolit PostgreSQL (przed) | Mikrousługi MongoDB (po) |
|---|---|---|
| Model danych | Relacyjny (tabele, klucze obce, JOIN) | Dokumentowy (kolekcje, referencje przez String) |
| Liczba baz | 1 instancja, 5 tabel | 4 instancje MongoDB (logicznie oddzielone) |
| Schemat | Ścisły (migracje DDL przez Prisma Migrate) | Elastyczny (schema-on-read, bez migracji) |
| Transakcje | ACID wielotabelowe | BASE; brak cross-service transakcji |
| Identyfikator użytkownika | `cognitoId` (AWS Cognito, Int lub String) | `keycloakId` (Keycloak `sub`, UUID) |
| Dostęp cross-domain | JOIN SQL | Zdarzenia Kafka + denormalizacja |
| Środowisko prod | Zarządzany PostgreSQL | Azure Cosmos DB (MongoDB API, per serwis) |
| Migracje schematu | Blokują całą bazę | Dodanie pola — brak przestojów |
| Monitoring | pgAdmin / pg_stat_activity | MongoDB Compass / Azure Cosmos DB Insights |

Rezygnacja z gwarancji ACID na poziomie przekrojowym (cross-service) jest świadomą
decyzją architektoniczną, wynikającą z przyjęcia modelu spójności ewentualnej
(ang. *eventual consistency*). Zdarzenia Kafka pełnią rolę mechanizmu synchronizacji
stanu między serwisami: po zdarzeniu `theralink.payment.completed` stan dokumentów
`Payment` i `Appointment` jest spójny po przetworzeniu zdarzenia przez konsumenta,
choć chwilowo może istnieć niespójność (np. przez kilkadziesiąt milisekund).
W kontekście systemu rezerwacji wizyt psychologicznych chwilowa niespójność
jest akceptowalna z perspektywy biznesowej — użytkownik nie odczuwa opóźnień
rzędu milisekund jako niespójności.

Elastyczność schematu MongoDB przynosi wymierną korzyść operacyjną: w monolicie
każda zmiana schematu danych wymagała przygotowania migracji Prisma, przetestowania
na kopii produkcyjnej i wdrożenia w oknie serwisowym. W architekturze dokumentowej
dodanie nowego pola do dokumentu (np. `yearsOfExperience` do `Psychologist`)
realizowane jest przez dodanie pola do klasy Java i opcjonalnego backfill skryptu
dla istniejących dokumentów — bez przestojów bazy i bez ryzyka blokady migracji DDL.
