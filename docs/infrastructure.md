# TheraLink — Infrastruktura: Dev (Docker Compose) + Prod (Azure AKS)

> **Dla kogo jest ten dokument?**
> Opisuje jak postawić cały stack lokalnie (docker-compose) i jak wdrożyć na produkcję (Azure Kubernetes Service). Zaczyna się od teorii, potem przechodzi do konkretnych kroków.

---

## Spis treści

1. [Architektura środowisk](#1-architektura-środowisk)
2. [Nowe repo: theralink-infrastructure](#2-nowe-repo-theralink-infrastructure)
3. [Dev: Docker Compose](#3-dev-docker-compose)
4. [Zarządzanie sekretami](#4-zarządzanie-sekretami)
5. [Prod: Azure AKS — teoria](#5-prod-azure-aks--teoria)
6. [Prod: Krok po kroku — Azure](#6-prod-krok-po-kroku--azure)
7. [CI/CD: GitHub Actions → ACR → AKS](#7-cicd-github-actions--acr--aks)
8. [Struktura Kubernetes manifests](#8-struktura-kubernetes-manifests)

---

## 1. Architektura środowisk

### Dev vs Prod — porównanie

| Komponent | Dev (local) | Prod (Azure) |
|---|---|---|
| Uruchamianie | `docker-compose up` | `kubectl apply` / Helm |
| Keycloak | docker-compose, `start-dev` | Kubernetes StatefulSet |
| MongoDB | jeden kontener na dev | Azure Cosmos DB (Mongo API) per serwis |
| Kafka | docker-compose (Kafka + Zookeeper) | Azure Event Hubs (Kafka-compatible) |
| Sekrety | `.env` (gitignored) | Azure Key Vault + K8s Secrets |
| Obrazy Docker | budowane lokalnie | Azure Container Registry (ACR) |
| Ingress | brak (porty bezpośrednio) | Azure Application Gateway / NGINX Ingress |
| TLS/SSL | brak | cert-manager + Let's Encrypt |

### Dlaczego Azure Cosmos DB zamiast self-hosted MongoDB?

MongoDB w Kubernetesie jako StatefulSet to **operacyjnie trudne** — trzeba zarządzać replikaset, persistence, backupami. Azure Cosmos DB for MongoDB to managed service który:
- Mówi językiem MongoDB (ten sam sterownik, Spring Data działa bez zmian)
- Ma automatyczne backupy
- Skaluje się automatycznie
- W Azure działa bez opóźnień sieciowych (są w tej samej sieci)

Jedyna zmiana w kodzie: `MONGODB_URI` w zmiennych środowiskowych.

### Dlaczego Azure Event Hubs zamiast self-hosted Kafka?

Azure Event Hubs ma **wbudowany protokół Kafka** — Twój kod Spring Boot z `spring-kafka` nie wymaga żadnych zmian. Zamiast stawiać Kafka + Zookeeper + zarządzać brokerami, używasz managed service.

```yaml
# Ta sama konfiguracja Spring Boot działa z Event Hubs:
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
    # dla Event Hubs: theralink.servicebus.windows.net:9093
```

---

## 2. Nowe repo: theralink-infrastructure

Osobne repozytorium `theralink-infrastructure` — **PRIVATE** — zawiera:
- docker-compose dla dev
- Kubernetes manifests / Helm charts dla prod
- Skrypty CI/CD i deploy
- Szablony sekretów (nigdy prawdziwe wartości)

```
theralink-infrastructure/
├── docker-compose/
│   ├── docker-compose.yml          ← główny plik dev (wszystkie serwisy)
│   ├── docker-compose.override.yml ← lokalne nadpisania (gitignored)
│   └── .env.example                ← szablon zmiennych (commit do repo)
│
├── k8s/
│   ├── namespace.yml
│   ├── keycloak/
│   │   ├── deployment.yml
│   │   ├── service.yml
│   │   └── configmap.yml
│   ├── user-service/
│   │   ├── deployment.yml
│   │   └── service.yml
│   ├── appointment-service/
│   ├── psychologist-service/
│   ├── notification-service/
│   ├── api-gateway/
│   └── ingress/
│       └── ingress.yml
│
├── helm/
│   └── theralink/
│       ├── Chart.yaml
│       ├── values.yaml             ← wartości domyślne (bez sekretów)
│       ├── values.prod.yaml        ← nadpisania dla prod (bez sekretów)
│       └── templates/
│           ├── _helpers.tpl
│           ├── deployment.yaml
│           └── service.yaml
│
├── scripts/
│   ├── setup-azure.sh              ← jednorazowe tworzenie zasobów Azure
│   ├── build-and-push.sh           ← buduj obrazy i push do ACR
│   └── deploy.sh                   ← deploy do AKS
│
└── .github/
    └── workflows/
        ├── build-push.yml          ← trigger: push do serwisu → ACR
        └── deploy-aks.yml          ← trigger: tag release → deploy do AKS
```

> ⚠️ `.env`, `values.secrets.yaml`, certyfikaty — NIGDY w repo. Używaj Azure Key Vault.

---

## 3. Dev: Docker Compose

### Teoria: jak docker-compose łączy serwisy

W docker-compose wszystkie serwisy są w jednej sieci `theralink-network`. Zamiast `localhost:8080` serwisy komunikują się przez **nazwę kontenera**:

```
# Z poziomu user-service kontener:
http://keycloak:8080/realms/theralink   ← nazwa serwisu z docker-compose
mongodb://mongo-users:27017             ← nie "localhost"!
kafka:9092                              ← nie "localhost"!
```

Dlatego `application.yml` używa zmiennych środowiskowych — dev vs prod mają inne hosty:

```yaml
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI:mongodb://localhost:27017/theralink-users}
      #                   ↑ dev local       ↑ prod: Cosmos DB URI
```

### Plik: `docker-compose/docker-compose.yml`

```yaml
version: '3.9'

networks:
  theralink-network:
    driver: bridge

volumes:
  keycloak_db_data:
  mongo_users_data:
  mongo_appointments_data:
  mongo_psychologists_data:
  mongo_notifications_data:
  mongo_payments_data:
  kafka_data:
  zookeeper_data:

services:

  # ─────────────────────────────────────────
  # KEYCLOAK (Auth)
  # ─────────────────────────────────────────
  keycloak-db:
    image: postgres:16
    container_name: thera-keycloak-db
    networks: [theralink-network]
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: ${KC_DB_PASSWORD:-keycloak_dev}
    volumes:
      - keycloak_db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U keycloak"]
      interval: 10s
      timeout: 5s
      retries: 5

  keycloak:
    build:
      context: ../../thera-keycloak   # ścieżka do repo thera-keycloak
    container_name: thera-keycloak
    networks: [theralink-network]
    ports:
      - "8080:8080"
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: ${KC_ADMIN_USERNAME:-admin}
      KC_BOOTSTRAP_ADMIN_PASSWORD: ${KC_ADMIN_PASSWORD:-admin_dev_password}
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak-db:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: ${KC_DB_PASSWORD:-keycloak_dev}
      KC_HOSTNAME: localhost
      KC_HOSTNAME_PORT: 8080
      KC_HTTP_ENABLED: "true"
      KC_HOSTNAME_STRICT: "false"
      KC_SPI_THEME_STATIC_MAX_AGE: "-1"
      KC_SPI_THEME_CACHE_THEMES: "false"
      KC_SPI_THEME_CACHE_TEMPLATES: "false"
    depends_on:
      keycloak-db:
        condition: service_healthy
    volumes:
      - ../../thera-keycloak/realm-export.json:/opt/keycloak/data/import/realm-export.json:ro
    command: ["start-dev", "--import-realm"]

  # ─────────────────────────────────────────
  # KAFKA (Message Bus)
  # ─────────────────────────────────────────
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    container_name: thera-zookeeper
    networks: [theralink-network]
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper_data:/var/lib/zookeeper

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: thera-kafka
    networks: [theralink-network]
    ports:
      - "9092:9092"
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    volumes:
      - kafka_data:/var/lib/kafka/data
    healthcheck:
      test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:29092", "--list"]
      interval: 15s
      timeout: 10s
      retries: 5

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: thera-kafka-ui
    networks: [theralink-network]
    ports:
      - "9093:8080"
    depends_on: [kafka]
    environment:
      KAFKA_CLUSTERS_0_NAME: theralink-dev
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092

  # ─────────────────────────────────────────
  # MONGODB (per serwis — osobne bazy)
  # ─────────────────────────────────────────
  mongo-users:
    image: mongo:7
    container_name: thera-mongo-users
    networks: [theralink-network]
    ports:
      - "27017:27017"
    volumes:
      - mongo_users_data:/data/db

  mongo-appointments:
    image: mongo:7
    container_name: thera-mongo-appointments
    networks: [theralink-network]
    ports:
      - "27018:27017"
    volumes:
      - mongo_appointments_data:/data/db

  mongo-psychologists:
    image: mongo:7
    container_name: thera-mongo-psychologists
    networks: [theralink-network]
    ports:
      - "27019:27017"
    volumes:
      - mongo_psychologists_data:/data/db

  mongo-notifications:
    image: mongo:7
    container_name: thera-mongo-notifications
    networks: [theralink-network]
    ports:
      - "27020:27017"
    volumes:
      - mongo_notifications_data:/data/db

  # ─────────────────────────────────────────
  # SPRING BOOT SERVICES
  # ─────────────────────────────────────────
  user-service:
    build:
      context: ../../thera-rest-service
    container_name: thera-user-service
    networks: [theralink-network]
    ports:
      - "8081:8081"
    depends_on:
      keycloak:
        condition: service_started
      mongo-users:
        condition: service_started
      kafka:
        condition: service_healthy
    environment:
      MONGODB_URI: mongodb://mongo-users:27017/theralink-users
      KEYCLOAK_ISSUER_URI: http://keycloak:8080/realms/theralink
      KAFKA_BOOTSTRAP_SERVERS: kafka:29092
    restart: on-failure

  # appointment-service, psychologist-service, notification-service
  # — analogiczne bloki (dodaj gdy repo będą gotowe)

  # ─────────────────────────────────────────
  # API GATEWAY
  # ─────────────────────────────────────────
  # api-gateway:
  #   build:
  #     context: ../../theralink-api-gateway
  #   ports:
  #     - "8090:8090"
  #   depends_on: [keycloak, user-service]
  #   networks: [theralink-network]
```

### Plik: `docker-compose/.env.example`

```env
# Keycloak
KC_DB_PASSWORD=keycloak_dev
KC_ADMIN_USERNAME=admin
KC_ADMIN_PASSWORD=admin_dev_password

# Stripe (tylko dla payment-service — osobne repo)
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

Skopiuj do `.env` (gitignored) i uzupełnij wartości:
```bash
cp .env.example .env
```

### Uruchamianie dev stacku

```bash
cd theralink-infrastructure/docker-compose

# Pierwszy start (build obrazów)
docker-compose up --build

# Kolejne starty
docker-compose up -d

# Logi konkretnego serwisu
docker-compose logs -f user-service

# Zatrzymanie
docker-compose down

# Zatrzymanie + usunięcie danych (!)
docker-compose down -v
```

### Kolejność startowania (zależy od `depends_on`)

```
zookeeper → kafka
keycloak-db → keycloak
mongo-users → user-service
kafka + keycloak + mongo → user-service
```

---

## 4. Zarządzanie sekretami

### Dev — `.env` (proste, tylko lokalnie)

```
docker-compose/
├── .env.example   ← commit (szablony, bez wartości)
└── .env           ← gitignored (prawdziwe wartości)
```

### Prod — Azure Key Vault + Kubernetes Secrets

**Teoria:** Kubernetes ma wbudowany mechanizm `Secret`, ale są one domyślnie base64-encoded — nie szyfrowane. Azure Key Vault to prawdziwy skarbiec sekretów z:
- szyfrowaniem at-rest
- pełnym audytem dostępu
- integracją z AKS przez CSI driver

**Przepływ:**
```
Azure Key Vault
    │  (CSI Secret Store Driver)
    ▼
Kubernetes Pod
    └── sekrety zamontowane jako pliki lub zmienne środowiskowe
```

**Instalacja CSI driver w AKS:**
```bash
az aks enable-addons \
  --resource-group rg-theralink \
  --name aks-theralink \
  --addons azure-keyvault-secrets-provider
```

**SecretProviderClass** (k8s manifest):
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: theralink-user-service-secrets
  namespace: theralink
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "<managed-identity-client-id>"
    keyvaultName: "kv-theralink-prod"
    tenantID: "<azure-tenant-id>"
    objects: |
      array:
        - |
          objectName: mongodb-uri-users
          objectType: secret
        - |
          objectName: keycloak-issuer-uri
          objectType: secret
```

---

## 5. Prod: Azure AKS — teoria

### Co to jest Azure AKS?

**Azure Kubernetes Service (AKS)** to managed Kubernetes — Microsoft zarządza warstwą control plane (API server, etcd, scheduler). Ty zarządzasz tylko węzłami roboczymi (worker nodes) i aplikacjami.

### Zasoby Azure które będziesz tworzyć

```
Azure Subscription
└── Resource Group: rg-theralink
    ├── AKS Cluster: aks-theralink
    │   ├── Node Pool: Standard_B2s (2 CPU, 4GB RAM) × 2-3 węzły
    │   └── Namespace: theralink (wszystkie serwisy)
    │
    ├── Azure Container Registry: acrtheralink
    │   └── Repozytoria obrazów Docker:
    │       ├── theralink/user-service:1.0.0
    │       ├── theralink/keycloak:1.0.0
    │       └── ...
    │
    ├── Azure Cosmos DB (Mongo API):
    │   ├── Account: cosmos-theralink
    │   └── Databases: theralink-users, theralink-appointments, ...
    │
    ├── Azure Event Hubs Namespace: evh-theralink
    │   └── Event Hubs (= Kafka topics):
    │       ├── theralink.user.created
    │       └── theralink.appointment.created
    │
    ├── Azure Key Vault: kv-theralink-prod
    │   └── Sekrety (wartości wpisane ręcznie lub przez pipeline)
    │
    └── Azure Application Gateway (Ingress)
        └── Domena: theralink.pl → AKS
```

---

## 6. Prod: Krok po kroku — Azure

### Wymagania wstępne

```bash
# Zainstaluj Azure CLI
brew install azure-cli

# Zainstaluj kubectl
brew install kubectl

# Zainstaluj Helm
brew install helm

# Zaloguj się do Azure
az login

# Ustaw subskrypcję
az account set --subscription "<twoja-subscription-id>"
```

### Krok 1 — Utwórz Resource Group

```bash
az group create \
  --name rg-theralink \
  --location polandcentral
```

> `polandcentral` to region Azure w Polsce — najniższe opóźnienia dla polskich użytkowników.

### Krok 2 — Azure Container Registry (ACR)

ACR to prywatny rejestr obrazów Docker — odpowiednik Docker Hub, ale w Azure.

```bash
# Utwórz rejestr (nazwa musi być globalnie unikalna)
az acr create \
  --resource-group rg-theralink \
  --name acrtheralink \
  --sku Basic

# Zaloguj lokalnego Dockera do ACR
az acr login --name acrtheralink

# Zbuduj i wypchnij obraz (przykład user-service)
docker build -t acrtheralink.azurecr.io/theralink/user-service:1.0.0 \
  /path/to/thera-rest-service

docker push acrtheralink.azurecr.io/theralink/user-service:1.0.0
```

### Krok 3 — AKS Cluster

```bash
az aks create \
  --resource-group rg-theralink \
  --name aks-theralink \
  --node-count 2 \
  --node-vm-size Standard_B2s \
  --enable-managed-identity \
  --attach-acr acrtheralink \
  --generate-ssh-keys \
  --location polandcentral

# Pobierz credentials do kubectl
az aks get-credentials \
  --resource-group rg-theralink \
  --name aks-theralink

# Sprawdź połączenie
kubectl get nodes
```

`--attach-acr` pozwala AKS pobierać obrazy z ACR bez dodatkowej konfiguracji.

### Krok 4 — Azure Key Vault

```bash
# Utwórz Key Vault
az keyvault create \
  --name kv-theralink-prod \
  --resource-group rg-theralink \
  --location polandcentral

# Dodaj sekret (przykład)
az keyvault secret set \
  --vault-name kv-theralink-prod \
  --name mongodb-uri-users \
  --value "mongodb://cosmos-theralink:PASSWORD@cosmos-theralink.mongo.cosmos.azure.com:10255/theralink-users?ssl=true"

# Włącz CSI Secret Store w AKS
az aks enable-addons \
  --resource-group rg-theralink \
  --name aks-theralink \
  --addons azure-keyvault-secrets-provider
```

### Krok 5 — Azure Cosmos DB (MongoDB API)

```bash
# Utwórz konto Cosmos DB z API MongoDB
az cosmosdb create \
  --resource-group rg-theralink \
  --name cosmos-theralink \
  --kind MongoDB \
  --server-version 7.0 \
  --locations regionName=polandcentral

# Utwórz bazę dla user-service
az cosmosdb mongodb database create \
  --account-name cosmos-theralink \
  --resource-group rg-theralink \
  --name theralink-users

# Pobierz connection string i wstaw do Key Vault
az cosmosdb keys list \
  --name cosmos-theralink \
  --resource-group rg-theralink \
  --type connection-strings
```

### Krok 6 — Azure Event Hubs (Kafka)

```bash
# Namespace Event Hubs
az eventhubs namespace create \
  --resource-group rg-theralink \
  --name evh-theralink \
  --sku Standard \
  --location polandcentral \
  --enable-kafka true      # ← włącz protokół Kafka!

# Utwórz topic (= Kafka topic)
az eventhubs eventhub create \
  --resource-group rg-theralink \
  --namespace-name evh-theralink \
  --name theralink.user.created \
  --partition-count 4

az eventhubs eventhub create \
  --resource-group rg-theralink \
  --namespace-name evh-theralink \
  --name theralink.appointment.created \
  --partition-count 4
```

**Spring Boot application.yml dla Event Hubs (bez zmian w kodzie!):**

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
    # prod: evh-theralink.servicebus.windows.net:9093
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: PLAIN
      sasl.jaas.config: >
        org.apache.kafka.common.security.plain.PlainLoginModule required
        username="$ConnectionString"
        password="${EVENT_HUBS_CONNECTION_STRING}";
```

### Krok 7 — Namespace i deploy

```bash
# Utwórz namespace w Kubernetes
kubectl create namespace theralink

# Wdróż serwis (Helm)
helm upgrade --install theralink ./helm/theralink \
  --namespace theralink \
  --values ./helm/theralink/values.prod.yaml \
  --set image.tag=1.0.0

# Sprawdź status
kubectl get pods -n theralink
kubectl get services -n theralink
```

---

## 7. CI/CD: GitHub Actions → ACR → AKS

### Plik: `.github/workflows/build-push.yml`

```yaml
name: Build & Push to ACR

on:
  push:
    branches: [main]
    paths:
      - 'thera-rest-service/**'   # trigger tylko gdy zmienił się ten serwis

env:
  ACR_REGISTRY: acrtheralink.azurecr.io
  SERVICE_NAME: user-service

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to ACR
        uses: azure/docker-login@v1
        with:
          login-server: ${{ env.ACR_REGISTRY }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}

      - name: Build and push
        run: |
          docker build -t $ACR_REGISTRY/theralink/$SERVICE_NAME:${{ github.sha }} \
            ./thera-rest-service
          docker push $ACR_REGISTRY/theralink/$SERVICE_NAME:${{ github.sha }}
```

### Plik: `.github/workflows/deploy-aks.yml`

```yaml
name: Deploy to AKS

on:
  workflow_dispatch:        # ręczny trigger
    inputs:
      image_tag:
        description: 'Docker image tag'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Get AKS credentials
        run: |
          az aks get-credentials \
            --resource-group rg-theralink \
            --name aks-theralink

      - name: Deploy with Helm
        run: |
          helm upgrade --install theralink ./helm/theralink \
            --namespace theralink \
            --set image.tag=${{ github.event.inputs.image_tag }}
```

---

## 8. Struktura Kubernetes manifests

### Przykład: `k8s/user-service/deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: theralink
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: acrtheralink.azurecr.io/theralink/user-service:1.0.0
          ports:
            - containerPort: 8081
          env:
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: user-service-secrets
                  key: mongodb-uri
            - name: KEYCLOAK_ISSUER_URI
              value: "https://keycloak.theralink.pl/realms/theralink"
            - name: KAFKA_BOOTSTRAP_SERVERS
              value: "evh-theralink.servicebus.windows.net:9093"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8081
            initialDelaySeconds: 60
            periodSeconds: 30
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: theralink
spec:
  selector:
    app: user-service
  ports:
    - port: 8081
      targetPort: 8081
```

### Ingress (domena → serwisy)

```yaml
# k8s/ingress/ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: theralink-ingress
  namespace: theralink
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - api.theralink.pl
        - auth.theralink.pl
      secretName: theralink-tls
  rules:
    - host: api.theralink.pl
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-gateway
                port:
                  number: 8090
    - host: auth.theralink.pl
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: keycloak
                port:
                  number: 8080
```

---

## Porządek wdrożenia (produkcja)

```
1. az group create + ACR + AKS            → jednorazowe (setup-azure.sh)
2. Azure Key Vault → wstaw sekrety ręcznie
3. Azure Cosmos DB → utwórz bazy per serwis
4. Azure Event Hubs → utwórz topiki Kafka
5. Build obrazów Docker → push do ACR
6. kubectl apply k8s/namespace.yml
7. Keycloak → deploy + import realm
8. Spring Boot serwisy → deploy
9. API Gateway → deploy
10. Ingress + cert-manager → TLS
11. Angular → Azure Static Web Apps lub AKS
```

---

## Powiązane dokumenty

- [[backend-migration]] — implementacja Spring Boot serwisów
- [[keycloak-setup]] — konfiguracja Keycloak
- [[payment-service]] — payment service (osobne repo PCI-DSS)
- [[index]] — mapa dokumentacji