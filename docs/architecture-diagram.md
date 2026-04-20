# TheraLink — Architektura systemu (diagramy)

> **Jak używać:**
> - **Miro** → zainstaluj plugin "Mermaid Diagrams for Miro" → wklej kod bloku
> - **mermaid.live** → otwórz stronę, wklej kod → screenshot do prezentacji
> - **Obsidian** → renderuje automatycznie

---

## 1. Widok całego systemu

```mermaid
graph TB
    subgraph CLIENT["👤 Użytkownik"]
        BROWSER["Przeglądarka\n(Angular 21)"]
    end

    subgraph EXTERNAL["🌐 Zewnętrzne API"]
        STRIPE["💳 Stripe\nPłatności"]
        ZOOM["📹 Zoom API\nMeetingi"]
        SENDGRID["📧 SendGrid\nEmail"]
    end

    subgraph INFRA["🔐 Infrastruktura"]
        KEYCLOAK["Keycloak\n:8080\nAutoryzacja / SSO"]
        GATEWAY["API Gateway\n:8090\nSpring Cloud Gateway"]
        KAFKA["Apache Kafka\n:9092\nMessage Bus"]
    end

    subgraph SERVICES["⚙️ Mikroserwisy"]
        USER["User Service\n:8081\nKlienci + Psycholodzy"]
        APPT["Appointment Service\n:8082\nWizyty + Zoom"]
        PSYCH["Psychologist Service\n:8083\nSloty dostępności"]
        NOTIF["Notification Service\n:8084\nEmail — tylko Kafka"]
        PAY["Payment Service\n:8085\n⚠️ PCI-DSS"]
    end

    subgraph DATABASES["🗄️ Bazy danych (MongoDB)"]
        DB_USERS[("theralink-users")]
        DB_APPT[("theralink-appointments")]
        DB_PSYCH[("theralink-psychologists")]
        DB_NOTIF[("theralink-notifications")]
        DB_PAY[("theralink-payments")]
    end

    BROWSER -->|"HTTPS + JWT"| GATEWAY
    BROWSER -->|"OIDC login"| KEYCLOAK
    GATEWAY -->|"weryfikuje JWT"| KEYCLOAK
    GATEWAY --> USER
    GATEWAY --> APPT
    GATEWAY --> PSYCH
    GATEWAY --> PAY

    USER --- DB_USERS
    APPT --- DB_APPT
    PSYCH --- DB_PSYCH
    NOTIF --- DB_NOTIF
    PAY --- DB_PAY

    PAY -->|"theralink.payment.completed"| KAFKA
    PAY -->|"theralink.payment.failed"| KAFKA
    APPT -->|"theralink.appointment.confirmed"| KAFKA
    APPT -->|"theralink.appointment.reminder"| KAFKA

    KAFKA -->|"konsument"| APPT
    KAFKA -->|"konsument"| NOTIF

    PAY <-->|"REST API"| STRIPE
    APPT <-->|"REST API"| ZOOM
    NOTIF -->|"REST API"| SENDGRID

    style PAY fill:#ffcccc,stroke:#cc0000
    style KEYCLOAK fill:#e6f3ff,stroke:#0066cc
    style KAFKA fill:#fff3cd,stroke:#856404
    style GATEWAY fill:#e6f3ff,stroke:#0066cc
```

---

## 2. Przepływ: Rezerwacja wizyty + Płatność + Zoom + Email

```mermaid
sequenceDiagram
    actor Klient
    participant Angular
    participant Gateway as API Gateway
    participant Appt as Appointment Service
    participant Pay as Payment Service
    participant Kafka
    participant Notif as Notification Service
    participant Zoom as Zoom API
    participant SendGrid

    Klient->>Angular: Wybiera wizytę i psychologa
    Angular->>Gateway: POST /appointments
    Gateway->>Appt: utwórz wizytę (status: PENDING)
    Appt-->>Angular: { appointmentId, status: PENDING }

    Angular->>Gateway: POST /payments/intent
    Gateway->>Pay: utwórz PaymentIntent w Stripe
    Pay-->>Angular: { clientSecret }

    Angular->>Angular: Stripe.js — formularz karty
    Klient->>Angular: Wpisuje kartę testową 4242...
    Angular->>Pay: Stripe webhook: payment_intent.succeeded

    Pay->>Kafka: theralink.payment.completed
    Pay-->>Angular: { status: SUCCESS }

    Kafka->>Appt: konsumuje payment.completed
    Appt->>Appt: status: PENDING → PAID

    alt Wizyta zdalna (REMOTE)
        Appt->>Zoom: POST /v2/users/me/meetings
        Zoom-->>Appt: { joinUrl, startUrl, meetingId }
    end

    Appt->>Appt: status: PAID → CONFIRMED
    Appt->>Kafka: theralink.appointment.confirmed (+ zoomJoinUrl)

    Kafka->>Notif: konsumuje appointment.confirmed
    Notif->>SendGrid: email do Klienta (potwierdzenie + link Zoom)
    Notif->>SendGrid: email do Psychologa (nowa wizyta + link start)
    Notif->>Notif: zapisz EmailLog w MongoDB
```

