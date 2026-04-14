# TheraLink — Backend Migration Guide
## Node.js/Express → Spring Boot Microservices

> **Dla kogo jest ten dokument?**
> Jesteś nowy w Spring Boot, ale masz doświadczenie z Node.js/Express. Każda sekcja zaczyna się od teorii — wyjaśniam *dlaczego* coś istnieje w Spring, a potem pokazuję *jak* to zaimplementować. Czytaj od góry do dołu.

---

## Spis treści

1. [Audyt obecnego backendu](#1-audyt-obecnego-backendu)
2. [Teoria Spring Boot dla programistów Node.js](#2-teoria-spring-boot-dla-programistów-nodejs)
3. [Wymagania i narzędzia](#3-wymagania-i-narzędzia)
4. [theralink-user-service](#4-theralink-user-service)
5. [theralink-appointment-service](#5-theralink-appointment-service)
6. [theralink-psychologist-service](#6-theralink-psychologist-service)
7. [theralink-notification-service](#7-theralink-notification-service)
8. [theralink-payment-service](#8-theralink-payment-service--oddzielne-repozytorium)
9. [Migracja danych PostgreSQL → MongoDB](#9-migracja-danych-postgresql--mongodb)
10. [Testowanie: JUnit 5 + Mockito + Testcontainers](#10-testowanie-junit-5--mockito--testcontainers)

---

## 1. Audyt obecnego backendu

### Mapa obecnych tras (Express)

Plik wejściowy: `backend/src/index.ts`

```
POST   /clients                         → clientController.createClient
GET    /clients/:cognitoId              → clientController.getClient

GET    /psychologists                   → psychologistController.getAllPsychologists
POST   /psychologists                   → psychologistController.createPsychologist
GET    /psychologists/:cognitoId        → psychologistController.getPsychologist
PUT    /psychologists/:cognitoId        → psychologistController.updatePsychologist

POST   /appointments/:cognitoId         → appointmentController.createAppointment
GET    /appointments/singleApt/:id      → appointmentController.getAppointment
GET    /appointments/psychologistApts/:cognitoId → appointmentController.getAllForPsychologist
GET    /appointments/clientApts/:cognitoId       → appointmentController.getAllForClient
PUT    /appointments/:id                → appointmentController.updateAppointment

POST   /availabilities/:psychologistId  → availabilityController.createAvailability (AUTH: psychologist)
GET    /availabilities/psychologist/:id → availabilityController.getForPsychologist (AUTH: psychologist/client)
GET    /availabilities/:id              → availabilityController.getAvailability    (AUTH: psychologist/client)
PUT    /availabilities/:id              → availabilityController.updateAvailability (AUTH: psychologist)

POST   /sync-role                       → syncRoleController
GET    /api/chat                        → chatRouter (OpenRouter — poza zakresem migracji)
```

### Krytyczne problemy bezpieczeństwa

**Problem 1 — `backend/src/middleware/authMiddleware.ts` linia 30:**
```typescript
// BŁĄD: jwt.decode() NIE weryfikuje podpisu!
// Każdy może podrobić token i uzyskać dostęp do API.
const decoded = jwt.decode(token) as DecodedToken;
```

`jwt.decode()` tylko dekoduje payload Base64 — nie sprawdza, czy token jest poprawnie podpisany przez AWS Cognito. Poprawna funkcja to `jwt.verify(token, publicKey)`, ale w Spring Boot to robi Spring Security automatycznie przez JWKS (JSON Web Key Set) z Keycloak — nie musisz nic pisać ręcznie.

**Problem 2 — brak transakcji:** Żadna operacja nie używa transakcji bazodanowych. W MongoDB Spring Data zapewnia wsparcie dla transakcji (replica set).

**Problem 3 — encje eksponowane bezpośrednio:** Kontrolery zwracają obiekty Prisma bezpośrednio. W Spring Boot stosujemy DTOs.

### Mapa migracji: serwis → trasy

| Serwis Spring Boot | Przejmuje trasy Express | Port |
|---|---|---|
| `theralink-user-service` | `/clients`, `/psychologists`, `/sync-role` | 8081 |
| `theralink-appointment-service` | `/appointments` | 8082 |
| `theralink-psychologist-service` | `/availabilities` | 8083 |
| `theralink-notification-service` | Konsument Kafka (brak HTTP) | 8084 |
| `theralink-payment-service` | Stripe (oddzielne repo) | 8085 |

---

## 2. Teoria Spring Boot dla programistów Node.js

### 2.1 Dependency Injection — odpowiednik `require()` na sterydach

W Node.js importujesz zależności ręcznie:
```typescript
// Node.js
import { PrismaClient } from "@prisma/client";
const prisma = new PrismaClient(); // TY tworzysz obiekt
```

W Spring Boot framework tworzy obiekty za Ciebie i wstrzykuje je tam gdzie potrzeba. Mechanizm nazywa się **Dependency Injection (DI)** i jest zarządzany przez **Spring IoC Container** (Inversion of Control).

```java
// Spring Boot — adnotacja @Service mówi Springowi: "zarządzaj tym obiektem"
@Service
public class ClientService {
    private final ClientRepository repository; // Spring wstrzykuje to automatycznie

    // Konstruktor = wstrzyknięcie przez konstruktor (zalecane)
    public ClientService(ClientRepository repository) {
        this.repository = repository;
    }
}
```

**Kluczowe adnotacje DI:**
| Adnotacja | Znaczenie | Odpowiednik Node.js |
|---|---|---|
| `@Component` | Generyczny bean zarządzany przez Spring | Eksportowana klasa |
| `@Service` | Bean warstwy logiki biznesowej | Plik serwisu (np. `userService.ts`) |
| `@Repository` | Bean dostępu do danych | Funkcje Prisma |
| `@RestController` | Bean obsługi żądań HTTP | Plik kontrolera Express |
| `@Configuration` | Bean konfiguracji | Plik `config.ts` |

### 2.2 Annotations — adnotacje zamiast middleware

Express używa middleware — funkcji pośredniczących. Spring Boot używa adnotacji na metodach i klasach.

```typescript
// Express
router.get("/psychologists/:cognitoId", authMiddleware(["psychologist"]), getPsychologist);
```

```java
// Spring Boot
@GetMapping("/psychologists/{keycloakId}")
@PreAuthorize("hasRole('PSYCHOLOGIST')")  // autoryzacja jako adnotacja!
public ResponseEntity<PsychologistResponse> getPsychologist(@PathVariable String keycloakId) {
    // logika
}
```

### 2.3 Spring MVC — architektura warstw

Spring Boot wymusza wyraźny podział na warstwy, którego Express nie narzuca:

```
HTTP Request
    ↓
Controller (@RestController)   ← "Route handler" z Express
    ↓
Service (@Service)             ← Logika biznesowa
    ↓
Repository (@Repository)       ← Prisma / ORM
    ↓
MongoDB
```

### 2.4 Spring Security + Keycloak

W Express ręcznie dekodowałeś JWT (i to źle, bo nie weryfikowałeś podpisu). Spring Security automatycznie:
1. Pobiera klucze publiczne z Keycloak JWKS endpoint (`/realms/theralink/protocol/openid-connect/certs`)
2. Weryfikuje podpis każdego tokenu
3. Wypakowuje role i udostępnia je przez `@AuthenticationPrincipal Jwt jwt`

```java
// W kontrolerze — Spring automatycznie wypełnia obiekt jwt z tokenu Bearer
@GetMapping("/me")
public ResponseEntity<ClientResponse> getMe(@AuthenticationPrincipal Jwt jwt) {
    String keycloakId = jwt.getSubject(); // odpowiednik decoded.sub
    // ...
}
```

### 2.5 Spring Data MongoDB — odpowiednik Prisma

```typescript
// Prisma
const client = await prisma.client.findUnique({ where: { cognitoId } });
```

```java
// Spring Data MongoDB
Optional<Client> client = clientRepository.findByKeycloakId(keycloakId);
// Spring automatycznie generuje zapytanie z nazwy metody!
```

Spring Data analizuje nazwę metody `findByKeycloakId` i automatycznie generuje zapytanie MongoDB. Nie musisz pisać żadnego SQL ani kodu zapytania.

### 2.6 application.yml — odpowiednik .env

```typescript
// Node.js .env
DATABASE_URL="postgresql://..."
PORT=3000
```

```yaml
# Spring Boot application.yml
server:
  port: 8081

spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}  # wartość z zmiennej środowiskowej
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER_URI}
```

### 2.7 Lombok — eliminacja boilerplate Java

Java wymaga pisania getterów, setterów, konstruktorów. Lombok generuje je automatycznie z adnotacji:

```java
@Data           // generuje gettery, settery, equals, hashCode, toString
@Builder        // wzorzec budowniczy: Client.builder().name("Jan").build()
@NoArgsConstructor  // konstruktor bez argumentów
@AllArgsConstructor // konstruktor ze wszystkimi polami
public class ClientResponse {
    private String id;
    private String name;
    private String email;
}
```

---

## 3. Wymagania i narzędzia

### 3.1 Instalacja

```bash
# Java 21 (LTS) — wymagane przez Spring Boot 3.x
# macOS z Homebrew:
brew install openjdk@21
echo 'export JAVA_HOME=/opt/homebrew/opt/openjdk@21' >> ~/.zshrc

# Weryfikacja
java --version  # powinno pokazać: openjdk 21.x.x

# Maven (jeśli nie masz)
brew install maven
mvn --version
```

### 3.2 Generowanie projektu — Spring Initializr

Każdy serwis generujesz na https://start.spring.io z następującymi ustawieniami:

| Pole | Wartość |
|---|---|
| Project | Maven |
| Language | Java |
| Spring Boot | 3.3.x (najnowsza stabilna) |
| Packaging | Jar |
| Java | 21 |

**Zależności dla user-service:**
- Spring Web
- Spring Data MongoDB
- Spring Security
- OAuth2 Resource Server
- Lombok
- Spring for Apache Kafka
- Validation
- Actuator

### 3.3 Struktura katalogów każdego serwisu

```
theralink-user-service/
├── src/
│   ├── main/
│   │   ├── java/com/theralink/userservice/
│   │   │   ├── UserServiceApplication.java      ← punkt wejścia (jak index.ts)
│   │   │   ├── config/
│   │   │   │   ├── SecurityConfig.java           ← zamiast authMiddleware.ts
│   │   │   │   └── KafkaConfig.java
│   │   │   ├── controller/
│   │   │   │   ├── ClientController.java         ← zamiast clientController.ts
│   │   │   │   └── PsychologistController.java
│   │   │   ├── service/
│   │   │   │   ├── ClientService.java
│   │   │   │   └── PsychologistService.java
│   │   │   ├── repository/
│   │   │   │   ├── ClientRepository.java         ← zamiast prisma.client.*
│   │   │   │   └── PsychologistRepository.java
│   │   │   ├── model/
│   │   │   │   ├── Client.java                   ← zamiast Prisma model
│   │   │   │   └── Psychologist.java
│   │   │   ├── dto/
│   │   │   │   ├── request/
│   │   │   │   │   ├── CreateClientRequest.java
│   │   │   │   │   └── UpdateClientRequest.java
│   │   │   │   └── response/
│   │   │   │       └── ClientResponse.java
│   │   │   └── kafka/
│   │   │       └── UserEventProducer.java
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       └── java/com/theralink/userservice/
│           └── service/ClientServiceTest.java
└── pom.xml
```

---

## 4. theralink-user-service

### Cel
Przejmuje trasy Express: `POST /clients`, `GET /clients/:id`, `GET/POST/PUT /psychologists`

### 4.1 pom.xml

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
    <artifactId>user-service</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>theralink-user-service</name>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <!-- Web (odpowiednik express) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- MongoDB (zamiast Prisma + PostgreSQL) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>

        <!-- Spring Security + OAuth2/JWT (zamiast jwt.decode) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
        </dependency>

        <!-- Walidacja DTO (@NotBlank, @Email itp.) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>

        <!-- Lombok (eliminuje boilerplate) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Monitoring (opcjonalne, ale dobre) -->
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

### 4.2 application.yml

```yaml
# src/main/resources/application.yml
server:
  port: 8081

spring:
  application:
    name: theralink-user-service

  data:
    mongodb:
      uri: ${MONGODB_URI:mongodb://localhost:27017/theralink-users}
      # Zmienna środowiskowa MONGODB_URI; jeśli nie ustawiona, użyje localhost

  security:
    oauth2:
      resourceserver:
        jwt:
          # Spring automatycznie pobierze klucze publiczne z Keycloak
          # i będzie weryfikował każdy token Bearer
          issuer-uri: ${KEYCLOAK_ISSUER_URI:http://localhost:8080/realms/theralink}

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

logging:
  level:
    com.theralink: DEBUG
    org.springframework.security: DEBUG  # usuń na produkcji
```

### 4.3 Punkt wejścia

```java
// src/main/java/com/theralink/userservice/UserServiceApplication.java
package com.theralink.userservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

// @SpringBootApplication = @Configuration + @EnableAutoConfiguration + @ComponentScan
// Odpowiednik: app.listen(port) z index.ts, ale Spring konfiguruje wszystko automatycznie
@SpringBootApplication
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

### 4.4 SecurityConfig — zastępuje authMiddleware.ts

**Teoria:** `SecurityConfig` to centralne miejsce konfiguracji bezpieczeństwa. Spring Security automatycznie:
- Weryfikuje podpis JWT przez JWKS z Keycloak
- Blokuje żądania bez tokenu
- Mapuje role Keycloak na uprawnienia Spring Security
- Udostępnia dane z tokenu przez `@AuthenticationPrincipal Jwt jwt`

```java
// src/main/java/com/theralink/userservice/config/SecurityConfig.java
package com.theralink.userservice.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.web.SecurityFilterChain;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Configuration
@EnableWebSecurity
// @EnableMethodSecurity pozwala używać @PreAuthorize na metodach kontrolera
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Wyłącz CSRF — API REST nie używa sesji przeglądarkowych
            .csrf(csrf -> csrf.disable())
            // Wyłącz sesje HTTP — JWT jest stateless
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            // Autoryzacja żądań
            .authorizeHttpRequests(auth -> auth
                // Zezwól na publiczną rejestrację
                .requestMatchers("POST", "/clients").permitAll()
                .requestMatchers("POST", "/psychologists").permitAll()
                // Actuator health check — publiczny
                .requestMatchers("/actuator/health").permitAll()
                // Reszta wymaga tokenu
                .anyRequest().authenticated()
            )
            // Skonfiguruj serwer zasobów OAuth2 (weryfikuje JWT przez Keycloak JWKS)
            .oauth2ResourceServer(oauth2 ->
                oauth2.jwt(jwt ->
                    jwt.jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );

        return http.build();
    }

    // Konwertuje role Keycloak na role Spring Security
    // Keycloak przechowuje role w: token.realm_access.roles[]
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            // Wyciągamy realm_access.roles z tokenu Keycloak
            Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
            if (realmAccess == null) return List.of();

            List<String> roles = (List<String>) realmAccess.get("roles");
            if (roles == null) return List.of();

            // Mapujemy np. "psychologist" → "ROLE_PSYCHOLOGIST"
            // Spring Security wymaga prefiksu ROLE_
            return roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                .collect(Collectors.toList());
        });
        return converter;
    }
}
```

### 4.5 Model Client — zastępuje Prisma model

**Teoria:** `@Document` to odpowiednik Prisma `model Client`. Oznacza że klasa jest dokumentem MongoDB. W odróżnieniu od PostgreSQL, MongoDB nie ma fixed schema — pola mogą być opcjonalne.

```java
// src/main/java/com/theralink/userservice/model/Client.java
package com.theralink.userservice.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

// @Document(collection = "clients") — odpowiednik: model Client {} w Prisma
// Mówi Spring Data: "ta klasa to dokument w kolekcji 'clients'"
@Document(collection = "clients")
@Data               // Lombok: gettery + settery + equals + hashCode + toString
@Builder            // Lombok: ClientBuilder dla wzorca budowniczego
@NoArgsConstructor  // Lombok: konstruktor bezargumentowy (wymagany przez MongoDB)
@AllArgsConstructor // Lombok: konstruktor ze wszystkimi polami
public class Client {

    // @Id — odpowiednik @id @default(autoincrement()) w Prisma
    // MongoDB generuje ObjectId automatycznie (np. "507f1f77bcf86cd799439011")
    @Id
    private String id;

    // @Indexed(unique = true) — odpowiednik @unique w Prisma
    // keycloakId zastępuje cognitoId (migracja z AWS Cognito na Keycloak)
    @Indexed(unique = true)
    private String keycloakId;

    private String name;
    private String email;
    private String phoneNumber;
    private String history; // opcjonalne — pole może nie istnieć w dokumencie
}
```

### 4.6 Model Psychologist

```java
// src/main/java/com/theralink/userservice/model/Psychologist.java
package com.theralink.userservice.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

@Document(collection = "psychologists")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
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
    private String description;
    private String specialization;
    private Integer yearsOfExperience;
}
```

### 4.7 DTOs — żądania i odpowiedzi

**Teoria:** DTO (Data Transfer Object) to klasa pośrednia. Kontroler przyjmuje DTO zamiast encji `@Document`. Dzięki temu:
- Możesz walidować dane wejściowe adnotacjami (`@NotBlank`, `@Email`)
- Nie eksponujesz wewnętrznej struktury bazy danych
- Możesz dodawać/ukrywać pola bez zmiany encji

```java
// src/main/java/com/theralink/userservice/dto/request/CreateClientRequest.java
package com.theralink.userservice.dto.request;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import lombok.Data;

@Data
public class CreateClientRequest {

    // @NotBlank — pole nie może być null ani pusty string
    // Odpowiednik: if (!cognitoId) { res.status(400)... } w Express
    @NotBlank(message = "keycloakId jest wymagany")
    private String keycloakId;

    @NotBlank(message = "Imię jest wymagane")
    private String name;

    @NotBlank(message = "Email jest wymagany")
    @Email(message = "Email musi być poprawnym adresem")
    private String email;

    @NotBlank(message = "Numer telefonu jest wymagany")
    private String phoneNumber;
}
```

```java
// src/main/java/com/theralink/userservice/dto/request/UpdateClientRequest.java
package com.theralink.userservice.dto.request;

import jakarta.validation.constraints.Email;
import lombok.Data;

@Data
public class UpdateClientRequest {
    @Email(message = "Email musi być poprawnym adresem")
    private String email;      // opcjonalne — może być null
    private String phoneNumber; // opcjonalne
}
```

```java
// src/main/java/com/theralink/userservice/dto/response/ClientResponse.java
package com.theralink.userservice.dto.response;

import com.theralink.userservice.model.Client;
import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class ClientResponse {
    private String id;
    private String keycloakId;
    private String name;
    private String email;
    private String phoneNumber;
    private String history;

    // Metoda fabryczna — konwertuje encję na DTO
    // Wywołanie: ClientResponse.from(client)
    public static ClientResponse from(Client client) {
        return ClientResponse.builder()
            .id(client.getId())
            .keycloakId(client.getKeycloakId())
            .name(client.getName())
            .email(client.getEmail())
            .phoneNumber(client.getPhoneNumber())
            .history(client.getHistory())
            .build();
    }
}
```

```java
// src/main/java/com/theralink/userservice/dto/request/CreatePsychologistRequest.java
package com.theralink.userservice.dto.request;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import lombok.Data;

@Data
public class CreatePsychologistRequest {

    @NotBlank(message = "keycloakId jest wymagany")
    private String keycloakId;

    @NotBlank(message = "Email jest wymagany")
    @Email
    private String email;

    private String name = "";
    private String phoneNumber = "";
    private String location = "";

    @Min(value = 0, message = "Stawka godzinowa nie może być ujemna")
    private Integer hourlyRate = 0;

    private String description = "";
    private String specialization = "";
    private Integer yearsOfExperience = 0;
}
```

```java
// src/main/java/com/theralink/userservice/dto/response/PsychologistResponse.java
package com.theralink.userservice.dto.response;

import com.theralink.userservice.model.Psychologist;
import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class PsychologistResponse {
    private String id;
    private String keycloakId;
    private String name;
    private String email;
    private String phoneNumber;
    private String location;
    private Integer hourlyRate;
    private String description;
    private String specialization;
    private Integer yearsOfExperience;

    public static PsychologistResponse from(Psychologist p) {
        return PsychologistResponse.builder()
            .id(p.getId())
            .keycloakId(p.getKeycloakId())
            .name(p.getName())
            .email(p.getEmail())
            .phoneNumber(p.getPhoneNumber())
            .location(p.getLocation())
            .hourlyRate(p.getHourlyRate())
            .description(p.getDescription())
            .specialization(p.getSpecialization())
            .yearsOfExperience(p.getYearsOfExperience())
            .build();
    }
}
```

### 4.8 Repository — zastępuje prisma.client.*

**Teoria:** `MongoRepository` to interfejs. Spring Data automatycznie generuje implementację w czasie uruchamiania aplikacji. Nie musisz pisać żadnego kodu — tylko deklarujesz metody.

Konwencja nazewnictwa:
- `findByKeycloakId(id)` → `db.clients.findOne({ keycloakId: id })`
- `findByEmail(email)` → `db.clients.findOne({ email: email })`
- `existsByKeycloakId(id)` → `db.clients.countDocuments({ keycloakId: id }) > 0`

```java
// src/main/java/com/theralink/userservice/repository/ClientRepository.java
package com.theralink.userservice.repository;

import com.theralink.userservice.model.Client;
import org.springframework.data.mongodb.repository.MongoRepository;

import java.util.Optional;

// MongoRepository<Client, String>
//   - Client = typ encji
//   - String = typ klucza głównego (@Id)
// Spring generuje implementację automatycznie!
public interface ClientRepository extends MongoRepository<Client, String> {

    // Spring analizuje nazwę: findBy + KeycloakId
    // Generuje zapytanie: { keycloakId: ?0 }
    // Odpowiednik Prisma: prisma.client.findUnique({ where: { cognitoId } })
    Optional<Client> findByKeycloakId(String keycloakId);

    // Odpowiednik: prisma.client.findUnique + checking != null
    boolean existsByKeycloakId(String keycloakId);
}
```

```java
// src/main/java/com/theralink/userservice/repository/PsychologistRepository.java
package com.theralink.userservice.repository;

import com.theralink.userservice.model.Psychologist;
import org.springframework.data.mongodb.repository.MongoRepository;

import java.util.Optional;

public interface PsychologistRepository extends MongoRepository<Psychologist, String> {
    Optional<Psychologist> findByKeycloakId(String keycloakId);
    boolean existsByKeycloakId(String keycloakId);
}
```

### 4.9 Serwisy — logika biznesowa

**Teoria:** Warstwa serwisu zawiera logikę biznesową. Kontroler NIE powinien mieć logiki — tylko przyjmuje żądanie i wywołuje serwis. Serwis jest testowalny niezależnie od warstwy HTTP.

```java
// src/main/java/com/theralink/userservice/service/ClientService.java
package com.theralink.userservice.service;

import com.theralink.userservice.dto.request.CreateClientRequest;
import com.theralink.userservice.dto.request.UpdateClientRequest;
import com.theralink.userservice.dto.response.ClientResponse;
import com.theralink.userservice.model.Client;
import com.theralink.userservice.repository.ClientRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

// @RequiredArgsConstructor — Lombok generuje konstruktor z wszystkimi final polami
// Spring widzi jeden konstruktor i automatycznie wstrzykuje zależności
@Service
@RequiredArgsConstructor
public class ClientService {

    private final ClientRepository clientRepository;

    // Odpowiednik: createClient() z clientController.ts
    public ClientResponse createClient(CreateClientRequest request) {
        // Budujemy encję z DTO
        Client client = Client.builder()
            .keycloakId(request.getKeycloakId())
            .name(request.getName())
            .email(request.getEmail())
            .phoneNumber(request.getPhoneNumber())
            .build();

        // .save() = INSERT (jeśli nowy) lub UPDATE (jeśli ma @Id)
        // Odpowiednik Prisma: prisma.client.create({ data: { ... } })
        Client saved = clientRepository.save(client);

        // Zawsze zwracamy DTO, nigdy encję!
        return ClientResponse.from(saved);
    }

    // Odpowiednik: getClient() z clientController.ts
    public ClientResponse getClientByKeycloakId(String keycloakId) {
        // orElseThrow = jeśli Optional pusty, rzuć wyjątek
        // Spring domyślnie konwertuje RuntimeException na HTTP 500
        // Ale lepiej mieć własny wyjątek (patrz niżej)
        Client client = clientRepository.findByKeycloakId(keycloakId)
            .orElseThrow(() -> new ClientNotFoundException("Klient nie znaleziony: " + keycloakId));

        return ClientResponse.from(client);
    }

    // Odpowiednik: updateClient() z clientController.ts
    public ClientResponse updateClient(String keycloakId, UpdateClientRequest request) {
        Client client = clientRepository.findByKeycloakId(keycloakId)
            .orElseThrow(() -> new ClientNotFoundException("Klient nie znaleziony: " + keycloakId));

        // Aktualizuj tylko pola, które zostały przekazane (nie null)
        if (request.getEmail() != null) {
            client.setEmail(request.getEmail());
        }
        if (request.getPhoneNumber() != null) {
            client.setPhoneNumber(request.getPhoneNumber());
        }

        Client updated = clientRepository.save(client);
        return ClientResponse.from(updated);
    }

    // Pomocnicza klasa wyjątku zagnieżdżona w serwisie
    // Możesz ją też wyciągnąć do osobnego pliku
    public static class ClientNotFoundException extends RuntimeException {
        public ClientNotFoundException(String message) {
            super(message);
        }
    }
}
```

```java
// src/main/java/com/theralink/userservice/service/PsychologistService.java
package com.theralink.userservice.service;

import com.theralink.userservice.dto.request.CreatePsychologistRequest;
import com.theralink.userservice.dto.request.UpdatePsychologistRequest;
import com.theralink.userservice.dto.response.PsychologistResponse;
import com.theralink.userservice.model.Psychologist;
import com.theralink.userservice.repository.PsychologistRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class PsychologistService {

    private final PsychologistRepository psychologistRepository;

    public PsychologistResponse createPsychologist(CreatePsychologistRequest request) {
        if (psychologistRepository.existsByKeycloakId(request.getKeycloakId())) {
            throw new PsychologistAlreadyExistsException("Psycholog już istnieje");
        }

        Psychologist psychologist = Psychologist.builder()
            .keycloakId(request.getKeycloakId())
            .name(request.getName())
            .email(request.getEmail())
            .phoneNumber(request.getPhoneNumber())
            .location(request.getLocation())
            .hourlyRate(request.getHourlyRate())
            .description(request.getDescription())
            .specialization(request.getSpecialization())
            .yearsOfExperience(request.getYearsOfExperience())
            .build();

        return PsychologistResponse.from(psychologistRepository.save(psychologist));
    }

    public PsychologistResponse getPsychologistByKeycloakId(String keycloakId) {
        return psychologistRepository.findByKeycloakId(keycloakId)
            .map(PsychologistResponse::from)
            .orElseThrow(() -> new PsychologistNotFoundException("Psycholog nie znaleziony"));
    }

    // Odpowiednik: prisma.psychologist.findMany()
    public List<PsychologistResponse> getAllPsychologists() {
        return psychologistRepository.findAll()
            .stream()
            .map(PsychologistResponse::from)
            .collect(Collectors.toList());
    }

    public PsychologistResponse updatePsychologist(String keycloakId, UpdatePsychologistRequest request) {
        Psychologist p = psychologistRepository.findByKeycloakId(keycloakId)
            .orElseThrow(() -> new PsychologistNotFoundException("Psycholog nie znaleziony"));

        if (request.getName() != null) p.setName(request.getName());
        if (request.getEmail() != null) p.setEmail(request.getEmail());
        if (request.getPhoneNumber() != null) p.setPhoneNumber(request.getPhoneNumber());
        if (request.getLocation() != null) p.setLocation(request.getLocation());
        if (request.getHourlyRate() != null) p.setHourlyRate(request.getHourlyRate());
        if (request.getDescription() != null) p.setDescription(request.getDescription());
        if (request.getSpecialization() != null) p.setSpecialization(request.getSpecialization());

        return PsychologistResponse.from(psychologistRepository.save(p));
    }

    public static class PsychologistNotFoundException extends RuntimeException {
        public PsychologistNotFoundException(String msg) { super(msg); }
    }

    public static class PsychologistAlreadyExistsException extends RuntimeException {
        public PsychologistAlreadyExistsException(String msg) { super(msg); }
    }
}
```

### 4.10 Global Exception Handler

**Teoria:** `@ControllerAdvice` to interceptor dla wyjątków — odpowiednik globalnego error handlera w Express. Zamiast `try/catch` w każdym serwisie, możemy rzucać wyjątki i tutaj je łapać, zwracając właściwe kody HTTP.

```java
// src/main/java/com/theralink/userservice/config/GlobalExceptionHandler.java
package com.theralink.userservice.config;

import com.theralink.userservice.service.ClientService;
import com.theralink.userservice.service.PsychologistService;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.Map;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // Łapie błędy walidacji (@Valid na DTO)
    // Odpowiednik: res.status(400).json({ message: "Missing required fields" })
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(MethodArgumentNotValidException ex) {
        String errors = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));

        return ResponseEntity.badRequest().body(Map.of("message", errors));
    }

    @ExceptionHandler(ClientService.ClientNotFoundException.class)
    public ResponseEntity<Map<String, String>> handleClientNotFound(ClientService.ClientNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(Map.of("message", ex.getMessage()));
    }

    @ExceptionHandler(PsychologistService.PsychologistNotFoundException.class)
    public ResponseEntity<Map<String, String>> handlePsychNotFound(PsychologistService.PsychologistNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(Map.of("message", ex.getMessage()));
    }

    @ExceptionHandler(PsychologistService.PsychologistAlreadyExistsException.class)
    public ResponseEntity<Map<String, String>> handlePsychExists(PsychologistService.PsychologistAlreadyExistsException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT).body(Map.of("message", ex.getMessage()));
    }
}
```

### 4.11 Kontrolery

**Teoria:** `@RestController` to połączenie `@Controller` (obsługa HTTP) i `@ResponseBody` (automatyczna serializacja do JSON). Metody zwracają `ResponseEntity<T>` — odpowiednik `res.status(201).json(data)`.

```java
// src/main/java/com/theralink/userservice/controller/ClientController.java
package com.theralink.userservice.controller;

import com.theralink.userservice.dto.request.CreateClientRequest;
import com.theralink.userservice.dto.request.UpdateClientRequest;
import com.theralink.userservice.dto.response.ClientResponse;
import com.theralink.userservice.service.ClientService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

// @RequestMapping("/clients") — odpowiednik: app.use("/clients", userRoutes)
@RestController
@RequestMapping("/clients")
@RequiredArgsConstructor
public class ClientController {

    private final ClientService clientService;

    // POST /clients
    // Odpowiednik Express:
    //   router.post("/", createClient)
    //   const { cognitoId, name, email, phoneNumber } = req.body;
    //
    // @Valid uruchamia walidację CreateClientRequest przed wywołaniem metody
    // Jeśli walidacja się nie powiedzie → GlobalExceptionHandler.handleValidation()
    @PostMapping
    public ResponseEntity<ClientResponse> createClient(@Valid @RequestBody CreateClientRequest request) {
        ClientResponse response = clientService.createClient(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    // GET /clients/{keycloakId}
    // @PathVariable — odpowiednik req.params.cognitoId
    @GetMapping("/{keycloakId}")
    @PreAuthorize("hasAnyRole('CLIENT', 'PSYCHOLOGIST', 'ADMIN')")
    public ResponseEntity<ClientResponse> getClient(@PathVariable String keycloakId) {
        return ResponseEntity.ok(clientService.getClientByKeycloakId(keycloakId));
    }

    // GET /clients/me — pobierz profil zalogowanego klienta na podstawie JWT
    @GetMapping("/me")
    @PreAuthorize("hasRole('CLIENT')")
    public ResponseEntity<ClientResponse> getMe(@AuthenticationPrincipal Jwt jwt) {
        // jwt.getSubject() = sub z tokenu Keycloak = keycloakId
        // Odpowiednik: decoded.sub z authMiddleware.ts
        String keycloakId = jwt.getSubject();
        return ResponseEntity.ok(clientService.getClientByKeycloakId(keycloakId));
    }

    // PUT /clients/{keycloakId}
    @PutMapping("/{keycloakId}")
    @PreAuthorize("hasAnyRole('CLIENT', 'ADMIN')")
    public ResponseEntity<ClientResponse> updateClient(
        @PathVariable String keycloakId,
        @Valid @RequestBody UpdateClientRequest request
    ) {
        return ResponseEntity.ok(clientService.updateClient(keycloakId, request));
    }
}
```

```java
// src/main/java/com/theralink/userservice/controller/PsychologistController.java
package com.theralink.userservice.controller;

import com.theralink.userservice.dto.request.CreatePsychologistRequest;
import com.theralink.userservice.dto.request.UpdatePsychologistRequest;
import com.theralink.userservice.dto.response.PsychologistResponse;
import com.theralink.userservice.service.PsychologistService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/psychologists")
@RequiredArgsConstructor
public class PsychologistController {

    private final PsychologistService psychologistService;

    // GET /psychologists — odpowiednik: prisma.psychologist.findMany()
    @GetMapping
    public ResponseEntity<List<PsychologistResponse>> getAllPsychologists() {
        return ResponseEntity.ok(psychologistService.getAllPsychologists());
    }

    // POST /psychologists
    @PostMapping
    public ResponseEntity<PsychologistResponse> createPsychologist(
        @Valid @RequestBody CreatePsychologistRequest request
    ) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(psychologistService.createPsychologist(request));
    }

    // GET /psychologists/{keycloakId}
    @GetMapping("/{keycloakId}")
    public ResponseEntity<PsychologistResponse> getPsychologist(@PathVariable String keycloakId) {
        return ResponseEntity.ok(psychologistService.getPsychologistByKeycloakId(keycloakId));
    }

    // PUT /psychologists/{keycloakId}
    @PutMapping("/{keycloakId}")
    @PreAuthorize("hasAnyRole('PSYCHOLOGIST', 'ADMIN')")
    public ResponseEntity<PsychologistResponse> updatePsychologist(
        @PathVariable String keycloakId,
        @Valid @RequestBody UpdatePsychologistRequest request
    ) {
        return ResponseEntity.ok(psychologistService.updatePsychologist(keycloakId, request));
    }
}
```

### 4.12 Kafka Event Producer

**Teoria:** Po zapisaniu klienta/psychologa do MongoDB, publikujemy zdarzenie Kafka. Inne serwisy (np. notification-service) mogą subskrybować ten topic bez bezpośredniego wywołania user-service. To jest luźne sprzężenie (loose coupling).

```java
// src/main/java/com/theralink/userservice/kafka/UserEventProducer.java
package com.theralink.userservice.kafka;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

import java.time.Instant;
import java.util.Map;

// @Slf4j — Lombok generuje logger: log.info(), log.error() itp.
@Component
@RequiredArgsConstructor
@Slf4j
public class UserEventProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    // Topic format: theralink.{domain}.{event}
    private static final String TOPIC_CLIENT_REGISTERED = "theralink.user.client-registered";
    private static final String TOPIC_PSYCHOLOGIST_REGISTERED = "theralink.user.psychologist-registered";

    public void publishClientRegistered(String keycloakId, String email) {
        Map<String, Object> event = Map.of(
            "keycloakId", keycloakId,
            "email", email,
            "timestamp", Instant.now().toString()
        );
        kafkaTemplate.send(TOPIC_CLIENT_REGISTERED, keycloakId, event);
        log.info("Opublikowano zdarzenie client-registered dla: {}", keycloakId);
    }

    public void publishPsychologistRegistered(String keycloakId, String email) {
        Map<String, Object> event = Map.of(
            "keycloakId", keycloakId,
            "email", email,
            "timestamp", Instant.now().toString()
        );
        kafkaTemplate.send(TOPIC_PSYCHOLOGIST_REGISTERED, keycloakId, event);
        log.info("Opublikowano zdarzenie psychologist-registered dla: {}", keycloakId);
    }
}
```

---

## 5. theralink-appointment-service

### Cel
Przejmuje trasy Express: `POST/GET/PUT /appointments`

Port: `8082`

### 5.1 application.yml

```yaml
server:
  port: 8082

spring:
  application:
    name: theralink-appointment-service
  data:
    mongodb:
      uri: ${MONGODB_URI:mongodb://localhost:27017/theralink-appointments}
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
```

### 5.2 Model Appointment

```java
// src/main/java/com/theralink/appointmentservice/model/Appointment.java
package com.theralink.appointmentservice.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

import java.time.Instant;

@Document(collection = "appointments")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Appointment {

    @Id
    private String id;

    @Indexed
    private String clientKeycloakId;   // był: clientCognitoId

    @Indexed
    private String psychologistKeycloakId; // był: psychologistId (cognitoId)

    private String meetingLink;
    private Instant date;              // Instant zamiast DateTime (lepsza obsługa UTC)
    private AppointmentStatus status;  // był: Status (ApplicationStatus)
}
```

```java
// src/main/java/com/theralink/appointmentservice/model/AppointmentStatus.java
package com.theralink.appointmentservice.model;

// Odpowiednik enum ApplicationStatus { Pending Denied Approved } z Prisma
public enum AppointmentStatus {
    PENDING,
    DENIED,
    APPROVED
}
```

### 5.3 DTOs

```java
// src/main/java/com/theralink/appointmentservice/dto/request/CreateAppointmentRequest.java
package com.theralink.appointmentservice.dto.request;

import jakarta.validation.constraints.Future;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.Data;

import java.time.Instant;

@Data
public class CreateAppointmentRequest {

    @NotBlank(message = "psychologistKeycloakId jest wymagany")
    private String psychologistKeycloakId;

    @NotNull(message = "Data wizyty jest wymagana")
    @Future(message = "Data wizyty musi być w przyszłości")
    private Instant date;

    private String meetingLink = "https://meet.theralink.pl/pending";
}
```

```java
// src/main/java/com/theralink/appointmentservice/dto/response/AppointmentResponse.java
package com.theralink.appointmentservice.dto.response;

import com.theralink.appointmentservice.model.Appointment;
import com.theralink.appointmentservice.model.AppointmentStatus;
import lombok.Builder;
import lombok.Data;

import java.time.Instant;

@Data
@Builder
public class AppointmentResponse {
    private String id;
    private String clientKeycloakId;
    private String psychologistKeycloakId;
    private String meetingLink;
    private Instant date;
    private AppointmentStatus status;

    public static AppointmentResponse from(Appointment a) {
        return AppointmentResponse.builder()
            .id(a.getId())
            .clientKeycloakId(a.getClientKeycloakId())
            .psychologistKeycloakId(a.getPsychologistKeycloakId())
            .meetingLink(a.getMeetingLink())
            .date(a.getDate())
            .status(a.getStatus())
            .build();
    }
}
```

### 5.4 Repository

```java
// src/main/java/com/theralink/appointmentservice/repository/AppointmentRepository.java
package com.theralink.appointmentservice.repository;

import com.theralink.appointmentservice.model.Appointment;
import org.springframework.data.mongodb.repository.MongoRepository;

import java.util.List;

public interface AppointmentRepository extends MongoRepository<Appointment, String> {
    // Odpowiednik: prisma.appointment.findMany({ where: { psychologistId: cognitoId } })
    List<Appointment> findByPsychologistKeycloakId(String psychologistKeycloakId);

    // Odpowiednik: prisma.appointment.findMany({ where: { clientCognitoId: cognitoId } })
    List<Appointment> findByClientKeycloakId(String clientKeycloakId);
}
```

### 5.5 Serwis

```java
// src/main/java/com/theralink/appointmentservice/service/AppointmentService.java
package com.theralink.appointmentservice.service;

import com.theralink.appointmentservice.dto.request.CreateAppointmentRequest;
import com.theralink.appointmentservice.dto.request.UpdateAppointmentRequest;
import com.theralink.appointmentservice.dto.response.AppointmentResponse;
import com.theralink.appointmentservice.kafka.AppointmentEventProducer;
import com.theralink.appointmentservice.model.Appointment;
import com.theralink.appointmentservice.model.AppointmentStatus;
import com.theralink.appointmentservice.repository.AppointmentRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class AppointmentService {

    private final AppointmentRepository appointmentRepository;
    private final AppointmentEventProducer eventProducer;

    public AppointmentResponse createAppointment(String clientKeycloakId, CreateAppointmentRequest request) {
        Appointment appointment = Appointment.builder()
            .clientKeycloakId(clientKeycloakId)
            .psychologistKeycloakId(request.getPsychologistKeycloakId())
            .date(request.getDate())
            .meetingLink(request.getMeetingLink())
            .status(AppointmentStatus.PENDING) // domyślnie PENDING, nie APPROVED
            .build();

        Appointment saved = appointmentRepository.save(appointment);

        // Publikuj zdarzenie Kafka — notification-service wyśle email
        eventProducer.publishAppointmentCreated(saved);

        return AppointmentResponse.from(saved);
    }

    public AppointmentResponse getAppointment(String id) {
        return appointmentRepository.findById(id)
            .map(AppointmentResponse::from)
            .orElseThrow(() -> new AppointmentNotFoundException("Wizyta nie znaleziona: " + id));
    }

    public List<AppointmentResponse> getAppointmentsForPsychologist(String keycloakId) {
        return appointmentRepository.findByPsychologistKeycloakId(keycloakId)
            .stream()
            .map(AppointmentResponse::from)
            .collect(Collectors.toList());
    }

    public List<AppointmentResponse> getAppointmentsForClient(String keycloakId) {
        return appointmentRepository.findByClientKeycloakId(keycloakId)
            .stream()
            .map(AppointmentResponse::from)
            .collect(Collectors.toList());
    }

    public AppointmentResponse updateAppointment(String id, UpdateAppointmentRequest request) {
        Appointment appointment = appointmentRepository.findById(id)
            .orElseThrow(() -> new AppointmentNotFoundException("Wizyta nie znaleziona: " + id));

        if (request.getDate() != null) appointment.setDate(request.getDate());
        if (request.getStatus() != null) appointment.setStatus(request.getStatus());
        if (request.getMeetingLink() != null) appointment.setMeetingLink(request.getMeetingLink());

        Appointment updated = appointmentRepository.save(appointment);

        // Jeśli status zmieniony, powiadom Kafką
        if (request.getStatus() != null) {
            eventProducer.publishAppointmentStatusChanged(updated);
        }

        return AppointmentResponse.from(updated);
    }

    public static class AppointmentNotFoundException extends RuntimeException {
        public AppointmentNotFoundException(String msg) { super(msg); }
    }
}
```

### 5.6 Kontroler

```java
// src/main/java/com/theralink/appointmentservice/controller/AppointmentController.java
package com.theralink.appointmentservice.controller;

import com.theralink.appointmentservice.dto.request.CreateAppointmentRequest;
import com.theralink.appointmentservice.dto.request.UpdateAppointmentRequest;
import com.theralink.appointmentservice.dto.response.AppointmentResponse;
import com.theralink.appointmentservice.service.AppointmentService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/appointments")
@RequiredArgsConstructor
public class AppointmentController {

    private final AppointmentService appointmentService;

    // POST /appointments
    // W Express: router.post("/:cognitoId", createAppointment)
    // Tutaj: clientKeycloakId pochodzi z JWT, nie z URL — bezpieczniej!
    @PostMapping
    @PreAuthorize("hasRole('CLIENT')")
    public ResponseEntity<AppointmentResponse> createAppointment(
        @AuthenticationPrincipal Jwt jwt,  // JWT z Bearer tokenu
        @Valid @RequestBody CreateAppointmentRequest request
    ) {
        String clientKeycloakId = jwt.getSubject();
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(appointmentService.createAppointment(clientKeycloakId, request));
    }

    // GET /appointments/{id}
    // W Express: router.get("/singleApt/:appointmentId", getAppointment)
    @GetMapping("/{id}")
    @PreAuthorize("hasAnyRole('CLIENT', 'PSYCHOLOGIST', 'ADMIN')")
    public ResponseEntity<AppointmentResponse> getAppointment(@PathVariable String id) {
        return ResponseEntity.ok(appointmentService.getAppointment(id));
    }

    // GET /appointments/psychologist/{keycloakId}
    // W Express: router.get("/psychologistApts/:cognitoId", ...)
    @GetMapping("/psychologist/{keycloakId}")
    @PreAuthorize("hasAnyRole('PSYCHOLOGIST', 'ADMIN')")
    public ResponseEntity<List<AppointmentResponse>> getForPsychologist(
        @PathVariable String keycloakId
    ) {
        return ResponseEntity.ok(appointmentService.getAppointmentsForPsychologist(keycloakId));
    }

    // GET /appointments/client/{keycloakId}
    // W Express: router.get("/clientApts/:cognitoId", ...)
    @GetMapping("/client/{keycloakId}")
    @PreAuthorize("hasAnyRole('CLIENT', 'ADMIN')")
    public ResponseEntity<List<AppointmentResponse>> getForClient(
        @PathVariable String keycloakId
    ) {
        return ResponseEntity.ok(appointmentService.getAppointmentsForClient(keycloakId));
    }

    // PUT /appointments/{id}
    @PutMapping("/{id}")
    @PreAuthorize("hasAnyRole('PSYCHOLOGIST', 'ADMIN')")
    public ResponseEntity<AppointmentResponse> updateAppointment(
        @PathVariable String id,
        @Valid @RequestBody UpdateAppointmentRequest request
    ) {
        return ResponseEntity.ok(appointmentService.updateAppointment(id, request));
    }
}
```

### 5.7 Kafka Producer

```java
// src/main/java/com/theralink/appointmentservice/kafka/AppointmentEventProducer.java
package com.theralink.appointmentservice.kafka;

import com.theralink.appointmentservice.model.Appointment;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

import java.time.Instant;
import java.util.Map;

@Component
@RequiredArgsConstructor
@Slf4j
public class AppointmentEventProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    private static final String TOPIC_CREATED = "theralink.appointment.created";
    private static final String TOPIC_STATUS_CHANGED = "theralink.appointment.status-changed";

    public void publishAppointmentCreated(Appointment appointment) {
        Map<String, Object> event = Map.of(
            "appointmentId", appointment.getId(),
            "clientKeycloakId", appointment.getClientKeycloakId(),
            "psychologistKeycloakId", appointment.getPsychologistKeycloakId(),
            "date", appointment.getDate().toString(),
            "status", appointment.getStatus().name(),
            "timestamp", Instant.now().toString()
        );
        kafkaTemplate.send(TOPIC_CREATED, appointment.getId(), event);
        log.info("Opublikowano appointment.created dla: {}", appointment.getId());
    }

    public void publishAppointmentStatusChanged(Appointment appointment) {
        Map<String, Object> event = Map.of(
            "appointmentId", appointment.getId(),
            "clientKeycloakId", appointment.getClientKeycloakId(),
            "newStatus", appointment.getStatus().name(),
            "timestamp", Instant.now().toString()
        );
        kafkaTemplate.send(TOPIC_STATUS_CHANGED, appointment.getId(), event);
        log.info("Opublikowano appointment.status-changed: {} → {}",
            appointment.getId(), appointment.getStatus());
    }
}
```

---

## 6. theralink-psychologist-service

### Cel
Przejmuje trasy Express: `POST/GET/PUT /availabilities`

Port: `8083`

### 6.1 Model AvailabilitySlot

```java
// src/main/java/com/theralink/psychologistservice/model/AvailabilitySlot.java
package com.theralink.psychologistservice.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.index.CompoundIndex;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.mapping.Document;

import java.time.Instant;

// CalendarAppointment z Prisma → AvailabilitySlot w MongoDB
// @CompoundIndex zapobiega duplikatom (jak unique constraint w SQL)
// Odpowiednik logiki: if (exists) { res.status(409)... } z availabilityController.ts
@Document(collection = "availability_slots")
@CompoundIndex(
    def = "{'psychologistKeycloakId': 1, 'date': 1, 'startHour': 1}",
    unique = true
)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class AvailabilitySlot {

    @Id
    private String id;

    @Indexed
    private String psychologistKeycloakId;

    private Instant date;
    private String startHour;  // format "HH:mm", np. "10:00"
    private String patientName; // puste = slot wolny, wypełnione = zarezerwowane
}
```

### 6.2 DTOs i Repository

```java
// src/main/java/com/theralink/psychologistservice/dto/request/CreateSlotRequest.java
package com.theralink.psychologistservice.dto.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Pattern;
import lombok.Data;

import java.time.Instant;

@Data
public class CreateSlotRequest {

    @NotNull(message = "Data jest wymagana")
    private Instant date;

    @NotBlank(message = "Godzina startu jest wymagana")
    @Pattern(regexp = "^([01]\\d|2[0-3]):([0-5]\\d)$", message = "Format godziny: HH:mm")
    private String startHour;

    private String patientName = ""; // pusty = dostępny slot
}
```

```java
// src/main/java/com/theralink/psychologistservice/repository/AvailabilitySlotRepository.java
package com.theralink.psychologistservice.repository;

import com.theralink.psychologistservice.model.AvailabilitySlot;
import org.springframework.data.mongodb.repository.MongoRepository;

import java.time.Instant;
import java.util.List;
import java.util.Optional;

public interface AvailabilitySlotRepository extends MongoRepository<AvailabilitySlot, String> {
    // findMany({ where: { psychologistId } }) z Prisma
    List<AvailabilitySlot> findByPsychologistKeycloakId(String psychologistKeycloakId);

    // Sprawdzenie duplikatu (zastępuje if (exists) { status 409 })
    // MongoDB @CompoundIndex obsłuży to automatycznie przy save()
    Optional<AvailabilitySlot> findByPsychologistKeycloakIdAndDateAndStartHour(
        String psychologistKeycloakId, Instant date, String startHour
    );
}
```

### 6.3 Serwis i Kontroler

```java
// src/main/java/com/theralink/psychologistservice/service/AvailabilityService.java
package com.theralink.psychologistservice.service;

import com.theralink.psychologistservice.dto.request.CreateSlotRequest;
import com.theralink.psychologistservice.dto.request.UpdateSlotRequest;
import com.theralink.psychologistservice.dto.response.SlotResponse;
import com.theralink.psychologistservice.model.AvailabilitySlot;
import com.theralink.psychologistservice.repository.AvailabilitySlotRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.dao.DuplicateKeyException;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class AvailabilityService {

    private final AvailabilitySlotRepository slotRepository;

    public SlotResponse createSlot(String psychologistKeycloakId, CreateSlotRequest request) {
        AvailabilitySlot slot = AvailabilitySlot.builder()
            .psychologistKeycloakId(psychologistKeycloakId)
            .date(request.getDate())
            .startHour(request.getStartHour())
            .patientName(request.getPatientName())
            .build();

        try {
            AvailabilitySlot saved = slotRepository.save(slot);
            return SlotResponse.from(saved);
        } catch (DuplicateKeyException e) {
            // @CompoundIndex wyrzuca DuplicateKeyException gdy slot już istnieje
            // Odpowiednik: if (exists) { res.status(409) } z availabilityController.ts
            throw new SlotAlreadyExistsException("Slot już istnieje dla tego czasu");
        }
    }

    public List<SlotResponse> getSlotsForPsychologist(String keycloakId) {
        return slotRepository.findByPsychologistKeycloakId(keycloakId)
            .stream()
            .map(SlotResponse::from)
            .collect(Collectors.toList());
    }

    public SlotResponse getSlot(String id) {
        return slotRepository.findById(id)
            .map(SlotResponse::from)
            .orElseThrow(() -> new SlotNotFoundException("Slot nie znaleziony: " + id));
    }

    public SlotResponse updateSlot(String id, UpdateSlotRequest request) {
        AvailabilitySlot slot = slotRepository.findById(id)
            .orElseThrow(() -> new SlotNotFoundException("Slot nie znaleziony: " + id));

        if (request.getDate() != null) slot.setDate(request.getDate());
        if (request.getStartHour() != null) slot.setStartHour(request.getStartHour());
        if (request.getPatientName() != null) slot.setPatientName(request.getPatientName());

        return SlotResponse.from(slotRepository.save(slot));
    }

    public static class SlotNotFoundException extends RuntimeException {
        public SlotNotFoundException(String msg) { super(msg); }
    }

    public static class SlotAlreadyExistsException extends RuntimeException {
        public SlotAlreadyExistsException(String msg) { super(msg); }
    }
}
```

```java
// src/main/java/com/theralink/psychologistservice/controller/AvailabilityController.java
package com.theralink.psychologistservice.controller;

import com.theralink.psychologistservice.dto.request.CreateSlotRequest;
import com.theralink.psychologistservice.dto.request.UpdateSlotRequest;
import com.theralink.psychologistservice.dto.response.SlotResponse;
import com.theralink.psychologistservice.service.AvailabilityService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/availabilities")
@RequiredArgsConstructor
public class AvailabilityController {

    private final AvailabilityService availabilityService;

    // POST /availabilities
    // W Express: router.post("/:psychologistId", authMiddleware(["psychologist"]), ...)
    // Tutaj: psychologistKeycloakId pochodzi z JWT — bezpieczniej
    @PostMapping
    @PreAuthorize("hasRole('PSYCHOLOGIST')")
    public ResponseEntity<SlotResponse> createSlot(
        @AuthenticationPrincipal Jwt jwt,
        @Valid @RequestBody CreateSlotRequest request
    ) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(availabilityService.createSlot(jwt.getSubject(), request));
    }

    // GET /availabilities/psychologist/{keycloakId}
    @GetMapping("/psychologist/{keycloakId}")
    @PreAuthorize("hasAnyRole('PSYCHOLOGIST', 'CLIENT')")
    public ResponseEntity<List<SlotResponse>> getSlotsForPsychologist(
        @PathVariable String keycloakId
    ) {
        return ResponseEntity.ok(availabilityService.getSlotsForPsychologist(keycloakId));
    }

    // GET /availabilities/{id}
    @GetMapping("/{id}")
    @PreAuthorize("hasAnyRole('PSYCHOLOGIST', 'CLIENT')")
    public ResponseEntity<SlotResponse> getSlot(@PathVariable String id) {
        return ResponseEntity.ok(availabilityService.getSlot(id));
    }

    // PUT /availabilities/{id}
    @PutMapping("/{id}")
    @PreAuthorize("hasRole('PSYCHOLOGIST')")
    public ResponseEntity<SlotResponse> updateSlot(
        @PathVariable String id,
        @Valid @RequestBody UpdateSlotRequest request
    ) {
        return ResponseEntity.ok(availabilityService.updateSlot(id, request));
    }
}
```

---

## 7. theralink-notification-service

### Cel
Konsument Kafka — brak żadnych endpointów HTTP. Subskrybuje zdarzenia i wysyła emaile/SMS.

Port: `8084` (tylko Actuator health check)

### 7.1 pom.xml — minimalne zależności

```xml
<!-- Usuwa spring-boot-starter-web — ten serwis nie ma HTTP API! -->
<!-- Dodaje tylko Kafka + mail -->
<dependencies>
    <!-- Kafka consumer -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>

    <!-- Email (JavaMailSender) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>

    <!-- Actuator dla health check -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

