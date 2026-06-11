# Rozdział 1 — Analiza obecnej architektury

> **Status:** streszczenie wygenerowane przez skill `kod-do-pracy`, gotowe do rozbudowania w Coworku przez skill `praca-inzynierska`.
> **Mapowanie na zakres pracy:** punkt 1.
> **Data wygenerowania:** 2026-06-09 (rewizja po feedbacku użytkownika)
> **Uwagi dla Cowork:** rozdział jest podzielony na 7 bloków `## [ROZDZIAŁ:...]` o tej samej nazwie — każdy ma stanowić odrębny podrozdział (1.1, 1.2, …, 1.7).

---

## [ROZDZIAŁ: Analiza obecnej architektury — przegląd ogólny]

**Temat:** Ogólna charakterystyka monolitycznej wersji systemu TheraLink, jej warstw, granic odpowiedzialności i stosu technologicznego, sprzed migracji do architektury mikroserwisowej.

**Technologie:** Node.js, Express.js 4.21, TypeScript 5.8, Prisma ORM 6.5, PostgreSQL z rozszerzeniem PostGIS, AWS Cognito, AWS Amplify 6.14, AWS S3 SDK, Next.js 15 (App Router), React 19, Redux Toolkit 2.6 + RTK Query, Google Maps API (`@react-google-maps/api`), Material UI 7.1, Radix UI, shadcn/ui, Tailwind CSS 4, DeepSeek przez OpenRouter, `@ai-sdk/react`, jsonwebtoken, bcrypt, helmet, morgan.

**Co zostało zaimplementowane:** System pierwotnie zrealizowano jako monolit dwuwarstwowy. Warstwę serwerową stanowi pojedynczy proces Node.js z frameworkiem Express.js i językiem TypeScript, obsługujący 18 endpointów REST pogrupowanych w sześciu plikach tras. Warstwę klienta tworzy aplikacja Next.js 15 wykorzystująca App Router, około 60 plików TypeScript/TSX (z czego 29 stanowi strony w katalogu `app/`), oraz zarządzanie stanem oparte o Redux Toolkit i RTK Query. Obie aplikacje współdzielą jedno repozytorium i wdrażane są wspólnie. Łącznie kod monolitycznego backendu zajmuje 729 linii w 13 plikach, kod frontendu — około 60 plików, plus 16 komponentów dedykowanych pływającemu czatowi AI. Tożsamość użytkowników zarządzana jest przez AWS Cognito; jako klucz korelacyjny w bazie danych występuje pole `cognitoId`, używane jako klucz obcy w czterech z pięciu modeli Prisma. Komunikacja z chmurą AWS realizowana jest po stronie klienta przez bibliotekę AWS Amplify, a po stronie serwera — bezpośrednio przez tokeny JWT wystawiane przez Cognito.

**Kluczowe decyzje techniczne:** Architektura monolityczna została wybrana na początkowym etapie projektu jako rozwiązanie minimalizujące koszt wdrożenia i przyspieszające iterację — jeden proces, jedno repozytorium, jeden schemat bazy. Wybór AWS Cognito jako dostawcy tożsamości umożliwiał wykorzystanie gotowych komponentów UI z biblioteki AWS Amplify, kosztem silnego sprzężenia z ekosystemem chmurowym jednego dostawcy (ang. *vendor lock-in*). Decyzja o użyciu rozszerzenia PostGIS w PostgreSQL podyktowana była planem wyszukiwania psychologów po lokalizacji geograficznej, lecz w finalnej implementacji nigdy nie została wykorzystana — schemat nie definiuje żadnych kolumn geograficznych.

**Pliki/komponenty:**
- `backend/src/index.ts` — punkt wejścia, konfiguracja Express, montowanie 6 routerów
- `backend/src/controllers/` — 5 kontrolerów
- `backend/src/routes/` — 6 plików tras + endpoint czatu
- `backend/src/middleware/authMiddleware.ts` — jedyny middleware projektu
- `backend/prisma/schema.prisma` — 5 modeli, 2 enumy, 11 migracji w historii
- `frontend/app/` — strony Next.js App Router
- `frontend/state/api.ts`, `state/redux.tsx`, `state/index.ts` — warstwa stanu
- `frontend/app/(auth)/authProvider.tsx` — konfiguracja AWS Amplify Authenticator

---

## [ROZDZIAŁ: Analiza obecnej architektury — warstwa serwerowa]

**Temat:** Szczegółowa analiza monolitycznej warstwy serwerowej zbudowanej w technologii Node.js + Express.js + Prisma, obejmująca strukturę kodu, listę endpointów, organizację kontrolerów i wzorce pracy z bazą danych.

**Technologie:** Node.js, Express.js 4.21, TypeScript 5.8, Prisma ORM 6.5, PostgreSQL, jsonwebtoken, axios, helmet, morgan, body-parser, cors.

