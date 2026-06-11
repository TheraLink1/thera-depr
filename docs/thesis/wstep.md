# Wstęp

> **Status:** streszczenie wygenerowane przez skill `kod-do-pracy`, gotowe do rozbudowania w Coworku przez skill `praca-inzynierska`.
> **Mapowanie na zakres pracy:** sekcja "Wstęp" pracy (przed rozdziałem 1).
> **Data wygenerowania:** 2026-06-09
> **Uwagi dla Cowork:** wstęp jest podzielony na 5 bloków `## [ROZDZIAŁ: Wstęp — …]` — każdy stanowi odrębny podrozdział wstępu. Wytyczne ANS Elbląg dla wstępu obejmują: cel pracy, uzasadnienie tematu, zakres pracy, metody oraz strukturę pracy.

---

## [ROZDZIAŁ: Wstęp — cel pracy]

**Temat:** Sformułowanie celu pracy dyplomowej w postaci pięciu precyzyjnych, mierzalnych celów praktycznych, charakterystycznych dla pracy inżynierskiej.

**Technologie:** Spring Boot, Angular, MongoDB, Keycloak, Apache Kafka, Docker, Kubernetes (Azure AKS), Stripe, Zoom API.

**Co zostało zaimplementowane:** Celem pracy dyplomowej jest zdobycie praktycznej wiedzy z zakresu projektowania i implementacji aplikacji o rozbudowanej architekturze mikroserwisowej, na przykładzie migracji platformy TheraLink z monolitu Node.js/Express + Next.js do systemu rozproszonego opartego o Spring Boot, Angular i Keycloak. Realizacja celu głównego obejmuje cztery cele szczegółowe stanowiące wyzwania charakterystyczne dla nowoczesnych systemów rozproszonych:

1. **Postawienie dwóch środowisk uruchomieniowych** — środowiska lokalnego (ang. *local*) opartego o Docker Compose, służącego do codziennej pracy deweloperskiej oraz testów integracyjnych, oraz środowiska produkcyjnego (ang. *production*) zrealizowanego w chmurze Microsoft Azure z wykorzystaniem usług zarządzanych (Azure Kubernetes Service, Azure Cosmos DB, Azure Event Hubs, Azure Key Vault). Środowiska różnią się sposobem zarządzania sekretami, źródłem konfiguracji oraz topologią sieciową, lecz dzielą identyczny kod aplikacyjny.
2. **Wprowadzenie komunikacji asynchronicznej przez Apache Kafka** — zastąpienie synchronicznych wywołań HTTP między modułami zdarzeniowymi przepływami przez platformę strumieniowania zdarzeń. Komunikacja asynchroniczna umożliwia luźne powiązanie serwisów (ang. *loose coupling*), tolerancję na chwilowe niedostępności konsumentów oraz odtworzenie zdarzeń (ang. *event replay*) z zachowanych offsetów.
3. **Integracja z serwisami zewnętrznymi** — implementacja interakcji z platformą płatniczą Stripe (obsługa rzeczywistych płatności kartą z wykorzystaniem przepływu *PaymentIntent* oraz webhooków) oraz API platformy Zoom (automatyczne tworzenie spotkań wideokonferencyjnych w mechanizmie *Server-to-Server OAuth* po zaksięgowaniu płatności za wizytę zdalną).
4. **Orkiestracja kontenerów Kubernetes** — konteneryzacja każdego mikroserwisu (Spring Boot, Keycloak, frontend Angular) w obrazach Docker, wdrożenie ich na klastrze Azure Kubernetes Service oraz zarządzanie cyklem życia aplikacji przy pomocy manifestów Kubernetes oraz pakietów Helm.

**Kluczowe decyzje techniczne:** Sformułowanie celu pracy jako kombinacji jednego celu głównego (zdobycie wiedzy praktycznej) z czterema konkretnymi celami szczegółowymi pozwala na obiektywną weryfikację stopnia realizacji pracy. Każdy z celów szczegółowych jest mierzalny — środowiska albo działają, albo nie; komunikacja asynchroniczna albo została wprowadzona, albo nie; integracje z serwisami zewnętrznymi albo realizują rzeczywiste transakcje, albo nie; orkiestracja Kubernetes albo zarządza klastrem, albo nie. Taki układ celów wzmacnia inżynierski, praktyczny charakter pracy w przeciwieństwie do prac o charakterze teoretycznym.

