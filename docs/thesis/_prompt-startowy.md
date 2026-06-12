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
- IN: Stripe (płatności)
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

> ROZDZIAŁ MIGRACYJNY: porównanie przed/po WPLECIONE w treść (każdy podrozdział mini-tabelka, na końcu zbiorczy podrozdział).
> Materiały źródłowe: [[azure-aks-deployment]], [[azure-deployment-metrics]], [[infrastructure]], `thera-infrastructure/helm/theralink/`, `thera-infrastructure/scripts/`, Dockerfile w każdym repo serwisu, `realm-export.json`. **Realne metryki** z świeżego deploy w `docs/azure-deployment-metrics.md` (sekcje 7, 8, 9 — rozmiary obrazów, czasy buildów, czasy startu podów).
> **Screeny:** 8 sztuk (Rys. 8.1–8.8). Pełne instrukcje w [[_screen-checklist-8-9-10]]. Wstawiać przez `> 📸 **[SCREEN DO DODANIA]**` zgodnie z numeracją.

**Proponowane podrozdziały:**

8.1. **Konteneryzacja — Docker** (teoria + przed/po)
- Co opisać: filozofia kontenerów (izolacja procesów + system plików, brak hypervisora jak w VM), warstwy obrazu, Dockerfile jako deklaratywny przepis, multi-stage builds dla małych obrazów produkcyjnych
- Przed/Po: monolit uruchamiany przez `npm start` lokalnie vs. każdy mikroserwis spakowany w obraz Docker z dokładnie określonym JRE/Node runtime
- ~3-4 listingi: Dockerfile thera-rest-service (multi-stage Java 25), Dockerfile thera-ui (multi-stage Node 22 + nginx), Dockerfile thera-keycloak (z `kc.sh build --db postgres`)
- **Screen 8.1** (warstwy obrazu): `docs/screens/rozdz-08/8.1-docker-history.png`

8.2. **Środowisko developerskie — Docker Compose**
- Co opisać: jak `docker-compose.yml` łączy serwisy w sieć `theralink-network`, komunikacja przez nazwy kontenerów (DNS), volumes dla persystencji, healthchecki, `depends_on`
- Przed/Po: lokalnie odpalane 4 procesy z różnymi portami vs. `docker-compose up` jedno polecenie postawia cały stack (Keycloak + Postgres + MongoDB + Kafka + Zookeeper + kafka-ui)
- Listingi: `thera-infrastructure/docker-compose/docker-compose.yml`, sekcja Kafka z dwoma listenerami (INTERNAL + EXTERNAL), `.env.example`
- **Screen 8.2** (docker-compose ps): `docs/screens/rozdz-08/8.2-docker-compose-ps.png`

8.3. **Orkiestracja — Kubernetes (teoria)**
- Co opisać: dlaczego Kubernetes (deklaratywność, self-healing, rolling updates, skalowanie), słownik pojęć: Pod, Deployment, StatefulSet, Service, Ingress, ConfigMap, Secret, PersistentVolume
- Architektura klastra: control plane (API server, scheduler, etcd) vs. worker nodes
- ~1-2 diagramy: relacja Pod ↔ Deployment ↔ Service ↔ Ingress
- **Screen 8.3** (k9s — pods view): `docs/screens/rozdz-08/8.3-k9s-pods.png`

8.4. **Manifesty Kubernetes dla TheraLink**
- Co opisać: stworzyliśmy manifesty per serwis — Deployment + Service + SecretProviderClass. StatefulSet dla Postgres (Keycloak DB) z `volumeClaimTemplates` (PVC tworzony automatycznie per replika)
- Listingi: `helm/theralink/templates/keycloak-deployment.yaml` (env vars, resources, probes), `keycloak-db-statefulset.yaml` (volumeClaimTemplates), `keycloak-service.yaml` (ClusterIP), `keycloak-db-service.yaml` (headless service)
- Przed/Po: docker-compose `depends_on: service_healthy` vs. Kubernetes readiness probes które blokują Service endpoint do momentu gotowości aplikacji

