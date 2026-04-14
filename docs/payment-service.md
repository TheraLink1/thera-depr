# theralink-payment-service — Przewodnik implementacji

> ⚠️ **RESTRICTED ACCESS / PCI-DSS**
> Ten dokument opisuje implementację **osobnego, prywatnego repozytorium** `theralink-payment-service`.
> Dostęp do repozytorium mają wyłącznie upoważnieni deweloperzy.
> **Nie wstawiaj żadnego kodu płatności do innych repozytoriów TheraLink.**

---

## Spis treści

1. [Kontekst i PCI-DSS](#1-kontekst-i-pci-dss)
2. [Teoria Stripe dla początkujących](#2-teoria-stripe-dla-początkujących)
3. [Struktura projektu](#3-struktura-projektu)
4. [pom.xml](#4-pomxml)
5. [application.yml](#5-applicationyml)
6. [Model Payment](#6-model-payment)
7. [DTOs](#7-dtos)
8. [Repository](#8-repository)
9. [StripeConfig](#9-stripeconfig)
10. [PaymentService](#10-paymentservice)
11. [PaymentController](#11-paymentcontroller)
12. [SecurityConfig](#12-securityconfig)
13. [Kafka — Producer i Consumer](#13-kafka--producer-i-consumer)
14. [Testowanie](#14-testowanie)

---

## 1. Kontekst i PCI-DSS

### Dlaczego payment service jest w osobnym repozytorium?

**PCI-DSS** (Payment Card Industry Data Security Standard) to zestaw wymagań bezpieczeństwa nałożonych przez firmy kartowe (Visa, Mastercard) na systemy przetwarzające dane płatnicze.

Kluczowe wymagania PCI-DSS wpływające na architekturę:

| Wymaganie PCI-DSS | Co to oznacza dla nas |
|---|---|
| Izolacja komponentów płatniczych | Osobne repo, osobny pipeline CI/CD, osobna baza MongoDB |
| Kontrola dostępu do kodu | Tylko upoważnieni deweloperzy mają dostęp do `theralink-payment-service` |
| Szyfrowanie danych wrażliwych | Klucze Stripe NIGDY w kodzie — tylko zmienne środowiskowe lub HashiCorp Vault |
| Audyt i monitoring | Osobny log, osobne alerty Sentry/DataDog |
| Minimalizacja zakresu | Inne serwisy NIE wiedzą nic o Stripe — komunikacja tylko przez Kafka |

### Jak payment-service komunikuje się z resztą systemu

```
[Angular]
    │
    │  POST /payments/intent  (JWT Bearer token)
    ▼
[API Gateway :8090]
    │
    ▼
[payment-service :8085]
    │
    ├──► [Stripe API] — tworzy PaymentIntent
    │         │
    │         │  (po sukcesie płatności)
    │         ▼
    │    [Stripe Webhook] ──► POST /payments/webhook (bez JWT)
    │                              │
    │                              ▼
    │                     [payment-service]
    │                              │
    ▼                              ▼
[MongoDB :27017]         [Kafka: theralink.payment.completed]
theralink-payments                │
                                  ▼
                    [notification-service] — wyślij email potwierdzenia
```

### Stripe zamiast własnego systemu płatności

Stripe (nie my) przechowuje dane kart. My **nigdy nie widzimy numeru karty** — Stripe.js na frontendzie tokenizuje kartę bezpośrednio, a my dostajemy tylko `paymentIntentId`.

---

## 2. Teoria Stripe dla początkujących

### PaymentIntent — centralny obiekt Stripe

`PaymentIntent` to obiekt Stripe reprezentujący zamiar zapłacenia określonej kwoty. Przepływ wygląda tak:

```
1. Backend (my) tworzy PaymentIntent w Stripe API
   → Stripe zwraca: { id: "pi_xxx", client_secret: "pi_xxx_secret_yyy" }

2. Backend zwraca client_secret do Angulara

3. Angular używa Stripe.js z client_secret do pokazania formularza karty
   → Stripe.js bezpośrednio komunikuje się ze Stripe (karta NIE przechodzi przez nasz backend!)

4. Po wpisaniu karty — Stripe przetwarza płatność

5. Stripe wywołuje nasz webhook: POST /payments/webhook
   → Sprawdzamy podpis (Stripe-Signature header) — to kluczowe zabezpieczenie!
   → Aktualizujemy status płatności w MongoDB
   → Publikujemy event Kafka
```

### Klucze Stripe

Stripe ma dwa środowiska:

| Klucz | Środowisko | Użycie |
|---|---|---|
| `sk_test_...` | Test (dev/staging) | Backend — nigdy nie pokazuj frontendowi! |
| `sk_live_...` | Produkcja | Backend — tylko prod environment |
| `pk_test_...` | Test | Frontend (Angular, Stripe.js) — możesz eksponować |
| `pk_live_...` | Produkcja | Frontend — możesz eksponować |
| Webhook secret | Oba | Weryfikacja podpisu webhooka |

> **Zasada:** Klucz `sk_` (secret key) NIGDY nie opuszcza backendu. Klucz `pk_` (publishable key) jest przeznaczony dla frontendu.

### Kwoty w groszach (najczęstszy błąd!)

Stripe operuje na **najmniejszej jednostce waluty**:
- PLN → grosze: `150 PLN = 15000`
- EUR → centy: `12.50 EUR = 1250`

```java
// BŁĄD:
long amount = 150; // Stripe potraktuje to jako 1,50 PLN!

// POPRAWNIE:
long amount = 15000; // 150,00 PLN
```

---

## 3. Struktura projektu

```
theralink-payment-service/         ← OSOBNE PRYWATNE REPO
├── src/
│   ├── main/
│   │   ├── java/com/theralink/paymentservice/
│   │   │   ├── PaymentServiceApplication.java
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.java      ← /webhook jest permitAll()
│   │   │   │   ├── StripeConfig.java        ← inicjalizuje Stripe.apiKey
│   │   │   │   └── KafkaConfig.java
│   │   │   ├── controller/
│   │   │   │   └── PaymentController.java
│   │   │   ├── service/
│   │   │   │   └── PaymentService.java
│   │   │   ├── repository/
│   │   │   │   └── PaymentRepository.java
│   │   │   ├── model/
│   │   │   │   ├── Payment.java
│   │   │   │   └── PaymentStatus.java
│   │   │   ├── dto/
│   │   │   │   ├── request/
│   │   │   │   │   └── CreatePaymentIntentRequest.java
│   │   │   │   └── response/
│   │   │   │       ├── PaymentIntentResponse.java
│   │   │   │       └── PaymentResponse.java
│   │   │   └── kafka/
│   │   │       ├── PaymentEventProducer.java
│   │   │       └── AppointmentEventConsumer.java
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       └── java/com/theralink/paymentservice/
│           ├── service/PaymentServiceTest.java
│           └── controller/PaymentControllerTest.java
└── pom.xml
```

---

## 4. pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.4</version>
        <relativePath/>
    </parent>

    <groupId>com.theralink</groupId>
    <artifactId>payment-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>theralink-payment-service</name>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <!-- Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- MongoDB — osobna baza dla płatności -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>

        <!-- Security + JWT (Keycloak) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
        </dependency>

        <!-- Walidacja DTO -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Stripe Java SDK -->
        <!-- Sprawdź najnowszą wersję: https://github.com/stripe/stripe-java/releases -->
        <dependency>
            <groupId>com.stripe</groupId>
            <artifactId>stripe-java</artifactId>
            <version>25.3.0</version>
        </dependency>

        <!-- Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Monitoring -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Testy -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>mongodb</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 5. application.yml

```yaml
# src/main/resources/application.yml
server:
  port: 8085

spring:
  application:
    name: theralink-payment-service

  data:
    mongodb:
      # OSOBNA baza MongoDB tylko dla płatności — nigdy nie dziel z innymi serwisami!
      uri: ${MONGODB_URI:mongodb://localhost:27017/theralink-payments}

  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER_URI:http://localhost:8080/realms/theralink}

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: theralink-payment-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"

# Klucze Stripe — ZAWSZE ze zmiennych środowiskowych, NIGDY hardcode!
# W produkcji: HashiCorp Vault lub Azure Key Vault
stripe:
  secret-key: ${STRIPE_SECRET_KEY}
  webhook-secret: ${STRIPE_WEBHOOK_SECRET}
  # Klucz publikowalny jest dla frontendu — tutaj tylko informacyjnie
  # publishable-key: ${STRIPE_PUBLISHABLE_KEY}

logging:
  level:
    com.theralink: DEBUG
    # Wyłącz na produkcji — może logować wrażliwe dane!
    com.stripe: WARN
```

---

## 6. Model Payment

**Teoria:** `PaymentStatus` to enum opisujący cykl życia płatności. Stripe może zwrócić wiele stanów — mapujemy je na nasze uproszczone statusy.

```java
// src/main/java/com/theralink/paymentservice/model/PaymentStatus.java
package com.theralink.paymentservice.model;

public enum PaymentStatus {
    PENDING,    // PaymentIntent stworzony, czekamy na płatność
    COMPLETED,  // Stripe webhook: payment_intent.succeeded
    FAILED,     // Stripe webhook: payment_intent.payment_failed
    REFUNDED    // Stripe webhook: charge.refunded
}
```

```java
// src/main/java/com/theralink/paymentservice/model/Payment.java
package com.theralink.paymentservice.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

import java.time.Instant;

@Document(collection = "payments")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Payment {

    @Id
    private String id;

    // ID wizyty z appointment-service (referencja przez String, nie JOIN)
    @Indexed(unique = true)
    private String appointmentId;

    // Keycloak ID klienta płacącego
    @Indexed
    private String clientKeycloakId;

    // ID obiektu PaymentIntent w Stripe (np. "pi_3QdXXX")
    // Używamy do identyfikacji webhooka i ewentualnych refundacji
    @Indexed(unique = true)
    private String stripePaymentIntentId;

    // Kwota w groszach! (np. 15000 = 150,00 PLN)
    private Long amount;

    // Kod waluty ISO 4217, małe litery (np. "pln", "eur")
    private String currency;

    private PaymentStatus status;

    // Czas stworzenia PaymentIntent (nie czas faktycznej płatności!)
    private Instant createdAt;

    // Czas potwierdzenia przez Stripe webhook (null do momentu sukcesu)
    private Instant paidAt;

    // Metadata — opcjonalne dodatkowe info
    private String description; // np. "Wizyta z Dr. Jan Kowalski"
}
```

---

## 7. DTOs

```java
// src/main/java/com/theralink/paymentservice/dto/request/CreatePaymentIntentRequest.java
package com.theralink.paymentservice.dto.request;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.Data;

@Data
public class CreatePaymentIntentRequest {

    @NotBlank(message = "appointmentId jest wymagany")
    private String appointmentId;

    // Kwota w GROSZACH — walidacja minimum 100 = 1,00 PLN
    @NotNull(message = "Kwota jest wymagana")
    @Min(value = 100, message = "Minimalna kwota to 1,00 PLN (100 groszy)")
    private Long amount;

    // Opcjonalny opis (np. "Wizyta z psychologiem")
    private String description;
}
```

```java
// src/main/java/com/theralink/paymentservice/dto/response/PaymentIntentResponse.java
package com.theralink.paymentservice.dto.response;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class PaymentIntentResponse {
    // client_secret — Angular przekazuje go do Stripe.js
    // Format: "pi_xxx_secret_yyy"
    // UWAGA: traktuj client_secret jak hasło — nie loguj go!
    private String clientSecret;

    // ID rekordu płatności w naszej bazie MongoDB
    private String paymentId;
}
```

```java
// src/main/java/com/theralink/paymentservice/dto/response/PaymentResponse.java
package com.theralink.paymentservice.dto.response;

import com.theralink.paymentservice.model.Payment;
import com.theralink.paymentservice.model.PaymentStatus;
import lombok.Builder;
import lombok.Data;

import java.time.Instant;

@Data
@Builder
public class PaymentResponse {
    private String id;
    private String appointmentId;
    private String clientKeycloakId;
    private Long amount;
    private String currency;
    private PaymentStatus status;
    private Instant createdAt;
    private Instant paidAt;
    private String description;

    // Nigdy nie zwracamy stripePaymentIntentId ani client_secret do frontendu po stworzeniu!
    // Zawierają wrażliwe informacje

    public static PaymentResponse from(Payment payment) {
        return PaymentResponse.builder()
            .id(payment.getId())
            .appointmentId(payment.getAppointmentId())
            .clientKeycloakId(payment.getClientKeycloakId())
            .amount(payment.getAmount())
            .currency(payment.getCurrency())
            .status(payment.getStatus())
            .createdAt(payment.getCreatedAt())
            .paidAt(payment.getPaidAt())
            .description(payment.getDescription())
            .build();
    }
}
```

---

## 8. Repository

```java
// src/main/java/com/theralink/paymentservice/repository/PaymentRepository.java
package com.theralink.paymentservice.repository;

import com.theralink.paymentservice.model.Payment;
import com.theralink.paymentservice.model.PaymentStatus;
import org.springframework.data.mongodb.repository.MongoRepository;

import java.util.List;
import java.util.Optional;

public interface PaymentRepository extends MongoRepository<Payment, String> {

    // Znajdź płatność dla danej wizyty
    Optional<Payment> findByAppointmentId(String appointmentId);

    // Wszystkie płatności klienta
    List<Payment> findByClientKeycloakId(String clientKeycloakId);

    // Po ID Stripe (używane w obsłudze webhooka)
    Optional<Payment> findByStripePaymentIntentId(String stripePaymentIntentId);

    // Płatności o danym statusie — przydatne do raportowania
    List<Payment> findByClientKeycloakIdAndStatus(String clientKeycloakId, PaymentStatus status);
}
```

---

## 9. StripeConfig

**Teoria:** `@PostConstruct` to adnotacja Javy (nie Spring) — metoda oznaczona `@PostConstruct` jest wywoływana automatycznie po tym, jak Spring stworzy bean i wstrzyknie wszystkie zależności. To idealne miejsce na inicjalizację bibliotek zewnętrznych takich jak Stripe.

```java
// src/main/java/com/theralink/paymentservice/config/StripeConfig.java
package com.theralink.paymentservice.config;

import com.stripe.Stripe;
import jakarta.annotation.PostConstruct;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

@Configuration
@Slf4j
public class StripeConfig {

    // @Value wczytuje wartość z application.yml → stripe.secret-key
    // Które z kolei bierze ją ze zmiennej środowiskowej STRIPE_SECRET_KEY
    @Value("${stripe.secret-key}")
    private String secretKey;

    @Value("${stripe.webhook-secret}")
    private String webhookSecret; // nie używamy tutaj, ale lepiej trzymać oba w jednym miejscu

    // @PostConstruct — wykona się raz, zaraz po stworzeniu beana StripeConfig
    // Podobne do: const stripe = new Stripe(process.env.STRIPE_SECRET_KEY) w Node.js
    @PostConstruct
    public void initStripe() {
        Stripe.apiKey = secretKey;
        // Ustawiamy wersję API Stripe — ważne dla stabilności
        // Stripe może zmieniać API między wersjami
        Stripe.apiVersion = "2024-06-20";
        log.info("Stripe SDK zainicjalizowany (środowisko: {})",
            secretKey.startsWith("sk_test_") ? "TEST" : "PRODUKCJA");
    }
}
```

---

## 10. PaymentService

```java
// src/main/java/com/theralink/paymentservice/service/PaymentService.java
package com.theralink.paymentservice.service;

import com.stripe.exception.SignatureVerificationException;
import com.stripe.exception.StripeException;
import com.stripe.model.Event;
import com.stripe.model.PaymentIntent;
import com.stripe.net.Webhook;
import com.stripe.param.PaymentIntentCreateParams;
import com.theralink.paymentservice.dto.request.CreatePaymentIntentRequest;
import com.theralink.paymentservice.dto.response.PaymentIntentResponse;
import com.theralink.paymentservice.dto.response.PaymentResponse;
import com.theralink.paymentservice.kafka.PaymentEventProducer;
import com.theralink.paymentservice.model.Payment;
import com.theralink.paymentservice.model.PaymentStatus;
import com.theralink.paymentservice.repository.PaymentRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.time.Instant;
import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class PaymentService {

    private final PaymentRepository paymentRepository;
    private final PaymentEventProducer eventProducer;

    // Webhook secret wstrzyknięty z application.yml
    @Value("${stripe.webhook-secret}")
    private String webhookSecret;

    // ─── TWORZENIE PŁATNOŚCI ──────────────────────────────────────────────────

    public PaymentIntentResponse createPaymentIntent(
        String clientKeycloakId,
        CreatePaymentIntentRequest request
    ) {
        // Sprawdź czy wizyta już ma płatność
        paymentRepository.findByAppointmentId(request.getAppointmentId())
            .ifPresent(existing -> {
                throw new PaymentAlreadyExistsException(
                    "Płatność dla wizyty " + request.getAppointmentId() + " już istnieje");
            });

        try {
            // 1. Stwórz PaymentIntent w Stripe
            PaymentIntentCreateParams params = PaymentIntentCreateParams.builder()
                .setAmount(request.getAmount())   // w groszach!
                .setCurrency("pln")
                // Stripe automatycznie dobiera metody płatności dla PLN
                .setAutomaticPaymentMethods(
                    PaymentIntentCreateParams.AutomaticPaymentMethods.builder()
                        .setEnabled(true)
                        .build()
                )
                // Metadata — pomoże nam zidentyfikować płatność w webhooku
                .putMetadata("appointmentId", request.getAppointmentId())
                .putMetadata("clientKeycloakId", clientKeycloakId)
                .setDescription(request.getDescription() != null
                    ? request.getDescription()
                    : "Wizyta TheraLink")
                .build();

            PaymentIntent intent = PaymentIntent.create(params);
            log.info("Stworzono PaymentIntent: {} dla wizyty: {}",
                intent.getId(), request.getAppointmentId());

            // 2. Zapisz rekord w MongoDB
            Payment payment = Payment.builder()
                .appointmentId(request.getAppointmentId())
                .clientKeycloakId(clientKeycloakId)
                .stripePaymentIntentId(intent.getId())
                .amount(request.getAmount())
                .currency("pln")
                .status(PaymentStatus.PENDING)
                .createdAt(Instant.now())
                .description(request.getDescription())
                .build();

            Payment saved = paymentRepository.save(payment);

            // 3. Zwróć client_secret do Angulara
            // Angular przekaże go do Stripe.js który obsłuży formularz karty
            return new PaymentIntentResponse(intent.getClientSecret(), saved.getId());

        } catch (StripeException e) {
            log.error("Błąd Stripe przy tworzeniu PaymentIntent: {}", e.getMessage());
            throw new PaymentCreationException("Błąd podczas inicjalizacji płatności: " + e.getMessage());
        }
    }

    // ─── WEBHOOK STRIPE ───────────────────────────────────────────────────────

    // Webhook to endpoint wywoływany przez Stripe po zakończeniu płatności.
    // Stripe wysyła HTTP POST z podpisanym payloadem.
    // Musimy ZAWSZE weryfikować podpis — inaczej każdy mógłby podszyć się pod Stripe!
    public void handleWebhook(String payload, String stripeSignatureHeader) {
        // Weryfikacja kryptograficzna podpisu
        // Stripe.Signature = HMAC-SHA256(payload, webhook_secret)
        Event event;
        try {
            event = Webhook.constructEvent(payload, stripeSignatureHeader, webhookSecret);
        } catch (SignatureVerificationException e) {
            // Ktoś próbuje podszyć się pod Stripe lub webhook secret jest błędny
            log.error("Nieprawidłowy podpis webhooka Stripe: {}", e.getMessage());
            throw new InvalidWebhookSignatureException("Nieprawidłowy podpis webhooka");
        }

        log.info("Otrzymano webhook Stripe: {} (ID: {})", event.getType(), event.getId());

        // Obsłuż konkretne typy zdarzeń
        switch (event.getType()) {
            case "payment_intent.succeeded" -> handlePaymentSucceeded(event);
            case "payment_intent.payment_failed" -> handlePaymentFailed(event);
            default -> log.debug("Nieobsługiwany typ webhooka: {}", event.getType());
        }
    }

    private void handlePaymentSucceeded(Event event) {
        // Wyciągnij ID PaymentIntent z eventu Stripe
        String paymentIntentId = extractPaymentIntentId(event);

        paymentRepository.findByStripePaymentIntentId(paymentIntentId)
            .ifPresentOrElse(
                payment -> {
                    // Zaktualizuj status w MongoDB
                    payment.setStatus(PaymentStatus.COMPLETED);
                    payment.setPaidAt(Instant.now());
                    paymentRepository.save(payment);

                    // Powiadom pozostałe serwisy przez Kafka
                    eventProducer.publishPaymentCompleted(payment);
                    log.info("Płatność {} zakończona sukcesem", payment.getId());
                },
                () -> log.warn("Nie znaleziono płatności dla PaymentIntent: {}", paymentIntentId)
            );
    }

    private void handlePaymentFailed(Event event) {
        String paymentIntentId = extractPaymentIntentId(event);

        paymentRepository.findByStripePaymentIntentId(paymentIntentId)
            .ifPresent(payment -> {
                payment.setStatus(PaymentStatus.FAILED);
                paymentRepository.save(payment);
                eventProducer.publishPaymentFailed(payment);
                log.warn("Płatność {} nie powiodła się", payment.getId());
            });
    }

    private String extractPaymentIntentId(Event event) {
        // Stripe pakuje obiekt PaymentIntent w event.data.object
        // Castujemy na PaymentIntent żeby dostać ID
        return event.getDataObjectDeserializer()
            .getObject()
            .map(obj -> ((PaymentIntent) obj).getId())
            .orElseThrow(() -> new RuntimeException("Nie można odczytać PaymentIntent z eventu"));
    }

    // ─── ODCZYT DANYCH ───────────────────────────────────────────────────────

    public PaymentResponse getPaymentForAppointment(String appointmentId) {
        return paymentRepository.findByAppointmentId(appointmentId)
            .map(PaymentResponse::from)
            .orElseThrow(() -> new PaymentNotFoundException("Płatność nie znaleziona dla wizyty: " + appointmentId));
    }

    public List<PaymentResponse> getPaymentsForClient(String clientKeycloakId) {
        return paymentRepository.findByClientKeycloakId(clientKeycloakId)
            .stream()
            .map(PaymentResponse::from)
            .collect(Collectors.toList());
    }

    // ─── WYJĄTKI ─────────────────────────────────────────────────────────────

    public static class PaymentNotFoundException extends RuntimeException {
        public PaymentNotFoundException(String msg) { super(msg); }
    }

    public static class PaymentAlreadyExistsException extends RuntimeException {
        public PaymentAlreadyExistsException(String msg) { super(msg); }
    }

    public static class PaymentCreationException extends RuntimeException {
        public PaymentCreationException(String msg) { super(msg); }
    }

    public static class InvalidWebhookSignatureException extends RuntimeException {
        public InvalidWebhookSignatureException(String msg) { super(msg); }
    }
}
```

---

## 11. PaymentController

```java
// src/main/java/com/theralink/paymentservice/controller/PaymentController.java
package com.theralink.paymentservice.controller;

import com.theralink.paymentservice.dto.request.CreatePaymentIntentRequest;
import com.theralink.paymentservice.dto.response.PaymentIntentResponse;
import com.theralink.paymentservice.dto.response.PaymentResponse;
import com.theralink.paymentservice.service.PaymentService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/payments")
@RequiredArgsConstructor
@Slf4j
public class PaymentController {

    private final PaymentService paymentService;

    // POST /payments/intent
    // Klient inicjuje płatność — dostaje client_secret do Stripe.js
    @PostMapping("/intent")
    @PreAuthorize("hasRole('CLIENT')")
    public ResponseEntity<PaymentIntentResponse> createPaymentIntent(
        @AuthenticationPrincipal Jwt jwt,
        @Valid @RequestBody CreatePaymentIntentRequest request
    ) {
        String clientKeycloakId = jwt.getSubject();
        PaymentIntentResponse response = paymentService.createPaymentIntent(clientKeycloakId, request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    // POST /payments/webhook
    // WAŻNE: Ten endpoint NIE wymaga JWT Bearer tokenu!
    // Stripe wywołuje go bezpośrednio — nie ma jak przesłać Keycloak token
    // Bezpieczeństwo zapewnia weryfikacja podpisu Stripe-Signature
    // SecurityConfig musi mieć: .requestMatchers("POST", "/payments/webhook").permitAll()
    @PostMapping("/webhook")
    public ResponseEntity<Map<String, String>> handleWebhook(
        // Pełny raw body jako String — KRYTYCZNE!
        // Stripe weryfikuje podpis na surowym body.
        // Jeśli Spring zdeserializuje JSON przed weryfikacją → podpis nie będzie się zgadzał!
        @RequestBody String payload,
        // Nagłówek Stripe-Signature zawiera HMAC-SHA256
        @RequestHeader("Stripe-Signature") String stripeSignature
    ) {
        try {
            paymentService.handleWebhook(payload, stripeSignature);
            // Stripe wymaga odpowiedzi HTTP 200 — inaczej ponowi żądanie!
            return ResponseEntity.ok(Map.of("status", "received"));
        } catch (PaymentService.InvalidWebhookSignatureException e) {
            return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(Map.of("error", "Invalid signature"));
        }
    }

    // GET /payments/appointment/{appointmentId}
    @GetMapping("/appointment/{appointmentId}")
    @PreAuthorize("hasAnyRole('CLIENT', 'PSYCHOLOGIST', 'ADMIN')")
    public ResponseEntity<PaymentResponse> getPaymentForAppointment(
        @PathVariable String appointmentId
    ) {
        return ResponseEntity.ok(paymentService.getPaymentForAppointment(appointmentId));
    }

    // GET /payments/me — historia płatności zalogowanego klienta
    @GetMapping("/me")
    @PreAuthorize("hasRole('CLIENT')")
    public ResponseEntity<List<PaymentResponse>> getMyPayments(
        @AuthenticationPrincipal Jwt jwt
    ) {
        return ResponseEntity.ok(paymentService.getPaymentsForClient(jwt.getSubject()));
    }
}
```

> ⚠️ **Ważne dla webhooka:** Spring Boot domyślnie czyta body żądania jako strumień — po odczytaniu przez `@RequestBody String` nie można go odczytać ponownie. Dlatego MUSI być `String payload` (nie `Object` ani `Map`) — Stripe SDK potrzebuje surowego stringa do weryfikacji podpisu.

---

## 12. SecurityConfig

**Teoria:** Endpoint `/payments/webhook` musi być `permitAll()` — Stripe nie ma tokenu Keycloak i nie może przesłać nagłówka `Authorization: Bearer`. Bezpieczeństwo tego endpointu zapewnia wyłącznie weryfikacja `Stripe-Signature` w serwisie.

```java
// src/main/java/com/theralink/paymentservice/config/SecurityConfig.java
package com.theralink.paymentservice.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.web.SecurityFilterChain;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                // KLUCZOWE: webhook musi być publiczny — Stripe nie ma JWT
                .requestMatchers("POST", "/payments/webhook").permitAll()
                // Actuator health check
                .requestMatchers("/actuator/health").permitAll()
                // Wszystko inne wymaga tokenu Keycloak
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 ->
                oauth2.jwt(jwt ->
                    jwt.jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
            if (realmAccess == null) return List.of();
            List<String> roles = (List<String>) realmAccess.get("roles");
            if (roles == null) return List.of();
            return roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                .collect(Collectors.toList());
        });
        return converter;
    }
}
```

---

## 13. Kafka — Producer i Consumer

### Producer — publikuje zdarzenia po zakończeniu płatności

```java
// src/main/java/com/theralink/paymentservice/kafka/PaymentEventProducer.java
package com.theralink.paymentservice.kafka;

import com.theralink.paymentservice.model.Payment;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

import java.time.Instant;
import java.util.Map;

@Component
@RequiredArgsConstructor
@Slf4j
public class PaymentEventProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    private static final String TOPIC_COMPLETED = "theralink.payment.completed";
    private static final String TOPIC_FAILED = "theralink.payment.failed";

    // Publikowane po otrzymaniu webhooka payment_intent.succeeded
    // Konsumenci: notification-service (wyślij email), appointment-service (oznacz wizytę jako opłaconą)
    public void publishPaymentCompleted(Payment payment) {
        Map<String, Object> event = Map.of(
            "paymentId", payment.getId(),
            "appointmentId", payment.getAppointmentId(),
            "clientKeycloakId", payment.getClientKeycloakId(),
            "amount", payment.getAmount(),
            "currency", payment.getCurrency(),
            "paidAt", payment.getPaidAt().toString(),
            "timestamp", Instant.now().toString()
        );
        kafkaTemplate.send(TOPIC_COMPLETED, payment.getAppointmentId(), event);
        log.info("Opublikowano payment.completed dla wizyty: {}", payment.getAppointmentId());
    }

    public void publishPaymentFailed(Payment payment) {
        Map<String, Object> event = Map.of(
            "paymentId", payment.getId(),
            "appointmentId", payment.getAppointmentId(),
            "clientKeycloakId", payment.getClientKeycloakId(),
            "timestamp", Instant.now().toString()
        );
        kafkaTemplate.send(TOPIC_FAILED, payment.getAppointmentId(), event);
        log.warn("Opublikowano payment.failed dla wizyty: {}", payment.getAppointmentId());
    }
}
```

### Consumer — subskrybuje nowe wizyty

```java
// src/main/java/com/theralink/paymentservice/kafka/AppointmentEventConsumer.java
package com.theralink.paymentservice.kafka;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

import java.util.Map;

// Payment service subskrybuje appointment.created żeby wiedzieć o nowych wizytach
// wymagających płatności — może np. wstępnie zarezerwować slot płatności
@Component
@RequiredArgsConstructor
@Slf4j
public class AppointmentEventConsumer {

    @KafkaListener(topics = "theralink.appointment.created")
    public void onAppointmentCreated(Map<String, Object> event) {
        String appointmentId = (String) event.get("appointmentId");
        String clientKeycloakId = (String) event.get("clientKeycloakId");

        log.info("Nowa wizyta wymagająca płatności: appointmentId={}, client={}",
            appointmentId, clientKeycloakId);

        // Tutaj możesz np.:
        // - Zarezerwować kwotę z portfela klienta (jeśli masz system prepaid)
        // - Wyliczyć kwotę na podstawie danych z psychologist-service
        // - Po prostu zalogować dla celów audytowych
        // W uproszczonej wersji: czekamy aż klient sam zainicjuje płatność przez POST /payments/intent
    }
}
```

---

## 14. Testowanie

### Test jednostkowy serwisu (z mockiem Stripe)

**Teoria:** Stripe SDK robi prawdziwe wywołania HTTP do `api.stripe.com`. W testach jednostkowych mockujemy te wywołania, żeby testy były szybkie i niezależne od sieci.

```java
// src/test/java/com/theralink/paymentservice/service/PaymentServiceTest.java
package com.theralink.paymentservice.service;

import com.stripe.exception.StripeException;
import com.stripe.model.PaymentIntent;
import com.theralink.paymentservice.dto.request.CreatePaymentIntentRequest;
import com.theralink.paymentservice.dto.response.PaymentIntentResponse;
import com.theralink.paymentservice.kafka.PaymentEventProducer;
import com.theralink.paymentservice.model.Payment;
import com.theralink.paymentservice.model.PaymentStatus;
import com.theralink.paymentservice.repository.PaymentRepository;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.MockedStatic;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock
    private PaymentRepository paymentRepository;

    @Mock
    private PaymentEventProducer eventProducer;

    @InjectMocks
    private PaymentService paymentService;

    @Test
    @DisplayName("createPaymentIntent — powinien zwrócić client_secret gdy płatność nie istnieje")
    void createPaymentIntent_success() throws StripeException {
        // Arrange
        CreatePaymentIntentRequest request = new CreatePaymentIntentRequest();
        request.setAppointmentId("apt-123");
        request.setAmount(15000L); // 150 PLN
        request.setDescription("Wizyta z psychologiem");

        // Brak istniejącej płatności
        when(paymentRepository.findByAppointmentId("apt-123"))
            .thenReturn(Optional.empty());

        // Mock zapisu
        Payment savedPayment = Payment.builder()
            .id("pay-456")
            .appointmentId("apt-123")
            .clientKeycloakId("kc-789")
            .stripePaymentIntentId("pi_test_xxx")
            .status(PaymentStatus.PENDING)
            .build();
        when(paymentRepository.save(any())).thenReturn(savedPayment);

        // Mock Stripe SDK — klasa statyczna wymaga MockedStatic
        PaymentIntent mockIntent = mock(PaymentIntent.class);
        when(mockIntent.getId()).thenReturn("pi_test_xxx");
        when(mockIntent.getClientSecret()).thenReturn("pi_test_xxx_secret_yyy");

        // MockedStatic pozwala mockować metody statyczne (PaymentIntent.create jest statyczna)
        try (MockedStatic<PaymentIntent> stripeStatic = mockStatic(PaymentIntent.class)) {
            stripeStatic.when(() -> PaymentIntent.create(any())).thenReturn(mockIntent);

            // Act
            PaymentIntentResponse result = paymentService.createPaymentIntent("kc-789", request);

            // Assert
            assertThat(result.getClientSecret()).isEqualTo("pi_test_xxx_secret_yyy");
            assertThat(result.getPaymentId()).isEqualTo("pay-456");
            verify(paymentRepository).save(any());
        }
    }

    @Test
    @DisplayName("createPaymentIntent — powinien rzucić wyjątek gdy płatność już istnieje")
    void createPaymentIntent_alreadyExists() {
        // Arrange
        CreatePaymentIntentRequest request = new CreatePaymentIntentRequest();
        request.setAppointmentId("apt-exists");
        request.setAmount(10000L);

        Payment existingPayment = Payment.builder().id("pay-already").build();
        when(paymentRepository.findByAppointmentId("apt-exists"))
            .thenReturn(Optional.of(existingPayment));

        // Act + Assert
        assertThatThrownBy(() -> paymentService.createPaymentIntent("kc-123", request))
            .isInstanceOf(PaymentService.PaymentAlreadyExistsException.class);

        // Stripe NIE powinien być wywołany
        verifyNoInteractions(eventProducer);
    }
}
```

### Test integracyjny repozytorium z Testcontainers

```java
// src/test/java/com/theralink/paymentservice/repository/PaymentRepositoryIntegrationTest.java
package com.theralink.paymentservice.repository;

import com.theralink.paymentservice.model.Payment;
import com.theralink.paymentservice.model.PaymentStatus;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.data.mongo.DataMongoTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.time.Instant;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

@DataMongoTest
@Testcontainers
class PaymentRepositoryIntegrationTest {

    @Container
    static MongoDBContainer mongodb = new MongoDBContainer("mongo:7.0");

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongodb::getReplicaSetUrl);
    }

    @Autowired
    private PaymentRepository paymentRepository;

    @AfterEach
    void cleanup() {
        paymentRepository.deleteAll();
    }

    @Test
    void shouldFindByAppointmentId() {
        Payment payment = Payment.builder()
            .appointmentId("apt-test-001")
            .clientKeycloakId("kc-client-001")
            .stripePaymentIntentId("pi_test_001")
            .amount(15000L)
            .currency("pln")
            .status(PaymentStatus.PENDING)
            .createdAt(Instant.now())
            .build();

        paymentRepository.save(payment);

        Optional<Payment> found = paymentRepository.findByAppointmentId("apt-test-001");
        assertThat(found).isPresent();
        assertThat(found.get().getAmount()).isEqualTo(15000L);
        assertThat(found.get().getStatus()).isEqualTo(PaymentStatus.PENDING);
    }

    @Test
    void shouldFindByStripePaymentIntentId() {
        Payment payment = Payment.builder()
            .appointmentId("apt-test-002")
            .clientKeycloakId("kc-client-001")
            .stripePaymentIntentId("pi_test_unique_id")
            .amount(10000L)
            .currency("pln")
            .status(PaymentStatus.PENDING)
            .createdAt(Instant.now())
            .build();

        paymentRepository.save(payment);

        Optional<Payment> found = paymentRepository.findByStripePaymentIntentId("pi_test_unique_id");
        assertThat(found).isPresent();
        assertThat(found.get().getAppointmentId()).isEqualTo("apt-test-002");
    }
}
```

### Test kontrolera (HTTP layer)

```java
// src/test/java/com/theralink/paymentservice/controller/PaymentControllerTest.java
package com.theralink.paymentservice.controller;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.theralink.paymentservice.dto.request.CreatePaymentIntentRequest;
import com.theralink.paymentservice.dto.response.PaymentIntentResponse;
import com.theralink.paymentservice.service.PaymentService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.when;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.jwt;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(PaymentController.class)
class PaymentControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private PaymentService paymentService;

    @Test
    void createPaymentIntent_shouldReturn201() throws Exception {
        CreatePaymentIntentRequest request = new CreatePaymentIntentRequest();
        request.setAppointmentId("apt-111");
        request.setAmount(15000L);

        PaymentIntentResponse mockResponse = new PaymentIntentResponse(
            "pi_test_secret_xxx", "pay-222");

        when(paymentService.createPaymentIntent(eq("kc-subject-id"), any()))
            .thenReturn(mockResponse);

        // jwt() buduje mock tokenu JWT z custom subject — symuluje zalogowanego klienta
        mockMvc.perform(
                post("/payments/intent")
                    .with(jwt().jwt(j -> j.subject("kc-subject-id")))
                    .with(csrf())
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request))
            )
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.clientSecret").value("pi_test_secret_xxx"))
            .andExpect(jsonPath("$.paymentId").value("pay-222"));
    }

    @Test
    void webhook_shouldReturn200_whenSignatureValid() throws Exception {
        String rawPayload = "{\"type\":\"payment_intent.succeeded\"}";

        // Webhook endpoint jest permitAll() — nie potrzebuje tokenu JWT
        mockMvc.perform(
                post("/payments/webhook")
                    .contentType(MediaType.APPLICATION_JSON)
                    .header("Stripe-Signature", "t=xxx,v1=yyy")
                    .content(rawPayload)
            )
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.status").value("received"));
    }
}
```

---

## Uruchamianie serwisu

```bash
# Wymagane zmienne środowiskowe (nigdy nie hardcode!)
export STRIPE_SECRET_KEY="sk_test_TWOJ_KLUCZ_TEST"
export STRIPE_WEBHOOK_SECRET="whsec_TWOJ_WEBHOOK_SECRET"
export MONGODB_URI="mongodb://localhost:27017/theralink-payments"
export KEYCLOAK_ISSUER_URI="http://localhost:8080/realms/theralink"
export KAFKA_BOOTSTRAP_SERVERS="localhost:9092"

# Build
mvn clean package -DskipTests

# Start
java -jar target/payment-service-0.0.1-SNAPSHOT.jar

# Health check
curl http://localhost:8085/actuator/health

# Test lokalny Stripe webhooka (wymaga Stripe CLI)
# https://stripe.com/docs/webhooks/test
stripe listen --forward-to localhost:8085/payments/webhook
```
