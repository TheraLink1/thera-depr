# Checklist screenów — rozdziały 8, 9, 10

> Zrzuty do zrobienia podczas pisania rozdziałów. Numeracja odpowiada numeracji rysunków w pracy.
>
> **Konwencja zapisu:** `docs/screens/rozdz-NN/N.M-opis.png`
> **Skrót do zrzutu:** `Shift+Cmd+5` na Mac, "Capture Selected Portion".

---

## Setup jednorazowy

```bash
mkdir -p /Users/desirecutieqb/IdeaProjects/TheraLink/docs/screens/rozdz-08
mkdir -p /Users/desirecutieqb/IdeaProjects/TheraLink/docs/screens/rozdz-09
mkdir -p /Users/desirecutieqb/IdeaProjects/TheraLink/docs/screens/rozdz-10
```

Warto też zainstalować **k9s** jeśli jeszcze nie masz (ładny CLI dashboard K8s):
```bash
brew install k9s
```

---

## Rozdział 8 — Infrastruktura (Docker + Kubernetes) — **8 screenów**

### Screen 8.1 — Hierarchia warstw obrazu Docker
- **Co pokazać:** Output `docker history acrtheralinkprod.azurecr.io/theralink/user-service:latest` — widoczne warstwy multi-stage build (base JRE → COPY src → RUN package → finalny JAR)
- **Komenda:**
  ```bash
  docker pull acrtheralinkprod.azurecr.io/theralink/user-service:latest
  docker history acrtheralinkprod.azurecr.io/theralink/user-service:latest --no-trunc --format "table {{.CreatedBy}}\t{{.Size}}"
  ```
- **Plik:** `docs/screens/rozdz-08/8.1-docker-history.png`
- **Podpis:** Rys. 8.1. Warstwy obrazu `theralink/user-service` po multi-stage build

### Screen 8.2 — Sieć docker-compose
- **Co pokazać:** Output `docker-compose ps` z poziomu `thera-docker-compose/` — wszystkie 6 serwisów Running (keycloak, keycloak-db, mongodb, zookeeper, kafka, kafka-ui)
- **Komenda:**
  ```bash
  cd ~/IdeaProjects/thera-docker-compose
  docker-compose up -d
  docker-compose ps
  ```
- **Plik:** `docs/screens/rozdz-08/8.2-docker-compose-ps.png`
- **Podpis:** Rys. 8.2. Status środowiska developerskiego TheraLink po `docker-compose up`

### Screen 8.3 — k9s — widok podów namespace `theralink`
- **Co pokazać:** k9s otwarty na namespace `theralink`, widok `:pods` z 5 podami `thera-*` w stanie Running, kolumna READY = 1/1, AGE i RESTARTS widoczne
- **Komenda:**
  ```bash
  k9s --context aks-theralink-prod-se-001
  # po otwarciu: :ns → wybierz theralink → :pods
  ```
- **Plik:** `docs/screens/rozdz-08/8.3-k9s-pods.png`
- **Podpis:** Rys. 8.3. Stan podów namespace `theralink` w klastrze AKS (widok k9s)

### Screen 8.4 — Manifesty wygenerowane przez Helm
- **Co pokazać:** `helm template` z fragmentem wygenerowanego Deployment YAML — widoczna sekcja `spec.template.spec.containers[0]` z env vars i resources. Zaznacz placeholder `{{ .Values.X }}` w lewym oknie + wynik w prawym (jeśli IDE pozwala na split, jeśli nie — pojedynczy screen wyniku)
- **Komenda:**
  ```bash
  cd ~/IdeaProjects/thera-infrastructure
  helm template theralink ./helm/theralink -f helm/theralink/values.prod.yaml \
    --set keycloak.tag=latest \
    --set user.tag=latest \
    --set payment.tag=latest \
    --set frontend.tag=latest \
    | less
  # przewijaj do sekcji Deployment thera-user-service
  ```
- **Plik:** `docs/screens/rozdz-08/8.4-helm-template-output.png`
- **Podpis:** Rys. 8.4. Manifest Deployment wygenerowany przez Helm z `values.prod.yaml`

