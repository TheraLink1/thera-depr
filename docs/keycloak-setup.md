# theralink-keycloak — Przewodnik konfiguracji

> **Repo:** `theralink-keycloak` — osobne repozytorium z własnym `Dockerfile` i motywem FTL.
> Keycloak zastępuje AWS Cognito jako system zarządzania tożsamością (Identity Provider).

---

## Spis treści

1. [Teoria Keycloak dla początkujących](#1-teoria-keycloak-dla-początkujących)
2. [Instalacja i konfiguracja (Docker)](#2-instalacja-i-konfiguracja-docker)
3. [Role i atrybuty użytkownika](#3-role-i-atrybuty-użytkownika)
4. [Własny motyw FTL (theralink-theme)](#4-własny-motyw-ftl-theralink-theme)
5. [Integracja z Angular (keycloak-angular)](#5-integracja-z-angular-keycloak-angular)
6. [Weryfikacja i troubleshooting](#6-weryfikacja-i-troubleshooting)

---

## 1. Teoria Keycloak dla początkujących

### Keycloak vs AWS Cognito — porównanie

| Cecha | AWS Cognito | Keycloak |
|---|---|---|
| Hosting | Tylko AWS cloud | Self-hosted (Docker, Kubernetes) |
| Koszt | Płatny po przekroczeniu limitu | Open-source, darmowy |
| UI logowania | Ograniczone customizacje | Pełna kontrola przez FTL templates |
| Vendor lock-in | Wysoki (cognitoId, Amplify) | Brak — standard OAuth2/OIDC |
| Konfiguracja | AWS Console / CDK | Keycloak Admin Console / JSON export |
| JWT format | Własny format AWS | Standard OIDC, pełna kontrola claims |

### Kluczowe pojęcia

**Realm** — izolowany przestrzeń tenantów. TheraLink będzie miał jeden realm: `theralink`.
Można to porównać do "projektu" w AWS Cognito (User Pool).

```
Keycloak server
└── realm: master          (tylko dla adminów Keycloak — nie tykamy!)
└── realm: theralink       ← nasz realm
    ├── users: klienci, psycholodzy, adminowie
    ├── clients: theralink-angular, theralink-backend
    └── roles: CLIENT, PSYCHOLOGIST, ADMIN
```

**Client (OAuth2 Client)** — aplikacja korzystająca z Keycloak. NIE myląć z "klientem" (pacjentem) w sensie biznesowym!

| Client ID | Typ | Użycie |
|---|---|---|
| `theralink-angular` | Public (PKCE) | Aplikacja Angular loguje użytkowników |
| `theralink-backend` | Confidential | Serwisy Spring Boot weryfikują tokeny |

**Typy tokenów:**
- **Access Token (JWT)** — krótkotrwały (15 min), wysyłany z każdym żądaniem HTTP w nagłówku `Authorization: Bearer`. Spring Boot weryfikuje jego podpis.
- **Refresh Token** — długotrwały (7 dni), służy do odświeżania access tokenu bez ponownego logowania.
- **ID Token** — zawiera dane profilu użytkownika (imię, email) dla frontendu.

### OAuth2 Authorization Code Flow z PKCE

To przepływ, który Angular używa do logowania. PKCE (Proof Key for Code Exchange) zabezpiecza aplikacje publiczne (Angular nie ma sekretu klienta).

```
1. Użytkownik klika "Zaloguj się" w Angular
   ↓
2. Angular przekierowuje na:
   http://localhost:8080/realms/theralink/protocol/openid-connect/auth
   ?client_id=theralink-angular
   &redirect_uri=http://localhost:4200/callback
   &response_type=code
   &scope=openid profile email
   &code_challenge=BASE64(SHA256(code_verifier))   ← PKCE
   &code_challenge_method=S256

3. Keycloak pokazuje stronę logowania (nasz motyw FTL!)
   ↓
4. Użytkownik wpisuje dane → Keycloak weryfikuje
   ↓
5. Keycloak przekierowuje z powrotem:
   http://localhost:4200/callback?code=AUTHORIZATION_CODE
   ↓
6. Angular wymienia code na tokeny:
   POST http://localhost:8080/realms/theralink/protocol/openid-connect/token
   Body: code=AUTHORIZATION_CODE & code_verifier=...
   ↓
7. Keycloak zwraca: { access_token, refresh_token, id_token }
   ↓
8. Angular zapisuje tokeny (memory lub sessionStorage)
   Każde żądanie HTTP: Authorization: Bearer ACCESS_TOKEN
   ↓
9. Spring Boot weryfikuje podpis access_token przez JWKS:
   http://localhost:8080/realms/theralink/protocol/openid-connect/certs
```

### Jak Spring Boot weryfikuje JWT bez rozmowy z Keycloak przy każdym żądaniu?

Keycloak publikuje klucze publiczne (JWKS — JSON Web Key Set) pod adresem:
```
http://localhost:8080/realms/theralink/protocol/openid-connect/certs
```

Spring Boot pobiera te klucze **raz przy starcie** i cachuje je. Każdy JWT jest podpisany kluczem prywatnym Keycloak — Spring weryfikuje podpis kluczem publicznym lokalnie, bez połączenia z Keycloak. Szybko i bezpiecznie.

Konfiguracja w `application.yml` każdego Spring Boot serwisu:
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8080/realms/theralink
          # Spring pobierze JWKS z: {issuer-uri}/.well-known/openid-configuration
```

---

## 2. Instalacja i konfiguracja (Docker)

### Struktura repozytorium `theralink-keycloak`

```
theralink-keycloak/
├── Dockerfile                  ← buduje obraz z wbudowanym motywem
├── docker-compose.yml          ← lokalny dev (Keycloak + PostgreSQL)
├── realm-export.json           ← konfiguracja realm (import automatyczny)
└── themes/
    └── theralink/
        ├── login/
        │   ├── theme.properties
        │   ├── template.ftl
        │   ├── login.ftl
        │   ├── register.ftl
        │   ├── error.ftl
        │   └── resources/
        │       ├── css/
        │       │   └── theralink.css
        │       └── img/
        │           └── logo.svg
        └── account/
            └── theme.properties
```

### docker-compose.yml — lokalny development

```yaml
# docker-compose.yml
version: '3.8'

services:
  keycloak-db:
    image: postgres:16
    container_name: theralink-keycloak-db
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: ${KC_DB_PASSWORD:-keycloak_dev_password}
    volumes:
      - keycloak_db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U keycloak"]
      interval: 10s
      timeout: 5s
      retries: 5

  keycloak:
    build: .   # używa lokalnego Dockerfile z motywem
    container_name: theralink-keycloak
    environment:
      # Dane admina Keycloak — zmień na produkcji!
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: ${KC_ADMIN_PASSWORD:-admin}
      # Baza danych
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak-db:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: ${KC_DB_PASSWORD:-keycloak_dev_password}
      # Hostname
      KC_HOSTNAME: localhost
      KC_HOSTNAME_PORT: 8080
      KC_HTTP_ENABLED: "true"
      KC_HOSTNAME_STRICT: "false"
      # Tryb deweloperski — nie używaj na produkcji!
      KC_LOG_LEVEL: info
    ports:
      - "8080:8080"
    depends_on:
      keycloak-db:
        condition: service_healthy
    command:
      # start-dev = tryb deweloperski (brak HTTPS, szybszy restart)
      # Na produkcji: start (wymaga certyfikatu TLS!)
      - start-dev
      # Automatyczny import realm przy starcie (jeśli realm nie istnieje)
      - --import-realm
    volumes:
      # Montujemy realm-export.json do katalogu importu
      - ./realm-export.json:/opt/keycloak/data/import/realm-export.json:ro

volumes:
  keycloak_db_data:
```

### Dockerfile — obraz z wbudowanym motywem

```dockerfile
# Dockerfile
FROM quay.io/keycloak/keycloak:25.0.4

# Kopiuj własny motyw do obrazu
# Keycloak szuka motywów w /opt/keycloak/themes/
COPY themes/theralink /opt/keycloak/themes/theralink

# Ustaw motyw jako domyślny (można też przez realm-export.json)
# Ten argument jest opcjonalny — można to skonfigurować w Admin Console
ENV KC_SPI_THEME_DEFAULT_LOCALE=pl

# Optymalizacja: pre-kompiluj templates przy budowaniu obrazu
# Sprawia że Keycloak startuje szybciej w produkcji
RUN /opt/keycloak/bin/kc.sh build

ENTRYPOINT ["/opt/keycloak/bin/kc.sh"]
```

### realm-export.json — automatyczna konfiguracja realm

Ten plik jest importowany przy starcie Keycloak. Pozwala na automatyczne tworzenie realm bez klikania w Admin Console. Wygeneruj go przez: Admin Console → Export → Export realm.

```json
{
  "realm": "theralink",
  "displayName": "TheraLink",
  "enabled": true,

  "loginTheme": "theralink",
  "accountTheme": "theralink",
  "emailTheme": "theralink",

  "sslRequired": "none",
  "registrationAllowed": true,
  "registrationEmailAsUsername": true,
  "loginWithEmailAllowed": true,
  "duplicateEmailsAllowed": false,
  "resetPasswordAllowed": true,
  "editUsernameAllowed": false,
  "rememberMe": true,

  "accessTokenLifespan": 900,
  "ssoSessionIdleTimeout": 1800,
  "ssoSessionMaxLifespan": 604800,

  "internationalizationEnabled": true,
  "supportedLocales": ["pl", "en"],
  "defaultLocale": "pl",

  "clients": [
    {
      "clientId": "theralink-angular",
      "name": "TheraLink Angular App",
      "enabled": true,
      "publicClient": true,
      "standardFlowEnabled": true,
      "directAccessGrantsEnabled": false,

      "redirectUris": [
        "http://localhost:4200/*",
        "https://app.theralink.pl/*"
      ],
      "webOrigins": [
        "http://localhost:4200",
        "https://app.theralink.pl"
      ],

      "attributes": {
        "pkce.code.challenge.method": "S256"
      },

      "defaultClientScopes": ["openid", "profile", "email", "roles"]
    },
    {
      "clientId": "theralink-backend",
      "name": "TheraLink Backend Services",
      "enabled": true,
      "publicClient": false,
      "serviceAccountsEnabled": true,
      "standardFlowEnabled": false,

      "secret": "${KC_BACKEND_CLIENT_SECRET}",

      "defaultClientScopes": ["openid", "roles"]
    }
  ],

  "roles": {
    "realm": [
      { "name": "CLIENT", "description": "Pacjent korzystający z platformy" },
      { "name": "PSYCHOLOGIST", "description": "Zarejestrowany psycholog" },
      { "name": "ADMIN", "description": "Administrator platformy" }
    ]
  },

  "defaultRoles": ["CLIENT"]
}
```

---

## 3. Role i atrybuty użytkownika

### Role realm

TheraLink używa trzech ról realm (nie ról klienta):

| Rola | Opis | Kto może nadać |
|---|---|---|
| `CLIENT` | Domyślna rola każdego nowego użytkownika | Automatycznie przy rejestracji |
| `PSYCHOLOGIST` | Weryfikowany psycholog | Admin po weryfikacji dokumentów |
| `ADMIN` | Administrator platformy | Tylko inny admin |

### Jak role trafiają do JWT

Keycloak umieszcza role realm w tokenie JWT pod kluczem `realm_access.roles`:

```json
{
  "sub": "550e8400-e29b-41d4-a716-446655440000",
  "preferred_username": "jan.kowalski@example.com",
  "email": "jan.kowalski@example.com",
  "given_name": "Jan",
  "family_name": "Kowalski",
  "realm_access": {
    "roles": [
      "CLIENT",
      "offline_access",
      "uma_authorization"
    ]
  },
  "exp": 1741516800,
  "iat": 1741516800
}
```

Spring Boot czyta `realm_access.roles` w `SecurityConfig.jwtAuthenticationConverter()` — patrz `docs/backend-migration.md`.

### Konfiguracja Protocol Mappers — dodawanie ról do tokenu

Keycloak domyślnie może nie wstawiać ról do access tokenu. Upewnij się że mapper jest skonfigurowany:

1. Admin Console → Clients → `theralink-angular` → Client scopes
2. Kliknij `theralink-angular-dedicated`
3. Add mapper → By configuration → User Realm Role
4. Token Claim Name: `realm_access.roles`
5. Add to access token: ON
6. Save

### Atrybuty użytkownika (opcjonalne)

Możesz dodać dodatkowe atrybuty do profilu użytkownika Keycloak:

```
Admin Console → Realm settings → User profile → Add attribute
```

Przykładowe atrybuty dla TheraLink:
- `phoneNumber` — numer telefonu (String)
- `specialization` — specjalizacja psychologa (String, tylko dla PSYCHOLOGIST)

Aby atrybuty pojawiły się w JWT, dodaj Protocol Mapper:
1. Clients → `theralink-angular` → Client scopes → `theralink-angular-dedicated`
2. Add mapper → User Attribute
3. User Attribute: `phoneNumber`, Token Claim Name: `phone_number`
4. Add to access token: ON

### Automatyczna rola DEFAULT po rejestracji

W `realm-export.json` ustawione jest `"defaultRoles": ["CLIENT"]` — każdy nowy użytkownik automatycznie dostaje rolę `CLIENT`.

Aby zmienić rolę użytkownika na `PSYCHOLOGIST`:
1. Admin Console → Users → wybierz użytkownika → Role mappings
2. Assign role: `PSYCHOLOGIST`
3. Opcjonalnie usuń: `CLIENT` (jeśli psycholog nie ma być też klientem)

---

## 4. Własny motyw FTL (theralink-theme)

### Teoria: jak działa system motywów Keycloak

Keycloak renderuje strony logowania przy użyciu **FreeMarker** (`.ftl`) — silnika szablonów dla Javy. FTL to odpowiednik Handlebars/Jinja2 — HTML z logiką: zmienne `${variable}`, warunki `<#if>`, pętle `<#list>`.

**Hierarchia dziedziczenia motywów:**
```
base          ← minimalistyczny bazowy motyw Keycloak
  └── keycloak   ← domyślny motyw z CSS Keycloak
        └── theralink  ← nasz motyw (nadpisuje pliki z keycloak)
```

W `theme.properties` definiujesz od którego motywu dziedziczysz. Pliki których NIE nadpiszesz będą wzięte z motywu nadrzędnego.

### Struktura katalogów (gotowa do skopiowania)

```
themes/theralink/
├── login/
│   ├── theme.properties       ← metadane motywu
│   ├── template.ftl           ← szkielet HTML (header, footer, meta)
│   ├── login.ftl              ← formularz logowania
│   ├── register.ftl           ← formularz rejestracji
│   ├── error.ftl              ← strona błędu
│   └── resources/
│       ├── css/
│       │   └── theralink.css  ← Twoje style — wypełnij sam!
│       └── img/
│           └── logo.svg       ← logo TheraLink
└── account/
    └── theme.properties       ← panel konta użytkownika
```

### theme.properties

```properties
# themes/theralink/login/theme.properties

# Dziedzicz z motywu keycloak (masz dostęp do jego CSS i logiki)
parent=keycloak

# Importuj własny CSS PRZED CSS Keycloak (nadpisuje style)
styles=css/login.css css/theralink.css

# Skrypty (opcjonalne)
# scripts=js/theralink.js

# Obsługiwane języki
locales=pl,en

# Domyślna czcionka (jeśli chcesz Google Fonts, dodaj link w template.ftl)
# intheme.font.url=https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600
```

### template.ftl — bazowy layout HTML

```html
<#-- themes/theralink/login/template.ftl -->
<#-- To jest layout bazowy — login.ftl, register.ftl itd. go "dziedziczą" przez <#include> -->

<#macro registrationLayout bodyClass="" displayInfo=false displayMessage=true displayRequiredFields=false>
<!DOCTYPE html>
<html lang="${locale.currentLanguageTag}"
      class="${properties.kcHtmlClass!}"
      <#if realm.internationalizationEnabled>dir="ltr"</#if>>

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="robots" content="noindex, nofollow">

    <title>${realm.displayName} — TheraLink</title>

    <link rel="icon" href="${url.resourcesPath}/img/logo.svg">

    <#-- Style Keycloak (z motywu nadrzędnego) -->
    <#list properties.styles?split(' ') as style>
        <link href="${url.resourcesPath}/${style}" rel="stylesheet"/>
    </#list>

    <#-- Jeśli chcesz Google Fonts: -->
    <#-- <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet"> -->
</head>

<body class="${properties.kcBodyClass!}">

    <#-- Główny kontener strony -->
    <div class="theralink-auth-wrapper">

        <#-- Panel boczny z logo (opcjonalny) -->
        <aside class="theralink-sidebar">
            <div class="theralink-logo">
                <img src="${url.resourcesPath}/img/logo.svg" alt="TheraLink" />
                <span>TheraLink</span>
            </div>
            <p class="theralink-tagline">
                Zadbaj o swoje zdrowie psychiczne.
            </p>
        </aside>

        <#-- Główna sekcja formularza -->
        <main class="theralink-form-section">

            <#-- Komunikaty błędów / sukcesów -->
            <#if displayMessage && message?has_content>
                <div class="alert alert-${message.type}">
                    <#if message.type = 'error'>
                        <span class="icon-error">⚠</span>
                    <#elseif message.type = 'success'>
                        <span class="icon-success">✓</span>
                    </#if>
                    <span class="kc-feedback-text">${kcSanitize(message.summary)?no_esc}</span>
                </div>
            </#if>

            <#-- Treść strony (login.ftl, register.ftl itp. wstrzykują tu swój HTML) -->
            <#nested "form">

            <#-- Linki pomocnicze (np. "Wróć do logowania") -->
            <#if displayInfo>
                <div id="kc-info" class="${properties.kcSignUpClass!}">
                    <div id="kc-info-wrapper">
                        <#nested "info">
                    </div>
                </div>
            </#if>

        </main>
    </div>

    <#-- Skrypty -->
    <#list properties.scripts?split(' ') as script>
        <script src="${url.resourcesPath}/${script}" type="text/javascript"></script>
    </#list>

</body>
</html>
</#macro>
```

### login.ftl — formularz logowania

```html
<#-- themes/theralink/login/login.ftl -->
<#-- Importuje makro z template.ftl -->
<#import "template.ftl" as layout>

<#-- ${realm.displayName} = "TheraLink" z konfiguracji realm -->
<@layout.registrationLayout displayMessage=!messagesPerField.existsError('username','password')>

    <#-- Wypełnienie sekcji "form" z template.ftl -->
    <#if section = "form">

        <div class="theralink-card">
            <h1 class="theralink-card-title">Zaloguj się</h1>
            <p class="theralink-card-subtitle">Witaj z powrotem w TheraLink</p>

            <#-- ${url.loginAction} = adres POST formularza (generowany przez Keycloak) -->
            <form id="kc-form-login" action="${url.loginAction}" method="post">

                <div class="form-group">
                    <label for="username" class="form-label">
                        <#-- ${msg("username")} = przetłumaczona etykieta -->
                        ${msg("email")}
                    </label>
                    <input
                        tabindex="1"
                        id="username"
                        name="username"
                        type="email"
                        class="form-control <#if messagesPerField.existsError('username')>is-invalid</#if>"
                        <#-- Wypełnij pole jeśli użytkownik już wpisał (po błędzie) -->
                        value="${(login.username!'')?html}"
                        autofocus
                        autocomplete="email"
                        placeholder="jan.kowalski@example.com"
                    />
                    <#if messagesPerField.existsError('username')>
                        <div class="invalid-feedback">
                            ${kcSanitize(messagesPerField.get('username'))?no_esc}
                        </div>
                    </#if>
                </div>

                <div class="form-group">
                    <div class="form-label-row">
                        <label for="password" class="form-label">${msg("password")}</label>
                        <#-- Link "Zapomniałem hasła" — tylko jeśli reset jest włączony w realm -->
                        <#if realm.resetPasswordAllowed>
                            <a href="${url.loginResetCredentialsUrl}" class="forgot-password-link">
                                Zapomniałeś hasła?
                            </a>
                        </#if>
                    </div>
                    <input
                        tabindex="2"
                        id="password"
                        name="password"
                        type="password"
                        class="form-control <#if messagesPerField.existsError('password')>is-invalid</#if>"
                        autocomplete="current-password"
                        placeholder="••••••••"
                    />
                    <#if messagesPerField.existsError('password')>
                        <div class="invalid-feedback">
                            ${kcSanitize(messagesPerField.get('password'))?no_esc}
                        </div>
                    </#if>
                </div>

                <#-- Checkbox "Zapamiętaj mnie" — tylko jeśli włączony w realm -->
                <#if realm.rememberMe>
                    <div class="form-check">
                        <input
                            tabindex="3"
                            id="rememberMe"
                            name="rememberMe"
                            type="checkbox"
                            class="form-check-input"
                            <#if login.rememberMe??>checked</#if>
                        />
                        <label for="rememberMe" class="form-check-label">
                            ${msg("rememberMe")}
                        </label>
                    </div>
                </#if>

                <button
                    tabindex="4"
                    type="submit"
                    class="btn btn-primary btn-block"
                >
                    Zaloguj się
                </button>

            </form>

            <#-- Link do rejestracji — tylko jeśli rejestracja włączona w realm -->
            <#if realm.registrationAllowed>
                <p class="register-link">
                    Nie masz konta?
                    <a href="${url.registrationUrl}">Zarejestruj się</a>
                </p>
            </#if>
        </div>

    </#if>

</@layout.registrationLayout>
```

### register.ftl — formularz rejestracji

```html
<#-- themes/theralink/login/register.ftl -->
<#import "template.ftl" as layout>

<@layout.registrationLayout>

    <#if section = "form">

        <div class="theralink-card">
            <h1 class="theralink-card-title">Załóż konto</h1>
            <p class="theralink-card-subtitle">Dołącz do TheraLink i zadbaj o swoje zdrowie</p>

            <#-- ${url.registrationAction} = adres POST formularza rejestracji -->
            <form id="kc-register-form" action="${url.registrationAction}" method="post">

                <div class="form-row">
                    <div class="form-group col-6">
                        <label for="firstName" class="form-label">${msg("firstName")}</label>
                        <input
                            type="text"
                            id="firstName"
                            name="firstName"
                            class="form-control <#if messagesPerField.existsError('firstName')>is-invalid</#if>"
                            value="${(register.formData.firstName!'')?html}"
                            autocomplete="given-name"
                            placeholder="Jan"
                        />
                        <#if messagesPerField.existsError('firstName')>
                            <div class="invalid-feedback">
                                ${kcSanitize(messagesPerField.get('firstName'))?no_esc}
                            </div>
                        </#if>
                    </div>

                    <div class="form-group col-6">
                        <label for="lastName" class="form-label">${msg("lastName")}</label>
                        <input
                            type="text"
                            id="lastName"
                            name="lastName"
                            class="form-control <#if messagesPerField.existsError('lastName')>is-invalid</#if>"
                            value="${(register.formData.lastName!'')?html}"
                            autocomplete="family-name"
                            placeholder="Kowalski"
                        />
                    </div>
                </div>

                <div class="form-group">
                    <label for="email" class="form-label">${msg("email")}</label>
                    <input
                        type="email"
                        id="email"
                        name="email"
                        class="form-control <#if messagesPerField.existsError('email')>is-invalid</#if>"
                        value="${(register.formData.email!'')?html}"
                        autocomplete="email"
                        placeholder="jan.kowalski@example.com"
                    />
                    <#if messagesPerField.existsError('email')>
                        <div class="invalid-feedback">
                            ${kcSanitize(messagesPerField.get('email'))?no_esc}
                        </div>
                    </#if>
                </div>

                <div class="form-group">
                    <label for="password" class="form-label">${msg("password")}</label>
                    <input
                        type="password"
                        id="password"
                        name="password"
                        class="form-control <#if messagesPerField.existsError('password','password-confirm')>is-invalid</#if>"
                        autocomplete="new-password"
                        placeholder="Minimum 8 znaków"
                    />
                </div>

                <div class="form-group">
                    <label for="password-confirm" class="form-label">${msg("passwordConfirm")}</label>
                    <input
                        type="password"
                        id="password-confirm"
                        name="password-confirm"
                        class="form-control <#if messagesPerField.existsError('password-confirm')>is-invalid</#if>"
                        autocomplete="new-password"
                        placeholder="Powtórz hasło"
                    />
                    <#if messagesPerField.existsError('password','password-confirm')>
                        <div class="invalid-feedback">
                            ${kcSanitize(messagesPerField.get('password-confirm'))?no_esc}
                        </div>
                    </#if>
                </div>

                <button type="submit" class="btn btn-primary btn-block">
                    Zarejestruj się
                </button>

            </form>

            <p class="register-link">
                Masz już konto? <a href="${url.loginUrl}">Zaloguj się</a>
            </p>
        </div>

    </#if>

</@layout.registrationLayout>
```

### error.ftl — strona błędu

```html
<#-- themes/theralink/login/error.ftl -->
<#import "template.ftl" as layout>

<@layout.registrationLayout displayMessage=false>

    <#if section = "form">
        <div class="theralink-card theralink-error-card">
            <div class="error-icon">⚠️</div>
            <h1 class="theralink-card-title">Wystąpił błąd</h1>

            <#-- ${message.summary} = opis błędu od Keycloak -->
            <p class="error-message">${kcSanitize(message.summary)?no_esc}</p>

            <#if client?? && client.baseUrl?has_content>
                <a href="${client.baseUrl}" class="btn btn-outline">
                    Wróć do aplikacji
                </a>
            <#else>
                <a href="${url.loginUrl}" class="btn btn-outline">
                    Wróć do logowania
                </a>
            </#if>
        </div>
    </#if>

</@layout.registrationLayout>
```

### theralink.css — placeholder stylów

```css
/* themes/theralink/login/resources/css/theralink.css */
/*
 * Ten plik zawiera podstawową strukturę CSS dla motywu TheraLink.
 * Dostosuj kolory, czcionki i style do Twojego designu.
 *
 * Zmienne CSS do personalizacji:
 */
:root {
    --color-primary: #4A90D9;       /* Główny kolor — zmień na kolor marki */
    --color-primary-dark: #2E6DB4;  /* Ciemniejszy wariant */
    --color-background: #F7F9FC;    /* Tło strony */
    --color-card: #FFFFFF;          /* Tło karty formularza */
    --color-text: #1A1A2E;         /* Kolor tekstu */
    --color-text-muted: #6B7280;   /* Szary tekst (podpisy) */
    --color-error: #E53E3E;        /* Czerwony dla błędów */
    --color-success: #38A169;       /* Zielony dla sukcesu */
    --color-border: #E2E8F0;       /* Kolor obramowania */
    --font-family: 'Inter', -apple-system, sans-serif;
    --border-radius: 8px;
    --shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

/* ── Layout ─────────────────────────────────────────── */
body {
    background-color: var(--color-background);
    font-family: var(--font-family);
    margin: 0;
    padding: 0;
}

.theralink-auth-wrapper {
    display: flex;
    min-height: 100vh;
}

.theralink-sidebar {
    /* Panel boczny z logo — ukryj na mobile */
    width: 400px;
    background: linear-gradient(135deg, var(--color-primary), var(--color-primary-dark));
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    padding: 40px;
    color: white;
}

.theralink-form-section {
    flex: 1;
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 40px 20px;
}

/* ── Karta formularza ───────────────────────────────── */
.theralink-card {
    background: var(--color-card);
    border-radius: var(--border-radius);
    box-shadow: var(--shadow);
    padding: 40px;
    width: 100%;
    max-width: 420px;
}

.theralink-card-title {
    font-size: 1.75rem;
    font-weight: 700;
    color: var(--color-text);
    margin: 0 0 8px;
}

.theralink-card-subtitle {
    color: var(--color-text-muted);
    margin: 0 0 32px;
}

/* ── Formularze ─────────────────────────────────────── */
.form-group {
    margin-bottom: 20px;
}

.form-label {
    display: block;
    font-size: 0.875rem;
    font-weight: 500;
    color: var(--color-text);
    margin-bottom: 6px;
}

.form-label-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 6px;
}

.form-control {
    width: 100%;
    padding: 10px 14px;
    border: 1px solid var(--color-border);
    border-radius: var(--border-radius);
    font-size: 1rem;
    color: var(--color-text);
    background: white;
    box-sizing: border-box;
    transition: border-color 0.2s;
}

.form-control:focus {
    outline: none;
    border-color: var(--color-primary);
    box-shadow: 0 0 0 3px rgba(74, 144, 217, 0.15);
}

.form-control.is-invalid {
    border-color: var(--color-error);
}

.invalid-feedback {
    font-size: 0.8125rem;
    color: var(--color-error);
    margin-top: 4px;
}

/* ── Przyciski ──────────────────────────────────────── */
.btn {
    display: inline-block;
    padding: 12px 24px;
    border-radius: var(--border-radius);
    font-size: 1rem;
    font-weight: 500;
    cursor: pointer;
    border: none;
    text-decoration: none;
    text-align: center;
    transition: background-color 0.2s;
}

.btn-primary {
    background-color: var(--color-primary);
    color: white;
}

.btn-primary:hover {
    background-color: var(--color-primary-dark);
}

.btn-block {
    width: 100%;
    display: block;
    margin-top: 24px;
}

.btn-outline {
    border: 1px solid var(--color-border);
    background: transparent;
    color: var(--color-text);
}

/* ── Alerty ─────────────────────────────────────────── */
.alert {
    padding: 12px 16px;
    border-radius: var(--border-radius);
    margin-bottom: 20px;
    font-size: 0.875rem;
    display: flex;
    align-items: center;
    gap: 8px;
}

.alert-error {
    background-color: #FEF2F2;
    color: var(--color-error);
    border: 1px solid #FECACA;
}

.alert-success {
    background-color: #F0FFF4;
    color: var(--color-success);
    border: 1px solid #C6F6D5;
}

/* ── Linki ──────────────────────────────────────────── */
.forgot-password-link,
.register-link a {
    color: var(--color-primary);
    font-size: 0.875rem;
    text-decoration: none;
}

.register-link {
    text-align: center;
    margin-top: 24px;
    font-size: 0.875rem;
    color: var(--color-text-muted);
}

/* ── Mobile ─────────────────────────────────────────── */
@media (max-width: 768px) {
    .theralink-sidebar {
        display: none;
    }

    .theralink-card {
        box-shadow: none;
        padding: 24px 16px;
    }
}
```

### Rejestracja motywu w Admin Console

Po uruchomieniu Keycloak z obrazu Docker (motyw jest już skopiowany):

1. Zaloguj się: `http://localhost:8080` (admin / admin)
2. Wybierz realm: `theralink`
3. Realm settings → Themes
4. Login theme: `theralink`
5. Save

Lub automatycznie przez `realm-export.json` (pole `"loginTheme": "theralink"`).

### Tryb deweloperski FTL — podgląd zmian bez restartu

Keycloak cachuje szablony FTL. W trybie deweloperskim możesz wyłączyć cache:

```bash
# Uruchom Keycloak z wyłączonym cache (TYLKO DEV!)
docker run -e KC_SPI_THEME_STATIC_MAX_AGE=-1 \
           -e KC_SPI_THEME_CACHE_THEMES=false \
           -e KC_SPI_THEME_CACHE_TEMPLATES=false \
           ...
```

Lub w `docker-compose.yml` dodaj do sekcji `environment`:
```yaml
KC_SPI_THEME_STATIC_MAX_AGE: "-1"
KC_SPI_THEME_CACHE_THEMES: "false"
KC_SPI_THEME_CACHE_TEMPLATES: "false"
```

---

## 5. Integracja z Angular (keycloak-angular)

### Instalacja

```bash
# W repo theralink-frontend
npm install keycloak-angular keycloak-js

# Sprawdź kompatybilność wersji:
# keycloak-angular 15.x → Angular 17+, keycloak-js 23+
# keycloak-angular 16.x → Angular 18+, keycloak-js 25+
```

### Inicjalizacja w app.config.ts

**Teoria:** `APP_INITIALIZER` to token Angular DI, który pozwala wykonać kod asynchroniczny **przed** renderowaniem jakiegokolwiek komponentu. Keycloak musi być zainicjalizowany zanim Angular spróbuje dostać się do chronionych tras.

```typescript
// src/app/app.config.ts
import { ApplicationConfig, APP_INITIALIZER } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { KeycloakService } from 'keycloak-angular';
import { keycloakBearerInterceptor } from './core/interceptors/keycloak-bearer.interceptor';
import { routes } from './app.routes';
import { environment } from '../environments/environment';

// Funkcja inicjalizująca Keycloak — wykonuje się przed startem Angular
function initializeKeycloak(keycloak: KeycloakService) {
  return () =>
    keycloak.init({
      config: {
        url: environment.keycloakUrl,         // 'http://localhost:8080'
        realm: 'theralink',
        clientId: 'theralink-angular',
      },
      initOptions: {
        // onLoad: 'check-sso' — sprawdź czy użytkownik jest zalogowany
        // (nie wymusza logowania od razu — użytkownik może przeglądać publiczne strony)
        onLoad: 'check-sso',

        // silentCheckSsoRedirectUri — strona do sprawdzenia SSO w tle (iframe)
        // Musi istnieć plik: src/assets/silent-check-sso.html
        silentCheckSsoRedirectUri: window.location.origin + '/assets/silent-check-sso.html',

        // PKCE — wymagane dla public clients (Angular nie ma client secret)
        pkceMethod: 'S256',
      },
    });
}

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),

    // HTTP client z interceptorem który dodaje Bearer token
    provideHttpClient(
      withInterceptors([keycloakBearerInterceptor])
    ),

    // Rejestracja KeycloakService
    KeycloakService,

    // APP_INITIALIZER — Keycloak musi być zainicjalizowany przed startem Angular
    {
      provide: APP_INITIALIZER,
      useFactory: initializeKeycloak,
      multi: true,
      deps: [KeycloakService],
    },
  ],
};
```

### Plik silent-check-sso.html

Utwórz plik `src/assets/silent-check-sso.html`:

```html
<!DOCTYPE html>
<html>
<body>
  <script>
    // Ten plik obsługuje sprawdzenie SSO w tle (w ukrytym iframe)
    // Angular ładuje go żeby sprawdzić czy użytkownik jest zalogowany
    // bez przekierowania na stronę Keycloak
    parent.postMessage(location.href, location.origin);
  </script>
</body>
</html>
```

### AuthGuard — zabezpieczenie tras

```typescript
// src/app/core/guards/auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { KeycloakService } from 'keycloak-angular';

// Nowoczesny functional guard (Angular 15+)
export const authGuard: CanActivateFn = async () => {
  const keycloak = inject(KeycloakService);
  const router = inject(Router);

  const isLoggedIn = await keycloak.isLoggedIn();

  if (isLoggedIn) {
    return true;
  }

  // Użytkownik nie jest zalogowany — przekieruj na stronę logowania Keycloak
  await keycloak.login({
    redirectUri: window.location.origin + '/dashboard',
  });

  return false;
};

// Guard dla konkretnej roli
export const psychologistGuard: CanActivateFn = async () => {
  const keycloak = inject(KeycloakService);

  const isLoggedIn = await keycloak.isLoggedIn();
  if (!isLoggedIn) {
    await keycloak.login();
    return false;
  }

  // Sprawdź rolę realm
  const hasRole = keycloak.isUserInRole('PSYCHOLOGIST');
  if (!hasRole) {
    // Przekieruj na stronę "Brak dostępu"
    const router = inject(Router);
    return router.createUrlTree(['/unauthorized']);
  }

  return true;
};
```

### Konfiguracja tras z Guard

```typescript
// src/app/app.routes.ts
import { Routes } from '@angular/router';
import { authGuard, psychologistGuard } from './core/guards/auth.guard';

export const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./features/home/home.component')
      .then(m => m.HomeComponent),
  },
  {
    path: 'dashboard',
    canActivate: [authGuard],   // wymaga zalogowania
    loadComponent: () => import('./features/dashboard/dashboard.component')
      .then(m => m.DashboardComponent),
  },
  {
    path: 'psychologist/panel',
    canActivate: [psychologistGuard],  // wymaga roli PSYCHOLOGIST
    loadComponent: () => import('./features/psychologist/panel/panel.component')
      .then(m => m.PanelComponent),
  },
  {
    path: 'unauthorized',
    loadComponent: () => import('./features/unauthorized/unauthorized.component')
      .then(m => m.UnauthorizedComponent),
  },
];
```

### Interceptor HTTP — automatyczne dodawanie Bearer token

```typescript
// src/app/core/interceptors/keycloak-bearer.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { KeycloakService } from 'keycloak-angular';
import { from, switchMap } from 'rxjs';
import { environment } from '../../../environments/environment';

// Functional interceptor (Angular 15+)
// Automatycznie dodaje "Authorization: Bearer TOKEN" do żądań do naszego API
export const keycloakBearerInterceptor: HttpInterceptorFn = (req, next) => {
  const keycloak = inject(KeycloakService);

  // Dodawaj token TYLKO do żądań do naszego API Gateway
  // Nie wysyłaj tokenu do zewnętrznych API (np. Stripe.js)
  if (!req.url.startsWith(environment.apiGatewayUrl)) {
    return next(req);
  }

  return from(keycloak.getToken()).pipe(
    switchMap(token => {
      if (token) {
        const cloned = req.clone({
          setHeaders: {
            Authorization: `Bearer ${token}`,
          },
        });
        return next(cloned);
      }
      return next(req);
    })
  );
};
```

### Użycie KeycloakService w komponencie

```typescript
// Przykład użycia w komponencie
import { Component, OnInit, inject, signal } from '@angular/core';
import { KeycloakService } from 'keycloak-angular';

@Component({
  selector: 'app-user-menu',
  template: `
    <div *ngIf="isLoggedIn()">
      <span>Witaj, {{ userName() }}</span>
      <span class="role-badge">{{ userRole() }}</span>
      <button (click)="logout()">Wyloguj</button>
    </div>
    <button *ngIf="!isLoggedIn()" (click)="login()">Zaloguj się</button>
  `,
})
export class UserMenuComponent implements OnInit {
  private keycloak = inject(KeycloakService);

  // Angular Signals dla lokalnego stanu komponentu
  isLoggedIn = signal(false);
  userName = signal('');
  userRole = signal('');

  async ngOnInit() {
    this.isLoggedIn.set(await this.keycloak.isLoggedIn());

    if (this.isLoggedIn()) {
      // Dane z Keycloak — id_token
      const profile = await this.keycloak.loadUserProfile();
      this.userName.set(`${profile.firstName} ${profile.lastName}`);

      // Sprawdź rolę
      if (this.keycloak.isUserInRole('PSYCHOLOGIST')) {
        this.userRole.set('Psycholog');
      } else if (this.keycloak.isUserInRole('CLIENT')) {
        this.userRole.set('Pacjent');
      }
    }
  }

  login() {
    this.keycloak.login();
  }

  logout() {
    // Wylogowanie z Keycloak (SSO — wylogowuje ze wszystkich aplikacji w realm)
    this.keycloak.logout(window.location.origin);
  }

  // Dostęp do surowego tokenu JWT (np. żeby zdekodować inne claims)
  async getToken(): Promise<string> {
    return this.keycloak.getToken();
  }
}
```

### environment.ts

```typescript
// src/environments/environment.ts (dev)
export const environment = {
  production: false,
  apiGatewayUrl: 'http://localhost:8090',
  keycloakUrl: 'http://localhost:8080',
};

// src/environments/environment.prod.ts (produkcja)
export const environment = {
  production: true,
  apiGatewayUrl: 'https://api.theralink.pl',
  keycloakUrl: 'https://auth.theralink.pl',
};
```

---

## 6. Weryfikacja i troubleshooting

### Test lokalny — uzyskaj token przez curl

```bash
# 1. Sprawdź czy Keycloak działa
curl http://localhost:8080/realms/theralink/.well-known/openid-configuration
# Powinno zwrócić duży JSON z endpointami

# 2. Uzyskaj token (Resource Owner Password — tylko do testów!)
# Na produkcji NIE używaj tego flow — tylko Authorization Code + PKCE
curl -X POST http://localhost:8080/realms/theralink/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=theralink-angular" \
  -d "grant_type=password" \
  -d "username=jan@example.com" \
  -d "password=SecretPass123" \
  -d "scope=openid"

# Zwraca: { "access_token": "eyJ...", "refresh_token": "...", ... }

# 3. Zdekoduj token (bez weryfikacji — tylko do debugowania)
# Weź część po pierwszej kropce (payload) i zdekoduj Base64
echo "eyJ..." | base64 -d 2>/dev/null | python3 -m json.tool

# 4. Wywołaj chroniony endpoint Spring Boot
ACCESS_TOKEN="eyJhbGc..."
curl -H "Authorization: Bearer $ACCESS_TOKEN" \
  http://localhost:8081/clients/me
```

### Najczęstsze błędy

**1. CORS error w przeglądarce**

Symptom: `Access to XMLHttpRequest at 'http://localhost:8080/...' blocked by CORS policy`

Przyczyna: Angular (`localhost:4200`) nie jest w dozwolonych Web Origins klienta Keycloak.

Rozwiązanie:
```
Admin Console → Clients → theralink-angular → Settings
Web Origins: http://localhost:4200  ← dodaj!
Save
```

---

**2. redirect_uri mismatch**

Symptom: `Error: redirect_uri mismatch` po próbie logowania

Przyczyna: URL do którego Keycloak ma przekierować po logowaniu nie jest na liście dozwolonych.

Rozwiązanie:
```
Admin Console → Clients → theralink-angular → Settings
Valid redirect URIs: http://localhost:4200/*  ← gwiazdka na końcu!
Save
```

---

**3. Invalid token — clock skew**

Symptom: Spring Boot zwraca `401 Unauthorized` mimo ważnego tokenu. Log: `JWT expired at...`

Przyczyna: Zegarki serwera Spring Boot i Keycloak się nie zgadzają (różnica > 30 sekund).

Rozwiązanie: Zsynchronizuj czas systemowy przez NTP lub dodaj tolerancję:
```yaml
# application.yml w Spring Boot serwisie
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Tolerancja 60 sekund różnicy zegarów
          jws-algorithms: RS256
```

---

**4. Role nie pojawiają się w JWT**

Symptom: `realm_access.roles` jest pusty lub nie istnieje w tokenie.

Przyczyna: Brak Protocol Mapper dla ról w konfiguracji client scope.

Rozwiązanie: Dodaj mapper jak opisano w sekcji [3. Role i atrybuty użytkownika](#3-role-i-atrybuty-użytkownika).

Weryfikacja — zdekoduj token i sprawdź czy zawiera:
```json
{
  "realm_access": {
    "roles": ["CLIENT", "offline_access"]
  }
}
```

---

**5. Motyw FTL nie jest widoczny**

Symptom: Keycloak nadal pokazuje domyślny motyw.

Możliwe przyczyny:
1. Motyw nie jest wybrany w realm settings (Admin Console → Realm settings → Themes)
2. Plik jest w złej ścieżce — musi być w `/opt/keycloak/themes/theralink/login/`
3. Cache FTL — uruchom z wyłączonym cache (patrz sekcja "Tryb deweloperski FTL")
4. Błąd składni w `.ftl` — sprawdź logi Keycloak: `docker logs theralink-keycloak`

```bash
# Sprawdź logi Keycloak
docker logs theralink-keycloak 2>&1 | grep -i "theme\|error\|warn"
```

---

### Eksport konfiguracji realm do pliku

Po skonfigurowaniu realm przez Admin Console, wyeksportuj go do `realm-export.json`:

```bash
# Eksport przez Keycloak CLI wewnątrz kontenera
docker exec theralink-keycloak \
  /opt/keycloak/bin/kc.sh export \
  --realm theralink \
  --file /tmp/realm-export.json

# Skopiuj plik na hosta
docker cp theralink-keycloak:/tmp/realm-export.json ./realm-export.json
```

Zapisz `realm-export.json` do repozytorium `theralink-keycloak` — będzie automatycznie importowany przy starcie kontenera (`--import-realm`).

---

## Powiązane

- [[index]] — Mapa dokumentacji
- [[backend-migration]] — Spring Boot serwisy korzystające z Keycloak JWT
- [[frontend-migration]] — Angular + keycloak-angular integracja
- [[payment-service]] — Serwis płatności (też używa Keycloak do autoryzacji)