8.5. **Helm — parametryzacja manifestów**
- Co opisać: problem 5 serwisów × 3 środowiska = 15 plików YAML do utrzymania; Helm rozwiązuje to przez templating Go (`{{ .Values.x }}`); pojęcia Chart, Release, Values
- Listingi: `helm/theralink/Chart.yaml`, `values.yaml` (defaults), `values.prod.yaml` (override), `_helpers.tpl` (wspólne labelki), fragment template z relacją do values
- Przed/Po: 15 plików YAML z kopiowaniem vs. 1 chart + 2 values files; cykl życia: `helm install`, `helm upgrade`, `helm rollback`
- **Screen 8.4** (helm template output): `docs/screens/rozdz-08/8.4-helm-template-output.png`
- **Screen 8.5** (helm history): `docs/screens/rozdz-08/8.5-helm-history.png`

8.6. **Healthchecki — Liveness vs Readiness probes**
- Co opisać: różnica między liveness (czy aplikacja żyje — jak nie, K8s ją restartuje) a readiness (czy aplikacja jest gotowa przyjmować ruch — jak nie, Service nie kieruje do niej traffic). Spring Boot Actuator endpoints. Pułapka: `/actuator/health/readiness` i `/actuator/health/liveness` jako sub-paths wymagają `permitAll("/actuator/health/**")` w SecurityConfig (u nas użyliśmy `/actuator/health` exact match).
- Listingi: deployment.yaml fragment z `readinessProbe`/`livenessProbe`, SecurityConfig.java
- Przed/Po: docker-compose `healthcheck:` (lokalna sprawdzka kontenera) vs. K8s probes (sprawdzane przez kubelet, wpływają na routing ruchu)
- **Screen 8.6** (pod probes describe): `docs/screens/rozdz-08/8.6-pod-probes.png`

8.7. **Zarządzanie sekretami — CSI Secret Store Driver**
- Co opisać: dlaczego sekrety nie mogą być w obrazach Docker ani w ConfigMap (base64, nie szyfrowane); Azure Key Vault Provider for CSI Driver — pobiera sekrety w runtime i montuje jako pliki + opcjonalnie synchronizuje do K8s Secret; AKS addon `azureKeyvaultSecretsProvider`
- Listingi: `keycloak-secretproviderclass.yaml` (objects array, secretObjects mapping), fragment deployment z `volumeMounts` i `secretKeyRef` w env
- Przed/Po: `.env` plik w docker-compose (gitignored) vs. SecretProviderClass + Managed Identity (rola Key Vault Secrets User przyznana AKS kubelet identity)
- **Screen 8.7** (secret w env podzie): `docs/screens/rozdz-08/8.7-csi-secret-env.png`

8.8. **Ingress + TLS — wystawienie aplikacji na świat**
- Co opisać: różnica między Service typu ClusterIP/LoadBalancer/NodePort vs. Ingress (warstwa L7 z routing po hostname + path); NGINX Ingress Controller w klastrze (instalowany przez Helm, wystawia jedno Public IP); cert-manager + Let's Encrypt automatyczne wystawianie TLS przez HTTP-01 challenge
- Listingi: `ingress.yaml` (rules dla 3 hosts: root, auth.*, api.*), `cluster-issuer-letsencrypt.yaml` (ACME server + email + solvers)
- Przed/Po: docker-compose mapuje porty bezpośrednio (8080:8080, 4200:4200) vs. K8s — wszystkie serwisy wewnątrz klastra, jeden Ingress wystawia 1 IP z 3 subdomenami i TLS
- **Screen 8.8** (kubectl get ingress + certificate): `docs/screens/rozdz-08/8.8-ingress-cert.png`

8.9. **Podsumowanie migracji infrastruktury — zbiorcza tabela**
- Co opisać: zbiorcze porównanie wszystkich elementów (uruchamianie, sieć, sekrety, persystencja, healthchecks, skalowanie, monitoring, koszt). Wnioski: dla 5 serwisów dev local jest prostszy, ale prod K8s daje rozproszenie, self-healing, deklaratywność, łatwy rollback
- Tabela porównawcza (~10-12 wierszy)

**Liczba planowanych screenów:** ~6-8 (Dockerfile output, docker-compose up, k9s widoki, kubectl get pods, helm history, ingress detail, certificate status).

---

### Rozdział 9 — Wdrożenie Azure