### Screen 8.5 — Helm history (rolling updates)
- **Co pokazać:** Output `helm history theralink -n theralink` — 8+ rewizji z `STATUS` (superseded/deployed) i `DESCRIPTION` (Install complete / Upgrade complete)
- **Komenda:**
  ```bash
  helm history theralink -n theralink
  ```
- **Plik:** `docs/screens/rozdz-08/8.5-helm-history.png`
- **Podpis:** Rys. 8.5. Historia rewizji Helm release `theralink` — 8 wdrożeń podczas iteracyjnego debugowania

### Screen 8.6 — Readiness/Liveness probes w działaniu
- **Co pokazać:** `kubectl describe pod thera-user-service-<id> -n theralink` — sekcja `Liveness:`, `Readiness:`, `Containers` z `Restart Count` i `Last State` (jeśli był restart) lub po prostu działająca probe configuration. Można też pokazać output `kubectl get pod thera-user-service-<id> -n theralink -o yaml | grep -A 8 -E 'readinessProbe|livenessProbe'`
- **Komenda:**
  ```bash
  POD=$(kubectl get pod -n theralink -l app.kubernetes.io/name=thera-user-service -o jsonpath='{.items[0].metadata.name}')
  kubectl describe pod $POD -n theralink | grep -A 30 "Containers:"
  ```
- **Plik:** `docs/screens/rozdz-08/8.6-pod-probes.png`
- **Podpis:** Rys. 8.6. Konfiguracja `liveness` i `readiness` probes dla podu `thera-user-service`

### Screen 8.7 — CSI Secret Store — sekret wstrzyknięty do Poda
- **Co pokazać:** `kubectl describe pod thera-user-service-<id> -n theralink` z widoczną sekcją `Environment:` zawierającą `MONGODB_URI` z `<set to the key 'mongodb-uri-users' in secret 'user-secrets'>` — pokazuje że env var pochodzi z K8s Secret, który był utworzony przez SecretProviderClass
- **Komenda:**
  ```bash
  POD=$(kubectl get pod -n theralink -l app.kubernetes.io/name=thera-user-service -o jsonpath='{.items[0].metadata.name}')
  kubectl describe pod $POD -n theralink | grep -A 20 "Environment:"
  ```
- **Plik:** `docs/screens/rozdz-08/8.7-csi-secret-env.png`
- **Podpis:** Rys. 8.7. Sekret `mongodb-uri-users` wstrzyknięty z Azure Key Vault do podu `thera-user-service` przez SecretProviderClass

### Screen 8.8 — Ingress + Certyfikat TLS
- **Co pokazać:** Output `kubectl get ingress,certificate -n theralink -o wide` — widoczny Ingress `theralink-ingress` z 3 hostami (theralink.pl, auth.theralink.pl, api.theralink.pl), ADDRESS = 51.12.157.106, PORTS = 80, 443. Plus Certificate `theralink-tls` w stanie READY=True
- **Komenda:**
  ```bash
  kubectl get ingress,certificate -n theralink -o wide
  ```
- **Plik:** `docs/screens/rozdz-08/8.8-ingress-cert.png`
- **Podpis:** Rys. 8.8. Konfiguracja Ingress z certyfikatem Let's Encrypt dla domen TheraLink

---

## Rozdział 9 — Wdrożenie Azure — **10 screenów**

### Screen 9.1 — Azure Portal: Resource Group
- **Co pokazać:** Azure Portal → Resource Group `rg-theralink-prod-se-001` → zakładka "Overview" lub "Resources" z listą 5 zasobów (ACR, Key Vault, Cosmos DB, Event Hubs, AKS), region Sweden Central widoczny w nagłówku
- **Skąd:** https://portal.azure.com/#@ans-elblag.pl/resource/subscriptions/d148d3bc-fff7-4d1b-9f08-fc5a633d0312/resourceGroups/rg-theralink-prod-se-001/overview
- **Plik:** `docs/screens/rozdz-09/9.1-resource-group.png`
- **Podpis:** Rys. 9.1. Zasoby TheraLink w resource group `rg-theralink-prod-se-001` (Azure Portal)

