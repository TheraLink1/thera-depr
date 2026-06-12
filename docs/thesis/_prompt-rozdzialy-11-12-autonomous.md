# Prompt do sesji autonomicznej — rozdz. 11 + rozdz. 12

> **Jak użyć:** otwórz nowe okno Claude Code w katalogu `/Users/desirecutieqb/IdeaProjects/TheraLink`, wybierz model **Opus 4.7**, wklej cały blok poniżej i uruchom. Agent pracuje sam do końca, na koniec commit + push do origin/main.

---

## Tryb pracy

Pracujesz autonomicznie — autor jest nieobecny. Twoje pełnomocnictwo:
- **Pełne**: możesz commit + push do origin/main we wszystkich repach
- **Edytujesz kod**: w `thera-rest-service` (dodajesz brakujące testy), w `thera-payment-service` tylko jeśli wykryjesz krytyczny brak w implementacji (zgłaszasz w komentarzu rozdziału — patrz "Zgłaszanie braków")
- **NIE wyłączasz testów ani nie dotykasz infrastruktury Azure** (klaster zostaje jak jest)
- **NIE rozszerzasz zakresu** — trzymasz się dokładnie tego co opisano w `_prompt-startowy.md` dla rozdz. 11 i 12

Czas oszacowany na <2h pracy. Jeśli przekraczasz, **zakończ to co masz, commituj, ZGŁOŚ w ostatnim commicie co nie zostało zrobione**.

---

## Zadanie

Napisz **dwa rozdziały pracy inżynierskiej** zgodnie ze strukturą rozpisaną w `docs/thesis/_prompt-startowy.md`:

1. **Rozdział 11 — Płatności (Stripe)** — podrozdziały 11.1-11.10
2. **Rozdział 12 — Testy funkcjonalne i wydajnościowe** — podrozdziały 12.1-12.9

Pisz analogicznie do gotowych wzorców:
- `docs/thesis/rozdzial-05-keycloak.md` (najbliższy tematycznie do rozdz. 11)
- `docs/thesis/rozdzial-10-separacja-srodowisk.md` (najbliższy stylistycznie po audycie ANS Elbląg)

## Materiały źródłowe

### Rozdział 11 (kod):
- `/Users/desirecutieqb/IdeaProjects/thera-payment-service/`
- Główne pliki: `service/PaymentService.java`, `controller/PaymentController.java`, `config/StripeConfig.java`, `config/SecurityConfig.java`, `kafka/PaymentEventProducer.java`, `model/Payment.java`, `repository/PaymentRepository.java`, `application.yml`, `pom.xml`
- Dokumentacja istniejąca: `/Users/desirecutieqb/IdeaProjects/TheraLink/docs/payment-service.md`

### Rozdział 12 (kod testów):
- `thera-payment-service/src/test/java/` — 7 plików testowych (kompletne)
- `thera-rest-service/src/test/java/` — **TYLKO 1 plik szkieletu, agent dodaje brakujące**
- `thera-ui/src/app/**/*.spec.ts` — 11 plików testowych (średnie pokrycie)

## Kroki pracy

### Faza 1 — research (równolegle, 2 Explore agenty)

Odpal **jednocześnie** (w jednym message z dwoma Agent tool calls):

**Agent A — analiza implementacji Stripe (rozdz. 11):**
- Przeczytaj wszystkie pliki w `thera-payment-service/src/main/java/`
- Zweryfikuj że implementacja zgadza się z opisem w `_prompt-startowy.md` §Rozdział 11
- Zgłoś braki krytyczne (jeśli są): brak walidacji idempotency, brak walidacji HMAC, hardkodowane sekrety, brak obsługi błędów
- Zwróć: lista konkretnych numerów linii do cytowania w listingach 11.1-11.10

**Agent B — inwentaryzacja testów (rozdz. 12):**
- Przeczytaj WSZYSTKIE pliki testowe w 3 repach (thera-payment-service, thera-rest-service, thera-ui)
- Per plik: co testuje (3-4 zdania), jakie biblioteki używa, czas wykonania (jeśli widzi w komentarzach)
- Wykryj braki w thera-rest-service: które kontrolery/serwisy/mappery/repozytoria NIE mają testów
- Zwróć: macierz pokrycia + lista plików do dodania

