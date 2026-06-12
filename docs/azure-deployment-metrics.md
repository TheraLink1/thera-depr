# Azure Deployment — Metryki produkcyjne (snapshot 2026-06-12)

> Surowe outputs `kubectl`, `helm`, `az` z działającego klastra TheraLink.
> Do bezpośredniego cytowania w rozdziałach 8, 9, 10 pracy inżynierskiej.

---

## 1. `kubectl get pods -n theralink -o wide`

```
NAME                                     READY   STATUS    RESTARTS   AGE   IP             NODE
thera-keycloak-576879888d-2g75k          1/1     Running   0          40m   10.244.0.121   aks-nodepool1-20710978-vmss000000
thera-keycloak-db-0                      1/1     Running   0          73m   10.244.0.235   aks-nodepool1-20710978-vmss000000
thera-payment-service-54689b5f4f-xfhld   1/1     Running   0          73m   10.244.0.177   aks-nodepool1-20710978-vmss000000
thera-ui-7d86646df4-lsc4q                1/1     Running   0          64m   10.244.0.128   aks-nodepool1-20710978-vmss000000
thera-user-service-798876c4cd-xl4tv      1/1     Running   0          73m   10.244.0.114   aks-nodepool1-20710978-vmss000000
```

**5 podów, wszystkie 1/1 Running, 0 restartów (po stabilizacji)**, IP wewnętrzne w podsieci `10.244.0.0/16`, węzeł `aks-nodepool1-20710978-vmss000000` (VMSS instance #0).

## 2. `kubectl get all -n theralink` (skrócony)

| Typ zasobu | Liczba |
|---|---|
| Pod | 5 |
| Service | 5 (4× ClusterIP + 1 headless dla StatefulSet) |
| Deployment | 4 (`thera-keycloak`, `thera-payment-service`, `thera-ui`, `thera-user-service`) |
| StatefulSet | 1 (`thera-keycloak-db`) |
| ReplicaSet | 10 (1 aktywny per Deployment + 6 historycznych — rolling updates) |
| Ingress | 1 (`theralink-ingress` — 3 hosty) |
| Certificate (cert-manager) | 1 (`theralink-tls`) |
| PersistentVolumeClaim | 1 (`postgres-data-thera-keycloak-db-0`, 8GB Azure Disk) |
| SecretProviderClass | 3 (keycloak, user, payment) |
| K8s Secret (z Key Vault) | 3 (synchronizowane przez CSI) |

## 3. `kubectl top nodes` — zużycie węzła

```
NAME                                CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
aks-nodepool1-20710978-vmss000000   113m         5%     3272Mi          56%
```

**Standard_B2s_v2** (2 vCPU = 2000m, 4 GiB RAM):
- CPU używane: **5%** (113m / ~2000m), idle workload
- Memory używane: **56%** (3272 MiB / ~5800 MiB allocatable) — głównie Java heap (Keycloak + 2× Spring Boot)

## 4. `kubectl top pods -n theralink` — zużycie per pod

| Pod | CPU | Memory |
|---|---|---|
| `thera-keycloak` | 2m | 429 MiB |
| `thera-keycloak-db` (Postgres 16) | 6m | 33 MiB |
| `thera-payment-service` (Spring Boot 4) | 2m | 214 MiB |
| `thera-ui` (nginx + static) | 1m | 3 MiB |
| `thera-user-service` (Spring Boot 4) | 2m | 208 MiB |
| **SUMA** | **13m CPU** | **887 MiB RAM** |

**Wniosek:** workload idle zużywa <1% klastra. JVM-y dominują w RAM (~850 MiB), nginx + Postgres znikome.

## 5. `helm history theralink -n theralink`

```
REVISION  UPDATED                    STATUS      DESCRIPTION
1         Fri Jun 12 00:40:21 2026   superseded  Install complete
2         Fri Jun 12 00:43:25 2026   superseded  Upgrade complete
3         Fri Jun 12 00:44:58 2026   superseded  Upgrade complete
4         Fri Jun 12 00:49:15 2026   superseded  Upgrade complete
5         Fri Jun 12 00:55:19 2026   superseded  Upgrade complete
6         Fri Jun 12 00:59:46 2026   superseded  Upgrade complete
7         Fri Jun 12 01:05:46 2026   superseded  Upgrade complete
8         Fri Jun 12 01:13:31 2026   deployed    Upgrade complete
```

**8 revisions w ~33 minutach** od pierwszego install do stabilnego stanu — każdy upgrade to debugowanie kolejnej pułapki (CPU limits → health probes → KC db driver → SPRING_APPLICATION_JSON ordering → property name → bootstrap admin → template.ftl fix).

## 6. `helm list --all-namespaces` (relevantne)

```
NAME            NAMESPACE        REVISION  CHART                  APP VERSION
cert-manager    cert-manager     1         cert-manager-v1.20.2   v1.20.2
ingress-nginx   ingress-nginx    1         ingress-nginx-4.15.1   1.15.1
theralink       theralink        8         theralink-0.1.0        1.0.0
```

**3 Helm releases:** własny chart `theralink` + 2 zewnętrzne (NGINX Ingress Controller, cert-manager) zainstalowane przed deployem aplikacji.

## 7. Obrazy Docker — ACR (Azure Container Registry)

```
acrtheralinkprod.azurecr.io
├── theralink/keycloak           225.6 MB  amd64  (Keycloak 25.0.4 + custom theme + Postgres JDBC)
├── theralink/user-service       161.9 MB  amd64  (Spring Boot 4.0.3, Java 25 JRE)
├── theralink/payment-service    169.7 MB  amd64  (Spring Boot 4.0.3, Java 21 JRE)
└── theralink/frontend            25.1 MB  amd64  (Angular bundle + nginx:alpine)
```

**Rozmiary obrazów:**
- Spring Boot serwisy: ~160–170 MB (Java JRE ~120 MB + Spring fat JAR ~40 MB)
- Keycloak: 225 MB (Quarkus + dziesiątki bibliotek + custom theme + Postgres JDBC)
- Frontend: 25 MB (najmniejszy — Alpine nginx + statyczne assety Angulara)

Łącznie ~580 MB obrazów w ACR (Basic SKU pozwala na 10 GB — z dużym zapasem).

## 8. Czas budowy obrazów (lokalny `docker buildx --platform linux/amd64`)

> Mierzone na MacBook Pro M-series (Apple Silicon → linux/amd64 emulacja QEMU).

| Obraz | Pierwszy build (cold) | Rebuild (warm cache) |
|---|---|---|
| `theralink/keycloak` | ~2 min (kc.sh build + Postgres JDBC dependency) | ~30 s |
| `theralink/user-service` | ~5 min (Maven dependency resolution + Spring Boot package) | ~1 min |
| `theralink/payment-service` | ~5 min (jw.) | ~1 min |
| `theralink/frontend` | ~7 min (pnpm install + Angular ng build production) | ~2 min |

**Łącznie pierwszy push do ACR:** ~19 min (z poziomu lokalnej maszyny, włącznie z push fazą).

**Push do ACR:** ~1 min per obraz przy lokalizacji `swedencentral` z połączeniem ~50 Mbit/s.

## 9. Czas startu Pod-ów (od `Pending` do `Ready`)

| Pod | Image pull | App start | Total Pending → 1/1 |
|---|---|---|---|
| `thera-keycloak-db-0` | ~10s (Postgres 16, 200 MB) | ~5s (Hibernate init) | ~15s |
| `thera-keycloak` | ~15s (225 MB image) | ~25s (Quarkus + Liquibase 134 changesets) | ~40-60s |
| `thera-user-service` | ~10s (162 MB) | ~25s (Spring Boot + Tomcat + MongoDB connect) | ~35s |
| `thera-payment-service` | ~10s (170 MB) | ~25s (j.w. + Stripe SDK init) | ~35s |
| `thera-ui` | ~3s (25 MB) | <1s (nginx) | ~5s |

**Całkowity start klastra od `helm install` do `5/5 Ready`:** ~3-4 min przy świeżej DB Postgres.

## 10. Azure resources w `rg-theralink-prod-se-001`

| Nazwa | Typ | Region |
|---|---|---|
| `acrtheralinkprod` | Microsoft.ContainerRegistry/registries | swedencentral |
| `kv-theralink-prod-se-01` | Microsoft.KeyVault/vaults | swedencentral |
| `cosmos-theralink-prod-se-001` | Microsoft.DocumentDB/databaseAccounts | swedencentral |
| `evh-theralink-prod-se-001` | Microsoft.EventHub/namespaces | swedencentral |
| `aks-theralink-prod-se-001` | Microsoft.ContainerService/managedClusters | swedencentral |

**5 zasobów w resource group** (+ 1 implicitny: VNet/NSG/LoadBalancer w resource group `MC_rg-theralink-prod-se-001_*_swedencentral` zarządzanym przez AKS).

## 11. Cosmos DB konfiguracja

```json
{
  "name": "cosmos-theralink-prod-se-001",
  "enableFreeTier": true,
  "location": "Sweden Central",
  "defaultConsistency": "Eventual"
}
```

**Free Tier ON** → **$0/mc na zawsze** dla 1000 RU/s + 25 GB. Mamy 2 bazy po 400 RU/s = 800 RU/s, mieści się.

## 12. AKS konfiguracja

```json
{
  "fqdn": "aks-theral-rg-theralink-pro-d148d3-9zk563do.hcp.swedencentral.azmk8s.io",
  "k8sVersion": "1.35",
  "nodeCount": 1,
  "vmSize": "Standard_B2s_v2",
  "addons": {
    "azureKeyvaultSecretsProvider": {
      "enabled": true,
      "clientId": "e19a38d6-d163-4cac-a826-a47652afe82b"
    }
  }
}
```

- **K8s 1.35** (najnowsza darmowa w sweden)
- **1× Standard_B2s_v2** = 2 vCPU, 4 GiB RAM (~$0.05/h ≈ $30/mc 24/7)
- Addon **azureKeyvaultSecretsProvider** włączony — CSI driver dla Azure Key Vault

## 13. Sieć Kubernetes

| Element | Wartość |
|---|---|
| Pod CIDR | `10.244.0.0/16` (5 podów używa 5 IP) |
| Service CIDR | `10.0.0.0/16` (5 ClusterIP-ów: 10.0.61.223, .150.13, .112.137, .147.8 + headless) |
| Public IP NGINX Ingress | `51.12.157.106` |
| Domena | `theralink.pl`, `auth.theralink.pl`, `api.theralink.pl` |

## 14. TLS / cert-manager

- **Issuer:** Let's Encrypt production (`https://acme-v02.api.letsencrypt.org/directory`)
- **Challenge type:** HTTP-01 (przez NGINX Ingress)
- **Domeny w certyfikacie:** 3 (root + auth + api)
- **Ważność:** 3 miesiące (do ~2026-09-09)
- **Auto-renew:** ~60 dni przed wygaśnięciem (cert-manager zarządza automatycznie)

## 15. Koszt miesięczny (szacunkowy)

| Pozycja | Cena 24/7 | Z auto-stop AKS (10h/dzień) |
|---|---|---|
| AKS węzeł Standard_B2s_v2 | ~$30 | ~$12 |
| Event Hubs Standard 1 TU | ~$21 | ~$21 (zawsze on) |
| Public IP (Standard) | ~$4 | ~$4 |
| ACR Basic | ~$5 | ~$5 |
| Cosmos DB (Free Tier) | $0 | $0 |
| Key Vault Standard | <$1 | <$1 |
| Log Analytics (mały) | ~$1 | ~$1 |
| **Suma** | **~$62/mc** | **~$44/mc** |

Kredyt Azure for Students $100 starczy na **~1.5 miesiąca 24/7** lub **~2.3 miesiąca z auto-stop**.

## 16. Sieć Azure (implicit — managed by AKS)

Resource group `MC_rg-theralink-prod-se-001_aks-theralink-prod-se-001_swedencentral` zawiera (zarządzane przez AKS bez bezpośredniej interwencji):
- VNet `aks-vnet-*` z podsiecią `10.224.0.0/12`
- Network Security Group
- Standard Load Balancer (alokowany przez Service `ingress-nginx-controller` typu LoadBalancer)
- 1× Public IP `51.12.157.106`
- Route Table

## Powiązane

- [[azure-aks-deployment]] — narracyjny opis stanu produkcji
- [[azure-deployment-lessons]] — 10 pułapek + rozwiązania