### 7.2 application.yml

```yaml
server:
  port: 8084

spring:
  application:
    name: theralink-notification-service

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    consumer:
      group-id: theralink-notification-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"

  mail:
    host: ${MAIL_HOST:smtp.gmail.com}
    port: ${MAIL_PORT:587}
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail.smtp.auth: true
      mail.smtp.starttls.enable: true

notification:
  from-email: ${NOTIFICATION_FROM_EMAIL:noreply@theralink.pl}
```

### 7.3 Kafka Consumer

```java
// src/main/java/com/theralink/notificationservice/kafka/AppointmentEventConsumer.java
package com.theralink.notificationservice.kafka;

import com.theralink.notificationservice.service.EmailService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
@RequiredArgsConstructor
@Slf4j
public class AppointmentEventConsumer {

    private final EmailService emailService;

    // @KafkaListener — Spring automatycznie subskrybuje topic i wywołuje metodę
    // gdy pojawi się nowa wiadomość
    // groupId = "theralink-notification-group" z application.yml
    @KafkaListener(topics = "theralink.appointment.created")
    public void onAppointmentCreated(Map<String, Object> event) {
        log.info("Otrzymano zdarzenie appointment.created: {}", event);

        String clientId = (String) event.get("clientKeycloakId");
        String date = (String) event.get("date");

        // W pełnej implementacji: pobierz email klienta z user-service
        // lub użyj eventu który zawiera email bezpośrednio
        emailService.sendAppointmentConfirmation(clientId, date);
    }

    @KafkaListener(topics = "theralink.appointment.status-changed")
    public void onAppointmentStatusChanged(Map<String, Object> event) {
        log.info("Otrzymano zdarzenie appointment.status-changed: {}", event);

        String clientId = (String) event.get("clientKeycloakId");
        String newStatus = (String) event.get("newStatus");

        emailService.sendStatusChangeNotification(clientId, newStatus);
    }

    @KafkaListener(topics = "theralink.user.client-registered")
    public void onClientRegistered(Map<String, Object> event) {
        log.info("Otrzymano zdarzenie client-registered: {}", event);
        String email = (String) event.get("email");
        emailService.sendWelcomeEmail(email);
    }
}
```