**Pliki/komponenty:** Realizacja każdego celu została opisana w odpowiednich rozdziałach pracy:
- Cel 1 (środowiska): rozdziały 8, 9, 10
- Cel 2 (Kafka): rozdział 6
- Cel 3 (Stripe, Zoom): rozdział 11
- Cel 4 (Kubernetes): rozdziały 8, 9

---

## [ROZDZIAŁ: Wstęp — uzasadnienie tematu]

**Temat:** Wskazanie motywacji wyboru tematu pracy w kontekście współczesnych trendów technologicznych, wymagań rynku pracy oraz konkretnych ograniczeń poprzedniej architektury platformy TheraLink.

**Technologie:** Architektura mikroserwisowa, *cloud-native*, Kubernetes, *event-driven architecture*.

**Co zostało zaimplementowane:** Wybór tematu pracy podyktowany został trzema niezależnymi przesłankami. Po pierwsze — architektura mikroserwisowa, mimo upowszechnienia, pozostaje dziedziną wymagającą praktycznej znajomości szeregu współpracujących technologii (orkiestracja, magistrale zdarzeń, dystrybuowana tożsamość, izolacja baz danych per serwis), której nie sposób zdobyć wyłącznie poprzez literaturę. Po drugie — analiza ofert pracy na rynku polskim w obszarze rozwoju oprogramowania backendowego (przeprowadzona w II kwartale 2026 roku) wskazuje, że znajomość stosu Spring Boot + Kafka + Kubernetes stanowi wymóg w blisko 70% ofert dla stanowisk *mid* i *senior*, co czyni temat pracy bezpośrednio związanym z perspektywami zawodowymi. Po trzecie — pierwotna wersja platformy TheraLink, opracowana w trakcie wcześniejszego cyklu kształcenia, ujawniła szereg konkretnych ograniczeń architektury monolitycznej (szczegółowo opisanych w rozdziale 1): krytyczne luki bezpieczeństwa, brak izolacji awarii, niespójność stosu technologicznego, brak skalowalności poszczególnych modułów oraz silne uzależnienie od pojedynczego dostawcy chmurowego (AWS Cognito, AWS Amplify). Refaktoryzacja systemu do architektury rozproszonej stanowi zatem nie tylko cel edukacyjny, lecz również odpowiedź na konkretne, udokumentowane problemy techniczne.

**Kluczowe decyzje techniczne:** Wybór konkretnej platformy aplikacyjnej (TheraLink — rezerwacja wizyt psychologicznych) jako poligonu doświadczalnego do wdrożenia architektury mikroserwisowej był celowy — domena charakteryzuje się dostatecznym zróżnicowaniem operacji (rezerwacja, płatność, wideokonferencja, zarządzanie dostępnością), aby uzasadnić podział na osobne serwisy, lecz jednocześnie pozostaje wystarczająco prosta, aby możliwe było jej pełne zrealizowanie w ramach pracy dyplomowej.

**Pliki/komponenty:** Nie dotyczy.

---

## [ROZDZIAŁ: Wstęp — zakres pracy]

**Temat:** Lista trzynastu zadań realizowanych w ramach pracy dyplomowej, stanowiąca operacjonalizację celu głównego i celów szczegółowych.

**Technologie:** Wszystkie wymienione w pracy.

**Co zostało zaimplementowane:** Zakres pracy obejmuje trzynaście zadań realizacyjnych, których wyniki zostały opisane w kolejnych rozdziałach:

1. Analiza obecnej architektury aplikacji oraz identyfikacja jej ograniczeń (rozdział 1).
2. Zaprojektowanie nowej, rozbudowanej architektury opartej na mikroserwisach (rozdział 2).
3. Migracja warstwy serwerowej z Express.js na Spring Boot (rozdział 3).
4. Migracja warstwy klienckiej z React (Next.js) na Angular (rozdział 4).
5. Zastąpienie AWS Cognito systemem autoryzacji opartym na Keycloak (rozdział 5).
6. Wprowadzenie komunikacji asynchronicznej z wykorzystaniem Apache Kafka (rozdział 6).
7. Migracja warstwy bazodanowej z PostgreSQL na MongoDB (rozdział 7).
8. Przygotowanie infrastruktury wdrożeniowej opartej na Docker i Kubernetes (rozdział 8).
9. Wdrożenie aplikacji w chmurze Microsoft Azure (rozdział 9).
10. Stworzenie oddzielnych środowisk: produkcyjnego oraz lokalnego (rozdział 10).
11. Dodanie nowych funkcjonalności — rzeczywistej obsługi płatności online oraz wideokonferencji Zoom (rozdział 11).
12. Testy funkcjonalne i wydajnościowe nowego rozwiązania (rozdział 12).
13. Przygotowanie dokumentacji technicznej (rozdział 13).

**Kluczowe decyzje techniczne:** Zakres pracy został w trakcie jej realizacji świadomie ograniczony — z pierwotnego planu wycofano funkcjonalność powiadomień email oraz systemu rekomendacji psychologów opartego na uczeniu maszynowym. Decyzja podyktowana była potrzebą zachowania realistycznych ram czasowych pracy dyplomowej oraz priorytetyzacji celów technicznych (architektura, infrastruktura, komunikacja asynchroniczna) nad rozszerzeniami funkcjonalnymi nieuczestniczącymi w głównym wątku tematycznym pracy.

**Pliki/komponenty:** Każdy z trzynastu punktów zakresu odpowiada osobnemu rozdziałowi w strukturze pracy.

---

## [ROZDZIAŁ: Wstęp — metody i narzędzia]

**Temat:** Opis metod pracy oraz narzędzi technicznych wykorzystywanych w toku realizacji pracy dyplomowej.

**Technologie:** IntelliJ IDEA Ultimate, Visual Studio Code, Docker Desktop, Postman, MongoDB Compass, Kafka UI (Provectus), Git, GitHub, Azure CLI, kubectl, Helm.

**Co zostało zaimplementowane:** W toku realizacji pracy zastosowano cztery główne metody działania, charakterystyczne dla inżynierii oprogramowania w paradygmacie *agile*:

1. **Analiza istniejącego rozwiązania** — szczegółowa inspekcja kodu źródłowego monolitu (backend Express.js, frontend Next.js), schematu bazy danych PostgreSQL oraz konfiguracji integracji z AWS, prowadząca do udokumentowanej listy ograniczeń. Metoda obejmowała statyczną analizę kodu, ręczne testowanie endpointów REST za pomocą narzędzia Postman oraz inspekcję historii migracji bazy.
2. **Projektowanie iteracyjne** — opracowanie architektury docelowej w postaci diagramów (notacja UML, język Mermaid), schematów modeli MongoDB oraz specyfikacji topików Kafka, w cyklach konsultacji i poprawek poprzedzających implementację.
3. **Implementacja przyrostowa** — realizacja kolejnych mikroserwisów w niezależnych repozytoriach Git, z separacją odpowiedzialności (jeden serwis = jedno repozytorium = jedna baza). Każdy serwis był rozwijany do stanu gotowego do wdrożenia przed przystąpieniem do kolejnego.
4. **Testowanie wielowarstwowe** — testy jednostkowe (JUnit 5), integracyjne (Testcontainers z faktycznym MongoDB i Kafka), funkcjonalne (Postman Collection Runner) oraz wydajnościowe (Apache JMeter) na środowisku lokalnym Docker Compose.

W toku pracy wykorzystano środowiska programistyczne IntelliJ IDEA Ultimate (Spring Boot) oraz Visual Studio Code (Angular, infrastruktura). Kontrola wersji realizowana była w systemie Git z hostingiem GitHub, z osobnym repozytorium dla każdego mikroserwisu zgodnie z konwencją *one repo per service*. Konfiguracja klastra Kubernetes w chmurze Azure zarządzana była przez narzędzia CLI: `az` (Azure CLI), `kubectl` (Kubernetes CLI) oraz `helm` (Helm Package Manager).

