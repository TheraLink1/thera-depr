# Plan: Notyfikacje email + Zoom + Format wizyty

## Narzędzia

### Email — SendGrid
**Dlaczego SendGrid:**
- Darmowy tier: 100 emaili/dzień (wystarczy na dev + małą produkcję)
- Oficjalny Java SDK (`sendgrid-java`)
- Dynamiczne szablony HTML w panelu sendgrid.com — możesz edytować wygląd emaili bez dotykania kodu
- Standard branżowy, prosta integracja ze Spring Boot

Alternatywa: Resend (nowszy, prostsze API, ale mniejsza społeczność).

### Zoom — Server-to-Server OAuth
**Dlaczego Server-to-Server OAuth (nie stary JWT):**
- Zoom wycofał aplikacje JWT w 2023 roku
- Server-to-Server OAuth = aplikacja działa w imieniu konta Zoom (bez interakcji użytkownika)
- Potrzebujesz jednej aplikacji w Zoom Marketplace — `Account ID`, `Client ID`, `Client Secret`

---

## Kafka — nowe topiki

| Topik | Producent | Konsumenci | Kiedy |
|---|---|---|---|
| `theralink.payment.completed` | payment-service ✓ | appointment-service, notification-service | po udanej płatności |
| `theralink.appointment.confirmed` | appointment-service | notification-service | po stworzeniu Zoom meetingu |
| `theralink.appointment.reminder` | appointment-service (scheduler) | notification-service | dzień przed wizytą |
| `theralink.payment.failed` | payment-service ✓ | notification-service | po nieudanej płatności |

---

## Nowe repozytoria do stworzenia

| Repo | Port |
|---|---|
| `theralink-appointment-service` | 8082 |
| `theralink-notification-service` | 8084 |

---

## SERWIS 1: thera-rest-service (Psychologist — format wizyty)

### Zadania

**1. Dodaj enum `SessionFormat`**
```
src/main/java/.../model/SessionFormat.java
```
Wartości: `REMOTE`, `IN_PERSON`, `BOTH`

**2. Zaktualizuj model `Psychologist`**
Nowe pole:
```java
private SessionFormat sessionFormat;  // domyślnie: IN_PERSON
```

**3. Zaktualizuj DTOs**
- `CreatePsychologistRequest` — dodaj `sessionFormat` z `@NotNull`
- `UpdatePsychologistRequest` — dodaj `sessionFormat` (opcjonalne)
- `PsychologistResponse` — dodaj `sessionFormat`

**4. Zaktualizuj mapper i serwis**
- `PsychologistMapper` — mapuj nowe pole
- `PsychologistService` — brak zmian w logice, mapper obsłuży

**5. Walidacja przy bookingu (opcjonalne, sprawdza appointment-service)**
- Jeśli psycholog ma `IN_PERSON` → nie można zarezerwować wizyty zdalnej
- Jeśli psycholog ma `REMOTE` → nie można zarezerwować wizyty stacjonarnej

---

## SERWIS 2: theralink-appointment-service (nowy)

### Struktura projektu
```
src/main/java/com/theralink/appointmentservice/
├── model/
│   ├── Appointment.java
│   ├── AppointmentStatus.java    (PENDING, PAID, CONFIRMED, CANCELLED)
│   └── SessionFormat.java        (REMOTE, IN_PERSON)
├── dto/
│   ├── request/CreateAppointmentRequest.java
│   └── response/AppointmentResponse.java
├── repository/AppointmentRepository.java
├── service/
│   ├── AppointmentService.java
│   └── ZoomService.java
├── kafka/
│   ├── PaymentEventConsumer.java     ← słucha theralink.payment.completed
│   └── AppointmentEventProducer.java ← publikuje confirmed + reminder
├── scheduler/AppointmentReminderScheduler.java
└── config/
    ├── SecurityConfig.java
    └── ZoomConfig.java
```