---

## 3. Przepływ: Autoryzacja (Keycloak)

```mermaid
sequenceDiagram
    actor Użytkownik
    participant Angular
    participant Keycloak
    participant Gateway as API Gateway
    participant Service as Mikroserwis

    Użytkownik->>Angular: Klika "Zaloguj się"
    Angular->>Keycloak: Redirect (Authorization Code + PKCE)
    Keycloak-->>Użytkownik: Strona logowania TheraLink
    Użytkownik->>Keycloak: Login + hasło
    Keycloak-->>Angular: Authorization Code
    Angular->>Keycloak: Wymień kod na tokeny
    Keycloak-->>Angular: Access Token (JWT) + Refresh Token

    Note over Angular: Token przechowywany w pamięci (nie localStorage!)

    Angular->>Gateway: Request + Authorization: Bearer <JWT>
    Gateway->>Keycloak: Pobierz JWKS (klucze publiczne)
    Gateway->>Gateway: Weryfikuj podpis JWT
    Gateway->>Service: Request (JWT zweryfikowany)
    Service->>Service: @AuthenticationPrincipal Jwt jwt<br/>keycloakId = jwt.getSubject()
    Service-->>Angular: Response
```

---

## 4. Przepływ: Przypomnienie dzień przed wizytą

```mermaid
sequenceDiagram
    participant Scheduler as Appointment Service\n(Scheduler — cron 10:00)
    participant MongoDB
    participant Kafka
    participant Notif as Notification Service
    participant SendGrid

    Note over Scheduler: Każdego dnia o 10:00
    Scheduler->>MongoDB: Znajdź wizyty na jutro (status: CONFIRMED)
    MongoDB-->>Scheduler: Lista wizyt

    loop Dla każdej wizyty
        Scheduler->>Kafka: theralink.appointment.reminder
        Kafka->>Notif: konsumuje reminder
        Notif->>Notif: Sprawdź EmailLog — czy już wysłano?
        alt Email nie był wysłany
            Notif->>SendGrid: Email do Klienta (przypomnienie + link Zoom)
            Notif->>SendGrid: Email do Psychologa (przypomnienie)
            Notif->>MongoDB: Zapisz EmailLog
        end
    end
```

---

## 5. Mapa serwisów — porty i repozytoria

```mermaid
graph LR
    subgraph REPOS["Repozytoria (GitHub)"]
        R1["thera-ui\nAngular 21 / port 4200"]
        R2["thera-keycloak\nKeycloak 25 / port 8080"]
        R3["theralink-api-gateway\nSpring Cloud / port 8090"]
        R4["thera-rest-service\nUser Service / port 8081"]
        R5["theralink-appointment-service\nport 8082"]
        R6["theralink-psychologist-service\nport 8083"]
        R7["theralink-notification-service\nport 8084 (brak HTTP)"]
        R8["thera-payment-service\n⚠️ PCI-DSS / port 8085"]
        R9["thera-docker-compose\nDev environment"]
        R10["thera-infrastructure\nHelm + K8s manifests"]
    end

    subgraph AZURE["☁️ Azure (Prod)"]
        AKS["Azure Kubernetes Service"]
        ACR["Container Registry\nacrtheralink.azurecr.io"]
        COSMOS["Cosmos DB\nMongoDB API"]
        EVH["Event Hubs\nKafka protocol"]
        KV["Key Vault\nSekrety"]
    end

    R1 & R2 & R3 & R4 & R5 & R6 & R7 & R8 --> ACR
    ACR --> AKS
    AKS --> COSMOS
    AKS --> EVH
    AKS --> KV
```

---

## Jak wygenerować obraz do prezentacji

### Opcja A — mermaid.live (najszybsze)
1. Otwórz **https://mermaid.live**
2. Wklej wybrany blok kodu (np. diagram 1 lub 2)
3. Kliknij **Download PNG** lub **Download SVG**

### Opcja B — Miro
1. W Miro: **Insert → Apps → Mermaid Diagrams**
2. Wklej kod → kliknij **Insert**
3. Edytuj kolory i układ bezpośrednio na tablicy

### Opcja C — Obsidian
Otwórz ten plik w Obsidian — wszystkie diagramy renderują się automatycznie.