> ROZDZIAŁ MIGRACYJNY: porównanie przed/po WPLECIONE.
> Materiały źródłowe: [[azure-aks-deployment]], [[azure-deployment-metrics]], [[azure-deployment-lessons]] (10 pułapek), `thera-infrastructure/scripts/setup-azure.sh`, `add-secrets.sh`, `build-and-push.sh`. **Region produkcyjny: swedencentral** (nie polandcentral — patrz Lessons §1 — Azure for Students policy `sys.regionrestriction`).
> **Screeny:** 10 sztuk (Rys. 9.1–9.10). Pełne instrukcje w [[_screen-checklist-8-9-10]]. Większość to widoki Azure Portal (Resource Group, ACR, AKS, Cosmos, Event Hubs, Key Vault) + DNS provider + przeglądarka.

**Proponowane podrozdziały:**

9.1. **Wybór dostawcy chmury i typ subskrypcji**
- Co opisać: Azure vs AWS vs GCP (kryterium: dostępność Azure for Students dla studenta uczelni, polski region, integracje z Microsoft Entra), typ subskrypcji "Azure for Students" — $100 kredytu, brak karty, `spendingLimit: On` (automatyczna blokada po wyczerpaniu)
- Wymienić: tenant `ans-elblag.pl`, subscription ID `d148d3bc-fff7-4d1b-9f08-fc5a633d0312`
- Pułapka do opisania: **polityka `sys.regionrestriction`** — Azure for Students ma whitelistę 5 regionów (losowych per subskrypcja), Polska zablokowana, wybrany `swedencentral`
- **Screen 9.2** (polityka regionów): `docs/screens/rozdz-09/9.2-region-policy.png`

9.2. **Architektura zasobów Azure**
- Co opisać: 6 zasobów w Resource Group `rg-theralink-prod-se-001`: ACR (registry obrazów), Key Vault (sekrety), Cosmos DB Mongo (baza), Event Hubs (Kafka), AKS (klaster), oraz implicitne (zarządzane przez AKS) — VNet, NSG, Load Balancer, Public IP
- Diagram: hierarchia Subscription → Resource Group → Resources + zarządzane MC_* RG
- Listingi: output `az resource list -g rg-theralink-prod-se-001 -o table`
- **Screen 9.1** (Azure Portal Resource Group): `docs/screens/rozdz-09/9.1-resource-group.png`

9.3. **Container Registry i build pipeline**
- Co opisać: ACR Basic SKU ($5/mc), 10 GB miejsca; w kontekście Azure for Students ACR Tasks (cloud build) jest **zablokowany** → build lokalny z `docker buildx --platform linux/amd64 --push` (krytyczne dla Apple Silicon)
- Listingi: `scripts/build-and-push.sh` (z fragmentem `docker buildx`), output `az acr repository list`
- Przed/Po: Docker Hub publiczny (każdy widzi) vs. ACR prywatny w tej samej sieci co AKS (szybki pull, attach przez Managed Identity)
- Metryki z [[azure-deployment-metrics]] §7: rozmiary obrazów (Keycloak 225 MB, Spring 162-170 MB, frontend 25 MB)
- **Screen 9.3** (ACR repositories): `docs/screens/rozdz-09/9.3-acr-repositories.png`

9.4. **AKS — managed Kubernetes**
- Co opisać: Azure zarządza control plane (API server, etcd, scheduler) za darmo, klient płaci tylko za worker nodes; konfiguracja: 1× Standard_B2s_v2 (2 vCPU, 4 GiB RAM), K8s 1.35, attach-acr (Managed Identity pull), addon azure-keyvault-secrets-provider (CSI driver)
- Pułapki do opisania: (a) **Standard_B2s v1 zablokowany** — używamy v2; (b) **K8s 1.30 to LTS-only** — używamy 1.35; (c) **Allocatable CPU 1900m**, system zjada ~1000m, dla aplikacji zostaje ~900m → wszystkie `resources.requests.cpu` agresywnie niskie (50-100m)
- Listingi: fragment `setup-azure.sh` z `az aks create`, output `az aks show`
- **Screen 9.4** (AKS Overview): `docs/screens/rozdz-09/9.4-aks-overview.png`