### Model `Appointment`
```java
@Document(collection = "appointments")
public class Appointment {
    @Id String id;
    String clientKeycloakId;
    String psychologistKeycloakId;
    LocalDateTime dateTime;         // data i godzina wizyty
    Integer durationMinutes;        // czas trwania (domyślnie 50)
    SessionFormat sessionFormat;    // REMOTE lub IN_PERSON
    String location;                // adres (gdy IN_PERSON)
    String zoomMeetingId;           // ID meetingu Zoom (gdy REMOTE)
    String zoomJoinUrl;             // link dla klienta
    String zoomStartUrl;            // link dla psychologa
    AppointmentStatus status;       // PENDING → PAID → CONFIRMED → CANCELLED
    String paymentId;               // ID płatności z payment-service
}
```

### Zadania

**1. Stwórz projekt Spring Boot**
- Zależności: web, mongodb, security, oauth2-resource-server, kafka, validation, actuator, lombok
- `application.yml`: port 8082, MONGODB_URI, KEYCLOAK_ISSUER_URI, KAFKA_BOOTSTRAP_SERVERS, zmienne Zoom

**2. Zoom — integracja Server-to-Server OAuth**

`ZoomConfig.java` — konfiguracja beana z tokenem:
```
ZOOM_ACCOUNT_ID, ZOOM_CLIENT_ID, ZOOM_CLIENT_SECRET → zmienne środowiskowe
```

`ZoomService.java` — metody:
- `createMeeting(appointmentId, psychologistName, dateTime, durationMinutes)` → zwraca `ZoomMeetingResult`
- `deleteMeeting(meetingId)` — do anulowania wizyty

Endpoint Zoom API: `POST https://api.zoom.us/v2/users/me/meetings`

Token OAuth: `POST https://zoom.us/oauth/token?grant_type=account_credentials&account_id={}`

**3. Kafka Consumer — `PaymentEventConsumer`**
Słucha: `theralink.payment.completed`

Logika po odebraniu eventu:
1. Znajdź wizytę po `appointmentId`
2. Zmień status: `PENDING → PAID`
3. Jeśli `sessionFormat == REMOTE` → wywołaj `ZoomService.createMeeting()`
4. Zmień status: `PAID → CONFIRMED`
5. Opublikuj `theralink.appointment.confirmed` (z linkami Zoom jeśli zdalna)

**4. Kafka Producer — `AppointmentEventProducer`**

Event `theralink.appointment.confirmed`:
```json
{
  "appointmentId": "...",
  "clientKeycloakId": "...",
  "psychologistKeycloakId": "...",
  "dateTime": "2026-04-25T14:00:00",
  "sessionFormat": "REMOTE",
  "zoomJoinUrl": "https://zoom.us/j/...",
  "location": null
}
```

Event `theralink.appointment.reminder`:
```json
{
  "appointmentId": "...",
  "clientKeycloakId": "...",
  "psychologistKeycloakId": "...",
  "dateTime": "2026-04-25T14:00:00",
  "sessionFormat": "REMOTE",
  "zoomJoinUrl": "https://zoom.us/j/...",
  "location": null
}
```

**5. Scheduler — `AppointmentReminderScheduler`**
```java
@Scheduled(cron = "0 0 10 * * *")  // Każdego dnia o 10:00
public void sendDayBeforeReminders() {
    // Znajdź wizyty zaplanowane na jutro ze statusem CONFIRMED
    // Dla każdej opublikuj theralink.appointment.reminder
}
```

**6. REST API — `AppointmentController`**
- `POST /appointments` — stwórz wizytę (status: PENDING)
- `GET /appointments/{id}` — szczegóły wizyty
- `GET /appointments/my` — wizyty zalogowanego użytkownika
- `GET /appointments/psychologist/{keycloakId}` — wizyty psychologa
- `DELETE /appointments/{id}` — anuluj wizytę (+ usuń Zoom meeting)

---

## SERWIS 3: theralink-notification-service (nowy)

