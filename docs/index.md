# TheraLink — Mapa dokumentacji

Centralne repozytorium wiedzy o architekturze i migracji platformy TheraLink.

## Backend

- [[backend-migration]] — Migracja Node.js/Express → Spring Boot microservices
- [[payment-service]] — Serwis płatności Stripe (osobne repo, PCI-DSS)

## Auth & Infrastruktura

- [[keycloak-setup]] — Konfiguracja Keycloak: Docker, motywy FTL, role, integracja Angular

## Frontend

- [[frontend-migration]] — Migracja Next.js → Angular 18
- [[angular-style-guide]] — Konwencje i wzorce Angular dla TheraLink
- [[typescript-style-guide]] — Standardy TypeScript w projekcie

## Porządek migracji

```
1. Infrastructure (Docker + Keycloak + MongoDB + Kafka)   → [[keycloak-setup]]
2. Frontend Angular                                        → [[frontend-migration]]
3. User Service + Appointment Service                      → [[backend-migration]]
4. Notification Service (Kafka consumer)                   → [[backend-migration]]
5. Payment Service (osobne repo)                           → [[payment-service]]
```
