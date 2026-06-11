# Azure AKS Deployment — TheraLink (2026-06-11)

> Stan po pierwszym pełnym wdrożeniu na Azure for Students.
> Domena docelowa: **theralink.pl** (TLS Let's Encrypt).

## Infrastruktura Azure

| Zasób | Nazwa | Region | Koszt |
|---|---|---|---|
| Subskrypcja | `Azure for Students` | — | $100 kredytu, spending limit On |
| Resource Group | `rg-theralink-prod-se-001` | swedencentral | $0 |
| Container Registry | `acrtheralinkprod.azurecr.io` | swedencentral | ~$5/mc (Basic) |
| Key Vault | `kv-theralink-prod-se-01` | swedencentral | ~$0 (RBAC mode) |
| Cosmos DB Mongo | `cosmos-theralink-prod-se-001` | swedencentral | **$0** (Free Tier ON, 1000 RU/s, 25 GB) |
| Event Hubs | `evh-theralink-prod-se-001` | swedencentral | ~$21/mc (Standard, Kafka) |
| AKS cluster | `aks-theralink-prod-se-001` | swedencentral | ~$30/mc (1× Standard_B2s_v2, K8s 1.35) |
| Public IP | 51.12.157.106 | swedencentral | ~$4/mc (LoadBalancer NGINX Ingress) |

**Łączny koszt ~$60/mc** 24/7. Z auto-stop AKS na noc spada do ~$15/mc.

> 📸 **[SCREEN DO DODANIA]**
> Co pokazać: Azure Portal → rg-theralink-prod-se-001 → Resources view ze wszystkimi 7 zasobami.
> Sugerowany podpis: "Resource Group `rg-theralink-prod-se-001` z kompletną infrastrukturą TheraLink".

## Kubernetes (namespace `theralink`)

```
thera-ui                  1/1 Running   :80    Angular 21 SPA + nginx
thera-keycloak            1/1 Running   :8080  Keycloak 25.0.4 (z PostgreSQL driver)
thera-keycloak-db         1/1 Running   :5432  PostgreSQL 16, StatefulSet + 8GB PVC
thera-user-service        1/1 Running   :8081  Spring Boot 4, Cosmos Mongo
thera-payment-service     1/1 Running   :8085  Spring Boot 4, Cosmos + Stripe SDK
```

**Wewnętrzny ruch:** ClusterIP Services + K8s DNS (`thera-keycloak.theralink.svc.cluster.local`).

**Ruch z internetu:** NGINX Ingress Controller → LoadBalancer Service → Public IP `51.12.157.106`.

## Routing HTTP/HTTPS

| Adres | Kierowane do | Status |
|---|---|---|
| `https://theralink.pl/` | thera-ui:80 | HTTP 200 |
| `https://auth.theralink.pl/realms/*` | thera-keycloak:8080 | HTTP 200 (login, OIDC discovery) |
| `https://api.theralink.pl/users/*` | thera-user-service:8081 | HTTP 401 (wymaga JWT) |
| `https://api.theralink.pl/payments/*` | thera-payment-service:8085 | HTTP 401 (wymaga JWT) |

**Certyfikat TLS:** Let's Encrypt (wystawiony przez cert-manager + HTTP-01 challenge przez NGINX Ingress). Auto-renewal przed wygaśnięciem (60 dni).

## Sekrety

Wszystkie sekrety w **Azure Key Vault `kv-theralink-prod-se-01`**, pobierane przez **CSI Secret Store Driver** do Pod-ów jako env vars (przez `SecretProviderClass` per serwis).

| Sekret | Konsument |
|---|---|
| `mongodb-uri-users` | thera-user-service |
| `mongodb-uri-payments` | thera-payment-service |
| `kafka-sasl-jaas-config` | user + payment (Event Hubs auth) |
| `eventhubs-connection-string` | — (referencja) |
| `keycloak-db-password` | thera-keycloak + thera-keycloak-db |
| `keycloak-admin-password` | thera-keycloak (manual SQL INSERT, patrz [[azure-deployment-lessons]]) |
| `stripe-secret-key` | thera-payment-service |
| `stripe-webhook-secret` | thera-payment-service (placeholder — do podmiany po skonfigurowaniu Stripe webhook) |

## Keycloak realm `theralink`

| Element | Wartość |
|---|---|
| Realm enabled | true |
| Default locale | pl |
| Languages | pl, en |
| Registration | enabled (nowi userzy automatycznie dostają role `CLIENT`) |
| Login theme | `theralink` (custom) |

**Clients:**
- `theralink-angular` (public, PKCE) — redirectUris: `https://theralink.pl/*`, `http://localhost:4200/*`
- `theralink-backend` (confidential, service account) — dla serwisów Spring Boot

**Roles:** CLIENT, PSYCHOLOGIST, ADMIN (+ defaults).

**Admin Console:**
- URL: https://auth.theralink.pl/admin/master/console/
- User: `admin`
- Hasło: `az keyvault secret show --vault-name kv-theralink-prod-se-01 --name keycloak-admin-password --query value -o tsv`

## Skrypty (w `theralink-infrastructure` repo)

| Skrypt | Co robi |
|---|---|
| `scripts/setup-azure.sh` | Idempotentnie tworzy wszystkie 6 zasobów Azure (RG, ACR, KV, Cosmos, EVH, AKS) + role assignment |
| `scripts/add-secrets.sh` | Pobiera connection stringi z Cosmos/EVH i wpisuje wszystkie sekrety do Key Vault. Generuje hasła Keycloak. SKIP_STRIPE=1 wstawia placeholdery |
| `scripts/build-and-push.sh` | Build 3 obrazów Docker (Keycloak, user, payment) lokalnie z `linux/amd64` i push do ACR |

## Helm chart (w `theralink-infrastructure/helm/theralink/`)

- `Chart.yaml`, `values.yaml`, `values.prod.yaml`
- `templates/*-deployment.yaml`, `*-service.yaml`, `*-secretproviderclass.yaml` per serwis
- `templates/keycloak-db-statefulset.yaml` (Postgres z volumeClaimTemplates)
- `templates/ingress.yaml` (NGINX + cert-manager)
- `templates/_helpers.tpl` (wspólne labelki)

**Deploy:**
```bash
helm install theralink ./helm/theralink \
  --namespace theralink --create-namespace \
  -f values.prod.yaml \
  --set global.domain=theralink.pl \
  --set keycloak.tag=<sha> --set user.tag=<sha> \
  --set payment.tag=<sha> --set frontend.tag=<sha>
```

## Powiązane

- [[azure-deployment-lessons]] — kluczowe lekcje z deploy (5 pułapek)
- [[infrastructure]] — pełny przewodnik dev + prod (oryginalny)
- [[keycloak-setup]] — konfiguracja Keycloak
- [[backend-migration]] — implementacja Spring Boot
- [[frontend-migration]] — Angular migration