9.5. **Baza danych — Azure Cosmos DB for MongoDB**
- Co opisać: managed MongoDB z protokołem Mongo (sterowniki Spring Data Mongo działają bez zmian); **Free Tier** (1× per subskrypcja, 1000 RU/s + 25 GB na zawsze za $0); 2 bazy: `theralink-users` (400 RU/s), `theralink-payments` (400 RU/s)
- Pułapka **krytyczna** do opisania: Spring Boot 4 + `${MONGODB_URI:default}` w application.yml NIE PODSTAWIA wartości env var gdy URI zawiera `:` (Cosmos hasło ma `:`) — placeholder resolver myli się. Rozwiązanie: `JDK_JAVA_OPTIONS=-Dspring.data.mongodb.connection-string=$(MONGODB_URI)` na poziomie JVM. Patrz Lessons §6.
- Listingi: fragment `setup-azure.sh` z `az cosmosdb create --enable-free-tier true`, deployment.yaml z JDK_JAVA_OPTIONS
- **Screen 9.5** (Cosmos DB Free Tier + bazy): `docs/screens/rozdz-09/9.5-cosmos-free-tier.png`

9.6. **Kafka — Azure Event Hubs**
- Co opisać: Event Hubs jest natywnym brokerem Azure, ale wystawia kompatybilny protokół Kafka (Spring Kafka działa bez zmian kodu); Standard SKU 1 TU, połączenie SASL_SSL + PLAIN; topics: `theralink.payment.completed`, `theralink.payment.failed`
- Listingi: fragment `setup-azure.sh` (`az eventhubs eventhub create --cleanup-policy Delete --retention-time 24`), JAAS config w `add-secrets.sh`
- Przed/Po: lokalna Kafka w docker-compose (Zookeeper + Kafka) vs. Event Hubs (zero infrastruktury, ten sam protokół, SASL_SSL zamiast PLAINTEXT)
- Pułapka: nowa składnia `az eventhubs eventhub create` w 2026 wymaga `--cleanup-policy` i `--retention-time` (godziny) zamiast deprecated `--message-retention` (dni)
- **Screen 9.6** (Event Hubs z topikami): `docs/screens/rozdz-09/9.6-event-hubs-kafka.png`

9.7. **Sekrety — Azure Key Vault + RBAC**
- Co opisać: Key Vault w trybie RBAC (nie Access Policies — nowsze, ról `Key Vault Secrets Officer` dla użytkownika, `Key Vault Secrets User` dla AKS Managed Identity); 8 sekretów w Key Vault; CSI Driver pobiera i synchronizuje do K8s Secret
- Listingi: fragment `add-secrets.sh` (z secret_data JSON), output `az keyvault secret list`
- Przed/Po: `.env` plik gitignored vs. Key Vault z audytem, szyfrowaniem at-rest, integracją z managed identities
- **Screen 9.7** (Key Vault z 8 sekretami): `docs/screens/rozdz-09/9.7-key-vault-secrets.png`

9.8. **DNS, Ingress i TLS produkcyjny**
- Co opisać: rekordy DNS A: `theralink.pl`, `auth.theralink.pl`, `api.theralink.pl` → Public IP `51.12.157.106`; NGINX Ingress Controller (LoadBalancer Service alokuje 1 Public IP — taniej niż Application Gateway $20/mc → $4/mc); cert-manager z Let's Encrypt prod issuer; HTTP-01 challenge przez NGINX
- Listingi: rekordy DNS (3× A → 51.12.157.106), `cluster-issuer-letsencrypt.yaml`, `kubectl get certificate` output
- Pułapka do opisania: nip.io jako tymczasowa "magiczna" domena gdy nie mieliśmy jeszcze theralink.pl
- **Screen 9.8** (DNS rekordy A): `docs/screens/rozdz-09/9.8-dns-records.png`
- **Screen 9.9** (przeglądarka theralink.pl + kłódka HTTPS): `docs/screens/rozdz-09/9.9-https-browser.png`