```java
// src/main/java/com/theralink/notificationservice/service/EmailService.java
package com.theralink.notificationservice.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
@Slf4j
public class EmailService {

    private final JavaMailSender mailSender;

    @Value("${notification.from-email}")
    private String fromEmail;

    public void sendWelcomeEmail(String toEmail) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom(fromEmail);
        message.setTo(toEmail);
        message.setSubject("Witaj w TheraLink!");
        message.setText("Dziękujemy za rejestrację w TheraLink. " +
            "Możesz teraz umówić wizytę z naszymi psychologami.");
        try {
            mailSender.send(message);
            log.info("Wysłano email powitalny do: {}", toEmail);
        } catch (Exception e) {
            log.error("Błąd wysyłania emaila do {}: {}", toEmail, e.getMessage());
        }
    }

    public void sendAppointmentConfirmation(String clientId, String date) {
        // W produkcji: pobierz email klienta przez Feign Client lub z eventu
        log.info("Potwierdzenie wizyty dla klienta {} na {}", clientId, date);
        // TODO: implementacja pełna po integracji z user-service
    }

    public void sendStatusChangeNotification(String clientId, String newStatus) {
        String statusPolish = switch (newStatus) {
            case "APPROVED" -> "zatwierdzona";
            case "DENIED" -> "odrzucona";
            default -> "zmieniona";
        };
        log.info("Powiadomienie o zmianie statusu dla klienta {}: {}", clientId, statusPolish);
        // TODO: implementacja pełna
    }
}
```

