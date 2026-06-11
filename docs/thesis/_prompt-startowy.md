# Prompt startowy do nowej sesji — pisanie rozdziału pracy inżynierskiej

> Skopiuj poniższy blok, podmień `{ROZDZIAŁ X}` i `{LISTA TEMATÓW}` i wklej do nowej sesji Claude Code.
> Sesja powinna być uruchomiona w katalogu `/Users/desirecutieqb/IdeaProjects/TheraLink`.

---

## 📋 Wersja A — pojedynczy rozdział

```
Piszę pracę inżynierską na ANS Elbląg (Instytut Informatyki Stosowanej im. Krzysztofa Brzeskiego) o migracji TheraLink z monolitu na mikrousługi.

KONTEKST DO PRZECZYTANIA NA START:
1. CLAUDE.md — konwencje projektu i stack
2. docs/thesis/wstep.md + wszystkie docs/thesis/rozdzial-*.md — co już napisane (spójność terminologii, brak duplikacji)
3. MEMORY.md — scope decisions i konwencje

ZASADY FORMATU:
- Skill kod-do-pracy: bloki `## [ROZDZIAŁ: ...]` z podrozdziałami
- Polski styl bezosobowy ("zaprojektowano", "przeanalizowano")
- Screeny: `> 📸 **[SCREEN DO DODANIA]**` z polami: Co pokazać / Sugerowany podpis / Źródło
- Pliki: zapisuj do docs/thesis/rozdzial-NN-{slug}.md
- Rozdziały migracyjne (3, 4, 5, 7) MUSZĄ mieć porównanie przed/po — WPLECIONE w treść:
  - w każdym podrozdziale technicznym krótka mini-tabelka lub akapit "Przed / Po"
  - na końcu rozdziału jeden krótki podrozdział-podsumowanie ze zbiorczą tabelą i wnioskami
  - NIE rób jednego dużego podrozdziału "Porównanie" — porównanie ma być w kontekście każdej kwestii

SCOPE PRACY:
- OUT: notification-service, powiadomienia email (wykluczone z zakresu)
- IN: Stripe (płatności), Zoom (linki do spotkań)
- Payment service to OSOBNE, restricted repo (PCI-DSS) — opisuj jako oddzielne
- Stack target: Spring Boot 4.0.3, Java 25/21, Angular 21, MongoDB 7, Keycloak 25, Kafka 7.6, Azure AKS

STYL:
- Nigdy nie commituj ani nie pushuj bez polecenia
- Krótkie, konkretne odpowiedzi; nie streszczaj co zrobiłeś po zapisaniu pliku

WORKFLOW:
1. Sprawdź sekcję "INPUT" niżej:
   - jeśli są tam gotowe PODROZDZIAŁY (oznaczone 4.1, 4.2…) → pisz rozdział od razu, bez kroku propozycji struktury
   - jeśli są tylko luźne TEMATY w bulletach → najpierw zaproponuj strukturę podrozdziałów (1-2 zdania na każdy + liczba planowanych screenów) i czekaj na akceptację, potem pisz
2. Pisz pełny rozdział i zapisz do pliku docs/thesis/rozdzial-NN-{slug}.md
3. Na koniec wypisz 3-5 kluczowych decyzji z tej sesji, które warto zapisać do memory

ROZDZIAŁ DO NAPISANIA: {ROZDZIAŁ X — TYTUŁ}

INPUT (wybierz jedną opcję, drugą usuń):

[OPCJA A — gotowe podrozdziały, Claude pisze od razu]
PODROZDZIAŁY:
{wklej tu blok "Proponowane podrozdziały" z pliku docs/thesis/_prompt-startowy.md
 dla danego rozdziału — np. 4.1, 4.2, 4.3… razem z opisami "Co opisać" i "Przed/Po"}

[OPCJA B — luźne tematy, Claude najpierw zaproponuje strukturę]
KLUCZOWE TEMATY DO POKRYCIA:
{wklej listę tematów w bulletach — każdy bullet to potencjalny podrozdział}
```

---

## 📋 Wersja B — grupa rozdziałów (gdy są tematycznie powiązane, np. 8+9+10)

```
Piszę pracę inżynierską na ANS Elbląg o migracji TheraLink z monolitu na mikrousługi.

[Wstaw cały blok KONTEKST + ZASADY FORMATU + SCOPE + STYL + WORKFLOW z wersji A]

GRUPA ROZDZIAŁÓW DO NAPISANIA W TEJ SESJI:
- Rozdział X — {tytuł}
- Rozdział Y — {tytuł}
- Rozdział Z — {tytuł}

