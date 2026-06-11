# TheraLink — Architektura systemu (diagramy)

> **Jak używać:**
> - **mermaid.live** → wklej kod bloku → Download PNG/SVG
> - **Miro** → Insert → Apps → Mermaid Diagrams → wklej kod
> - **Obsidian** → renderuje automatycznie

---

## 1. Widok całego systemu

```mermaid
graph TB
    BROWSER["🖥️ Angular 21\n:4200"]

    subgraph AUTH_GW["Auth & Gateway"]
        direction LR
        KEYCLOAK["🔐 Keycloak\n:8080"]
        GATEWAY["⚡ API Gateway\n:8090"]
    end

    subgraph SERVICES["Mikroserwisy"]
        direction LR
        USER["User Service\n:8081"]
        APPT["Appointment Service\n:8082"]
        PSYCH["Psychologist Service\n:8083"]
        PAY["💳 Payment Service\n:8085 ⚠️"]
    end

    KAFKA[["🔄 Apache Kafka\n:9092"]]

    subgraph DBS["🗄️ MongoDB — bazy danych"]
        direction LR
        DB1[("users")]
        DB2[("appointments")]
        DB3[("psychologists")]
        DB5[("payments")]
    end

    subgraph EXT["🌐 Zewnętrzne API"]
        direction LR
        STRIPE["💳 Stripe"]
        ZOOM["📹 Zoom"]
    end

    BROWSER -->|"HTTPS + JWT"| GATEWAY
    BROWSER <-->|"OIDC login"| KEYCLOAK
    GATEWAY -->|"weryfikuje JWT"| KEYCLOAK
    GATEWAY --> USER & APPT & PSYCH & PAY

    PAY -->|"payment.completed / failed"| KAFKA
    KAFKA --> APPT

    USER --- DB1
    APPT --- DB2
    PSYCH --- DB3
    PAY --- DB5

    PAY <-->|"REST"| STRIPE
    APPT <-->|"REST"| ZOOM

    style PAY fill:#ffcccc,stroke:#cc0000
    style KAFKA fill:#fff3cd,stroke:#856404
    style KEYCLOAK fill:#dce8f5,stroke:#0066cc
    style GATEWAY fill:#dce8f5,stroke:#0066cc
```

---

## 2. Kafka — przepływ eventów

```mermaid
graph LR
    subgraph PROD["📤 Producenci"]
        direction TB
        PAY["💳 Payment Service"]
    end

    subgraph KAFKA_BOX["🔄 Apache Kafka"]
        T1(["theralink.payment.completed"])
        T2(["theralink.payment.failed"])
    end

    subgraph CONS["📥 Konsumenci"]
        direction TB
        APPT_C["📅 Appointment Service"]
    end

    PAY --> T1 & T2

    T1 --> APPT_C
    T2 --> APPT_C

    style KAFKA_BOX fill:#fff3cd,stroke:#856404
    style PAY fill:#ffcccc,stroke:#cc0000
```

---

## 3. Przepływ: Rezerwacja + Płatność + Zoom

```mermaid
sequenceDiagram
    actor Klient
    participant Angular
    participant Gateway as API Gateway
    participant Appt as Appointment\nService
    participant Pay as Payment\nService
    participant Kafka
    participant Zoom as Zoom API

    Klient->>Angular: Wybiera wizytę
    Angular->>Gateway: POST /appointments
    Gateway->>Appt: utwórz wizytę
    Appt-->>Angular: { appointmentId, PENDING }

    Angular->>Gateway: POST /payments/intent
    Gateway->>Pay: utwórz PaymentIntent
    Pay-->>Angular: { clientSecret }

    Klient->>Angular: Podaje kartę (4242...)
    Angular-->>Pay: Stripe webhook:\npayment_intent.succeeded
    Pay->>Kafka: payment.completed

    Kafka->>Appt: payment.completed
    Appt->>Appt: PENDING → PAID

    alt Wizyta zdalna
        Appt->>Zoom: Utwórz meeting
        Zoom-->>Appt: { joinUrl, startUrl }
    end

    Appt->>Appt: PAID → CONFIRMED
```

---

## 4. Przepływ: Autoryzacja (Keycloak PKCE)

```mermaid
sequenceDiagram
    actor Użytkownik
    participant Angular
    participant Keycloak
    participant Gateway as API Gateway
    participant Service as Mikroserwis

    Użytkownik->>Angular: Klik "Zaloguj się"
    Angular->>Keycloak: Redirect + code_challenge (PKCE)
    Keycloak-->>Użytkownik: Strona logowania TheraLink
    Użytkownik->>Keycloak: Login + hasło
    Keycloak-->>Angular: Authorization Code
    Angular->>Keycloak: Wymień kod → tokeny
    Keycloak-->>Angular: Access Token (JWT 15min)\n+ Refresh Token (7 dni)

    Angular->>Gateway: Request +\nAuthorization: Bearer JWT
    Gateway->>Keycloak: Pobierz JWKS
    Gateway->>Gateway: Zweryfikuj podpis JWT
    Gateway->>Service: Przekaż request
    Service->>Service: jwt.getSubject()\n→ keycloakId
    Service-->>Angular: Response
```

---

## 5. Mapa repozytoriów i infrastruktury Azure

```mermaid
graph TB
    subgraph REPOS["📦 Repozytoria"]
        direction LR
        R1["thera-ui\nAngular :4200"]
        R2["thera-keycloak\nKeycloak :8080"]
        R3["api-gateway\n:8090"]
        R4["thera-rest-service\nUser :8081"]
        R5["appointment-service\n:8082"]
        R6["psychologist-service\n:8083"]
        R8["thera-payment-service\n⚠️ :8085"]
    end

    subgraph INFRA_REPOS["🛠️ Infrastruktura"]
        direction LR
        I1["thera-docker-compose\nDev environment"]
        I2["thera-infrastructure\nHelm + K8s"]
    end

    subgraph AZURE["☁️ Azure (Prod)"]
        direction LR
        ACR["Container Registry\nacrtheralink"]
        AKS["Kubernetes\nAKS"]
        COSMOS["Cosmos DB\nMongoDB API"]
        EVH["Event Hubs\nKafka protocol"]
        KV["Key Vault\nSekrety"]
    end

    REPOS --> ACR
    I2 --> AKS
    ACR --> AKS
    AKS --> COSMOS & EVH & KV

    style R8 fill:#ffcccc,stroke:#cc0000
    style AKS fill:#dce8f5,stroke:#0066cc
    style ACR fill:#dce8f5,stroke:#0066cc
```
