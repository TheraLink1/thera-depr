# STORY-02 — Transloco Scopes, AppointmentSlotPicker, Testy jednostkowe

**Data:** 2026-06-10  
**Commit:** `8dc5826`  
**Branch:** `main` (thera-ui)  
**Status:** ✅ Zrealizowane — build i 37 testów zielone

---

## Co i dlaczego

Trzy niezależne problemy rozwiązane w jednym sprincie:

1. **i18n był monolityczny** — dwa pliki `pl.json`/`en.json` ładowały się w całości przy starcie aplikacji, niwecząc lazy-loading tras.
2. **Hardkodowane polskie stringi** — komunikaty snackbara w `confirm-booking` były literałami w TypeScript zamiast kluczy Transloco.
3. **Brak komponentu domenowego** — logika wyboru terminu była wmieszana w `details-panel`; brak testów jednostkowych.

---

## A. Transloco Scopes — teoria i implementacja

### Jak działa scope w Transloco

Transloco scope to podział tłumaczeń na pliki per-feature. Loader HTTP automatycznie przekształca parametr `lang` z `"pl"` na `"booking/pl"`, więc istniejący `TranslocoHttpLoader` (`/assets/i18n/${lang}.json`) **nie wymaga zmian** — scope param `booking/pl` naturalnie mapuje się do `assets/i18n/booking/pl.json`.

Klucze w pliku scope są **bez prefiksu sekcji** — scope samo staje się prefiksem:

```json
// assets/i18n/booking/pl.json
{
  "title": "Potwierdź rezerwację",
  "success": "Wizyta została zarezerwowana!"
}
```

W template: `{{ 'booking.title' | transloco }}` — Transloco widzi prefix `booking`, ładuje scope i zwraca klucz `title`.

### Podział plików

| Plik root (zawsze) | Scope (lazy, z trasą) |
|---|---|
| `nav`, `common` | `home`, `browse`, `booking`, `clientDashboard`, `psychologistDashboard` |

Każdy scope ma: `assets/i18n/{scope}/{pl,en}.json`

### Rejestracja scope w trasach

```typescript
// features/booking/booking.routes.ts
import { provideTranslocoScope } from '@jsverse/transloco';

export const BOOKING_ROUTES: Routes = [
  { path: 'confirm', component: ConfirmBookingComponent, providers: [provideTranslocoScope('booking')] }
];
```

Wzorzec identyczny dla wszystkich 5 plików tras.

---

## B. Fix hardkodowanych tekstów

`confirm-booking.component.ts` — przed:
```typescript
this.snackBar.open('Wizyta została zarezerwowana!', 'OK', { duration: 4000 });
```

Po:
```typescript
private transloco = inject(TranslocoService);
// ...
this.snackBar.open(this.transloco.translate('booking.success'), 'OK', { duration: 4000 });
```

Klucze `booking.success` i `booking.error` istniały już w tłumaczeniach — wystarczyło podpiąć serwis.

---

## C. AppointmentSlotPickerComponent

**Lokalizacja:** `src/app/features/booking/appointment-slot-picker/`

### API komponentu

```typescript
@Component({ selector: 'thera-appointment-slot-picker', ... })
export class AppointmentSlotPickerComponent {
  psychologistId = input.required<string>();
  slotSelected   = output<{ date: string; startHour: string }>();
}
```

Typ `startHour: string` dopasowany do `CalendarSlot.startHour` z `AvailabilityService`.

### Wzorzec ładowania danych (signals + RxJS)

```typescript
private availabilityResult = toSignal(
  toObservable(this.psychologistId).pipe(
    switchMap(id =>
      this.availabilityService.getForPsychologist(id).pipe(
        map(slots => groupByDate(slots)),
        startWith(emptyResult(true)),
        catchError(() => of(emptyResult(false)))
      )
    )
  ),
  { initialValue: emptyResult(true) }
);

loading        = computed(() => this.availabilityResult().loading);
availableDates = computed(() => this.availabilityResult().availableDates);
```

`toObservable(signal)` reaguje na każdą zmianę `psychologistId`. `switchMap` anuluje poprzednie żądanie. `startWith` + `catchError` zapewniają stany loading/error bez dodatkowego boilerplate.