**Co zostało zaimplementowane:** Backend monolityczny udostępnia łącznie 18 endpointów REST rozdzielonych pomiędzy 6 grupami tras: `/clients` (2 endpointy), `/psychologists` (4 endpointy), `/availabilities` (4 endpointy), `/sync-role` (1 endpoint), `/appointments` (5 endpointów), `/api/chat` (1 endpoint integrujący platformę OpenRouter z modelem DeepSeek Reasoner). Wszystkie kontrolery operują bezpośrednio na kliencie Prisma, bez wyodrębnionej warstwy serwisowej i bez obiektów transferu danych (ang. *Data Transfer Object*). W aplikacji nie zdefiniowano centralnego punktu utworzenia klienta Prisma — każdy z pięciu kontrolerów (`clientController.ts:4`, `appointmentController.ts:4`, `psychologistController.ts:4`, `availabilityController.ts:4`, `syncRoleController.ts:6`) instancjonuje własny obiekt `new PrismaClient()`. W całym kodzie nie zaimplementowano żadnego endpointu typu DELETE — jedyna operacja usuwania (`prisma.client.delete` w `syncRoleController.ts:24`) wykonywana jest jako część zmiany roli użytkownika, bez owinięcia w transakcję, co w przypadku awarii drugiej operacji (utworzenia psychologa) prowadzi do utraty danych konta klienta. Warstwa walidacji wejścia nie istnieje — projekt nie używa żadnej z bibliotek typu Joi, Zod ani express-validator; dane z `req.body` przekazywane są bezpośrednio do operacji Prisma. Obsługa błędów realizowana jest indywidualnie w każdym kontrolerze przez blok `try/catch` zwracający status 500 z wiadomością z pola `error.message` — bez centralnego *error handlera* i bez rozróżnienia kodów odpowiedzi w zależności od typu błędu. Wykryto bezpośrednie duplikaty funkcjonalności: cztery funkcje (`createAvailability`, `getAvailabilitiesForPsychologist`, `getAvailability`, `updateAvailability`) zdefiniowane są zarówno w `availabilityController.ts`, jak i w `psychologistController.ts`, w identycznym brzmieniu.

**Kluczowe decyzje techniczne:** Brak warstwy serwisowej oraz tworzenie wielu instancji `PrismaClient` to typowe konsekwencje minimalnego początkowego projektu, lecz w produkcji prowadzą do wyczerpania puli połączeń bazy oraz utrudniają jednostkowe testowanie logiki biznesowej. Brak walidacji wejścia oraz brak transakcji w operacji zmiany roli stanowią poważne ograniczenia integralności danych. Wybór protokołu wyłącznie synchronicznego (REST) oznacza brak możliwości realizacji asynchronicznych przepływów (potwierdzenia płatności, zlecenia zewnętrzne), które blokowałyby wątek obsługujący żądanie HTTP klienta.

**Pliki/komponenty:**
- `backend/src/index.ts:1-44` — punkt wejścia z konfiguracją CORS, helmet, morgan, body-parser
- `backend/src/routes/userRoutes.ts`, `psychologistsRoutes.ts`, `availabilityRoutes.ts`, `syncRoleRoutes.ts`, `appointmentRoutes.ts`, `chat.ts` — 6 plików tras
- `backend/src/controllers/*.ts` — 5 plików kontrolerów z duplikatami funkcji
- `backend/src/middleware/authMiddleware.ts` — jedyny middleware projektu

---

## [ROZDZIAŁ: Analiza obecnej architektury — warstwa kliencka]

**Temat:** Szczegółowa analiza monolitycznej warstwy klienckiej zbudowanej w Next.js 15 z React 19, Redux Toolkit i AWS Amplify, obejmująca strukturę aplikacji, zarządzanie stanem, integracje zewnętrzne i jakość kodu.

**Technologie:** Next.js 15 (App Router), React 19, TypeScript 5, Redux Toolkit 2.6 + RTK Query, AWS Amplify 6.14 (Authenticator), `@react-google-maps/api`, `@ai-sdk/react`, Material UI 7.1, Radix UI (`@radix-ui/react-collapsible`, `@radix-ui/react-slot`), shadcn/ui, Tailwind CSS 4, React Hook Form + Zod (zainstalowane, lecz nieużywane), Framer Motion, Lucide React, Sonner.