---

## 8. theralink-payment-service — oddzielne repozytorium

> ⚠️ **PCI-DSS:** Payment service jest w **osobnym, prywatnym repozytorium** (`theralink-payment-service`) z ograniczonym dostępem. Kod płatności NIE należy do tego repo ani żadnego innego serwisu TheraLink.

**Pełna dokumentacja implementacji:** [`docs/payment-service.md`](./payment-service.md)

### Dlaczego osobne repo?

PCI-DSS (Payment Card Industry Data Security Standard) wymaga:
- Izolacji kodu obsługującego płatności od reszty systemu
- Osobnych uprawnień dostępu (tylko wybrani deweloperzy)
- Osobnych audytów bezpieczeństwa
- Pełnej kontroli nad tym, kto ma dostęp do kluczy Stripe

### Komunikacja z resztą systemu

Payment service **nie jest wywoływany bezpośrednio przez inne serwisy HTTP**. Komunikacja odbywa się wyłącznie przez Kafka:

```
appointment-service  ──[theralink.appointment.created]──►  payment-service
                                                                    │
                                                             (Stripe webhook)
                                                                    │
payment-service      ──[theralink.payment.completed]──►  notification-service
```

---

## 9. Migracja danych PostgreSQL → MongoDB

### 9.1 Teoria