9.9. **Keycloak w produkcji — pułapki Keycloak 25.0.4**
- Co opisać: realm `theralink` zaimportowany przez REST API `POST /admin/realms`; klient `theralink-angular` (public, PKCE); role CLIENT, PSYCHOLOGIST, ADMIN
- Pułapka **krytyczna**: KC_BOOTSTRAP_ADMIN_USERNAME/PASSWORD **nie tworzą admina** mimo że pojawiają się w `kc.sh show-config`. Rozwiązanie: manualny SQL INSERT do `user_entity` + `credential` + `user_role_mapping` z PBKDF2-SHA256 hash (27500 iteracji, 16-byte salt). Patrz Lessons §7.
- Pułapka pomniejsza: custom theme `template.ftl` zawiera `<#list properties.scripts?split(' ')>` ale `theme.properties` nie ma klucza `scripts` → FreeMarker `InvalidReferenceException` → 500 na każdym `/realms/*/auth`. Naprawa: usunięcie bloku.
- Listingi: PBKDF2 Python script + SQL INSERT, `realm-export.json` redirectUris z `https://theralink.pl/*`
- **Screen 9.10** (Keycloak login z custom motywem): `docs/screens/rozdz-09/9.10-keycloak-login.png`

9.10. **Push obrazów i pierwszy deploy**
- Co opisać: kolejność: `setup-azure.sh` → `add-secrets.sh` → `build-and-push.sh` → `kubectl apply cluster-issuer` → `helm install ingress-nginx` + `helm install cert-manager` → `helm install theralink`; 8 revisions Helm (debugowanie kolejnych pułapek)
- Metryki z [[azure-deployment-metrics]] §5: helm history, §8 czas budowy obrazów (pierwszy build ~19 min, rebuild ~5 min), §9 czas startu podów (~3-4 min do 5/5 Ready)
- Listingi: pełny `helm install theralink ... --set keycloak.tag=... ...` z 4 image tagami
- Output: `kubectl get pods -n theralink` 5/5 Running

9.11. **Logowanie i monitoring**
- Co opisać: Azure Monitor + Log Analytics (włączone defaultowo w AKS), Container Insights, Prometheus metrics (Spring Actuator + Keycloak metrics endpoint); k9s do lokalnego podglądu klastra
- Cytować rzeczywiste outputs: `kubectl top nodes` (5% CPU, 56% memory), `kubectl top pods` (13m CPU łącznie, 887 MiB RAM)

9.12. **Koszt wdrożenia — analiza ekonomiczna**
- Tabela z [[azure-deployment-metrics]] §15: szczegółowy koszt per usługa 24/7 vs z auto-stop AKS (~$62/mc → ~$44/mc), kredyt $100 starczy na 1.5-2.3 mc
- Free Tier Cosmos = $0 forever (oszczędność $5/mc)
- NGINX Ingress = 1 Public IP $4/mc vs Application Gateway $20/mc (świadoma decyzja)
- StatefulSet Postgres w klastrze = ~$1/mc PVC vs Azure DB for PostgreSQL $15/mc
- Wniosek: dla thesis demo koszt ~$50-60/mc, infrastrukturę można uruchomić → demo → wyłączyć

9.13. **Podsumowanie wdrożenia — zbiorcze przed/po**
- Co opisać: tabela 6-8 wierszy (rejestr obrazów, baza, message broker, sekrety, ingress, TLS, monitoring) lokal vs Azure
- Wnioski: 10 pułapek z [[azure-deployment-lessons]] jako materiał do dyskusji o realiach wdrożeń chmurowych (dokumentacja vs rzeczywistość, ograniczenia darmowych tierów, debugowanie property resolution Spring Boot)

**Liczba planowanych screenów:** ~8-10 (Azure Portal: resource group, Cosmos DB, AKS overview, Key Vault z sekretami; certificate w cert-manager; k9s widok; przeglądarka theralink.pl + auth.theralink.pl login screen; kubectl get all output; helm history).

---

### Rozdział 10 — Środowiska prod vs lokalne

