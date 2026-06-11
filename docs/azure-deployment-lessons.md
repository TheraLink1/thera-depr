# Azure Deployment — Kluczowe lekcje (2026-06-11)

> 7 pułapek napotkanych podczas pierwszego deploy TheraLink na Azure AKS.
> Wszystkie naprawione, ale warto pamiętać przy kolejnych deploy'ach.

---

## 1. Azure for Students — polityka regionów

**Problem:** Próba utworzenia ACR w `polandcentral` → `RequestDisallowedByAzure`. To samo w `germanywestcentral`.

**Przyczyna:** Microsoft narzuca subskrypcjom Azure for Students **politykę systemową `sys.regionrestriction`** z whitelistą losowych 5 regionów per subskrypcja. Sprawdzenie:

```bash
az policy assignment list --query "[?displayName=='Allowed resource deployment regions']" -o json
```

W tej konkretnej subskrypcji dozwolone: `italynorth, uaenorth, spaincentral, swedencentral, austriaeast`.

**Rozwiązanie:** wybrać region z whitelisty. **`swedencentral`** najbliżej Polski (~50ms) z pełną gamą usług.

---

## 2. ACR Tasks zablokowane dla Azure for Students

**Problem:** `az acr build --image ...` zwraca `TasksOperationsNotAllowed`.

**Przyczyna:** Cloud build na ACR Tasks jest płatną feature wykluczoną na Students tier.

**Rozwiązanie:** budujemy lokalnie z `docker buildx`:

```bash
docker buildx build \
  --platform linux/amd64 \    # AKS to amd64, Mac M-series to arm64
  --tag acrtheralinkprod.azurecr.io/theralink/SERVICE:TAG \
  --push \
  /path/to/repo
```

`--platform linux/amd64` jest **krytyczne** na Apple Silicon — bez tego obraz arm64 nie uruchomi się w AKS.

---

## 3. AKS na Standard_B2s (v1) zablokowany w swedencentral

**Problem:** `az aks create --node-vm-size Standard_B2s` → `VM size not allowed`.

**Przyczyna:** Microsoft wyłączył starsze rodziny VM (B-series v1) w nowszych regionach. Dostępne tylko **`Standard_B2s_v2`** (2 vCPU, 4 GB RAM, ta sama cena).

**Rozwiązanie:** wszędzie `Standard_B2s_v2` zamiast `Standard_B2s`.

---

## 4. Kubernetes 1.30 to LTS-only

**Problem:** `az aks create --kubernetes-version 1.30` → `K8sVersionNotSupported`.

**Przyczyna:** AKS 1.30 jest tylko w **Long-Term Support tier** (Premium AKS, dodatkowa opłata). Free tier ma 1.33+.

**Rozwiązanie:** używać `--kubernetes-version 1.35` (najnowsza stabilna w swedencentral).

Sprawdzenie dostępnych wersji:
```bash
az aks get-versions --location swedencentral --query "values[?!isPreview].version" -o tsv
```

---

## 5. AKS węzeł B2s_v2 ma tylko 1900m allocatable CPU

**Problem:** Pody Pending z `Insufficient cpu`.

**Przyczyna:** 1 vCPU Standard_B2s_v2 = 2000m, ale K8s rezerwuje ~100m dla system + kube-system zjada ~900m → dla aplikacji zostaje **~900m total**.

**Rozwiązanie:** wszystkie `resources.requests.cpu` agresywnie niskie:
- keycloak: 100m
- keycloak-db: 50m
- user-service: 100m
- payment-service: 100m
- frontend (nginx): 50m

Razem ~400m, mieści się.

---

## 6. Spring Boot 4 + Cosmos Mongo — `${MONGODB_URI:default}` nie działa

**Problem:** Spring Boot 4.0.3 w application.yml ma:
```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI:mongodb://localhost:27017/theralink-users}
```

Env var `MONGODB_URI` w Pod ma poprawny Cosmos URI. Ale aplikacja loguje:
```
MongoClient ... clusterSettings={hosts=[localhost:27017]
```

**Przyczyna:** Cosmos connection string zawiera `:` (separator hasło/host) — Spring property placeholder resolver myli to z separatorem default value w `${VAR:default}`. Wartość się **nie podstawia** i fallback'uje do default.

`SPRING_DATA_MONGODB_URI` env var ani `SPRING_APPLICATION_JSON` też nie pomogły (problem był z Kubernetes `$(VAR)` substitution order — `SPRING_APPLICATION_JSON` był PRZED `MONGODB_URI` w env list).

**Rozwiązanie:** `JDK_JAVA_OPTIONS` z explicit `-D` flags na poziomie JVM (literalne, omijają Spring placeholder resolver):