MongoDB jest bazą dokumentową — nie ma tabel, ma kolekcje. Nie ma relacji (JOIN), ma osadzone dokumenty lub referencje przez ID. Schemat jest elastyczny.

**Różnice mapowania:**

| PostgreSQL (Prisma) | MongoDB (Spring Data) |
|---|---|
| Table `Client` | Collection `clients` |
| `@id @default(autoincrement())` Int | `@Id` String (ObjectId) |
| `@unique cognitoId` | `@Indexed(unique=true) keycloakId` |
| Relacje przez klucze obce | Referencje przez String ID |
| `Appointment.clientCognitoId` FK | `Appointment.clientKeycloakId` String |

### 9.2 Strategia migracji ID

`cognitoId` (AWS Cognito) → `keycloakId` (Keycloak)

Podczas migracji użytkowników do Keycloak, Keycloak przypisze nowe UUID. Musisz:
1. Wyeksportować użytkowników z Cognito
2. Zaimportować do Keycloak — Keycloak nada nowe `sub` (UUID)
3. Zachować mapowanie: `cognitoId` → `keycloakId`
4. Uruchomić skrypt migracji danych

### 9.3 Skrypt migracji (Python)

```python
#!/usr/bin/env python3
"""
migracja_danych.py
Migruje dane z PostgreSQL do MongoDB.
Wymaga: pip install psycopg2-binary pymongo
"""

import psycopg2
import pymongo
from datetime import datetime, timezone

# Konfiguracja
PG_DSN = "postgresql://user:password@localhost:5432/theralink"
MONGO_URI = "mongodb://localhost:27017"

# Mapowanie cognitoId → keycloakId (wypełnij po migracji użytkowników do Keycloak)
# Format: { "cognito-uuid-1": "keycloak-uuid-1", ... }
COGNITO_TO_KEYCLOAK = {
    # "aws-cognito-id": "keycloak-subject-id"
}

def migrate():
    pg = psycopg2.connect(PG_DSN)
    mongo = pymongo.MongoClient(MONGO_URI)

    pg_cur = pg.cursor()

    # ── Migracja Klientów ────────────────────────────────────────────────
    print("Migracja klientów...")
    pg_cur.execute('SELECT "cognitoId", name, email, "phoneNumber", history FROM "Client"')
    clients_col = mongo["theralink-users"]["clients"]
    clients_col.create_index("keycloakId", unique=True)

    for row in pg_cur.fetchall():
        cognito_id, name, email, phone, history = row
        keycloak_id = COGNITO_TO_KEYCLOAK.get(cognito_id, cognito_id)  # fallback na cognitoId

        clients_col.update_one(
            {"keycloakId": keycloak_id},
            {"$setOnInsert": {
                "keycloakId": keycloak_id,
                "name": name,
                "email": email,
                "phoneNumber": phone,
                "history": history,
                "_migratedFrom": "postgresql",
                "_originalCognitoId": cognito_id,
            }},
            upsert=True
        )
    print(f"  Przeniesiono klientów: {clients_col.count_documents({})}")

    # ── Migracja Psychologów ─────────────────────────────────────────────
    print("Migracja psychologów...")
    pg_cur.execute("""
        SELECT "cognitoId", name, email, "phoneNumber", location,
               "hourlyRate", "Description", "Specialization"
        FROM "Psychologist"
    """)
    psychologists_col = mongo["theralink-users"]["psychologists"]
    psychologists_col.create_index("keycloakId", unique=True)

    for row in pg_cur.fetchall():
        cid, name, email, phone, loc, rate, desc, spec = row
        kid = COGNITO_TO_KEYCLOAK.get(cid, cid)

        psychologists_col.update_one(
            {"keycloakId": kid},
            {"$setOnInsert": {
                "keycloakId": kid,
                "name": name,
                "email": email,
                "phoneNumber": phone,
                "location": loc,
                "hourlyRate": rate,
                "description": desc,
                "specialization": spec,
                "_migratedFrom": "postgresql",
            }},
            upsert=True
        )
    print(f"  Przeniesiono psychologów: {psychologists_col.count_documents({})}")

    # ── Migracja Wizyt ───────────────────────────────────────────────────
    print("Migracja wizyt...")
    pg_cur.execute("""
        SELECT id, "clientCognitoId", "psychologistId", "meetingLink", date, "Status"
        FROM "Appointment"
    """)
    appointments_col = mongo["theralink-appointments"]["appointments"]

    STATUS_MAP = {"Pending": "PENDING", "Approved": "APPROVED", "Denied": "DENIED"}

    for row in pg_cur.fetchall():
        apt_id, client_cid, psych_cid, link, date, status = row
        client_kid = COGNITO_TO_KEYCLOAK.get(client_cid, client_cid)
        psych_kid = COGNITO_TO_KEYCLOAK.get(psych_cid, psych_cid)

        appointments_col.insert_one({
            "clientKeycloakId": client_kid,
            "psychologistKeycloakId": psych_kid,
            "meetingLink": link,
            "date": date.replace(tzinfo=timezone.utc) if date else None,
            "status": STATUS_MAP.get(status, "PENDING"),
            "_migratedFrom": "postgresql",
            "_originalId": apt_id,
        })
    print(f"  Przeniesiono wizyty: {appointments_col.count_documents({})}")

    # ── Migracja Slotów Dostępności ──────────────────────────────────────
    print("Migracja slotów dostępności...")
    pg_cur.execute("""
        SELECT id, "psychologistId", date, "startHour", "patientName"
        FROM appointments
    """)
    slots_col = mongo["theralink-psychologist"]["availability_slots"]

    for row in pg_cur.fetchall():
        slot_id, psych_cid, date, start_hour, patient = row
        psych_kid = COGNITO_TO_KEYCLOAK.get(psych_cid, psych_cid)

        slots_col.insert_one({
            "psychologistKeycloakId": psych_kid,
            "date": date.replace(tzinfo=timezone.utc) if date else None,
            "startHour": start_hour,
            "patientName": patient or "",
            "_migratedFrom": "postgresql",
            "_originalId": slot_id,
        })
    print(f"  Przeniesiono sloty: {slots_col.count_documents({})}")

    pg_cur.close()
    pg.close()
    mongo.close()
    print("\nMigracja zakończona!")

if __name__ == "__main__":
    migrate()
```