### Faza 2 — uzupełnienie testów thera-rest-service (rozdz. 12 — zmiany w kodzie)

Na podstawie wyników Agenta B dodaj brakujące testy:
- `*ServiceTest` — jednostkowe Mockito (mockowane repozytoria + Kafka producer)
- `*ControllerTest` — `@WebMvcTest` z mock JWT, weryfikacja statusów HTTP i autoryzacji
- `*RepositoryIntegrationTest` — `@DataMongoTest` + Testcontainers MongoDB
- `*MapperTest` — dla mapperów MapStruct (jeśli mają `@Mapping` z custom logic)

**Konwencje:**
- Pakiety zgodne ze strukturą main: `com.example.therarestservice.service.ClientServiceTest`
- Lombok-friendly (constructor injection — wstrzyknięcie przez `@InjectMocks` lub manualnie w `@BeforeEach`)
- JUnit 5 (`@Test` z `org.junit.jupiter.api`), AssertJ dla asercji, Mockito dla mocków
- Testcontainers: użyj `@Container` static + Spring Boot 3+ `@ServiceConnection`

**Uruchom `mvn test`** w thera-rest-service po dodaniu testów. Jeśli failują — popraw kod testów (NIE kod main). Zapisz liczby (zielone/czerwone/pominięte/czas) — wykorzystasz w tabeli 12.4.

**Jeśli zostaje czas po thera-rest-service** — uzupełnij brakujące testy w thera-ui (np. brakujące state NGXS dla psychologa, brakujące spec dla komponentów dashboardu). Jeśli nie ma czasu — pomiń i zaznacz w rozdziale jako kierunek rozwoju.

### Faza 3 — pisanie rozdziałów

Pisz **równolegle** rozdz. 11 i rozdz. 12, w jednym oknie sesji. Konwencje stylistyczne (jak rozdz. 10 po audycie):

1. **Styl bezosobowy** ("opisano", "wprowadzono", "zaimplementowano"), nigdy "ja zrobiłem"
2. **Polski rejestr akademicki**: zero anglicyzmów bez nawiasów `(ang. …)`, zero potoczyzmów, zero wykrzykników
3. **Numeracja**: tabele 11.1, 11.2…, rysunki 11.1, 11.2… w kolejności występowania w tekście, listingi 11.1, 11.2…
4. **Podpisy tabel** — nad tabelą, format `**Tabela X.Y.** Opis bez kropki końcowej`
5. **Podpisy rysunków** — pod rysunkiem, bez kropki, w nowej linii `źródło: opracowanie własne`
6. **Akronimy** — rozwijać przy pierwszym użyciu: `HMAC (ang. *Hash-based Message Authentication Code*)`, `PCI-DSS (ang. *Payment Card Industry Data Security Standard*)`
7. **Listingi z numeracją linii** zgodną z plikiem źródłowym (jak w rozdz. 4 — wzorzec): `(plik `…/PaymentService.java`, linie 67–127)`
8. **Cytowania `[X]`** — wstawiaj placeholder w miejscach wymagających odsyłacza do literatury/dokumentacji (lista poniżej)
9. **Screeny** — wzorzec rozdz. 10:
   ```
   > 📸 **[SCREEN DO DODANIA]**
   > **Co pokazać:** …
   > **Sugerowany podpis:** Rys. X.Y. …
   > **źródło:** opracowanie własne
   ```
10. **Tylda `~`** tylko w tabelach — w tekście ciągłym "około"

### Faza 4 — uruchom testy ostatecznie, zbieraj metryki

```bash
cd /Users/desirecutieqb/IdeaProjects/thera-payment-service && ./mvnw test 2>&1 | tail -20
cd /Users/desirecutieqb/IdeaProjects/thera-rest-service && ./mvnw test 2>&1 | tail -20
cd /Users/desirecutieqb/IdeaProjects/thera-ui && pnpm test --watch=false 2>&1 | tail -30
```

Wstaw faktyczne liczby do Tabeli 12.4 (rozdz. 12.7). Nie zgaduj.

### Faza 5 — commit + push

**Trzy osobne commity** (po jednym per repo):

