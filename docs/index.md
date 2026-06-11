# TheraLink — Mapa dokumentacji

Centralne repozytorium wiedzy o architekturze i migracji platformy TheraLink.

## Backend

- [[backend-migration]] — Migracja Node.js/Express → Spring Boot microservices
- [[payment-service]] — Serwis płatności Stripe (osobne repo, PCI-DSS)

## Architektura

- [[architecture-diagram]] — Diagramy Mermaid: widok systemu, przepływy, mapa repozytoriów

## Plany implementacji

- [[plan-notifications-zoom]] — Email (SendGrid) + Zoom + format wizyty (zdalnie/stacjonarnie)

## Auth & Infrastruktura

- [[keycloak-setup]] — Konfiguracja Keycloak: Docker, motywy FTL, role, integracja Angular
- [[infrastructure]] — Dev (Docker Compose) + Prod (Azure AKS): pełny przewodnik wdrożenia

## Frontend

- [[frontend-migration]] — Migracja Next.js → Angular 18
- [[angular-style-guide]] — Konwencje i wzorce Angular dla TheraLink
- [[typescript-style-guide]] — Standardy TypeScript w projekcie

## Stories (thera-ui)

- [[story-02-transloco-scopes-slot-picker]] — Transloco scopes, AppointmentSlotPicker, testy (2026-06-10)

## Wdrożenie produkcyjne Azure (2026-06-11)

- [[azure-aks-deployment]] — snapshot stanu produkcyjnego (zasoby, koszt, Pody, sekrety, realm Keycloak)
- [[azure-deployment-lessons]] — 10 kluczowych pułapek napotkanych podczas deploy + rozwiązania

## Porządek migracji

```
1. Infrastructure (Docker Compose dev + Azure setup)       → [[infrastructure]]
2. Keycloak (Auth)                                         → [[keycloak-setup]]
3. Frontend Angular                                        → [[frontend-migration]]
4. User Service + Appointment Service                      → [[backend-migration]]
5. Notification Service (Kafka consumer)                   → [[backend-migration]]
6. Payment Service (osobne repo PCI-DSS)                   → [[payment-service]]
7. Kubernetes prod deploy (AKS)                            → [[infrastructure]]
```