> ROZDZIAŁ MIGRACYJNY: porównanie przed/po jest TYM ROZDZIAŁEM — całe są o różnicach. Mini-tabelki per kwestia, na końcu zbiorcza.
> Materiały źródłowe: [[azure-aks-deployment]] vs `thera-infrastructure/docker-compose/docker-compose.yml`; pliki `application.yml` vs `application-prod.yml` w serwisach Spring Boot; `environment.ts` vs `environment.prod.ts` w Angular; `values.yaml` vs `values.prod.yaml` w Helm.
> **Screeny:** 6 sztuk (Rys. 10.1–10.6). Pełne instrukcje w [[_screen-checklist-8-9-10]]. Większość to side-by-side porównania (dev vs prod) wstawiane jako dwa osobne rysunki obok siebie w pracy.

**Proponowane podrozdziały:**

10.1. **Filozofia "dev/prod parity" — 12-factor app**
- Co opisać: zasada "dev/prod parity" z 12-factor: środowiska powinny być **maksymalnie podobne** żeby uniknąć "działa u mnie", ale **niektóre różnice są nieuniknione** (koszt, latencja, dostęp do prawdziwych zewnętrznych usług)
- Strategia TheraLink: ten sam stack technologiczny w dev i prod (Java, Spring Boot, Mongo, Kafka), różnica tylko w *managed vs self-hosted*

10.2. **Uruchamianie — `docker-compose up` vs `helm install`**
- Mini-tabela: dev `docker-compose up -d` (jeden komputer, ~1 GB RAM łącznie) vs prod `helm install theralink ./helm/theralink -f values.prod.yaml --set ...` (klaster Azure)
- Listingi: docker-compose serwis Keycloak vs Helm deployment Keycloak (równolegle pokazać oba)
- Wniosek: Helm = `docker-compose for K8s`, ale z templating i rolling updates
- **Screen 10.1** (docker-compose ps — dev): `docs/screens/rozdz-10/10.1-docker-compose-ps.png`
- **Screen 10.2** (kubectl get pods — prod): `docs/screens/rozdz-10/10.2-kubectl-pods.png`

10.3. **Baza danych — MongoDB kontener vs Azure Cosmos DB**
- Mini-tabela: dev `mongo:7` w docker-compose (jedna instancja, brak replikacji, brak SSL) vs prod Cosmos DB (managed, replikacja regionalna w wbudowana, SSL wymagany, RU-based scaling)
- Identyczny sterownik Spring Data MongoDB — kod aplikacji bez zmian
- Konfiguracja: `MONGODB_URI=mongodb://mongodb:27017/theralink-users` (dev) vs `MONGODB_URI=mongodb://cosmos-theralink-...:...@cosmos-...:10255/theralink-users?ssl=true&replicaSet=globaldb&retrywrites=false` (prod)

10.4. **Kafka — self-hosted vs Azure Event Hubs**
- Mini-tabela: dev Confluent Kafka + Zookeeper (~500 MB RAM) vs prod Event Hubs (managed, brak Zookeepera, SASL_SSL)
- Konfiguracja Spring: `KAFKA_BOOTSTRAP_SERVERS=kafka:29092` (dev) vs `evh-...servicebus.windows.net:9093` + SASL_SSL JAAS (prod)
- Decyzja: brak abstrakcji nad protokołem — używamy oficjalnego Kafka SDK, Event Hubs to drop-in replacement

10.5. **Auth — Keycloak Docker vs Keycloak StatefulSet**
- Dev: Keycloak 25 + Postgres w docker-compose, `start-dev` mode (bootstrap admin z env, single instance)
- Prod: ten sam obraz Keycloak (zbudowany z `kc.sh build --db postgres --health-enabled true`), `start --optimized`, Postgres jako K8s StatefulSet z 8GB PVC Azure Disk
- Pułapka: KC_BOOTSTRAP_ADMIN_* nie działa w 25.0.4 — manualny SQL INSERT (jednorazowo per środowisko)

10.6. **Sekrety — `.env` vs Azure Key Vault + CSI**
- Dev: `.env` plik gitignored, ładowany przez docker-compose (`environment:` z `${VAR}`)
- Prod: Azure Key Vault z RBAC, SecretProviderClass montuje pliki/synchronizuje K8s Secret, env vars z `secretKeyRef`
- Lista 8 sekretów (mongodb-uri-*, kafka-sasl-jaas-config, eventhubs-*, keycloak-*, stripe-*)
- Wniosek: w obu przypadkach aplikacja czyta sekrety jako env vars — kod identyczny
- **Screen 10.4** (porównanie .env vs Key Vault): `docs/screens/rozdz-10/10.4-secrets-dev-vs-prod.png`