1. `thera-rest-service` (jeśli dodałeś testy):
   ```
   test: dodanie testów jednostkowych i integracyjnych

   [opis dodanych plików]
   ```

2. `thera-ui` (tylko jeśli dodałeś testy):
   ```
   test: uzupełnienie pokrycia testami komponentów Angular
   ```

3. `TheraLink` (rozdziały + ewentualne aktualizacje docs):
   ```
   docs(rozdz. 11, 12): napisanie rozdziałów Płatności i Testy

   Rozdział 11 — Płatności (Stripe):
   - 10 podrozdziałów zgodnie z _prompt-startowy.md
   - X listingów z kodu thera-payment-service
   - Y tabel, Z rysunków
   - Cytowania [X] do uzupełnienia przez skill cytowania-bibliografia

   Rozdział 12 — Testy:
   - 9 podrozdziałów
   - Faktyczne metryki z mvn test i pnpm test
   - Stan przed i po (dodano N testów do thera-rest-service)
   - Testy wydajnościowe opisane jako kierunek rozwoju (k6)
   ```

Push każdego repo do origin/main. **Hook post-commit automatycznie wpisze body commit msg jako AI summary w Obsidian vault.**

## Zgłaszanie braków

Jeśli w trakcie pracy wykryjesz krytyczny brak (np. webhook bez walidacji HMAC, hardkodowany klucz, brakujący endpoint), wstaw w odpowiednim podrozdziale blok:

```markdown
> ⚠️ **Wykryto podczas pracy nad rozdziałem:**
> [opis problemu]
> Sugerowana poprawka: [konkretne kroki]
> Status: [zgłoszone do uzupełnienia po powrocie autora / poprawione w tym commicie]
```

Jeśli **agent poprawia** problem — dodaj tę zmianę do odpowiedniego commitu z prefiksem `fix:` zamiast `docs:`.

## Cytowania `[X]` — lista pozycji oczekiwanych

Rozdział 11:
- Stripe API Reference — PaymentIntent
- PCI-DSS standard (oficjalny dokument)
- HMAC-SHA256 specification (RFC 2104)
- Stripe Webhooks signing documentation
- OAuth 2.0 Resource Server (RFC 6749) — odsyłacz wstecz do rozdz. 5
- Spring Security OAuth2 — odsyłacz do rozdz. 5

Rozdział 12:
- Piramida testów (Mike Cohn, "Succeeding with Agile" 2010)
- Testcontainers documentation
- Jest documentation
- JMeter user manual
- k6 documentation
- Pact / Spring Cloud Contract (kierunek rozwoju)

Wszystkie wstawiaj jako `[X]` — Twórca pracy uzupełni przez `skill cytowania-bibliografia`.

## Końcowy raport

Po push'u napisz krótki raport (do 200 słów) w czasie tekstowym, NIE jako commit:

- Co udało się zrobić (rozdz. 11 i 12 — gotowe)
- Ile testów dodanych do thera-rest-service (i co konkretnie)
- Czy mvn test / pnpm test przechodzą zielone na końcu
- Co zgłoszone do uzupełnienia (braki)
- Linki do commitów (`origin/main`)

---

## Wskazówki techniczne

- **Pakiet thera-rest-service**: `com.example.therarestservice` (uwaga, nie `com.theralink.userservice` jak by się intuicyjnie wydawało)
- **mvnw**: używaj `./mvnw test` (wrapper Maven), nie systemowy `mvn`
- **Java w testach**: thera-rest-service używa Java 25, thera-payment-service Java 21 — sprawdź `pom.xml` jeśli kompilacja failuje
- **Testcontainers**: wymaga działającego Docker daemon — jeśli `docker ps` zwraca błąd, zatrzymaj się i wstaw blok ⚠️
- **MapStruct w testach**: testujesz wygenerowany `*MapperImpl`, NIE interface — wstrzykuj `Mappers.getMapper(ClientMapper.class)` w `@BeforeEach`
- **Test pełnego flow Stripe**: pomijasz (wymagałoby działającego konta Stripe + przekierowania webhooków przez `stripe listen`)

---

**Powodzenia. Pracuj systematycznie, jakość ważniejsza od ilości. Jeśli musisz wybrać między pełnym rozdz. 11 a oboma rozdziałami pobieżnie — wybierz pełny 11.**