### 9.4 Walidacja po migracji

```bash
# Sprawdź liczby — powinny się zgadzać z PostgreSQL
mongosh --eval "
  use theralink-users;
  print('Klienci:', db.clients.countDocuments());
  print('Psycholodzy:', db.psychologists.countDocuments());
"

mongosh --eval "
  use theralink-appointments;
  print('Wizyty:', db.appointments.countDocuments());
"
```

---

## 10. Testowanie: JUnit 5 + Mockito + Testcontainers

### 10.1 Teoria

| Narzędzie | Cel | Odpowiednik Node.js |
|---|---|---|
| JUnit 5 | Framework testów | Jest/Mocha |
| Mockito | Mockowanie zależności | jest.mock() |
| Testcontainers | Prawdziwe MongoDB w Dockerze podczas testów | jest + testenv + docker-compose |
| `@SpringBootTest` | Pełny test integracyjny | supertest |
| `@WebMvcTest` | Test tylko warstwy HTTP | supertest z mockami |

### 10.2 Test jednostkowy serwisu

```java
// src/test/java/com/theralink/userservice/service/ClientServiceTest.java
package com.theralink.userservice.service;

import com.theralink.userservice.dto.request.CreateClientRequest;
import com.theralink.userservice.dto.response.ClientResponse;
import com.theralink.userservice.model.Client;
import com.theralink.userservice.repository.ClientRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

// @ExtendWith(MockitoExtension.class) — uruchamia Mockito
// Odpowiednik: describe("ClientService", () => { ... }) z Jest
@ExtendWith(MockitoExtension.class)
class ClientServiceTest {

    // @Mock — tworzy mocka repozytorium (nie łączy się z prawdziwą bazą)
    // Odpowiednik: jest.mock("../repository/clientRepository")
    @Mock
    private ClientRepository clientRepository;

    // @InjectMocks — tworzy ClientService i wstrzykuje @Mock do konstruktora
    @InjectMocks
    private ClientService clientService;

    private CreateClientRequest validRequest;

    @BeforeEach
    void setUp() {
        // Odpowiednik: beforeEach(() => { ... }) z Jest
        validRequest = new CreateClientRequest();
        validRequest.setKeycloakId("keycloak-123");
        validRequest.setName("Jan Kowalski");
        validRequest.setEmail("jan@example.com");
        validRequest.setPhoneNumber("+48 123 456 789");
    }

    @Test
    @DisplayName("createClient — powinien zapisać i zwrócić klienta")
    void createClient_shouldSaveAndReturnClient() {
        // Arrange — konfigurujemy zachowanie mocka
        // Odpowiednik: jest.spyOn(repo, "save").mockResolvedValue(...)
        Client savedClient = Client.builder()
            .id("mongo-id-456")
            .keycloakId("keycloak-123")
            .name("Jan Kowalski")
            .email("jan@example.com")
            .phoneNumber("+48 123 456 789")
            .build();

        when(clientRepository.save(any(Client.class))).thenReturn(savedClient);

        // Act
        ClientResponse result = clientService.createClient(validRequest);

        // Assert
        assertThat(result.getKeycloakId()).isEqualTo("keycloak-123");
        assertThat(result.getName()).isEqualTo("Jan Kowalski");
        assertThat(result.getEmail()).isEqualTo("jan@example.com");

        // Weryfikujemy że save() zostało wywołane dokładnie raz
        // Odpowiednik: expect(repo.save).toHaveBeenCalledTimes(1)
        verify(clientRepository, times(1)).save(any(Client.class));
    }

    @Test
    @DisplayName("getClientByKeycloakId — powinien rzucić wyjątek gdy klient nie istnieje")
    void getClient_shouldThrowWhenNotFound() {
        // Arrange
        when(clientRepository.findByKeycloakId("nieistniejacy-id"))
            .thenReturn(Optional.empty());

        // Act + Assert
        // Odpowiednik: expect(() => service.getClient("x")).rejects.toThrow()
        assertThatThrownBy(() -> clientService.getClientByKeycloakId("nieistniejacy-id"))
            .isInstanceOf(ClientService.ClientNotFoundException.class)
            .hasMessageContaining("nieistniejacy-id");
    }
}
```