### Screen 9.2 — Polityka regionów (przyczyna wyboru swedencentral)
- **Co pokazać:** Azure Portal → Subscriptions → `Azure for Students` → Policies → znajdź `Allowed resource deployment regions` z listą 5 dozwolonych regionów (italynorth, uaenorth, spaincentral, swedencentral, austriaeast). LUB output terminala `az policy assignment list --query "[?displayName=='Allowed resource deployment regions'].parameters.listOfAllowedLocations" -o json`
- **Komenda (jeśli terminal):**
  ```bash
  az policy assignment list --query "[?displayName=='Allowed resource deployment regions'].parameters.listOfAllowedLocations" -o json
  ```
- **Plik:** `docs/screens/rozdz-09/9.2-region-policy.png`
- **Podpis:** Rys. 9.2. Polityka `sys.regionrestriction` Azure for Students ograniczająca regiony wdrożenia

### Screen 9.3 — Azure Container Registry z repozytoriami
- **Co pokazać:** Azure Portal → `acrtheralinkprod` → "Repositories" — lista 4 repozytoriów (`theralink/keycloak`, `theralink/user-service`, `theralink/payment-service`, `theralink/frontend`) z liczbą tagów i rozmiarem
- **Plik:** `docs/screens/rozdz-09/9.3-acr-repositories.png`
- **Podpis:** Rys. 9.3. Repozytoria obrazów Docker w Azure Container Registry `acrtheralinkprod`

### Screen 9.4 — AKS Overview
- **Co pokazać:** Azure Portal → `aks-theralink-prod-se-001` → "Overview" — widoczne: K8s version 1.35, Node count 1, VM size Standard_B2s_v2, Resource group, Status: Running. Można też przewinąć do "Properties" i pokazać "API server address"
- **Plik:** `docs/screens/rozdz-09/9.4-aks-overview.png`
- **Podpis:** Rys. 9.4. Klaster AKS `aks-theralink-prod-se-001` — przegląd konfiguracji

### Screen 9.5 — Cosmos DB Free Tier
- **Co pokazać:** Azure Portal → `cosmos-theralink-prod-se-001` → "Overview" — widoczny baner/tag "Free Tier" lub w "Pricing tier" wartość. Plus zakładka "Data Explorer" z dwoma bazami: `theralink-users`, `theralink-payments`
- **Plik:** `docs/screens/rozdz-09/9.5-cosmos-free-tier.png`
- **Podpis:** Rys. 9.5. Konto Azure Cosmos DB w trybie Free Tier z bazami `theralink-users` i `theralink-payments`

### Screen 9.6 — Event Hubs z protokołem Kafka
- **Co pokazać:** Azure Portal → `evh-theralink-prod-se-001` → "Event Hubs" — lista 2 topics: `theralink.payment.completed`, `theralink.payment.failed`. Plus "Settings" → "Shared access policies" lub baner pokazujący Kafka enabled
- **Plik:** `docs/screens/rozdz-09/9.6-event-hubs-kafka.png`
- **Podpis:** Rys. 9.6. Azure Event Hubs z włączonym protokołem Kafka i topikami TheraLink