10.7. **Konfiguracja aplikacji — Spring profile + Angular fileReplacements**
- Spring Boot: `application.yml` (dev defaults) + `application-prod.yml` (overrides) — wybierane przez `SPRING_PROFILES_ACTIVE` env var
- Angular: `environment.ts` (dev) vs `environment.prod.ts` (prod) — `ng build --configuration production` zamienia plik dzięki `fileReplacements` w `angular.json` (pułapka: bez tego konfiguracja produkcji NIE jest aplikowana!)
- Listingi: oba pliki environment.ts (porównanie obok siebie)
- **Screen 10.3** (environment.ts vs environment.prod.ts w IDE): `docs/screens/rozdz-10/10.3-environment-comparison.png`
- **Screen 10.5** (Spring application.yml z profile): `docs/screens/rozdz-10/10.5-spring-profiles.png`

10.8. **Sieć — docker-compose network vs Kubernetes Services + Ingress**
- Dev: jedna sieć `theralink-network`, serwisy komunikują się przez nazwy (`http://keycloak:8080`); porty wystawione na host (`8080:8080`, `4200:4200`, `27017:27017`)
- Prod: wewnętrzne ClusterIP DNS (`http://thera-keycloak.theralink.svc.cluster.local:8080`), zewnętrzny ruch przez NGINX Ingress + 3 subdomeny + TLS
- Mini-diagram: dev (host → port:port → container) vs prod (internet → DNS → Public IP → Ingress → Service → Pod)

10.9. **Domena i TLS — localhost vs Let's Encrypt**
- Dev: `localhost:4200` (Angular), `localhost:8080` (Keycloak) — brak TLS
- Prod: `theralink.pl` (root), `auth.theralink.pl`, `api.theralink.pl` z TLS Let's Encrypt (ważny 3 miesiące, auto-renew przez cert-manager)
- Pułapka: redirect URIs w Keycloak realm muszą zawierać OBA — `http://localhost:4200/*` (dev) i `https://theralink.pl/*` (prod)

10.10. **Workflow developera — w jednym dniu**
- Cykl: edit kodu → `docker-compose up` (test lokalny) → `git commit` → `git push` → `docker buildx build --push` (lokalnie do ACR) → `helm upgrade theralink --set X.tag=<new-sha>` (deploy do prod)
- Średni czas iteracji: lokalny test ~5s, deploy do prod ~3-4 min (build + push + rolling update)
- Wniosek: rozłożenie środowisk dev (szybkie iteracje, mock services) vs prod (prawdziwa konfiguracja, prawdziwy koszt)
- **Screen 10.6** (workflow git → build → helm upgrade): `docs/screens/rozdz-10/10.6-dev-to-prod-workflow.png`

10.11. **Skalowanie — fixed dev vs HPA prod**
- Dev: 1 replika per serwis (więcej nie ma sensu na laptopie)
- Prod: na razie też 1 replika (oszczędność kosztu, B2s_v2 ma 1900m allocatable CPU), ale chart wspiera `--set keycloak.replicas=3` bez zmian kodu
- Krótka wzmianka o HPA (Horizontal Pod Autoscaler) jako kierunek na przyszłość — definiowanie cpu/memory threshold dla auto-scaling

10.12. **Zbiorcza tabela porównawcza i wnioski**
- 15-20 wierszy: każdy element infrastruktury w dwóch kolumnach (dev local, prod Azure)
- Wnioski końcowe:
  - Ten sam kod aplikacji w obu środowiskach (sukces dev/prod parity)
  - Różnice tylko w warstwie konfiguracji (env vars, K8s manifesty)
  - Helm chart + values.prod.yaml jako materialny artefakt zachowujący różnice
  - Koszt prod ~$60/mc vs dev $0 — uzasadnione skalą i izolacją

**Liczba planowanych screenów:** ~5-7 (docker-compose ps + kubectl get pods obok siebie, environment.ts vs prod side-by-side, Azure Portal Cosmos DB connection string, .env vs Key Vault secret list, browser localhost:4200 vs theralink.pl).

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
