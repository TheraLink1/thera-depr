# Rozdział 4 — Migracja warstwy klienckiej (React/Next.js → Angular)

> **Status:** streszczenie wygenerowane przez skill `kod-do-pracy`, gotowe do rozbudowania w Coworku przez skill `praca-inzynierska`.
> **Mapowanie na zakres pracy:** punkt 4.
> **Data wygenerowania:** 2026-06-10
> **Uwagi dla Cowork:** rozdział podzielony na 9 bloków `## [ROZDZIAŁ:...]` (podrozdziały 4.1–4.9). Pole "Porównanie przed/po" jest obowiązkowe (rozdział migracyjny). Listingi kodu mają numerację linii ZGODNĄ z plikami źródłowymi — zachować numerację w pracy (kod: Courier New 10pt). Metryki buildów zmierzono 2026-06-10 na maszynie deweloperskiej. Moduł czatu AI oraz powiadomienia e-mail są poza zakresem pracy — nie rozbudowywać tych wątków. Pełny przepływ OAuth 2.0/PKCE opisano w rozdziale 2 (architektura bezpieczeństwa), konfigurację serwera Keycloak opisuje rozdział 5 — w 4.7 tylko strona kliencka.

---

## [ROZDZIAŁ: Migracja frontendu — filozofia frameworka: biblioteka a framework opinionated]

**Temat:** Porównanie modelu architektonicznego React (biblioteka komponentów wymagająca samodzielnego doboru ekosystemu) z modelem Angular (kompletny framework narzucający strukturę), jako fundament decyzji migracyjnej rozwiniętej w kolejnych podrozdziałach.

**Technologie:** React 19 + Next.js 15.3.2 (monolit), Angular 21.2.0 (architektura docelowa), wstrzykiwanie zależności (ang. *Dependency Injection*, DI), RxJS 7.8, Angular CLI.