**Co zostało zaimplementowane:** Aplikacja kliencka realizuje sześć głównych obszarów funkcjonalnych dostępnych przez Next.js App Router: stronę główną z formularzem wyszukiwania, rejestrację (z możliwością wyboru roli w polu `custom:role` formularza Amplify Authenticator — `authProvider.tsx:67-76`), logowanie, listę psychologów z filtrowaniem i mapą Google Maps (`view/page.tsx:14-163`), profil użytkownika oraz potwierdzenie rezerwacji. Niezależne dashboardy `ClientDashboard.tsx` (z czterema sekcjami: ustawienia konta, wizyty, płatności, weryfikacja) i `PsychologistDashboard.tsx` (z sześcioma sekcjami, w tym dostępność i kalendarz) udostępniają funkcje zależne od roli. Warstwa stanu opiera się na Redux Store z pojedynczym `globalSlice` (`state/index.ts:35-49`) przechowującym filtry wyszukiwania oraz preferencje UI, oraz na 14 endpointach RTK Query zdefiniowanych w `state/api.ts`. Endpointy podzielone są na pięć grup: autentykacja (`getAuthUser`, `syncUserRole`), zarządzanie psychologami (`getAllPsychologists`, `updatePsychologist`), zarządzanie klientami (`updateClient`), dostępności (4 endpointy) i wizyty (5 endpointów) — bez endpointów DELETE i bez zaawansowanego wyszukiwania. Token tożsamości pobierany jest przy każdym żądaniu przez `fetchAuthSession()` z AWS Amplify i dołączany w nagłówku `Authorization: Bearer` w `prepareHeaders` (`state/api.ts:19-26`). Pływający widget czatu AI (`FloatingChat.tsx`, `chat-demo.tsx` plus 16 komponentów w `components/chatbot-ui/`) realizuje strumieniową komunikację z modelem DeepSeek Reasoner przez własny *route handler* Next.js (`app/api/chat/route.ts`). Aplikacja wykorzystuje trzy równoległe systemy komponentów UI: Material UI dla głównych ekranów, Radix UI dla prymitywów dostępności oraz shadcn/ui dla widgetów czatu, bez spójnego systemu *design tokens* — kolory hardkodowane są w komponentach (`#2b6369`, `#256269`, `#00bfa5`). Cały projekt nie zawiera ani jednego pliku testowego (`*.test.ts`, `*.spec.ts`), zaś w produkcyjnym kodzie pozostają wywołania `console.log` (`chat-demo.tsx:25, 29, 66`).

**Kluczowe decyzje techniczne:** Decyzja o wykorzystaniu RTK Query zamiast samodzielnego klienta HTTP pozwala na deklaratywne cachowanie odpowiedzi i automatyczne unieważnianie tagów po mutacjach. Wybór AWS Amplify jako gotowego klienta tożsamości zaoszczędził pracy implementacyjnej, lecz w praktyce ścisle związał frontend z chmurą AWS — biblioteka `aws-amplify` używana jest w 12 plikach źródłowych. Współistnienie trzech systemów komponentów UI oraz instalacja bibliotek React Hook Form i Zod, które w analizowanym stanie kodu nie są wykorzystywane, wskazują na niespójne decyzje technologiczne i niedokończone implementacje. Zmiana biblioteki mapowej z Mapbox (zadeklarowanej w `package.json`) na Google Maps (faktycznie używaną w `GMap.tsx`) nie została odzwierciedlona w dokumentacji projektu.

**Pliki/komponenty:**
- `frontend/app/(auth)/authProvider.tsx` — konfiguracja Amplify Authenticator, custom header i footer, sync roli
- `frontend/app/providers.tsx` — kompozycja `StoreProvider`, `Authenticator.Provider`, `Auth`
- `frontend/state/api.ts` — 14 endpointów RTK Query z pobieraniem IdToken przez `fetchAuthSession()`
- `frontend/state/redux.tsx` — konfiguracja Redux Store z `setupListeners` dla RTK Query
- `frontend/state/index.ts` — `globalSlice` z hardkodowanymi koordynatami Elbląga (54.1522, 19.4088)
- `frontend/app/components/GMap.tsx` — integracja Google Maps z geokodowaniem on-demand, bez cache
- `frontend/app/dashboard/client/ClientDashboard.tsx`, `psychologist/PsychologistDashboard.tsx` — dwa dashboardy zależne od roli
- `frontend/components/chatbot-ui/` — 16 komponentów warstwy czatu (`chat`, `message-list`, `message-input`, `markdown-renderer`, `audio-visualizer`, …)
- `frontend/app/api/chat/route.ts` — *route handler* strumienia DeepSeek
- `frontend/types/prismaTypes.d.ts` — typy generowane z backendowej Prisma, kopiowane przez skrypt `postprisma:generate` w `backend/package.json`

---

## [ROZDZIAŁ: Analiza obecnej architektury — warstwa danych i schemat bazy]

**Temat:** Analiza warstwy danych monolitu opartej na PostgreSQL i Prisma ORM, obejmująca schemat modeli, relacje, historię migracji oraz wykorzystanie rozszerzenia PostGIS.

**Technologie:** PostgreSQL 16, Prisma ORM 6.5, rozszerzenie PostGIS, `@terraformer/wkt` (zainstalowany, nieużywany).