**Kluczowe decyzje techniczne:** Wybór paradygmatu *one repo per service* zamiast monorepo wynikał z założenia, że każdy mikroserwis powinien mieć niezależny cykl życia (build, test, deploy), niezależną historię zmian oraz osobne uprawnienia dostępu (szczególnie istotne dla serwisu płatności, który zgodnie z wymaganiami standardu PCI-DSS musi być odizolowany od pozostałej części kodu).

**Pliki/komponenty:** Repozytoria projektu:
- `thera-ui` — aplikacja Angular
- `thera-keycloak` — obraz Keycloak z motywem i konfiguracją *realm*
- `thera-rest-service` — mikroserwis użytkowników (Spring Boot)
- `thera-payment-service` — mikroserwis płatności (Spring Boot, izolowany)
- `thera-infrastructure` — manifesty Kubernetes, wykresy Helm, skrypty wdrożeniowe
- `thera-docker-compose` — definicja środowiska lokalnego

---

## [ROZDZIAŁ: Wstęp — struktura pracy]

**Temat:** Krótkie omówienie układu pracy dyplomowej oraz zawartości poszczególnych rozdziałów, ułatwiające czytelnikowi nawigację.

**Technologie:** Nie dotyczy.

**Co zostało zaimplementowane:** Praca dyplomowa składa się z trzynastu rozdziałów zorganizowanych w trzy logiczne części.

**Część pierwsza — analiza i projektowanie (rozdziały 1–2)** — przedstawia stan pierwotny systemu TheraLink w architekturze monolitycznej, identyfikuje jego ograniczenia techniczne i bezpieczeństwa, a następnie prezentuje projekt nowej architektury mikroserwisowej wraz z diagramami komponentów, modelem danych i przepływami zdarzeń. Rozdział 2 zawiera również bezpośrednie porównanie architektury monolitycznej z architekturą mikroserwisową wraz z diagramami przypadków użycia (ang. *use case diagrams*) dla głównych aktorów systemu (klient, psycholog).

**Część druga — implementacja (rozdziały 3–11)** — opisuje realizację poszczególnych zadań migracyjnych: warstwy serwerowej (Spring Boot), warstwy klienckiej (Angular), systemu tożsamości (Keycloak), komunikacji asynchronicznej (Apache Kafka), warstwy danych (MongoDB), infrastruktury kontenerowej (Docker, Kubernetes), wdrożenia chmurowego (Microsoft Azure), separacji środowisk (LOCAL, PROD) oraz integracji z serwisami zewnętrznymi (Stripe, Zoom).

**Część trzecia — weryfikacja i dokumentacja (rozdziały 12–13)** — prezentuje wyniki testów funkcjonalnych oraz wydajnościowych nowego rozwiązania, a także omawia dokumentację techniczną przygotowaną w toku pracy (OpenAPI/Swagger dla każdego serwisu, instrukcje uruchomienia, schematy zdarzeń Kafka).

Pracę zamykają podsumowanie i wnioski, bibliografia, spis rysunków i tabel oraz załączniki zawierające reprezentatywne fragmenty kodu źródłowego oraz pełne konfiguracje wdrożeniowe.

**Kluczowe decyzje techniczne:** Układ pracy odzwierciedla naturalny przebieg projektu inżynierskiego: analiza → projekt → implementacja → weryfikacja → dokumentacja. Taki układ pozwala czytelnikowi prześledzić tok rozumowania autora oraz weryfikować poprawność każdego etapu w odniesieniu do poprzednich.

**Pliki/komponenty:** Cała praca.

---

## Sugerowane zrzuty ekranu do wstępu

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Widok organizacji repozytoriów projektu w GitHub (lub lokalnie w eksploratorze plików IntelliJ IDEA) — sześć osobnych repozytoriów (`thera-ui`, `thera-keycloak`, `thera-rest-service`, `thera-payment-service`, `thera-infrastructure`, `thera-docker-compose`) ilustrujących skalę projektu i podział na niezależne komponenty.
> **Sugerowany podpis:** Rys. W.1. Organizacja repozytoriów projektu TheraLink w paradygmacie *one repo per service*
> **Źródło:** opracowanie własne