### Screen 9.7 — Azure Key Vault z 8 sekretami
- **Co pokazać:** Azure Portal → `kv-theralink-prod-se-01` → "Secrets" — lista 8 sekretów (mongodb-uri-users, mongodb-uri-payments, kafka-sasl-jaas-config, eventhubs-connection-string, keycloak-db-password, keycloak-admin-password, stripe-secret-key, stripe-webhook-secret), wszystkie Enabled. Wartości NIE odsłonięte (kropki/****)
- **Plik:** `docs/screens/rozdz-09/9.7-key-vault-secrets.png`
- **Podpis:** Rys. 9.7. Sekrety TheraLink w Azure Key Vault `kv-theralink-prod-se-01` (RBAC mode)

### Screen 9.8 — DNS rekordy A
- **Co pokazać:** Panel Twojego DNS providera (gdzie masz theralink.pl) z 3 rekordami A: `@`, `auth`, `api` → wszystkie wskazujące na `51.12.157.106`, TTL widoczne
- **Plik:** `docs/screens/rozdz-09/9.8-dns-records.png`
- **Podpis:** Rys. 9.8. Rekordy DNS A dla domeny `theralink.pl` wskazujące na IP publiczne NGINX Ingress

### Screen 9.9 — Działająca aplikacja przez HTTPS
- **Co pokazać:** Przeglądarka (Chrome/Safari) otwarta na `https://theralink.pl/` z widocznym SPA Angular + **kłódka HTTPS** w pasku adresu (potwierdzenie TLS Let's Encrypt). Można kliknąć kłódkę żeby pokazać szczegóły certyfikatu (Issuer: Let's Encrypt, Valid until 2026-09-09).
- **Plik:** `docs/screens/rozdz-09/9.9-https-browser.png`
- **Podpis:** Rys. 9.9. Aplikacja TheraLink uruchomiona produkcyjnie pod adresem `https://theralink.pl/` z certyfikatem Let's Encrypt

### Screen 9.10 — Keycloak login w przeglądarce (z motywem `theralink`)
- **Co pokazać:** Przeglądarka na `https://auth.theralink.pl/realms/theralink/protocol/openid-connect/auth?client_id=theralink-angular&response_type=code&redirect_uri=https://theralink.pl/&scope=openid&code_challenge=...&code_challenge_method=S256` (lub po kliknięciu "Login" w aplikacji TheraLink) — formularz logowania z motywem `theralink` (custom CSS, logo TheraLink, polskie napisy)
- **Plik:** `docs/screens/rozdz-09/9.10-keycloak-login.png`
- **Podpis:** Rys. 9.10. Formularz logowania Keycloak z custom motywem `theralink` (subdomena `auth.theralink.pl`)

---

## Rozdział 10 — Środowiska prod vs lokalne — **6 screenów**

> Sklejanie odrzucone — robimy **pojedyncze obrazy**. Mimo tego dwa screeny tej samej komendy z różnych środowisk (np. dev `docker-compose ps` + prod `kubectl get pods`) wstawiamy jako dwa osobne rysunki obok siebie w pracy.

### Screen 10.1 — docker-compose ps (dev)
- **Co pokazać:** Output `docker-compose ps` na lokalnym macu — 5-6 serwisów dev środowiska Running (mongodb, keycloak, keycloak-db, kafka, zookeeper, kafka-ui)
- **Komenda:**
  ```bash
  cd ~/IdeaProjects/thera-docker-compose
  docker-compose up -d
  docker-compose ps --format "table {{.Service}}\t{{.State}}\t{{.Ports}}"
  ```
- **Plik:** `docs/screens/rozdz-10/10.1-docker-compose-ps.png`
- **Podpis:** Rys. 10.1. Środowisko developerskie TheraLink uruchomione lokalnie przez Docker Compose

### Screen 10.2 — kubectl get pods (prod) — odpowiednik 10.1
- **Co pokazać:** Output `kubectl get pods -n theralink -o wide` — 5 podów thera-* z prod Azure
- **Komenda:**
  ```bash
  kubectl get pods -n theralink -o wide
  ```
- **Plik:** `docs/screens/rozdz-10/10.2-kubectl-pods.png`
- **Podpis:** Rys. 10.2. Środowisko produkcyjne TheraLink w klastrze AKS — równowartość Rys. 10.1

### Screen 10.3 — environment.ts vs environment.prod.ts
- **Co pokazać:** IDE (VS Code / IntelliJ) z otwartymi **dwoma** zakładkami obok siebie: `thera-ui/src/environments/environment.ts` (production: false, localhost URLs) i `environment.prod.ts` (production: true, theralink.pl URLs)
- **Plik:** `docs/screens/rozdz-10/10.3-environment-comparison.png`
- **Podpis:** Rys. 10.3. Porównanie plików konfiguracyjnych Angular: `environment.ts` (dev) i `environment.prod.ts` (prod)

### Screen 10.4 — .env (dev) i Key Vault Secrets (prod)
- **Co pokazać:** Po lewej terminal z `cat ~/IdeaProjects/thera-docker-compose/.env` (po zamaskowaniu prawdziwych wartości — np. `KC_ADMIN_PASSWORD=***`). Po prawej Azure Portal `kv-theralink-prod-se-01` → Secrets lista. **Jeden screen jeśli zmieścisz oba side-by-side w IDE, lub dwa osobne.**
- **Komenda:**
  ```bash
  cat ~/IdeaProjects/thera-docker-compose/.env | sed 's/=.*/=***MASKED***/'
  ```
- **Plik:** `docs/screens/rozdz-10/10.4-secrets-dev-vs-prod.png` (lub `.4a-env-dev.png` + `.4b-keyvault-prod.png`)
- **Podpis:** Rys. 10.4. Porównanie zarządzania sekretami: `.env` lokalnie vs Azure Key Vault w prod

### Screen 10.5 — Spring profile dev w intellij
- **Co pokazać:** IntelliJ / VS Code → `application.yml` (default profile, dev URLs `localhost:27017`) — sekcja `spring.data.mongodb.uri` z `${MONGODB_URI:mongodb://localhost:27017/theralink-users}` widoczna. Albo IntelliJ → Run Configuration z `SPRING_PROFILES_ACTIVE=prod`
- **Plik:** `docs/screens/rozdz-10/10.5-spring-profiles.png`
- **Podpis:** Rys. 10.5. Konfiguracja Spring profile w `application.yml` z fallback dev URI

### Screen 10.6 — Workflow dev: zmiana kodu → deploy do prod
- **Co pokazać:** Terminal z pełnym workflow (3-4 komendy w jednym screenshocie):
  ```
  $ git commit -am "fix: bug w slot pickerze"
  $ git push origin main
  $ ./scripts/build-and-push.sh
  ✓ Pushed: acrtheralinkprod.azurecr.io/theralink/frontend:abc1234
  $ helm upgrade theralink ./helm/theralink --set frontend.tag=abc1234
  Release "theralink" has been upgraded. Happy Helming!
  ```
- **Plik:** `docs/screens/rozdz-10/10.6-dev-to-prod-workflow.png`
- **Podpis:** Rys. 10.6. Workflow developera — od zmiany kodu lokalnie do wdrożenia w klastrze AKS

---

## Łącznie

| Rozdział | Liczba screenów |
|---|---|
| 8 — Infrastruktura | 8 |
| 9 — Wdrożenie Azure | 10 |
| 10 — Środowiska prod vs lokalne | 6 |
| **TOTAL** | **24 screeny** |

## Tipsy techniczne

- **Skrót Mac:** `Shift+Cmd+5` → "Capture Selected Portion" → drag obszar → po zwolnieniu klawiszy zapisze do Schowka i pokaże miniaturkę w prawym dolnym rogu. Klik miniaturki → "Save to..." → wybierz folder w `docs/screens/rozdz-NN/`
- **Wysokie DPI:** `defaults write com.apple.screencapture include-date -bool false` (opcjonalne — usuwa datę z nazwy pliku) + `Shift+Cmd+5` → "Options" → "Save to: Desktop/Other" → ustaw raz na `docs/screens/`
- **Zaciemnienie hasła:** dla `.env` lub innych miejsc z hasłami — użyj `Preview.app` po zrzuceniu, narzędzie "Rectangle" → wypełnij na czarno, lub w CleanShot X funkcja "Blur"
- **Output terminala:** powiększ font w terminalu (Cmd+) przed screenshotem dla czytelności. Włącz dark mode dla jednolitego stylu screenów
- **Azure Portal:** używaj 100% zoom, ukrywaj boczny pasek (`F11` lub menu) jeśli zmieści się więcej treści

## Po zrobieniu wszystkich screenów

Powiedz nowemu Claude (w sesji pisania rozdziałów):
> "Wszystkie 24 screeny w `docs/screens/rozdz-{08,09,10}/`. Wstaw je w odpowiednie miejsca w rozdziale podstawiając ścieżki bezwzględne pod `> 📸 [SCREEN DO DODANIA]` zgodnie z numeracją."

Claude wtedy zamieni placeholder na rzeczywisty markdown `![](rozdz-08/8.4-helm-template-output.png)`.