### 10.3 Test integracyjny z Testcontainers

```java
// src/test/java/com/theralink/userservice/repository/ClientRepositoryIntegrationTest.java
package com.theralink.userservice.repository;

import com.theralink.userservice.model.Client;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.data.mongo.DataMongoTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MongoDBContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

// @Testcontainers — Testcontainers automatycznie startuje Docker przed testami
// Uruchamia prawdziwą instancję MongoDB w Dockerze — nie potrzebujesz zainstalowanej bazy!
@DataMongoTest
@Testcontainers
class ClientRepositoryIntegrationTest {

    // MongoDB 7.0 w kontenerze Docker
    @Container
    static MongoDBContainer mongodb = new MongoDBContainer("mongo:7.0");

    // Dynamicznie podmień URI MongoDB na adres kontenera
    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.data.mongodb.uri", mongodb::getReplicaSetUrl);
    }

    @Autowired
    private ClientRepository clientRepository;

    @AfterEach
    void cleanup() {
        clientRepository.deleteAll();
    }

    @Test
    void shouldSaveAndFindByKeycloakId() {
        // Arrange
        Client client = Client.builder()
            .keycloakId("test-keycloak-id")
            .name("Anna Nowak")
            .email("anna@test.com")
            .phoneNumber("+48 987 654 321")
            .build();

        // Act
        clientRepository.save(client);
        Optional<Client> found = clientRepository.findByKeycloakId("test-keycloak-id");

        // Assert
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Anna Nowak");
        assertThat(found.get().getId()).isNotNull(); // MongoDB nadał ObjectId
    }

    @Test
    void existsByKeycloakId_shouldReturnFalseWhenNotExists() {
        boolean exists = clientRepository.existsByKeycloakId("nieistniejacy");
        assertThat(exists).isFalse();
    }
}
```