### Struktura projektu
```
src/main/java/com/theralink/notificationservice/
├── kafka/
│   ├── PaymentEventConsumer.java        ← theralink.payment.completed
│   └── AppointmentEventConsumer.java    ← theralink.appointment.confirmed
│                                           theralink.appointment.reminder
│                                           theralink.payment.failed
├── service/
│   └── EmailService.java                ← SendGrid
├── model/EmailLog.java                  ← zapis wysłanych emaili (MongoDB)
└── config/SendGridConfig.java
```

**WAŻNE:** notification-service to **Kafka consumer only** — brak REST API, brak portu publicznego.

### Zadania

**1. Stwórz projekt Spring Boot**
- Zależności: mongodb, kafka, actuator, lombok + `sendgrid-java` (Maven)
- Brak: web, security (nie ma HTTP endpointów)

**2. SendGrid — konfiguracja**

`pom.xml`:
```xml
<dependency>
    <groupId>com.sendgrid</groupId>
    <artifactId>sendgrid-java</artifactId>
    <version>4.10.2</version>
</dependency>
```

`application.yml`:
```yaml
sendgrid:
  api-key: ${SENDGRID_API_KEY}
  from-email: noreply@theralink.pl
  from-name: TheraLink
```

**3. EmailService — metody**
- `sendPaymentConfirmation(clientEmail, appointmentDetails)` → szablon "Płatność potwierdzona"
- `sendPaymentFailed(clientEmail)` → szablon "Płatność nieudana"
- `sendAppointmentConfirmation(clientEmail, psychologistEmail, appointmentDetails)` → do obu stron
- `sendAppointmentReminder(clientEmail, psychologistEmail, appointmentDetails)` → do obu stron

Każdy email wysyłany przez SendGrid Dynamic Templates (ID szablonu z env vars) lub inline HTML.

**4. Kafka Consumers**

`PaymentEventConsumer` — słucha `theralink.payment.completed` i `theralink.payment.failed`:
- `PAYMENT_COMPLETED` → `sendPaymentConfirmation()`
- `PAYMENT_FAILED` → `sendPaymentFailed()`

`AppointmentEventConsumer` — słucha `theralink.appointment.confirmed` i `theralink.appointment.reminder`:
- `appointment.confirmed` → `sendAppointmentConfirmation()` — do klienta I psychologa
- `appointment.reminder` → `sendAppointmentReminder()` — do klienta I psychologa

**5. EmailLog — model MongoDB**
```java
@Document(collection = "email_logs")
public class EmailLog {
    String id;
    String recipientEmail;
    String emailType;           // PAYMENT_CONFIRMATION, REMINDER, etc.
    String relatedEntityId;     // appointmentId lub paymentId
    Instant sentAt;
    boolean success;
    String errorMessage;        // jeśli błąd SendGrid
}
```
Przydatne do debugowania i audytu (kto dostał jaki email i kiedy).

---

## Zmienne środowiskowe — nowe

### theralink-appointment-service
```env
ZOOM_ACCOUNT_ID=...
ZOOM_CLIENT_ID=...
ZOOM_CLIENT_SECRET=...
```

### theralink-notification-service
```env
SENDGRID_API_KEY=SG.xxx...
```

### Gdzie wpisać:
- `thera-docker-compose/.env` — do docker-compose.yml (gdy uruchomione w Docker)
- `application-local.yml` w każdym serwisie — do lokalnego dev w IDE

---

## Kolejność implementacji

```
1. thera-rest-service        — dodaj SessionFormat do Psychologist (30 min)
2. theralink-appointment-service — stwórz projekt + model + CRUD API (2-3h)
3. theralink-appointment-service — Zoom integration (1-2h)
4. theralink-appointment-service — Kafka consumer (PaymentCompleted) (1h)
5. theralink-appointment-service — Scheduler (reminders) (30 min)
6. theralink-notification-service — stwórz projekt + SendGrid (1-2h)
7. theralink-notification-service — Kafka consumers (1h)
8. Testy end-to-end: płatność → Zoom → email (1h)
```

---

## Powiązane dokumenty

- [[backend-migration]] — konwencje Spring Boot
- [[payment-service]] — payment-service (PCI-DSS)
- [[infrastructure]] — docker-compose + Azure AKS