KOLEJNOŚĆ: piszemy po jednym, czekam na "ok" przed kolejnym.
Każdy rozdział = osobny plik docs/thesis/rozdzial-NN-{slug}.md
```

---

## 🗂 Planowany podział na sesje

| Sesja | Rozdziały | Powód grupowania |
|---|---|---|
| ✅ 1 | Wstęp + 1 + 2 + 3 | Analiza monolitu + projekt + migracja backendu (zrobione) |
| 2 | 4 | Migracja frontendu (React/Next.js → Angular) — osobny stack |
| 3 | 5 | Autoryzacja (Cognito → Keycloak) — osobny temat |
| 4 | 6 + 7 | Kafka + Migracja bazy (PostgreSQL → MongoDB) — komunikacja i dane |
| 5 | 8 + 9 + 10 | Docker + Kubernetes + Azure + porównanie środowisk — infra |
| 6 | 11 | Płatności Stripe (osobny restricted-access kontekst) |
| 7 | 12 + 13 | Testy + dokumentacja — domknięcie |

---

## 📝 Lista kluczowych tematów per rozdział (do wklejenia w sekcję "KLUCZOWE TEMATY")

### Rozdział 4 — Migracja frontendu (React/Next.js → Angular)

**Proponowane podrozdziały (gotowa struktura — wklej w sekcję KLUCZOWE TEMATY jako "PODROZDZIAŁY"):**

- **4.1. Filozofia frameworka — biblioteka vs framework opinionated**
  Co opisać: React jako biblioteka (decyzje o routerze, state, DI po stronie developera) vs Angular jako "batteries-included" framework. Dependency Injection, RxJS jako standardowa warstwa async. Konsekwencje dla architektury.
  Przed/Po: krótki akapit porównujący swobodę i koszt utrzymania.

- **4.2. Struktura projektu i routing**
  Co opisać: Next.js App Router (file-based routing, server vs client components) → Angular standalone components + lazy-loaded routes (`loadChildren`). Czemu lazy loading jest kluczowy dla mikrousług.
  Przed/Po: mini-tabela — struktura folderów + sposób definiowania trasy.

- **4.3. Zarządzanie stanem aplikacji**
  Co opisać: Redux Toolkit + RTK Query (slices, queries, cache) → NGXS (akcje, stany, selektory) + Angular Signals do stanu lokalnego komponentu. Kiedy używać NGXS, kiedy Signals.
  Przed/Po: tabela — stan globalny / lokalny / kasowanie cache.

- **4.4. Komunikacja z API i interceptory HTTP**
  Co opisać: w monolicie ręczne ustawianie `Authorization` headera w każdym fetch. W Angularze `HttpInterceptor` (jwt.interceptor.ts) — JWT, refresh token, error handling, retry — w jednym miejscu.
  Przed/Po: fragment Express fetch vs Angular HttpClient z interceptorem.

- **4.5. System komponentów UI — konsolidacja**
  Co opisać: monolit ma TRZY systemy (Material UI + Radix + shadcn) — niespójność wizualna, duplikacja zależności. W Angularze: Angular Material jako baza + własne komponenty domenowe (np. `PsychologistCard`, `AppointmentSlotPicker`).
  Przed/Po: tabela — liczba zależności, design tokens, spójność.

- **4.6. Internacjonalizacja (PL/EN)**
  Co opisać: Transloco — ngx-translate-style API, lazy-loaded translation files per moduł, scopes. Aplikacja docelowo polska, ale architektura przygotowana na EN.
  Przed/Po: brak i18n w monolicie → pełne wsparcie wielojęzyczne.

- **4.7. Integracja z Keycloak**
  Co opisać: `keycloak-angular` — silent SSO, automatyczne dołączanie tokenu do żądań, guard'y route'ów (`AuthGuard`, `RoleGuard`). W monolicie: AWS Amplify + ręczna obsługa sesji w Redux.
  Przed/Po: tabela — flow logowania, miejsce przechowywania tokenu, refresh.

- **4.8. Konfiguracja środowisk i build**
  Co opisać: `environment.ts` / `environment.prod.ts` zamiast `.env` (Next.js). File replacement w `angular.json`. Bezpieczeństwo — zmienne build-time vs runtime.
  Przed/Po: gdzie są klucze, jak wskazywany jest API gateway URL.

- **4.9. Podsumowanie migracji — zbiorcze porównanie**
  Co opisać: krótka tabela zbiorcza po wszystkich wymiarach (stack, bundle size, czas startu, liczba zależności, testy), 2-3 wnioski końcowe. Tu obowiązkowa duża tabela "Przed / Po".

Planowane screeny: ~6-8 (struktura projektu, lazy routing config, Redux DevTools vs NGXS DevTools, system UI w monolicie vs Angular Material, flow logowania Keycloak, zrzut z aplikacji Angular).

### Rozdział 5 — System autoryzacji (Cognito → Keycloak)
- Teoria: OAuth 2.0 + OIDC + JWT (RS256, JWKS)
- Dlaczego Keycloak: self-hosted, kontrola nad rolami (problem z Cognito #11)
- Architektura: realm `theralink`, klienci (frontend public, backend confidential)
- Flow: Authorization Code + PKCE S256
- Tokeny: access (15 min) + refresh (7 dni), claims, scopes
- Spring Security OAuth2 Resource Server — walidacja JWT po JWKS
- `@AuthenticationPrincipal Jwt jwt` → `jwt.getSubject()` jako `keycloakId`
- Custom theme i realm-export.json
- Porównanie przed/po z monolitem (jwt.decode vs jwt.verify, brak ról vs realm roles)

### Rozdział 6 — Komunikacja asynchroniczna (Apache Kafka)
- Teoria: event-driven architecture, brokery, topiki, partycje, consumer groups
- Dlaczego Kafka (vs RabbitMQ, vs synchroniczne REST)
- Konwencja topiców: `theralink.{domain}.{event}`
- Producer: Spring Kafka `KafkaTemplate`, serializacja JSON
- Consumer: `@KafkaListener`, acknowledge mode, error handling
- Use case: `payment.completed` → appointment-service aktualizuje status appointmenta
- Schemat eventów w `theralink-contracts` (Maven library)
- Dev: docker-compose (Kafka 7.6 + Zookeeper) / Prod: Azure Event Hubs (Kafka protocol, no code changes)

### Rozdział 7 — Migracja bazy danych (PostgreSQL → MongoDB)
- Teoria: relacyjne vs dokumentowe, ACID vs BASE, schema-on-read
- Database per service — granice bounded context
- 4 bazy: `theralink-users`, `theralink-appointments`, `theralink-psychologist-schedules`, `theralink-payments`
- Modelowanie: embedding vs referencing, agregaty DDD
- Spring Data MongoDB: `@Document`, `MongoRepository`, queries
- Brak joinów — jak to obejść (denormalizacja, eventy Kafka)
- Migracja danych z PostgreSQL (skrypt / ETL)
- Porównanie przed/po (schema, relacje, indeksy)
- Dev: MongoDB 7.0 (docker) / Prod: Azure Cosmos DB (MongoDB API)

### Rozdział 8 — Infrastruktura (Docker + Kubernetes)
- Teoria: konteneryzacja, orkiestracja, deklaratywna konfiguracja
- Dockerfile per serwis (multi-stage build, JRE slim)
- docker-compose.yml dev — wszystkie serwisy + MongoDB + Kafka + Keycloak
- Kubernetes: Pody, Deploymenty, Services, ConfigMaps, Secrets, Ingress
- Helm charts — szablonizacja per środowisko
- CSI Secret Store Driver dla Azure Key Vault
- Healthchecki: liveness + readiness probes (Spring Actuator)
- HPA (Horizontal Pod Autoscaler)

### Rozdział 9 — Wdrożenie Azure
- AKS (Azure Kubernetes Service) — managed K8s
- Resource group `rg-theralink`, region `polandcentral`
- Azure Container Registry `acrtheralink.azurecr.io`
- Azure Cosmos DB (MongoDB API) per serwis
- Azure Event Hubs (Kafka protocol)
- Azure Key Vault + CSI Driver
- Application Gateway / Ingress Controller
- Logi: Azure Monitor + Log Analytics
- Koszt vs lokalne docker-compose

### Rozdział 10 — Środowiska prod vs lokalne
- Dev: docker-compose, `.env`, mock danych
- Prod: AKS, Key Vault, prawdziwe usługi Azure
- Konfiguracja: Spring Profiles (`dev`, `prod`)
- Zarządzanie sekretami
- CI/CD pipeline (jeśli w zakresie)
- Tabela porównawcza zasobów dev vs prod

### Rozdział 11 — Płatności (Stripe)
- Teoria: PaymentIntent flow, idempotency, webhook bezpieczeństwo
- Architektura: thera-payment-service jako osobne restricted repo (PCI-DSS)
- Stripe SDK (Java)
- Webhook walidacja: HMAC-SHA256 podpis
- Event: `theralink.payment.completed` → appointment-service confirm
- Klucze przez env vars / Azure Key Vault — nigdy hardcoded
- Refundy i obsługa błędów

### Rozdział 12 — Testy funkcjonalne i wydajnościowe
- Unit testy (JUnit 5 + Mockito)
- Integracyjne (Testcontainers — MongoDB, Kafka)
- E2E (Cypress/Playwright dla Angular)
- Testy wydajnościowe (JMeter / k6) — porównanie monolit vs mikrousługi
- Pokrycie testami w monolicie: 0% — startujemy od zera

### Rozdział 13 — Dokumentacja techniczna
- OpenAPI / Swagger UI per serwis
- Diagramy architektury (Mermaid)
- README per repo
- Postman collection
- Onboarding nowego dewelopera

---

## 🔄 Po zakończeniu sesji

Na końcu każdej sesji poproś:

```
Wypisz krótko (3-5 punktów) najważniejsze decyzje techniczne lub zmiany scope z tej sesji, które warto zapisać do MEMORY.md, żeby następne sesje miały aktualny kontekst.
```

Potem te decyzje wrzuć do memory ręcznie albo poproś, żeby je zapisał.