**Co zostało zaimplementowane:** Schemat bazy danych zdefiniowany w pliku `prisma/schema.prisma` obejmuje pięć modeli (`Client`, `Psychologist`, `Appointment`, `Payment`, `CalendarAppointment`) oraz dwa typy wyliczeniowe (`ApplicationStatus`, `PaymentStatus`). Klucz unikalny `cognitoId` typu `String` występuje jako identyfikator domenowy w czterech z pięciu modeli i jest używany jako klucz obcy w relacjach `Appointment.clientCognitoId → Client.cognitoId`, `Appointment.psychologistId → Psychologist.cognitoId`, `Payment.clientCognitoId → Client.cognitoId` (pole opcjonalne — co semantycznie dopuszcza płatność bez przypisanego klienta), `CalendarAppointment.psychologistId → Psychologist.cognitoId`. Model `Payment` powiązany jest z `Appointment` relacją 1:1 przez `appointmentId @unique`. Schemat zawiera niespójność nazewniczą — model `CalendarAppointment` posiada dyrektywę `@@map("appointments")`, mapującą go na tabelę bazodanową o nazwie identycznej z tabelą modelu `Appointment` w nazwie, lecz innej strukturze. Pola modeli stosują niespójną konwencję nazewniczą: jednocześnie `camelCase` (`hourlyRate`, `phoneNumber`) i `PascalCase` (`Description`, `Specialization`, `Status`). Historia migracji bazy danych obejmuje 11 wpisów rozłożonych w okresie marzec – czerwiec 2025, w tym powtarzające się migracje dotyczące tego samego pola (`add_cognito_id` występuje dwukrotnie, `change_psychologistid_to_string` i `appointment_psychologist_id_to_string` realizują podobną zmianę). Rozszerzenie PostGIS zostało jawnie włączone w bloku `extensions = [postgis]` (`schema.prisma:9`), lecz w żadnym modelu nie zdefiniowano kolumn typu geograficznego — funkcjonalność wyszukiwania psychologów po lokalizacji nie została zaimplementowana. Schemat nie definiuje żadnych indeksów wtórnych — pole `email` w modelach `Client` i `Psychologist` jest często używane jako kryterium wyszukiwania, lecz nie posiada indeksu.

**Kluczowe decyzje techniczne:** Wybór relacyjnej bazy PostgreSQL z rozszerzeniem PostGIS podyktowany był wymaganiem wyszukiwania geograficznego — w praktyce wymaganie to nigdy nie zostało zaspokojone na poziomie schematu. Użycie `cognitoId` jako klucza obcego w czterech modelach (zamiast wewnętrznego identyfikatora bazodanowego) silnie powiązało schemat danych z konkretnym dostawcą tożsamości, czyniąc migrację na inną technologię (Keycloak, Auth0) operacją wymagającą zmiany schematu wszystkich powiązanych tabel.

**Pliki/komponenty:**
- `backend/prisma/schema.prisma` — definicja modeli, enumów, włączenie PostGIS
- `backend/prisma/migrations/` — 11 katalogów migracji z historią zmian schematu
- `backend/prisma/seed.ts` i `prisma/seedData/` — dane testowe ładowane do bazy lokalnej

---

## [ROZDZIAŁ: Analiza obecnej architektury — bezpieczeństwo i zarządzanie sekretami]

**Temat:** Analiza krytycznych luk bezpieczeństwa oraz nieprawidłowości w zarządzaniu sekretami zidentyfikowanych w monolitycznej wersji systemu TheraLink.

**Technologie:** jsonwebtoken, AWS Cognito (przyjmowane tokeny JWT), AWS Amplify, OpenRouter API, zmienne `NEXT_PUBLIC_*` w Next.js.

**Co zostało zaimplementowane:** Autoryzacja w warstwie serwerowej realizowana jest przez pojedynczy middleware `authMiddleware(allowedRoles[])` (`backend/src/middleware/authMiddleware.ts`), który ekstrahuje token z nagłówka `Authorization: Bearer`, dekoduje payload funkcją `jwt.decode(token)` (`linia 30`) i porównuje pole `custom:role` z listą dozwolonych ról. Wykryto pięć osobnych problemów bezpieczeństwa: po pierwsze — funkcja `jwt.decode()` jedynie dekoduje payload Base64 bez weryfikacji kryptograficznego podpisu tokenu; w praktyce dowolny użytkownik może utworzyć ważnie wyglądający token z atrybutem `custom:role` ustawionym na `psychologist` lub `admin` i uzyskać nieautoryzowany dostęp do API. Po drugie — jedenaście z osiemnastu endpointów (`/clients/*`, większość `/psychologists/*`, wszystkie `/appointments/*`, `/api/chat`) w ogóle nie wymaga przejścia przez middleware autoryzacji. Po trzecie — w pliku `backend/src/routes/chat.ts:7` zakodowano na stałe rzeczywisty klucz API platformy OpenRouter w postaci stałej `OPENROUTER_API_KEY = 'sk-or-v1-...8002'`, znajdujący się w repozytorium kontroli wersji. Po czwarte — plik `frontend/public/.env` umieszczony został w katalogu publicznym Next.js, który framework serwuje pod ścieżką statyczną; każdy klient HTTP może pobrać zawartość tego pliku bez uwierzytelnienia. Po piąte — pola `req.body` (np. `req.body.Status` w `appointmentController.ts:11`) odczytywane są bez walidacji i bez schemato wania, co umożliwia *mass assignment* oraz wprowadzanie nielegalnych wartości do bazy. Po stronie klienta autoryzacja realizowana jest wyłącznie warunkowo, w komponencie `Auth` (`authProvider.tsx:144-146`) sprawdzającym `pathname` — bez dedykowanego *middleware* Next.js i bez chroniących tras *route guards*.