### 10.4 Test kontrolera (HTTP layer)

```java
// src/test/java/com/theralink/userservice/controller/ClientControllerTest.java
package com.theralink.userservice.controller;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.theralink.userservice.dto.request.CreateClientRequest;
import com.theralink.userservice.dto.response.ClientResponse;
import com.theralink.userservice.service.ClientService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

// @WebMvcTest — testuje tylko warstwę HTTP (Spring MVC), mockuje serwisy
// Szybszy niż @SpringBootTest bo nie startuje całego kontekstu
@WebMvcTest(ClientController.class)
class ClientControllerTest {

    // MockMvc symuluje żądania HTTP bez uruchamiania serwera
    // Odpowiednik: supertest(app).post("/clients")
    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    // @MockBean — tworzy mocka Spring Bean (zastępuje prawdziwy ClientService)
    @MockBean
    private ClientService clientService;

    @Test
    @WithMockUser // symuluje zalogowanego użytkownika (Spring Security)
    void createClient_shouldReturn201() throws Exception {
        // Arrange
        CreateClientRequest request = new CreateClientRequest();
        request.setKeycloakId("kc-123");
        request.setName("Piotr Wiśniewski");
        request.setEmail("piotr@test.com");
        request.setPhoneNumber("+48 111 222 333");

        ClientResponse mockResponse = ClientResponse.builder()
            .id("mongo-789")
            .keycloakId("kc-123")
            .name("Piotr Wiśniewski")
            .email("piotr@test.com")
            .build();

        when(clientService.createClient(any())).thenReturn(mockResponse);

        // Act + Assert
        mockMvc.perform(
                post("/clients")
                    .with(csrf())
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request))
            )
            .andExpect(status().isCreated())              // HTTP 201
            .andExpect(jsonPath("$.keycloakId").value("kc-123"))
            .andExpect(jsonPath("$.name").value("Piotr Wiśniewski"));
    }

    @Test
    @WithMockUser
    void createClient_shouldReturn400_whenEmailInvalid() throws Exception {
        CreateClientRequest request = new CreateClientRequest();
        request.setKeycloakId("kc-123");
        request.setName("Test");
        request.setEmail("niepoprawny-email"); // brak @
        request.setPhoneNumber("+48 111 222 333");

        mockMvc.perform(
                post("/clients")
                    .with(csrf())
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request))
            )
            .andExpect(status().isBadRequest()); // @Email walidacja zwraca 400
    }
}
```

### 10.5 Uruchamianie testów

```bash
# Wszystkie testy
mvn test

# Konkretna klasa
mvn test -Dtest=ClientServiceTest

# Z raportem coverage (Jacoco)
mvn verify

# Pomiń testy (tylko build)
mvn package -DskipTests
```

---

## Podsumowanie — kolejność implementacji

```
Tydzień 1:
  ✅ theralink-user-service — Client + Psychologist
     (najważniejszy, reszta od niego zależy)

Tydzień 2:
  ✅ theralink-appointment-service
  ✅ Kafka integration (producer w appointment, consumer w notification)

Tydzień 3:
  ✅ theralink-psychologist-service — Availability slots
  ✅ theralink-notification-service — Email

Tydzień 4:
  ✅ Migracja danych PostgreSQL → MongoDB
  ✅ Testy integracyjne + Testcontainers

Oddzielnie (restricted repo):
  ⚠️  theralink-payment-service — Stripe
```

## Przydatne komendy deweloperskie

```bash
# Start wszystkich serwisów lokalnie (wymaga docker-compose z MongoDB + Kafka + Keycloak)
docker-compose -f docker-compose.dev.yml up -d

# Build serwisu
cd theralink-user-service
mvn clean package

# Start serwisu
java -jar target/user-service-0.0.1-SNAPSHOT.jar

# Lub przez Maven
mvn spring-boot:run

# Logi w trybie debug
mvn spring-boot:run -Dspring-boot.run.jvmArguments="-Dlogging.level.com.theralink=DEBUG"

# Health check
curl http://localhost:8081/actuator/health
```