```yaml
env:
  - name: MONGODB_URI         # MUSI być przed JDK_JAVA_OPTIONS dla K8s substitution
    valueFrom:
      secretKeyRef: {...}
  - name: JDK_JAVA_OPTIONS
    value: '-Dspring.data.mongodb.uri=$(MONGODB_URI) -Dspring.data.mongodb.connection-string=$(MONGODB_URI)'
```

`connection-string` to nowy alias w Spring Boot 4 (preferowany nad `uri`).

---

## 7. Keycloak 25.0.4 — bootstrap admin bug

**Problem:** Mimo ustawienia `KC_BOOTSTRAP_ADMIN_USERNAME` i `KC_BOOTSTRAP_ADMIN_PASSWORD` w Pod env vars i pustej bazy PostgreSQL, admin user **nie jest tworzony** przy starcie. `kc.sh show-config` widzi te env vars, ale Keycloak ich nie używa.

- `start --optimized` — admin nie tworzony
- `start` (bez optimized) — admin nie tworzony
- `start-dev` — admin nie tworzony

**`kc.sh bootstrap-admin user`** subcommand pojawił się dopiero w Keycloak 26+.

**Rozwiązanie:** ręczny SQL INSERT do bazy z PBKDF2-SHA256 hashed hasłem:

```python
iterations = 27500
salt = secrets.token_bytes(16)
hash_bytes = hashlib.pbkdf2_hmac('sha256', password.encode('utf-8'), salt, iterations, dklen=64)
# salt + hash → base64 → secret_data JSON
```

Wstawiamy do 3 tabel: `user_entity` (user), `credential` (password), `user_role_mapping` (admin role).

**Bonus problem:** custom theme `theralink/login/template.ftl` używał `properties.scripts?split(' ')` ale `theme.properties` nie miał klucza `scripts` → FreeMarker `InvalidReferenceException` przy każdym `/realms/*/protocol/openid-connect/auth` request. Login flow zwracał 500.

**Rozwiązanie:** usunięcie `#list properties.scripts` z template (theme nie używa custom JS).

---

## 8. Dockerfile Keycloak — brak `--db postgres` przy build

**Problem:** Keycloak Pod CrashLoopBackOff z:
```
Driver does not support the provided URL: jdbc:postgresql://thera-keycloak-db:5432/keycloak
```

**Przyczyna:** `RUN /opt/keycloak/bin/kc.sh build` (bez argumentów) buduje obraz **bez sterownika PostgreSQL**. `start --optimized` później nie może się połączyć z bazą.

**Rozwiązanie:** w Dockerfile:
```dockerfile
RUN /opt/keycloak/bin/kc.sh build \
    --db postgres \
    --health-enabled true \
    --metrics-enabled true
```

---

## 9. macOS bash 3.2 — brak `declare -A`

**Problem:** Skrypt `build-and-push.sh` używał associative arrays (`declare -A SERVICES=...`) → `unbound variable` na macOS.

**Przyczyna:** Apple nie aktualizuje bash od 2007 (licencja GPLv3). System bash to wersja 3.2 — bez associative arrays (dodane w bash 4.0).

**Rozwiązanie:** zwykła tablica z parami `image<TAB>repo` + parsing z `${ENTRY%TAB*}` / `${ENTRY#*TAB}`.

---

## 10. K8s `$(VAR)` substitution — kolejność env vars ma znaczenie

**Problem:** `$(MONGODB_URI)` w `SPRING_APPLICATION_JSON` env vars nie był podstawiany — pozostał literalny.

**Przyczyna:** Kubernetes podstawia `$(VAR)` w env tylko jeśli `VAR` jest **zadeklarowana WCZEŚNIEJ** na tej samej liście env.

**Rozwiązanie:** porządek deklaracji:
```yaml
env:
  - name: MONGODB_URI         # 1. źródłowa
    valueFrom: ...
  - name: SPRING_APPLICATION_JSON  # 2. używa $(MONGODB_URI)
    value: '{"...":{"uri":"$(MONGODB_URI)"}}'
```

---

## Konfiguracja awaryjna

Jeśli infrastruktura zostanie zniszczona, kolejność odtworzenia:
1. `./scripts/setup-azure.sh` — Azure resources
2. `./scripts/add-secrets.sh` — Key Vault secrets
3. **Ręczny INSERT admin do Postgres** — patrz wyżej PBKDF2
4. `./scripts/build-and-push.sh` — obrazy Docker
5. `kubectl apply -f k8s/cluster-issuer-letsencrypt.yaml`
6. `helm install theralink ./helm/theralink ...`
7. `curl POST /admin/realms` z realm-export.json → import realm `theralink`

## Powiązane

- [[azure-aks-deployment]] — pełny snapshot stanu produkcyjnego
- [[infrastructure]] — oryginalny przewodnik
- [[keycloak-setup]] — konfiguracja Keycloak