**Kluczowe decyzje techniczne:** Użycie `jwt.decode()` zamiast `jwt.verify()` było rezultatem braku znajomości różnicy semantycznej między obiema funkcjami w trakcie pierwotnej implementacji — funkcja `verify` wymagałaby pobierania kluczy publicznych z punktu JWKS (ang. *JSON Web Key Set*) AWS Cognito, co stanowi nietrywialny wysiłek implementacyjny. Hardkodowanie klucza API w pliku źródłowym wynikało z presji czasowej w trakcie pierwszego prototypu chatbota AI i nie zostało następnie skorygowane.

**Pliki/komponenty:**
- `backend/src/middleware/authMiddleware.ts:30` — wywołanie `jwt.decode()` bez weryfikacji podpisu
- `backend/src/routes/chat.ts:7` — zakodowany klucz `OPENROUTER_API_KEY`
- `frontend/public/.env` — publicznie dostępny plik zawierający klucze API
- `frontend/.env.local` — drugi plik z duplikatami tych samych kluczy
- `frontend/app/(auth)/authProvider.tsx:140-188` — warunkowa kontrola dostępu po stronie klienta

---

## [ROZDZIAŁ: Analiza obecnej architektury — zidentyfikowane ograniczenia]

**Temat:** Systematyczna lista ograniczeń technicznych, organizacyjnych i bezpieczeństwa pierwotnej architektury monolitycznej, stanowiących uzasadnienie podjęcia decyzji o migracji.

**Technologie:** Wszystkie wymienione powyżej.

**Co zostało zaimplementowane:** W toku analizy kodu źródłowego, schematu bazy danych oraz konfiguracji projektu zidentyfikowano jedenaście istotnych ograniczeń architektury monolitycznej:

1. **Brak izolacji awarii.** Całość logiki backendowej (autoryzacja, rezerwacje, profile, czat AI) działa w jednym procesie Node.js. Awaria pojedynczego modułu — w szczególności blokujące wywołanie do `openrouter.ai` z chatbota — skutkuje niedostępnością całej aplikacji, w tym krytycznych funkcji rezerwacji wizyt.
2. **Brak niezależnego skalowania modułów.** Moduł czatu AI generujący znaczne obciążenie sieciowe oraz obsługa rezerwacji o niskim ruchu skalują się wspólnie, co prowadzi do nieefektywnego wykorzystania zasobów.
3. **Krytyczna luka bezpieczeństwa — brak weryfikacji podpisu tokenu JWT.** W pliku `backend/src/middleware/authMiddleware.ts:30` użyto funkcji `jwt.decode()` zamiast `jwt.verify()`, co umożliwia podszycie się pod dowolną rolę (szczegółowo opisane w podrozdziale dotyczącym bezpieczeństwa).
4. **Zakodowany na stałe klucz API w kodzie źródłowym.** W pliku `backend/src/routes/chat.ts:7` zdefiniowano stałą `OPENROUTER_API_KEY` z rzeczywistym kluczem, znajdującym się w repozytorium.
5. **Eksponowanie zmiennych środowiskowych przez Next.js.** Plik `frontend/public/.env` umieszczony w katalogu publicznym serwowany jest pod adresem statycznym bez uwierzytelnienia.
6. **Vendor lock-in z AWS Cognito.** Identyfikacja użytkowników (`cognitoId`) jest częścią schematu bazy danych jako klucz obcy w czterech z pięciu modeli; w warstwie klienckiej biblioteka `aws-amplify` używana jest w 12 plikach źródłowych.
7. **Brak komunikacji asynchronicznej.** Wszystkie operacje między modułami realizowane są synchronicznie. System nie posiada ani kolejki wiadomości (ang. *message queue*), ani magistrali zdarzeń (ang. *event bus*), ani zadań cyklicznych (ang. *scheduled jobs*).
8. **Brak rozdzielenia logiki biznesowej od warstwy dostępu do danych.** Kontrolery operują bezpośrednio na `prisma.appointment.create()` i podobnych wywołaniach, bez warstwy serwisowej ani obiektów DTO. Każdy z pięciu kontrolerów instancjonuje własny obiekt `PrismaClient`, co stanowi antywzorzec — pula połączeń bazy nie jest współdzielona.
9. **Brak walidacji wejścia.** Projekt nie używa żadnej biblioteki walidacyjnej; dane z `req.body` przekazywane są bezpośrednio do operacji bazodanowych, co umożliwia *mass assignment* oraz zapis nieprawidłowych wartości.
10. **Brak transakcji w operacjach wielokrokowych.** Operacja zmiany roli klienta w psychologa (`syncRoleController.ts`) wykonuje sekwencyjnie `delete` rekordu klienta i `create` rekordu psychologa bez owinięcia w transakcję — w przypadku awarii drugiego kroku konto użytkownika zostaje utracone.
11. **Ograniczona kontrola nad zarządzaniem rolami w AWS Cognito.** Definicja ról oraz zarządzanie nimi w AWS Cognito wymaga dodawania *custom attributes* (`custom:role`) w konsoli AWS, bez pełnej kontroli programowej. Lokalne środowisko deweloperskie nie ma dostępu do realnego Cognito — każda zmiana roli wymaga rekonfiguracji w chmurze. Brak wsparcia dla zaawansowanych przepływów autoryzacyjnych (np. *role hierarchies*, dynamiczne uprawnienia per zasób, polityki ABAC). Każde aktywne konto użytkownika (MAU — *Monthly Active User*) wiąże się z opłatą po przekroczeniu darmowego progu, co przy modelu cenowym typowego startupu jest barierą skalowania.