**Co zostało zaimplementowane:** Przeanalizowano konsekwencje dwóch przeciwstawnych filozofii budowy warstwy klienckiej. React jest biblioteką odpowiadającą wyłącznie za renderowanie komponentów — decyzje o routerze, zarządzaniu stanem, kliencie HTTP, formularzach i strukturze projektu pozostają po stronie zespołu. W analizowanym monolicie skutkowało to współistnieniem trzech systemów komponentów UI, bibliotek zainstalowanych i nieużywanych (React Hook Form, Zod, Mapbox GL — opisane w rozdziale 1) oraz brakiem jednolitych konwencji między plikami. Angular natomiast jest frameworkiem typu *opinionated* („batteries included"): router, klient HTTP z interceptorami, formularze reaktywne, mechanizm DI oraz narzędzie wiersza poleceń Angular CLI (generatory kodu, budowanie, testy) stanowią integralną część platformy o wspólnym cyklu wydawniczym. Wbudowany kontener DI zarządza cyklem życia serwisów (`@Injectable({ providedIn: 'root' })`), a biblioteka RxJS pełni rolę standardowej warstwy programowania asynchronicznego — każde żądanie `HttpClient` zwraca strumień `Observable`, co ujednolica obsługę operacji sieciowych w całej aplikacji. Nowa aplikacja `thera-ui` wykorzystuje wyłącznie oficjalne pakiety `@angular/*` uzupełnione o trzy świadomie wybrane biblioteki zewnętrzne: NGXS (stan globalny), Transloco (internacjonalizacja) i keycloak-angular (integracja z serwerem tożsamości).

**Kluczowe decyzje techniczne:** Decyzję o zmianie frameworka uzasadniono w rozdziale 1 (podrozdział o wyborze technologii migracji) — niniejszy rozdział dokumentuje jej realizację. Z perspektywy architektury mikroserwisowej kluczowy okazał się mechanizm DI: serwisy domenowe (`PsychologistService`, `AppointmentService`, `AvailabilityService`) odwzorowują podział na mikroserwisy backendu, a ich wymiana lub mockowanie w testach nie wymaga zmian w komponentach. Koszt przyjętej filozofii to wyższy próg wejścia (TypeScript w trybie *strict*, RxJS, dekoratory) oraz mniejsza elastyczność doboru bibliotek — zaakceptowany świadomie w zamian za wymuszoną spójność kodu rozwijanego w pojedynkę.

**Pliki/komponenty:** `thera-ui/package.json` (18 zależności produkcyjnych), `thera-ui/src/app/core/` (serwisy, interceptory, guard — wzorce frameworkowe opisane w 4.4 i 4.7).

**Porównanie przed/po:** W monolicie swoboda doboru bibliotek doprowadziła do 44 zależności (33 produkcyjne + 11 deweloperskich), w tym trzech równoległych systemów UI i pakietów martwych (zainstalowanych, nieużywanych). W aplikacji Angular liczba zależności spadła do 25 (18 + 7), a każda pełni jednoznaczną rolę — framework eliminuje kategorię decyzji „którą bibliotekę wybrać", przenosząc ją na poziom „czy w ogóle potrzebna jest biblioteka zewnętrzna".

---

## [ROZDZIAŁ: Migracja frontendu — struktura projektu i routing]

**Temat:** Migracja z trasowania opartego na systemie plików (Next.js App Router, komponenty serwerowe i klienckie) na jawną konfigurację tras Angular z komponentami *standalone* i leniwym ładowaniem (ang. *lazy loading*) obszarów funkcjonalnych.

**Technologie:** Next.js App Router, React Server Components, Angular Router (`loadChildren`, trasy zagnieżdżone), komponenty standalone (bez NgModules), esbuild (podział kodu na fragmenty, ang. *code splitting*).

**Co zostało zaimplementowane:** W monolicie trasy wyznaczała struktura katalogu `app/` (`signin/`, `signup/`, `view/`, `profile/`, `confirm-booking/`, `dashboard/client/`, `dashboard/psychologist/`, grupa `(auth)/`) — konwencja wygodna, lecz rozmywająca granicę między routingiem a strukturą kodu; dodatkowo każdy plik wymagał decyzji o trybie renderowania (komponent serwerowy lub kliencki — dyrektywa `"use client"` występowała w 14 plikach). Nowa aplikacja definiuje routing jawnie w pliku `app.routes.ts` (Listing 4.1): pięć obszarów funkcjonalnych ładowanych leniwie przez `loadChildren`, każdy z własnym plikiem tras, a wewnątrz dashboardów trasy zagnieżdżone (np. `dashboard/client/account-settings`). Strukturę katalogów `src/app/` oparto na podziale odpowiedzialności: `core/` (auth, serwisy HTTP, interceptory, i18n — kod infrastrukturalny ładowany raz), `features/` (home, psychologists, booking — obszary domenowe), `dashboards/` (client, psychologist — widoki zależne od roli), `shared/` (navbar, footer, modele). Wszystkie komponenty są standalone — projekt nie zawiera ani jednego NgModule, a zależności komponentu deklarowane są w jego tablicy `imports`.

**Listing 4.1.** Jawna definicja tras z leniwym ładowaniem obszarów funkcjonalnych (plik `thera-ui/src/app/app.routes.ts`, linie 3–25)

```
 3  export const routes: Routes = [
 4    {
 5      path: '',
 6      loadChildren: () => import('./features/home/home.routes').then(m => m.HOME_ROUTES)
 7    },
 8    {
 9      path: 'psychologists',
10      loadChildren: () => import('./features/psychologists/psychologists.routes').then(m => m.PSYCHOLOGISTS_ROUTES)
11    },
12    {
13      path: 'booking',
14      loadChildren: () => import('./features/booking/booking.routes').then(m => m.BOOKING_ROUTES)
15    },
16    {
17      path: 'dashboard/client',
18      loadChildren: () => import('./dashboards/client/dashboard-client.routes').then(m => m.CLIENT_DASHBOARD_ROUTES)
19    },
20    {
21      path: 'dashboard/psychologist',
22      loadChildren: () => import('./dashboards/psychologist/dashboard-psychologist.routes').then(m => m.PSYCHOLOGIST_DASHBOARD_ROUTES)
23    },
24    { path: '**', redirectTo: '' }
25  ];
```

**Kluczowe decyzje techniczne:** Leniwe ładowanie zastosowano dla każdego obszaru funkcjonalnego, ponieważ współgra ono z architekturą mikroserwisową systemu: granice fragmentów kodu frontendu pokrywają się z granicami serwisów backendowych (obszar `psychologists` komunikuje się z serwisem psychologów, `booking` — z serwisem wizyt). Kompilator dzieli aplikację na fragmenty inicjalne i leniwe — pomiar buildu produkcyjnego wykazał, że fragment `psychologists-routes` (111,67 kB raw / 22,11 kB po kompresji) pobierany jest dopiero przy wejściu na listę psychologów, a fragment inicjalny aplikacji to 467,88 kB raw / 119,87 kB transferu. Rezygnacja z renderowania serwerowego (Next.js SSR) była świadoma: aplikacja po zalogowaniu operuje wyłącznie na danych prywatnych użytkownika, gdzie SSR nie przynosi korzyści SEO, a komplikuje wdrożenie (osobny proces Node.js zamiast statycznych plików serwowanych przez serwer HTTP w kontenerze).

**Pliki/komponenty:**
- `thera-ui/src/app/app.routes.ts` — trasy główne (Listing 4.1)
- `thera-ui/src/app/features/*/[nazwa].routes.ts`, `thera-ui/src/app/dashboards/*/dashboard-*.routes.ts` — trasy obszarów z guardami i scope'ami tłumaczeń
- `frontend/app/` — katalog tras monolitu (struktura w sugerowanym zrzucie ekranu)

**Porównanie przed/po:**

| Wymiar | Monolit Next.js | Aplikacja Angular |
|---|---|---|
| **Definicja trasy** | Niejawna — katalog w `app/` z plikiem `page.tsx` | Jawna — wpis w `app.routes.ts` z `loadChildren` |
| **Struktura katalogów** | Pochodna adresów URL (`app/view/`, `app/confirm-booking/`) | Pochodna odpowiedzialności (`core/`, `features/`, `dashboards/`, `shared/`) |
| **Podział kodu** | Automatyczny per strona (Next.js) | Świadomy per obszar funkcjonalny — 5 fragmentów leniwych zbieżnych z granicami mikroserwisów |
| **Tryby renderowania** | Mieszane — komponenty serwerowe + 14 plików `"use client"` | Jednolity — SPA renderowana w przeglądarce |
| **Moduły** | Nie dotyczy | Brak NgModules — 100% komponentów standalone |

---

## [ROZDZIAŁ: Migracja frontendu — zarządzanie stanem aplikacji]

**Temat:** Zastąpienie Redux Toolkit + RTK Query dwupoziomowym modelem stanu: NGXS dla stanu współdzielonego między widokami oraz Angular Signals dla stanu lokalnego komponentów.

**Technologie:** Redux Toolkit 2.6 + RTK Query (monolit), NGXS 21 (`@ngxs/store`, wtyczki logger i router), Angular Signals (`signal`, `computed`, `effect`, `linkedSignal`), `@angular/core/rxjs-interop` (`toSignal`, `toObservable`).

**Co zostało zaimplementowane:** W monolicie stan globalny tworzyły pojedynczy `globalSlice` (filtry wyszukiwania, preferencje UI) oraz 14 endpointów RTK Query z deklaratywnym cachowaniem opartym na tagach (opisane w rozdziale 1). W nowej aplikacji przyjęto jawny podział na dwa poziomy. **Poziom globalny** realizują trzy stany NGXS rejestrowane w `app.config.ts`: `AuthState` (tożsamość i role zalogowanego użytkownika), `PsychologistsState` (lista psychologów, stan ładowania, wybrany rekord) oraz `AppointmentsState` (wizyty klienta i psychologa). Każdy stan definiuje klasy akcji, model, selektory statyczne `@Selector()` oraz metody `@Action()` wykonujące efekty uboczne — np. `LoadPsychologists` deleguje do serwisu HTTP i aktualizuje stan przez `ctx.patchState()` (Listingi 4.2 i 4.3). **Poziom lokalny** realizują sygnały: pola formularzy i filtrów to `signal()`/`linkedSignal()`, wartości pochodne to `computed()` (przeliczane wyłącznie przy zmianie zależności), a mosty do świata RxJS i routera zapewniają `toSignal()`/`toObservable()`. Komponent `BrowseComponent` (Listing 4.4) łączy oba poziomy: parametry zapytania URL trafiają przez `toSignal` do sygnałów `keyword`/`location` (`linkedSignal` pozwala nadpisywać wartość lokalnie po zmianie źródła), lista psychologów czytana jest ze stanu NGXS przez `store.selectSignal()`, filtrowanie realizuje `computed`, a `effect()` w konstruktorze automatycznie wysyła akcję `LoadPsychologists` przy każdej zmianie filtrów. Wszystkie komponenty pracują w strategii detekcji zmian `OnPush`.

**Listing 4.2.** Klasy akcji stanu psychologów (plik `thera-ui/src/app/features/psychologists/state/psychologists.state.ts`, linie 7–15)

```
 7  export class LoadPsychologists {
 8    static readonly type = '[Psychologists] Load';
 9    constructor(public keyword?: string, public location?: string) {}
10  }
11
12  export class SelectPsychologist {
13    static readonly type = '[Psychologists] Select';
14    constructor(public id: string | null) {}
15  }
```

**Listing 4.3.** Obsługa akcji — efekt uboczny HTTP i aktualizacja stanu (plik `thera-ui/src/app/features/psychologists/state/psychologists.state.ts`, linie 47–61)

```
47  @Action(LoadPsychologists)
48  load(ctx: StateContext<PsychologistsStateModel>, action: LoadPsychologists) {
49    ctx.patchState({ loading: true, error: null });
50    return this.service.getAll(action.keyword, action.location).pipe(
51      tap({
52        next: (items) => ctx.patchState({ items, loading: false }),
53        error: (err) => ctx.patchState({ loading: false, error: err.message }),
54      })
55    );
56  }
57
58  @Action(SelectPsychologist)
59  select(ctx: StateContext<PsychologistsStateModel>, action: SelectPsychologist) {
60    ctx.patchState({ selectedId: action.id });
61  }
```

**Listing 4.4.** Połączenie sygnałów lokalnych ze stanem NGXS w komponencie listy psychologów (plik `thera-ui/src/app/features/psychologists/browse/browse.component.ts`, linie 33–60)

```
33  private route  = inject(ActivatedRoute);
34  private router = inject(Router);
35  private store  = inject(Store);
36
37  private queryParams = toSignal(this.route.queryParams, { initialValue: {} as Params });
38
39  keyword  = linkedSignal<string>(() => this.queryParams()['keyword'] ?? '');
40  location = linkedSignal<string>(() => this.queryParams()['location'] ?? '');
41
42  all      = this.store.selectSignal(PsychologistsState.items);
43  loading  = this.store.selectSignal(PsychologistsState.loading);
44  selected = this.store.selectSignal(PsychologistsState.selected);
45
46  filtered = computed(() => {
47    const kw  = this.keyword().toLowerCase();
48    const loc = this.location().toLowerCase();
49    return this.all().filter((p: Psychologist) => {
50      const matchKw  = !kw  || (p.Specialization || '').toLowerCase().includes(kw)  || (p.name || '').toLowerCase().includes(kw);
51      const matchLoc = !loc || (p.location || '').toLowerCase().includes(loc);
52      return matchKw && matchLoc;
53    });
54  });
55
56  constructor() {
57    effect(() => {
58      this.store.dispatch(new LoadPsychologists(this.keyword(), this.location()));
59    });
60  }
```

**Kluczowe decyzje techniczne:** Przyjęto regułę: NGXS wyłącznie dla stanu współdzielonego przez co najmniej dwa niezależne widoki (tożsamość użytkownika, listy domenowe), Signals dla wszystkiego, co lokalne (filtry, zaznaczenia, stany formularzy). Reguła zapobiega dwóm antywzorcom: „wszystko w store" (nadmiarowy kod akcji dla ulotnego stanu UI) oraz „wszystko w komponencie" (duplikacja pobrań i rozjazd danych między widokami). Wybór NGXS zamiast NgRx podyktowany był zwięzłością — definicja stanu, akcji i selektorów w jednym pliku klasowym zamiast rozproszenia na akcje/reducery/efekty/selektory. Metoda `store.selectSignal()` integruje oba światy bez ręcznych subskrypcji, co w połączeniu z `OnPush` ogranicza detekcję zmian do komponentów faktycznie dotkniętych zmianą.

**Pliki/komponenty:**
- `thera-ui/src/app/core/auth/state/auth.state.ts`, `features/psychologists/state/psychologists.state.ts`, `dashboards/client/state/appointments.state.ts` — trzy stany NGXS
- `thera-ui/src/app/features/psychologists/browse/browse.component.ts` — wzorcowe połączenie obu poziomów (Listing 4.4)
- `frontend/state/index.ts` (`globalSlice`), `frontend/state/api.ts` (RTK Query) — stan monolitu

**Porównanie przed/po:**

| Wymiar | Monolit (Redux Toolkit + RTK Query) | Angular (NGXS + Signals) |
|---|---|---|
| **Stan globalny** | 1 slice + cache RTK Query (14 endpointów) | 3 stany NGXS z jawnymi akcjami i selektorami |
| **Stan lokalny** | `useState` w komponentach, częściowo dublowany w slice | Sygnały: `signal()`, `linkedSignal()` per komponent |
| **Wartości pochodne** | Przeliczane przy każdym renderze lub memoizowane ręcznie (`useMemo`) | `computed()` — przeliczane wyłącznie przy zmianie zależności |
| **Unieważnianie danych** | Tagi RTK Query (`invalidatesTags`) | Akcje NGXS wywoływane jawnie (np. ponowny `LoadPsychologists`) |
| **Reakcja na zmiany filtrów** | `useEffect` z tablicą zależności | `effect()` śledzący sygnały automatycznie |
| **Narzędzia diagnostyczne** | Redux DevTools | NGXS Logger Plugin (akcje i migawki stanu w konsoli) |
| **Detekcja zmian** | Re-render drzewa od korzenia stanu | `OnPush` + sygnały — aktualizacja tylko dotkniętych komponentów |

---

## [ROZDZIAŁ: Migracja frontendu — komunikacja z API i interceptory HTTP]

**Temat:** Centralizacja przekrojowych aspektów komunikacji HTTP (dołączanie tokenu, odświeżanie sesji, obsługa błędów autoryzacji) w interceptorach Angular, w miejsce logiki rozproszonej po definicjach zapytań monolitu.

**Technologie:** RTK Query `fetchBaseQuery` + AWS Amplify `fetchAuthSession()` (monolit), Angular `HttpClient`, interceptory funkcyjne (`HttpInterceptorFn`, `withInterceptors`), keycloak-js (`updateToken`), RxJS.

**Co zostało zaimplementowane:** W monolicie token tożsamości dołączany był w funkcji `prepareHeaders` konfiguracji RTK Query (Listing 4.5) — rozwiązanie scentralizowane tylko częściowo: każde żądanie wywoływało asynchronicznie `fetchAuthSession()`, obsługa błędów HTTP pozostawała w gestii poszczególnych komponentów, a przepływ rejestracji wykonywał dodatkowo ręczne wywołania `fetch` z własnym nagłówkiem `Authorization` (`authProvider.tsx:155-161`). W nowej aplikacji komunikację HTTP obsługuje wyłącznie `HttpClient`, a aspekty przekrojowe realizują dwa interceptory funkcyjne zarejestrowane w `app.config.ts` (`provideHttpClient(withInterceptors([jwtInterceptor, errorInterceptor]))`). Interceptor `jwtInterceptor` (Listing 4.6) przed każdym żądaniem wywołuje `keycloak.updateToken(5)` — proaktywne odświeżenie tokenu, jeżeli wygasa w ciągu 5 sekund — po czym klonuje żądanie z nagłówkiem `Authorization: Bearer`. Interceptor `errorInterceptor` (Listing 4.7) przechwytuje odpowiedzi 401/403 i przekierowuje na stronę główną, propagując pozostałe błędy do warstwy wywołującej. Serwisy domenowe (`core/services/`) zawierają wyłącznie czyste wywołania `HttpClient` na adresach budowanych z `environment.apiGatewayUrl` — żaden komponent ani serwis nie zarządza tokenem samodzielnie.

**Listing 4.5.** Stan przed migracją — dołączanie tokenu w konfiguracji RTK Query (plik `frontend/state/api.ts`, linie 12–27)

```
12  export const api = createApi({
13    baseQuery: fetchBaseQuery({
14      baseUrl: process.env.NEXT_PUBLIC_API_BASE_URL,
15      responseHandler: (response) =>
16        response.headers.get("content-type")?.includes("application/json")
17          ? response.json()
18          : response.text(),
19      prepareHeaders: async (headers) => {
20        const session = await fetchAuthSession();
21        const { idToken } = session.tokens ?? {};
22        if (idToken) {
23          headers.set("Authorization", `Bearer ${idToken}`);
24        }
25        return headers;
26      },
27    }),
```

**Listing 4.6.** Stan po migracji — interceptor dołączający token z proaktywnym odświeżeniem (plik `thera-ui/src/app/core/interceptors/jwt.interceptor.ts`, linie 6–19)

```
 6  export const jwtInterceptor: HttpInterceptorFn = (req: HttpRequest<unknown>, next: HttpHandlerFn) => {
 7    const keycloak = inject(Keycloak);
 8
 9    return from(keycloak.updateToken(5).catch(() => false)).pipe(
10      switchMap(() => {
11        const token = keycloak.token;
12        if (token) {
13          const cloned = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
14          return next(cloned);
15        }
16        return next(req);
17      })
18    );
19  };
```

**Listing 4.7.** Centralna obsługa błędów autoryzacji (plik `thera-ui/src/app/core/interceptors/error.interceptor.ts`, linie 6–17)

```
 6  export const errorInterceptor: HttpInterceptorFn = (req: HttpRequest<unknown>, next: HttpHandlerFn) => {
 7    const router = inject(Router);
 8
 9    return next(req).pipe(
10      catchError((error) => {
11        if (error.status === 401 || error.status === 403) {
12          router.navigate(['/']);
13        }
14        return throwError(() => error);
15      })
16    );
17  };
```

**Kluczowe decyzje techniczne:** Zastosowano interceptory funkcyjne (`HttpInterceptorFn`) zamiast klasowych — są zwięźlejsze, korzystają z `inject()` i stanowią bieżącą rekomendację frameworka. Rozdzielenie odpowiedzialności na dwa interceptory (token osobno, błędy osobno) upraszcza testowanie jednostkowe — każdy aspekt weryfikowany jest niezależnie (testy opisane w 4.9). Wywołanie `updateToken(5)` eliminuje klasę błędów „token wygasł między pobraniem a użyciem", która w monolicie objawiała się sporadycznymi odpowiedziami 401 wymagającymi ręcznego odświeżenia strony.

**Pliki/komponenty:**
- `thera-ui/src/app/core/interceptors/jwt.interceptor.ts`, `error.interceptor.ts` — interceptory (Listingi 4.6, 4.7)
- `thera-ui/src/app/core/services/appointment.service.ts`, `psychologist.service.ts`, `availability.service.ts` — serwisy domenowe na `HttpClient`
- `frontend/state/api.ts` — konfiguracja RTK Query monolitu (Listing 4.5)

**Porównanie przed/po:**

| Wymiar | Monolit | Aplikacja Angular |
|---|---|---|
| **Dołączanie tokenu** | `prepareHeaders` w RTK Query + ręczne `fetch` w przepływie rejestracji | Jeden interceptor dla wszystkich żądań `HttpClient` |
| **Odświeżanie tokenu** | Niejawne, wewnątrz SDK Amplify, bez kontroli aplikacji | Jawne `updateToken(5)` przed każdym żądaniem |
| **Obsługa 401/403** | Brak centralnej — reakcja zależna od komponentu | `errorInterceptor` — jednolite przekierowanie |
| **Adres bazowy API** | `process.env.NEXT_PUBLIC_API_BASE_URL` | `environment.apiGatewayUrl` (brama Spring Cloud Gateway) |
| **Liczba miejsc z logiką auth w HTTP** | 2 (api.ts + authProvider.tsx) | 1 (jwt.interceptor.ts) |

---

## [ROZDZIAŁ: Migracja frontendu — konsolidacja systemu komponentów UI]

**Temat:** Zastąpienie trzech równoległych systemów komponentów monolitu jednym spójnym systemem: Angular Material z centralnym motywem oraz własnymi komponentami domenowymi.

**Technologie:** Material UI 7.1 + Radix UI + shadcn/ui + Tailwind CSS 4 (monolit), Angular Material 21.2 + Angular CDK, SCSS z API motywów `mat.define-theme()`, sygnałowe API komponentów (`input.required()`, `output()`).

**Co zostało zaimplementowane:** Monolit wykorzystywał równolegle trzy systemy komponentów (Material UI dla głównych ekranów, Radix UI dla prymitywów dostępności, shadcn/ui dla widgetów czatu) oraz Tailwind CSS, bez wspólnego systemu *design tokens* — kolory marki hardkodowane były w komponentach (`#2b6369`, `#00bfa5`), co udokumentowano w rozdziale 1 jako jedną z wad jakościowych. W nowej aplikacji przyjęto jeden system bazowy: Angular Material z motywem zdefiniowanym centralnie w `src/styles.scss` przez `mat.define-theme()` — paleta barw marki (primary `#2b6369`, accent `#00bfa5`) zadeklarowana jest w jednym miejscu i propagowana do wszystkich komponentów. Ponad warstwą bazową zbudowano komponenty domenowe odwzorowujące pojęcia biznesowe platformy: `PsychologistCardComponent` (karta psychologa na liście wyników, Listing 4.8), `DetailsPanelComponent` (panel szczegółów z osadzonym wyborem terminu) oraz `AppointmentSlotPickerComponent` (wybór terminu wizyty na podstawie faktycznych dostępności psychologa — Listing 4.9). Komponenty domenowe komunikują się przez sygnałowe API: `input.required<T>()` wymusza przekazanie danych na poziomie kompilatora, `output<T>()` emituje zdarzenia domenowe. Picker grupuje dostępności po dniach, prezentuje wybór dnia na kalendarzu Material (`MatDatepicker` z filtrem `isDateAvailable`) i emituje zdarzenie `slotSelected` z wybraną datą i godziną — eliminuje to wcześniejsze ręczne przekazywanie dowolnej daty w parametrach URL. Pływający widget czatu AI z monolitu (16 komponentów shadcn/ui) świadomie pominięto w migracji — funkcjonalność czatu pozostaje poza zakresem pracy.

**Listing 4.8.** Sygnałowe API komponentu domenowego karty psychologa (plik `thera-ui/src/app/features/psychologists/browse/psychologist-card/psychologist-card.component.ts`, linie 13–20)

```
13  export class PsychologistCardComponent {
14    psychologist = input.required<Psychologist>();
15    selected = output<Psychologist>();
16
17    select() {
18      this.selected.emit(this.psychologist());
19    }
20  }
```

**Listing 4.9.** Logika wyboru terminu w komponencie `AppointmentSlotPicker` (plik `thera-ui/src/app/features/booking/appointment-slot-picker/appointment-slot-picker.component.ts`, linie 71–94)

```
71  selectedDate = linkedSignal<Date | null>(() =>
72    this.availableDates().length > 0 ? new Date(this.availableDates()[0]) : null
73  );
74  selectedTime = signal<string | null>(null);
75
76  formattedDate = computed(() =>
77    this.selectedDate() ? format(this.selectedDate()!, 'yyyy-MM-dd') : ''
78  );
79
80  availableTimes = computed(() =>
81    this.selectedDate() ? (this.availability()[this.formattedDate()] ?? []) : []
82  );
83
84  isDateAvailable = (d: Date) => this.availableDates().includes(format(d, 'yyyy-MM-dd'));
85
86  onDateChange(d: Date | null) {
87    this.selectedDate.set(d);
88    this.selectedTime.set(null);
89  }
90
91  selectTime(t: string) {
92    this.selectedTime.set(t);
93    this.slotSelected.emit({ date: this.formattedDate(), startHour: t });
94  }
```

**Kluczowe decyzje techniczne:** Konsolidacja na Angular Material zamiast przenoszenia trzech systemów wynikała wprost z analizy rozdziału 1 — duplikacja zależności UI i brak design tokens zostały sklasyfikowane jako dług techniczny. Komponenty domenowe zaprojektowano jako „głupie" (prezentacyjne): nie znają routera ani store'u, komunikują się wyłącznie przez `input`/`output`, dzięki czemu są wielokrotnego użytku (picker osadzony w panelu szczegółów może zostać użyty również w kalendarzu psychologa) i testowalne w izolacji. Wybór terminu oparty na dostępnościach (zamiast dowolnej daty wpisywanej ręcznie) przeniósł walidację reguły biznesowej „wizyta tylko w wolnym terminie" z backendu również do warstwy UX.

**Pliki/komponenty:**
- `thera-ui/src/styles.scss` — centralny motyw `mat.define-theme()` z paletą marki
- `thera-ui/src/app/features/psychologists/browse/psychologist-card/`, `details-panel/` — komponenty domenowe listy
- `thera-ui/src/app/features/booking/appointment-slot-picker/` — komponent wyboru terminu (Listing 4.9)
- `frontend/package.json` — trzy systemy UI monolitu (Material UI, Radix, shadcn/ui + Tailwind)

**Porównanie przed/po:**

| Wymiar | Monolit | Aplikacja Angular |
|---|---|---|
| **Systemy komponentów** | 3 (Material UI, Radix UI, shadcn/ui) + Tailwind | 1 (Angular Material + CDK) |
| **Pakiety UI w zależnościach** | ~14 (mui, emotion ×2, radix ×2, tailwind ×3, lucide, cva, clsx, framer-motion, sonner, amplify-ui) | 2 (`@angular/material`, `@angular/cdk`) |
| **Design tokens** | Brak — kolory hardkodowane w komponentach | Centralny motyw SCSS (`mat.define-theme`) |
| **Komponenty domenowe** | Brak — strony budowane ad hoc | `PsychologistCard`, `DetailsPanel`, `AppointmentSlotPicker` |
| **Wybór terminu wizyty** | Dowolna data/godzina w parametrach URL | Wyłącznie wolne sloty z dostępności psychologa |

---

## [ROZDZIAŁ: Migracja frontendu — internacjonalizacja]

**Temat:** Wprowadzenie pełnej internacjonalizacji (ang. *internationalization*, i18n) z językiem polskim jako domyślnym i angielskim jako dodatkowym, z tłumaczeniami dzielonymi na zakresy (ang. *scopes*) ładowane wraz z leniwymi trasami.

**Technologie:** Transloco 8.2 (`@jsverse/transloco`), `provideTranslocoScope`, ładowanie tłumaczeń przez HTTP (`TranslocoHttpLoader`).

**Co zostało zaimplementowane:** Monolit nie posiadał żadnego mechanizmu i18n — teksty interfejsu były hardkodowane, w dodatku niespójnie (formularze logowania po polsku, strona główna częściowo po angielsku). W nowej aplikacji zastosowano bibliotekę Transloco skonfigurowaną w `app.config.ts` (Listing 4.10): języki `pl` (domyślny) i `en`, ponowne renderowanie po zmianie języka (`reRenderOnLangChange`). Tłumaczenia podzielono na zakresy odpowiadające obszarom funkcjonalnym: klucze wspólne (`nav`, `footer`) pozostają w plikach głównych `src/assets/i18n/{pl,en}.json`, natomiast każdy obszar posiada własny katalog (`assets/i18n/booking/pl.json` itd.) deklarowany w pliku tras przez `provideTranslocoScope` (Listing 4.11) — plik tłumaczeń pobierany jest dopiero przy wejściu na trasę, równolegle z leniwym fragmentem kodu. Loader (`core/i18n/transloco-loader.ts`) pobiera pliki przez `HttpClient` ze ścieżki `/assets/i18n/${lang}.json`, przy czym dla zakresów parametr `lang` przyjmuje postać `booking/pl` — jeden loader obsługuje pliki główne i zakresy. W szablonach tłumaczenia realizuje potok `| transloco` (np. `{{ 'nav.signIn' | transloco }}` w pasku nawigacji), a w kodzie TypeScript — `TranslocoService.translate()` (komunikaty snackbara w `confirm-booking.component.ts:78,82`); w plikach źródłowych nie pozostał żaden hardkodowany tekst interfejsu.

**Listing 4.10.** Konfiguracja Transloco (plik `thera-ui/src/app/app.config.ts`, linie 38–46)

```
38  provideTransloco({
39    config: {
40      availableLangs: ['pl', 'en'],
41      defaultLang: 'pl',
42      reRenderOnLangChange: true,
43      prodMode: environment.production,
44    },
45    loader: TranslocoHttpLoader,
46  }),
```

**Listing 4.11.** Zakres tłumaczeń ładowany wraz z leniwą trasą rezerwacji (plik `thera-ui/src/app/features/booking/booking.routes.ts`, linie 6–8)

```
 6  export const BOOKING_ROUTES: Routes = [
 7    { path: 'confirm', component: ConfirmBookingComponent, canActivate: [authGuard], providers: [provideTranslocoScope('booking')] }
 8  ];
```

**Kluczowe decyzje techniczne:** Wybrano Transloco (tłumaczenia rozwiązywane w czasie działania) zamiast wbudowanego mechanizmu `$localize` (tłumaczenia wkompilowane w build), ponieważ `$localize` wymaga osobnego artefaktu budowania dla każdego języka — przy wdrożeniu kontenerowym oznaczałoby to dwa obrazy Docker lub podwójny rozmiar obrazu. Transloco pozwala zmienić język bez przeładowania aplikacji i utrzymuje jeden artefakt produkcyjny. Podział na zakresy przyjęto z tego samego powodu co lazy loading kodu — użytkownik pobiera wyłącznie tłumaczenia ekranów, które faktycznie odwiedza. Aplikacja kierowana jest do polskiego użytkownika (język domyślny `pl`), lecz architektura i18n przygotowuje rozszerzenie na kolejne języki bez zmian w kodzie — wyłącznie przez dodanie plików JSON.

**Pliki/komponenty:**
- `thera-ui/src/assets/i18n/` — pliki główne + 5 katalogów zakresów (`home`, `browse`, `booking`, `clientDashboard`, `psychologistDashboard`)
- `thera-ui/src/app/core/i18n/transloco-loader.ts` — loader HTTP
- `thera-ui/src/app/app.config.ts` (Listing 4.10), pliki `*.routes.ts` z `provideTranslocoScope` (Listing 4.11)

**Porównanie przed/po:**

| Wymiar | Monolit | Aplikacja Angular |
|---|---|---|
| **Mechanizm i18n** | Brak | Transloco 8.2 |
| **Języki** | Mieszanka pl/en hardkodowana w komponentach | `pl` (domyślny) + `en`, komplet kluczy w obu |
| **Teksty w kodzie źródłowym** | Rozproszone po 14+ plikach | 0 — wyłącznie klucze tłumaczeń |
| **Ładowanie tłumaczeń** | Nie dotyczy | Pliki główne przy starcie + zakresy przy wejściu na trasę |

---

## [ROZDZIAŁ: Migracja frontendu — integracja z Keycloak po stronie klienckiej]

**Temat:** Zastąpienie klienckiej integracji AWS Amplify/Cognito biblioteką keycloak-angular: ciche logowanie jednokrotne (ang. *Single Sign-On*, SSO), guard tras z kontrolą ról oraz pełne odseparowanie aplikacji od dostawcy tożsamości.

**Technologie:** AWS Amplify 6.14 + Amplify Authenticator (monolit), keycloak-angular 21 + keycloak-js 26.2, mechanizm `check-sso` z ramką `silent-check-sso.html`, funkcyjny `CanActivateFn`.

**Co zostało zaimplementowane:** W monolicie uwierzytelnienie realizował komponent `Authenticator` z AWS Amplify — formularze logowania renderowane wewnątrz aplikacji, konfiguracja Cognito wkompilowana w kod (Listing 4.12), rola użytkownika przechowywana w niestandardowym atrybucie `custom:role`, a „ochrona" tras sprowadzała się do warunkowego sprawdzania `pathname` w komponencie (rozdział 1 wykazał brak faktycznych guardów). W nowej aplikacji uwierzytelnienie w całości delegowano do serwera Keycloak. Funkcja `provideKeycloak` w `app.config.ts` (Listing 4.13) inicjalizuje klienta OIDC w trybie `check-sso`: przy starcie aplikacja sprawdza w ukrytej ramce (`silentCheckSsoRedirectUri`) istnienie aktywnej sesji SSO — użytkownik zalogowany w innej karcie zostaje rozpoznany bez interakcji, a niezalogowany pozostaje na stronie publicznej bez wymuszania logowania. Trasy chronione deklarują funkcyjny `authGuard` (Listing 4.14): niezalogowanego użytkownika guard przekierowuje na ekran logowania Keycloak z powrotem na żądany adres (`redirectUri`), a zalogowanemu weryfikuje role realm'owe (`keycloak.realmAccess.roles`) względem wymagań trasy zadeklarowanych w `data.roles` — np. trasy `dashboard/client` wymagają roli `client`. Po zalogowaniu stan tożsamości (identyfikator, nazwa, role z tokenu) trafia do `AuthState` (NGXS) akcją `Login`, skąd czytają go navbar i dashboardy. Formularze logowania i rejestracji nie istnieją w kodzie aplikacji — renderuje je Keycloak z customowym motywem `theralink` (konfiguracja serwera w rozdziale 5, pełny przepływ Authorization Code + PKCE w rozdziale 2).

**Listing 4.12.** Stan przed migracją — konfiguracja Cognito wkompilowana w aplikację (plik `frontend/app/(auth)/authProvider.tsx`, linie 17–25)

```
17  Amplify.configure({
18    Auth: {
19      Cognito: {
20        userPoolId: process.env.NEXT_PUBLIC_AWS_COGNITO_USER_POOL_ID!,
21        userPoolClientId:
22          process.env.NEXT_PUBLIC_AWS_COGNITO_USER_POOL_CLIENT_ID!,
23      },
24    },
25  });
```

**Listing 4.13.** Stan po migracji — inicjalizacja klienta Keycloak z cichym SSO (plik `thera-ui/src/app/app.config.ts`, linie 27–37)

```
27  provideKeycloak({
28    config: {
29      url: environment.keycloak.url,
30      realm: environment.keycloak.realm,
31      clientId: environment.keycloak.clientId,
32    },
33    initOptions: {
34      onLoad: 'check-sso',
35      silentCheckSsoRedirectUri: window.location.origin + '/assets/silent-check-sso.html',
36    },
37  }),
```

**Listing 4.14.** Guard tras z kontrolą ról realm'owych (plik `thera-ui/src/app/core/auth/guards/auth.guard.ts`, linie 5–19)

```
 5  export const authGuard: CanActivateFn = async (route: ActivatedRouteSnapshot, state: RouterStateSnapshot) => {
 6    const keycloak = inject(Keycloak);
 7    const router = inject(Router);
 8
 9    if (!keycloak.authenticated) {
10      await keycloak.login({ redirectUri: window.location.origin + state.url });
11      return false;
12    }
13
14    const requiredRoles: string[] = route.data['roles'] ?? [];
15    if (requiredRoles.length === 0) return true;
16
17    const roles = keycloak.realmAccess?.roles ?? [];
18    return requiredRoles.every((role) => roles.includes(role));
19  };
```

**Kluczowe decyzje techniczne:** Tryb `check-sso` (zamiast `login-required`) pozostawia stronę główną i listę publicznych informacji dostępną bez logowania — logowanie wymuszane jest dopiero przez guard na trasach chronionych, co odpowiada modelowi biznesowemu platformy (przeglądanie przed rejestracją). Role odczytywane są bezpośrednio z tokenu JWT (claim `realm_access.roles`) — to samo źródło prawdy, które weryfikują serwisy backendowe (rozdział 2), eliminujące rozjazd ról między warstwami znany z monolitu (atrybut `custom:role` w Cognito kontra pole w bazie). Jeden parametryzowany guard (role z `data.roles`) zamiast osobnych klas per rola upraszcza konfigurację tras i testowanie.

**Pliki/komponenty:**
- `thera-ui/src/app/app.config.ts` — inicjalizacja Keycloak (Listing 4.13)
- `thera-ui/src/app/core/auth/guards/auth.guard.ts` — guard (Listing 4.14), `core/auth/state/auth.state.ts` — stan tożsamości
- `thera-ui/public/assets/silent-check-sso.html` — ramka cichego SSO
- `frontend/app/(auth)/authProvider.tsx` — integracja Amplify monolitu (Listing 4.12)

**Porównanie przed/po:**

| Wymiar | Monolit (Amplify/Cognito) | Angular (keycloak-angular) |
|---|---|---|
| **Formularze logowania** | Renderowane w aplikacji (Amplify Authenticator) | Po stronie Keycloak (motyw `theralink`) — poza kodem aplikacji |
| **Wykrycie sesji** | `fetchAuthSession()` przy każdym żądaniu | Ciche SSO raz przy starcie (`check-sso` + ukryta ramka) |
| **Ochrona tras** | Warunkowe sprawdzanie `pathname` w komponencie | Deklaratywny `authGuard` z rolami w `data.roles` |
| **Źródło ról** | Atrybut `custom:role` (Cognito) | Claim `realm_access.roles` tokenu JWT — wspólny z backendem |
| **Odświeżanie tokenu** | Ukryte w SDK Amplify | Jawne `updateToken(5)` w interceptorze (4.4) |
| **Uzależnienie od dostawcy** | `aws-amplify` w 12 plikach źródłowych | Konfiguracja w 1 pliku (`app.config.ts`) + standard OIDC |

---

## [ROZDZIAŁ: Migracja frontendu — konfiguracja środowisk i proces budowania]

**Temat:** Zastąpienie konfiguracji opartej na zmiennych środowiskowych Next.js (`.env`, prefiks `NEXT_PUBLIC_`) plikami środowiskowymi Angular podmienianymi w czasie budowania, wraz z porównaniem procesu budowania obu aplikacji.

**Technologie:** Pliki `.env` + zmienne `NEXT_PUBLIC_*` (monolit), `src/environments/environment.ts` / `environment.prod.ts` (Angular), builder `@angular/build:application` (esbuild), pnpm.

**Co zostało zaimplementowane:** W monolicie konfiguracja pochodziła z plików `.env` — z udokumentowanym w rozdziale 1 incydentem umieszczenia pliku `.env` z kluczami API w publicznym katalogu `frontend/public/`, serwowanym przez framework bez uwierzytelnienia. Mechanizm `NEXT_PUBLIC_*` dodatkowo zaciera granicę między konfiguracją serwerową a kliencką: ta sama konwencja pliku `.env` przechowuje sekrety serwera i wartości jawnie wkompilowywane w bundel przeglądarki. W nowej aplikacji konfiguracja kliencka jest jawnie publiczna i typowana: plik `environment.ts` (Listing 4.15) definiuje adres bramy API, parametry Keycloak i klucz publiczny Stripe dla środowiska deweloperskiego, a `environment.prod.ts` — odpowiedniki produkcyjne (`https://api.theralink.com`, `https://auth.theralink.com`). Wyboru pliku dokonuje builder w czasie budowania na podstawie konfiguracji (`ng build --configuration production`), a kompilator TypeScript wyklucza literówki w kluczach na etapie kompilacji. W plikach środowiskowych nie ma żadnych sekretów — wyłącznie wartości z definicji publiczne (adresy URL, identyfikator klienta OIDC typu *Public Client*, klucz publikowalny Stripe); sekrety systemu (klucz prywatny Stripe, hasła baz) istnieją wyłącznie po stronie backendu i infrastruktury (Azure Key Vault — rozdziały 2 i 9). Proces budowania oparty jest na esbuild — pełna generacja bundli produkcyjnych zajmuje ok. 3 s, a wynikowy katalog `dist/` (ok. 0,80 MB raw / 222 kB po kompresji gzip) serwowany jest jako statyczne pliki, bez procesu Node.js w środowisku produkcyjnym.

**Listing 4.15.** Publiczna konfiguracja środowiska deweloperskiego — klucz Stripe skrócony (plik `thera-ui/src/environments/environment.ts`, linie 1–12)

```
 1  export const environment = {
 2    production: false,
 3    apiGatewayUrl: 'http://localhost:8090',
 4    keycloak: {
 5      url: 'http://localhost:8080',
 6      realm: 'theralink',
 7      clientId: 'theralink-angular',
 8    },
 9    stripe: {
10      publicKey: 'pk_test_51TMrDmKz...',
11    },
12  };
```

**Kluczowe decyzje techniczne:** Przyjęto zasadę, że bundel frontendu z definicji jest publiczny — więc konfiguracja kliencka nie może zawierać niczego, co publiczne nie jest. Mechanizm plików `environment.*.ts` wymusza tę zasadę strukturalnie: nie istnieje ścieżka, którą sekret serwerowy mógłby „przeciec" do bundla, bo frontend nie współdzieli z backendem żadnego pliku konfiguracyjnego (w monolicie współdzielił konwencję `.env`, co poskutkowało incydentem `public/.env`). Podczas próby zbudowania monolitu na potrzeby pomiarów ujawniła się dodatkowa różnica jakościowa: `next build` nie kończy się bez modyfikacji projektu — kolejno blokowały go błędy ESLint (m.in. `no-explicit-any` w komponentach czatu), błąd kompilacji typów w `GMap.tsx` (konflikt dwóch kopii `@types/react` wynikający z dwóch plików `package-lock.json` w repozytorium) oraz błąd prerenderingu `/confirm-booking` (`useSearchParams()` bez granicy `Suspense`). Build aplikacji Angular przechodzi bez ostrzeżeń blokujących wraz z pełnym zestawem testów.

**Pliki/komponenty:**
- `thera-ui/src/environments/environment.ts` (Listing 4.15), `environment.prod.ts` — konfiguracja per środowisko
- `thera-ui/angular.json` — konfiguracje budowania (development/production)
- `frontend/.env.local`, `frontend/public/.env` — konfiguracja monolitu (incydent bezpieczeństwa opisany w rozdziale 1)

**Porównanie przed/po:**

| Wymiar | Monolit Next.js | Aplikacja Angular |
|---|---|---|
| **Źródło konfiguracji** | Pliki `.env` (wspólna konwencja z backendem) | Typowane pliki `environment.*.ts` per środowisko |
| **Granica publiczne/sekretne** | Konwencja nazewnicza `NEXT_PUBLIC_` | Strukturalna — frontend nie ma dostępu do żadnych sekretów |
| **Incydenty ekspozycji** | `public/.env` z kluczami API dostępny przez HTTP | Brak możliwości — w bundlu tylko wartości publiczne |
| **Adres backendu** | `NEXT_PUBLIC_API_BASE_URL` (bezpośrednio backend) | `environment.apiGatewayUrl` (brama Spring Cloud Gateway) |
| **Wynik builda produkcyjnego** | Wymaga procesu Node.js (SSR) | Statyczne pliki — dowolny serwer HTTP / kontener nginx |
| **Przebieg builda** | Nie kończy się bez wyłączenia lintera i typecheck | Przechodzi w całości wraz z testami |

---

## [ROZDZIAŁ: Migracja frontendu — podsumowanie i zbiorcze porównanie]

**Temat:** Zbiorcze zestawienie wszystkich wymiarów migracji warstwy klienckiej wraz z mierzalnymi parametrami obu aplikacji oraz wnioskami.

**Technologie:** Wszystkie wymienione w podrozdziałach 4.1–4.8; pomiary: `next build` (Next.js 15.3.2), `ng build` (builder esbuild), Vitest 4.

**Co zostało zaimplementowane:** Migrację warstwy klienckiej zrealizowano w całości — wszystkie funkcjonalności monolitu objęte zakresem pracy (wyszukiwanie i przeglądanie psychologów, rezerwacja wizyt, dashboardy obu ról, rozliczenia, zarządzanie dostępnością) działają w aplikacji Angular, rozszerzone o elementy wcześniej nieobecne: internacjonalizację, wybór terminu z faktycznych dostępności, deklaratywną ochronę tras z kontrolą ról oraz testy jednostkowe (11 plików, 37 testów — interceptory, guard, trzy stany NGXS, trzy serwisy HTTP, komponent wyboru terminu; komplet przechodzi w 3,4 s). Pomiary buildów produkcyjnych z 2026-06-10 zestawiono w tabeli zbiorczej.

**Kluczowe decyzje techniczne (wnioski):**
1. **Konsolidacja przewyższa elastyczność w projekcie jednoosobowym.** Zastąpienie swobodnie dobieranego ekosystemu React jednym frameworkiem zredukowało zależności z 44 do 25, systemy UI z trzech do jednego i wyeliminowało kategorię błędów „niespójne konwencje między plikami" — kosztem wyższego progu wejścia, który w kontekście pracy dyplomowej był korzyścią edukacyjną.
2. **Rozmiar dostarczanego kodu spadł o rząd wielkości.** Łączny JavaScript klienta zmniejszył się z 11,53 MB raw / 2,47 MB gzip (312 plików) do 0,80 MB raw / 0,22 MB gzip (~20 plików) — głównie dzięki eliminacji nieużywanych bibliotek, konsolidacji UI i wycięciu modułu czatu AI; fragment inicjalny aplikacji Angular to 119,87 kB transferu, resztę dostarczają leniwe fragmenty per obszar.
3. **Jakość wymuszana przez narzędzia, nie dyscyplinę.** Monolit nie przechodził własnego builda produkcyjnego (lint, typy, prerendering) — wady ujawniały się dopiero przy próbie wdrożenia. W aplikacji Angular kompilacja w trybie strict, budowanie i 37 testów stanowią jedną bramkę jakości wykonywaną w całości w ok. 7 s.
4. **Standard zamiast dostawcy.** Wymiana Amplify/Cognito (12 plików z zależnością od SDK) na OIDC/Keycloak (1 plik konfiguracji) odcięła ostatnie uzależnienie warstwy klienckiej od konkretnej chmury — warunek przenośności całego systemu (rozdziały 2 i 9).

**Pliki/komponenty:** Całość repozytorium `thera-ui` w zestawieniu z katalogiem `frontend/` monolitu.

**Porównanie przed/po (tabela zbiorcza):**

| Wymiar | Monolit (Next.js/React) | Aplikacja Angular | Podrozdział |
|---|---|---|---|
| **Framework** | Next.js 15.3.2 + React 19 (biblioteka + dobór ekosystemu) | Angular 21.2 (framework zintegrowany) | 4.1 |
| **Zależności (prod + dev)** | 33 + 11 = 44 | 18 + 7 = 25 | 4.1 |
| **Routing** | File-based (App Router), 14 plików `"use client"` | Jawny, 5 obszarów lazy, 100% standalone | 4.2 |
| **Stan globalny** | Redux Toolkit (1 slice) + RTK Query (14 endpointów) | NGXS (3 stany: auth, psychologists, appointments) | 4.3 |
| **Stan lokalny** | `useState`/`useMemo`/`useEffect` | Signals: `signal`, `computed`, `linkedSignal`, `effect` | 4.3 |
| **Token w żądaniach** | `prepareHeaders` + ręczne `fetch` (2 miejsca) | 1 interceptor z `updateToken(5)` | 4.4 |
| **Obsługa błędów HTTP** | Per komponent | Centralny `errorInterceptor` (401/403) | 4.4 |
| **Systemy UI** | 3 (MUI + Radix + shadcn) + Tailwind, kolory hardkodowane | 1 (Angular Material), centralny motyw SCSS | 4.5 |
| **Komponenty domenowe** | Brak | `PsychologistCard`, `DetailsPanel`, `AppointmentSlotPicker` | 4.5 |
| **i18n** | Brak (mieszanka pl/en w kodzie) | Transloco: pl + en, 5 zakresów lazy | 4.6 |
| **Uwierzytelnienie** | Amplify/Cognito w aplikacji, `custom:role` | Keycloak OIDC, ciche SSO, role z JWT | 4.7 |
| **Ochrona tras** | Warunki na `pathname` w komponencie | `authGuard` + `data.roles` na trasach | 4.7 |
| **Konfiguracja** | `.env` + `NEXT_PUBLIC_*` (incydent `public/.env`) | `environment.*.ts` — tylko wartości publiczne | 4.8 |
| **JS klienta (raw / gzip)** | 11,53 MB / 2,47 MB (312 plików) | 0,80 MB / 0,22 MB (fragment inicjalny 119,87 kB gzip) | 4.9 |
| **Czas budowania** | Kompilacja 8,0 s; pełny build nie kończy się bez modyfikacji | Pełny build 3,0 s (esbuild) | 4.8–4.9 |
| **Testy jednostkowe** | 0 | 11 plików / 37 testów (Vitest), 3,4 s | 4.9 |

---

## Sugerowane zrzuty ekranu do tego rozdziału

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Dwa drzewa katalogów obok siebie: `frontend/app/` monolitu (katalogi-trasy: `signin`, `signup`, `view`, `dashboard`, `(auth)`, `api/chat`) oraz `thera-ui/src/app/` (podział `core/`, `features/`, `dashboards/`, `shared/`) — rozwinięte do drugiego poziomu w IntelliJ IDEA / VS Code.
> **Sugerowany podpis:** Rys. 4.1. Struktura katalogów warstwy klienckiej przed migracją (Next.js, struktura pochodna tras) i po migracji (Angular, struktura pochodna odpowiedzialności)
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Zakładka Network w Chrome DevTools podczas nawigacji ze strony głównej na `/psychologists` — moment dociągnięcia leniwego fragmentu `chunk-…js (psychologists-routes)` oraz pliku tłumaczeń `browse/pl.json`.
> **Sugerowany podpis:** Rys. 4.2. Leniwe ładowanie fragmentu kodu i zakresu tłumaczeń przy wejściu na trasę listy psychologów
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Konsola przeglądarki z wpisami NGXS Logger Plugin — akcja `[Psychologists] Load` z migawkami stanu przed/po (loading: true → items: […], loading: false).
> **Sugerowany podpis:** Rys. 4.3. Rejestr akcji i zmian stanu NGXS w konsoli przeglądarki
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Dwa zrzuty tej samej funkcjonalności (np. lista psychologów): monolit (mieszanka Material UI/Tailwind) i aplikacja Angular (spójny motyw Material z paletą `#2b6369`/`#00bfa5`, karty `PsychologistCard`).
> **Sugerowany podpis:** Rys. 4.4. Interfejs listy psychologów przed migracją i po konsolidacji systemu komponentów
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Komponent `AppointmentSlotPicker` w panelu szczegółów psychologa — kalendarz Material z aktywnymi tylko dniami z dostępnościami oraz lista godzin wybranego dnia.
> **Sugerowany podpis:** Rys. 4.5. Komponent domenowy wyboru terminu wizyty oparty na dostępnościach psychologa
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Strona logowania Keycloak z motywem `theralink` po przekierowaniu przez `authGuard` z trasy chronionej (widoczny adres `…/realms/theralink/protocol/openid-connect/auth` na pasku przeglądarki).
> **Sugerowany podpis:** Rys. 4.6. Ekran logowania serwera Keycloak wywołany przez guard trasy chronionej
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Terminal z wynikiem `ng build` (tabela Initial/Lazy chunk files z rozmiarami raw i transfer, czas 3,0 s) oraz wynikiem `pnpm test` (11 plików, 37 testów, zielone).
> **Sugerowany podpis:** Rys. 4.7. Wynik budowania produkcyjnego i pełnego zestawu testów aplikacji Angular
> **Źródło:** opracowanie własne

> 📸 **[SCREEN DO DODANIA]**
> **Co pokazać:** Terminal z nieudaną próbą `next build` monolitu — błędy ESLint (`no-explicit-any`) lub błąd typów w `GMap.tsx` (konflikt `@types/react`), jako kontrast do Rys. 4.7.
> **Sugerowany podpis:** Rys. 4.8. Nieudany przebieg budowania produkcyjnego monolitu — błędy ujawniane dopiero na etapie builda
> **Źródło:** opracowanie własne