### Integracja z details-panel

`details-panel` oddał całą logikę dostępności pickerowi. Teraz tylko:
```typescript
onSlotSelected(slot: { date: string; startHour: string }) {
  this.router.navigate(['/booking/confirm'], {
    queryParams: { psychologistId: this.psychologist().cognitoId, date: slot.date, time: slot.startHour },
  });
}
```

Template:
```html
<thera-appointment-slot-picker
  [psychologistId]="psychologist().cognitoId"
  (slotSelected)="onSlotSelected($event)"
/>
```

---

## D. Testy jednostkowe

**37 testów, 11 plików, 0 failures.**

### Wzorce testowe (Angular 21 + Vitest)

**Interceptory (function-based):**
```typescript
provideHttpClient(withInterceptors([jwtInterceptor]))
provideHttpClientTesting()
{ provide: Keycloak, useValue: mockKeycloak }
// Interceptor używa Promise — potrzeba await przed expectOne:
const flushPromises = () => new Promise<void>(resolve => setTimeout(resolve));
http.get('/api/test').subscribe();
await flushPromises();
httpTesting.expectOne('/api/test');
```

> `fakeAsync`/`flushMicrotasks` **nie działa** w tym projekcie (brak `zone-testing.js`). Zamiast tego `setTimeout(resolve)` flush'uje mikrotaski.

**Guard (function-based `CanActivateFn`):**
```typescript
const result = await TestBed.runInInjectionContext(() => authGuard(route, state));
```

**NGXS State:**
```typescript
provideStore([AuthState])
{ provide: AuthService, useValue: { login: vi.fn() } }
store.dispatch(new Login()).toPromise()
store.selectSnapshot(AuthState.isLoggedIn)
```

**HTTP Services:**
```typescript
httpTesting.expectOne(`${environment.apiGatewayUrl}/api/appointments`);
req.request.method === 'POST'
req.request.body  === payload
```

**Standalone komponent z sygnałowym inputem:**
```typescript
fixture.componentRef.setInput('psychologistId', 'p-1');
fixture.detectChanges();
// slotSelected output:
component.slotSelected.subscribe(v => emitted.push(v));
fixture.nativeElement.querySelector('.time-btn').click();
```

### Tabela plików testowych

| Plik | Scenariusze |
|---|---|
| `jwt.interceptor.spec.ts` | token → Bearer; brak tokenu; updateToken reject |
| `error.interceptor.spec.ts` | 401 → navigate(/); 403 → navigate(/); 500 → propaguje |
| `auth.guard.spec.ts` | authenticated + rola → true; brak roli → false; niezalogowany → login() |
| `auth.state.spec.ts` | Login → authService.login(); Logout; default selectors |
| `psychologists.state.spec.ts` | LoadPsychologists → items; SelectPsychologist → selectedId; selected selector |
| `appointments.state.spec.ts` | LoadClient; LoadPsychologist; service error → items puste |
| `appointment.service.spec.ts` | create POST; getForClient GET; getForPsychologist GET; update PUT |
| `psychologist.service.spec.ts` | getAll; getAll z params; getById; update |
| `availability.service.spec.ts` | getForPsychologist GET; create POST |
| `appointment-slot-picker.component.spec.ts` | render slotów; klik → emit; loading state; error state |
| `app.spec.ts` | creates; navbar+footer widoczne |

---

## Kluczowe decyzje techniczne

- **`TranslocoHttpLoader` bez zmian** — scope param `booking/pl` automatycznie tworzy ścieżkę `assets/i18n/booking/pl.json`.
- **`startHour: string`** w output — dopasowany do `CalendarSlot.startHour`, nie `number` jak sugerował opis.
- **`NO_ERRORS_SCHEMA`** w teście pickera — eliminuje konieczność full setup Angular Material w środowisku testowym.
- **`TranslocoTestingModule.forRoot()`** z `@jsverse/transloco` (nie `/testing`) — ten subpath nie istnieje w v8.

---

## Powiązane dokumenty

- [[angular-style-guide]] — wzorce signals, OnPush, input()/output()
- [[frontend-migration]] — kontekst migracji do Angular