**Dodatkowo zidentyfikowane braki jakościowe:**
- **Zero testów** — w repozytorium nie znaleziono żadnego pliku testowego (`*.test.ts`, `*.spec.ts`).
- **Duplikaty kodu** — cztery funkcje obsługi dostępności (`createAvailability`, `getAvailabilitiesForPsychologist`, `getAvailability`, `updateAvailability`) zdefiniowane są równolegle w `availabilityController.ts` i `psychologistController.ts`.
- **Niespójność stosu UI** — frontend wykorzystuje równolegle trzy systemy komponentów (Material UI, Radix UI, shadcn/ui), bez wspólnego systemu *design tokens*.
- **Martwe zależności** — pakiety `@aws-sdk/client-s3`, `@aws-sdk/lib-storage`, `@terraformer/wkt` w backendzie oraz `mapbox-gl`, `react-hook-form`, `zod` w frontendzie są zainstalowane, lecz nie używane.
- **Chaotyczna ewolucja schematu** — 11 migracji bazy w 3-miesięcznym okresie z powtarzającymi się zmianami tego samego pola wskazuje na brak początkowego planu modelowania danych.
- **Brak strukturyzowanego logowania i obserwowalności** — jedyne logowanie realizowane jest przez `morgan("common")` oraz pojedyncze wywołania `console.log` / `console.error`, bez integracji z systemem zbierania metryk czy śledzenia żądań.
- **Brak paginacji** — endpoint `getAllPsychologists()` zwraca pełną listę bez paginacji, co przy wzroście liczby psychologów prowadzi do liniowego wzrostu zużycia pamięci i przepustowości.

**Kluczowe decyzje techniczne:** Suma jedenastu wymienionych ograniczeń stanowi wystarczające uzasadnienie dla podjęcia decyzji o pełnej refaktoryzacji architektury w kierunku rozproszonej, zdarzeniowej i izolowanej warstwowo. Pozostawienie systemu w obecnej postaci uniemożliwiałoby uzyskanie wymaganego poziomu bezpieczeństwa (wymóg poprawnej weryfikacji JWT), skalowalności (niezależne skalowanie modułów o różnych profilach obciążenia) oraz zgodności z wymaganiami zewnętrznych integracji (PCI-DSS dla obsługi płatności kartą).

**Pliki/komponenty:** Wszystkie wymienione w poprzednich blokach.

---

## [ROZDZIAŁ: Analiza obecnej architektury — uzasadnienie wyboru technologii migracji]

**Temat:** Mapowanie zidentyfikowanych ograniczeń pierwotnej architektury na decyzje o wyborze technologii zastępczych w nowej architekturze mikroserwisowej, wraz z uzasadnieniem każdego wyboru.

**Technologie:** Angular 21, Spring Boot 4.0, Spring Security OAuth2, MongoDB 7.0 / Azure Cosmos DB, Keycloak 25, Apache Kafka / Azure Event Hubs, Docker, Kubernetes (Azure AKS), Stripe, Zoom Server-to-Server OAuth.

**Co zostało zaimplementowane:** Każda z kluczowych decyzji technologicznych nowej architektury została podyktowana konkretnym ograniczeniem zidentyfikowanym w architekturze pierwotnej:

**Next.js → Angular.** Angular jako framework typu *opinionated* narzuca spójną strukturę projektu opartą na modułach, wstrzykiwaniu zależności (ang. *Dependency Injection*, DI) oraz serwisach, eliminując problem niespójności trzech równoległych systemów komponentów UI obserwowany w monolicie. Wbudowane narzędzie Angular CLI eliminuje konieczność ręcznego dobierania bibliotek do trasowania (router), formularzy (reactive forms) i zarządzania stanem aplikacji, czym adresuje obserwowany w monolicie problem zainstalowanych, lecz nieużywanych bibliotek (React Hook Form, Zod).

**Node.js + Express → Spring Boot.** Ekosystem Spring dostarcza gotowe komponenty dla architektury mikroserwisowej: Spring Cloud Gateway dla bramki API, Spring Data MongoDB dla warstwy persystencji, Spring Kafka dla komunikacji asynchronicznej. Kluczowym uzasadnieniem migracji jest wbudowana funkcjonalność Spring Security OAuth2 Resource Server, która automatycznie weryfikuje podpisy tokenów JWT przez pobieranie kluczy publicznych z punktu JWKS dostawcy tożsamości. W Expressie była to manualna integracja, która w pierwotnej implementacji doprowadziła do krytycznej luki bezpieczeństwa — użycia `jwt.decode()` zamiast `jwt.verify()`. W Spring Boot deweloper nie pisze żadnego kodu weryfikacji — odpowiada za to framework.

**PostgreSQL → MongoDB.** Decyzja podyktowana trzema czynnikami: brak wymagania transakcji wielodokumentowych w domenie biznesowej (każda operacja dotyczy jednego dokumentu), dostępność zarządzanej usługi *cloud-native* (Azure Cosmos DB z API zgodnym z MongoDB), oraz mapowanie obiekt-dokument w stosunku 1:1 (jeden agregat domenowy = jeden dokument), upraszczające warstwę dostępu do danych w porównaniu do relacyjnego modelu z licznymi tabelami i kluczami obcymi. Dodatkowo wzorzec *database per service* z natury wspiera niezależną ewolucję schematu każdego mikroserwisu.

**AWS Cognito → Keycloak.** Keycloak jako rozwiązanie *open source* zapewnia pełną kontrolę nad definicjami ról, *realm* środowiskowymi i przepływami autoryzacji, eliminując ograniczenia opisane w punkcie 11 listy ograniczeń (konieczność rekonfiguracji w konsoli AWS, brak lokalnego środowiska deweloperskiego, koszty MAU). Keycloak można uruchomić zarówno w środowisku deweloperskim (przez Docker Compose), jak i w produkcji na klastrze Kubernetes, bez ponoszenia dodatkowych opłat licencyjnych.

**Apache Kafka.** Nowa architektura wprowadza komunikację asynchroniczną na bazie Apache Kafka (lokalnie) i Azure Event Hubs z kompatybilnym protokołem Kafka (produkcja). Adresuje to ograniczenie nr 7 — brak izolacji modułów oraz brak możliwości realizacji przepływów zdarzeniowych (potwierdzenie płatności, tworzenie spotkania Zoom po opłaceniu wizyty).

**Docker + Kubernetes (Azure AKS).** Kontenery Docker zapewniają izolację runtime'u każdego mikroserwisu, eliminując problem współdzielonego procesu Node.js z punktu 1 listy ograniczeń. Orkiestracja Kubernetes na klastrze Azure Kubernetes Service umożliwia niezależne skalowanie poszczególnych serwisów w odpowiedzi na ich indywidualne profile obciążenia.

**Zewnętrzne serwisy:** Stripe stanowi platformę obsługi rzeczywistych płatności kartą — funkcjonalność, której pierwotny monolit nie posiadał. Zoom (przez interfejs *Server-to-Server OAuth*) realizuje automatyczne tworzenie spotkań wideo dla wizyt zdalnych po opłaceniu rezerwacji.

**Kluczowe decyzje techniczne:** Wybór każdej z technologii zastępczych ma jednoznaczne uzasadnienie wynikające z ograniczenia obserwowanego w monolicie. Decyzje nie wynikają z mody technologicznej, lecz z konkretnych potrzeb wynikających z analizy stanu pierwotnego. Wszystkie wybrane technologie posiadają zarządzane odpowiedniki w chmurze Microsoft Azure (Azure Cosmos DB dla MongoDB, Azure Event Hubs dla Kafka, Azure AKS dla Kubernetes), co umożliwia spójną strategię wdrożenia produkcyjnego.

**Pliki/komponenty:** Decyzje technologiczne są realizowane w odrębnych repozytoriach mikroserwisów, opisanych szczegółowo w rozdziałach 3–11 pracy.

---

## Sugerowane zrzuty ekranu do tego rozdziału

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Drzewo katalogów monolitu — `backend/src/` z podkatalogami `controllers`, `routes`, `middleware`, oraz `frontend/app/` z podkatalogami Next.js App Router (`signin`, `signup`, `dashboard`, `profile`, `view`, `confirm-booking`, `(auth)`). Idealnie zrzut z IntelliJ IDEA lub VS Code z rozwiniętym drzewem do drugiego poziomu zagłębienia.
> **Sugerowany podpis:** Rys. 1.1. Struktura katalogów monolitycznej wersji systemu TheraLink
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram architektury monolitu — pojedynczy blok "Backend Node.js/Express (18 endpointów)" połączony z PostgreSQL/PostGIS, AWS Cognito, AWS S3, OpenRouter API, oraz blok "Frontend Next.js" (Redux + RTK Query + AWS Amplify + Google Maps). Diagram można wygenerować z bloku Mermaid (`graph LR`) i wyeksportować do PNG przez mermaid.live.
> **Sugerowany podpis:** Rys. 1.2. Architektura systemu TheraLink w wersji monolitycznej
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Fragment kodu `backend/src/middleware/authMiddleware.ts`, linie 20–50, z wyraźnie widocznym wywołaniem `jwt.decode(token)` (linia 30). Najlepiej zrzut z IDE z podświetloną składnią — linia z `jwt.decode` ma być wyróżniona (czerwone tło, strzałka lub komentarz "BRAK WERYFIKACJI PODPISU").
> **Sugerowany podpis:** Rys. 1.3. Krytyczna luka bezpieczeństwa w pierwotnej implementacji middleware autoryzacji — brak weryfikacji podpisu tokenu JWT
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Fragment kodu `backend/src/routes/chat.ts`, linie 1–15, z wyraźnie widoczną zakodowaną na stałe stałą `OPENROUTER_API_KEY`. Klucz na zrzucie powinien być częściowo zamazany (np. pokazane pierwsze i ostatnie 4 znaki: `sk-or-v1-8d32...8002`), aby rzeczywisty klucz nie trafił do treści pracy dyplomowej.
> **Sugerowany podpis:** Rys. 1.4. Klucz API platformy OpenRouter zakodowany na stałe w kodzie źródłowym
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Zrzut katalogu `frontend/public/` w eksploratorze plików IDE — z widocznym plikiem `.env` obok plików `.svg` (`file.svg`, `globe.svg`, `next.svg`, `vercel.svg`). Zrzut ma wizualnie pokazać, że plik z sekretami leży w katalogu publicznych zasobów Next.js.
> **Sugerowany podpis:** Rys. 1.5. Plik konfiguracyjny ze zmiennymi środowiskowymi umieszczony w publicznym katalogu zasobów Next.js
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Diagram ERD wygenerowany z `prisma/schema.prisma` — pięć modeli (`Client`, `Psychologist`, `Appointment`, `Payment`, `CalendarAppointment`) z relacjami i kluczami obcymi opartymi na polu `cognitoId`. Można wygenerować przez Prisma Editor, DBeaver lub dbdiagram.io.
> **Sugerowany podpis:** Rys. 1.6. Schemat relacyjnej bazy danych PostgreSQL w wersji monolitycznej — model danych pierwotnej aplikacji
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Lista 11 katalogów migracji w `backend/prisma/migrations/` z widocznymi nazwami zawierającymi daty oraz powtarzające się elementy (`add_cognito_id`, `change_psychologistid_to_string`, `appointment_psychologist_id_to_string`). Zrzut ilustruje chaotyczną ewolucję schematu.
> **Sugerowany podpis:** Rys. 1.7. Historia migracji bazy danych monolitu — 11 migracji w okresie marzec–czerwiec 2025
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Konsola AWS Cognito z widoczną zakładką *User Pool Attributes* i polem `custom:role` jako *custom attribute*. Zrzut ilustruje, że zarządzanie rolami wymaga ręcznej konfiguracji w panelu administracyjnym AWS, bez programowej kontroli z poziomu aplikacji.
> **Sugerowany podpis:** Rys. 1.8. Konfiguracja atrybutu niestandardowego `custom:role` w panelu AWS Cognito
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Tabela porównawcza technologii pierwotnych i zastępczych — kolumny: "Komponent", "Monolit", "Mikroserwisy", "Uzasadnienie migracji". Można wykonać jako tabelę w Wordzie lub diagram. Wiersze: Frontend (Next.js → Angular), Backend (Express → Spring Boot), Baza danych (PostgreSQL → MongoDB), Tożsamość (Cognito → Keycloak), Komunikacja (REST sync → Kafka async), Orkiestracja (brak → Docker+K8s), Płatności (brak → Stripe), Wideokonferencje (brak → Zoom).
> **Sugerowany podpis:** Rys. 1.9. Zestawienie technologii pierwotnej architektury monolitycznej i nowej architektury mikroserwisowej
> **Źródło:** opracowanie własne
