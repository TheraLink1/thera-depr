# TheraLink — Frontend Migration Plan (Next.js → Angular 18)

## Context

The current frontend is a **Next.js 15 + React 19** application using AWS Amplify for authentication, Redux Toolkit + RTK Query for state/data fetching, and Material-UI for components. The goal is to migrate it to a standalone **Angular 18+** application (`theralink-frontend` repository) that connects to the new Spring Boot microservices through Spring Cloud Gateway and uses Keycloak for authentication.

**Why Angular?** Angular is a complete opinionated framework: built-in DI, router, reactive forms, HttpClient, and TypeScript-first. Better suited for a multi-team enterprise product vs React's pick-your-own-everything approach.

**Repository:** `theralink-frontend` (new, empty repo — full greenfield, NOT a migration of existing files)

---

## What Exists in the Current Frontend (Map to Migrate)

| Current File / Page | Angular Target |
|---|---|
| `app/(auth)/authProvider.tsx` | `core/auth/auth.service.ts` + Keycloak-Angular |
| `app/signin/page.tsx` | Keycloak handles login UI (custom theme in `theralink-keycloak`) |
| `app/signup/page.tsx` | Keycloak handles registration UI |
| `app/view/page.tsx` + `Card.tsx` + `DetailsPanel.tsx` | `features/psychologists/browse/` |
| `app/confirm-booking/page.tsx` | `features/booking/confirm-booking/` |
| `app/dashboard/client/*` | `features/dashboard-client/` |
| `app/dashboard/psychologist/*` | `features/dashboard-psychologist/` |
| `app/components/Navbar.tsx` | `shared/components/navbar/` |
| `app/components/GMap.tsx` + `AddressAutocomplete.tsx` | `shared/components/map/` |
| `frontend/state/api.ts` (RTK Query) | `core/services/*.service.ts` + NGXS `@Action` methods |
| `frontend/state/index.ts` (Redux slices) | `store/` (NGXS `@State` classes) |
| `types/prismaTypes.d.ts` | `shared/models/*.model.ts` |

**Current API base URL:** `http://localhost:3001` (Express backend)
**New API base URL:** `http://localhost:8090` (Spring Cloud Gateway)

---

## Step 0: Prerequisites

Before starting Angular work, these must exist locally:
- Keycloak running at `http://localhost:8080` (from `theralink-keycloak` `docker-compose.yml`)
- Keycloak realm `theralink` imported with roles `client` and `psychologist`
- Spring Cloud Gateway running at `http://localhost:8090` (or at minimum, the old Express backend available)

---

## Step 1: Project Scaffolding

### 1.1 Create Angular Project
```bash
npm install -g @angular/cli@21
ng new theralink-frontend \
  --routing=true \
  --style=scss \
  --strict=true \
  --standalone=false    # Use NgModules (not standalone components) for enterprise structure
cd theralink-frontend
```

### 1.2 Install Dependencies
```bash
# Angular Material (UI — replaces MUI)
ng add @angular/material
# Choose: custom theme → yes global typography → yes animations

# Keycloak (replaces AWS Amplify)
npm install keycloak-angular@21 keycloak-js@26

# NGXS (state management — replaces Redux Toolkit)
npm install @ngxs/store @ngxs/form-plugin @ngxs/router-plugin @ngxs/logger-plugin

# i18n (internationalization)
ng add @jsverse/transloco      # Transloco — runtime i18n (Polish + English)

# Date/time
npm install date-fns @danielmoncada/angular-datetime-picker

# UI extras
npm install @swimlane/ngx-charts   # Charts for billings/analytics
npm install ngx-mask               # Input masks (phone numbers, etc.)
npm install ngx-editor             # Rich text editor (psychologist profiles, notes)
npm install ngx-cookie-service     # Cookie management
npm install emoji-picker-element   # Emoji picker

# Utilities
npm install @popperjs/core mime ngx-papaparse

# YouTube player (Angular Material component)
# — bundled with @angular/youtube-player (already in @angular/* packages)
```

### 1.3 Final `package.json` Key Dependencies
```json
{
  "dependencies": {
    "@angular/animations": "^21.1.0",
    "@angular/cdk": "^21.1.0",
    "@angular/common": "^21.1.0",
    "@angular/compiler": "^21.1.0",
    "@angular/core": "^21.1.0",
    "@angular/forms": "^21.1.0",
    "@angular/material": "^21.1.0",
    "@angular/platform-browser": "^21.1.0",
    "@angular/platform-browser-dynamic": "^21.1.0",
    "@angular/router": "^21.1.0",
    "@angular/youtube-player": "^21.1.0",
    "@danielmoncada/angular-datetime-picker": "^20.0.1",
    "@jsverse/transloco": "^8.2.0",
    "@ngxs/form-plugin": "^20.1.0",
    "@ngxs/logger-plugin": "^20.1.0",
    "@ngxs/router-plugin": "^20.1.0",
    "@ngxs/store": "^20.1.0",
    "@popperjs/core": "^2.11.8",
    "@swimlane/ngx-charts": "^23.1.0",
    "date-fns": "^4.1.0",
    "emoji-picker-element": "^1.28.1",
    "keycloak-angular": "^21.0.0",
    "keycloak-js": "^26.2.2",
    "mime": "^4.1.0",
    "ngx-cookie-service": "^21.1.0",
    "ngx-editor": "19.0.0-beta.1",
    "ngx-mask": "^20.0.3",
    "ngx-papaparse": "^8.0.0",
    "prosemirror-model": "^1.25.4",
    "prosemirror-state": "^1.4.4",
    "rxjs": "~7.8.2"
  }
}
```

---

## Step 2: Project Structure (Full Directory Layout)

```
theralink-frontend/
├── src/
│   ├── app/
│   │   ├── core/                               # Imported ONCE in AppModule
│   │   │   ├── auth/
│   │   │   │   ├── auth.service.ts             # Keycloak wrapper service
│   │   │   │   ├── auth.guard.ts               # Protects authenticated routes
│   │   │   │   ├── role.guard.ts               # Protects role-specific routes
│   │   │   │   └── keycloak-init.factory.ts    # APP_INITIALIZER function
│   │   │   ├── interceptors/
│   │   │   │   ├── jwt.interceptor.ts          # Attaches Bearer token
│   │   │   │   └── error.interceptor.ts        # Global error handling (401, 403, 500)
│   │   │   ├── services/
│   │   │   │   └── api-error.service.ts        # Centralized error messages
│   │   │   └── core.module.ts
│   │   │
│   │   ├── shared/                             # Imported in feature modules
│   │   │   ├── components/
│   │   │   │   ├── navbar/
│   │   │   │   │   ├── navbar.component.ts
│   │   │   │   │   ├── navbar.component.html
│   │   │   │   │   └── navbar.component.scss
│   │   │   │   ├── footer/
│   │   │   │   ├── loading-spinner/
│   │   │   │   ├── confirmation-dialog/        # Reusable Angular Material dialog
│   │   │   │   └── map/
│   │   │   │       ├── map.component.ts        # Google Maps wrapper
│   │   │   │       ├── map.component.html
│   │   │   │       └── address-autocomplete/
│   │   │   │           └── address-autocomplete.component.ts
│   │   │   ├── models/
│   │   │   │   ├── client.model.ts
│   │   │   │   ├── psychologist.model.ts
│   │   │   │   ├── appointment.model.ts
│   │   │   │   └── payment.model.ts
│   │   │   ├── pipes/
│   │   │   │   └── currency-pln.pipe.ts        # 20000 groszy → "200,00 zł"
│   │   │   │   # status labels handled via Transloco keys (status.PENDING etc.)
│   │   │   └── shared.module.ts
│   │   │
│   │   ├── features/
│   │   │   ├── auth/                           # Post-login redirects + role selection
│   │   │   │   ├── role-select/                # After Keycloak login, choose client/psychologist
│   │   │   │   │   ├── role-select.component.ts
│   │   │   │   │   └── role-select.component.html
│   │   │   │   └── auth.module.ts
│   │   │   │
│   │   │   ├── psychologists/                  # Replaces /view page
│   │   │   │   ├── browse/
│   │   │   │   │   ├── browse.component.ts     # Main browse page with map + list
│   │   │   │   │   ├── browse.component.html
│   │   │   │   │   ├── psychologist-card/
│   │   │   │   │   │   ├── psychologist-card.component.ts
│   │   │   │   │   │   └── psychologist-card.component.html
│   │   │   │   │   └── detail-panel/
│   │   │   │   │       ├── detail-panel.component.ts
│   │   │   │   │       └── detail-panel.component.html
│   │   │   │   ├── services/
│   │   │   │   │   └── psychologist.service.ts # HTTP calls to /api/psychologists
│   │   │   │   └── psychologists.module.ts
│   │   │   │
│   │   │   ├── booking/                        # Replaces /confirm-booking
│   │   │   │   ├── confirm-booking/
│   │   │   │   │   ├── confirm-booking.component.ts
│   │   │   │   │   └── confirm-booking.component.html
│   │   │   │   ├── services/
│   │   │   │   │   └── booking.service.ts      # HTTP calls to /api/appointments
│   │   │   │   └── booking.module.ts
│   │   │   │
│   │   │   ├── dashboard-client/               # Replaces dashboard/client/*
│   │   │   │   ├── client-dashboard/           # Main dashboard layout + sidenav
│   │   │   │   ├── appointment-history/
│   │   │   │   ├── billings/
│   │   │   │   ├── account-settings/
│   │   │   │   ├── verify-form/
│   │   │   │   └── dashboard-client.module.ts
│   │   │   │
│   │   │   ├── dashboard-psychologist/         # Replaces dashboard/psychologist/*
│   │   │   │   ├── psychologist-dashboard/
│   │   │   │   ├── calendar/                   # Calendar view for sessions
│   │   │   │   ├── set-availability/           # Replaces SetAvailability.tsx
│   │   │   │   ├── appointments/
│   │   │   │   ├── billings/
│   │   │   │   ├── ratings/
│   │   │   │   ├── account-settings/
│   │   │   │   └── dashboard-psychologist.module.ts
│   │   │   │
│   │   │   └── payment/                        # Stripe payment flow
│   │   │       ├── checkout/
│   │   │       │   ├── checkout.component.ts   # Stripe Elements card form
│   │   │       │   └── checkout.component.html
│   │   │       ├── payment-history/
│   │   │       ├── services/
│   │   │       │   └── payment.service.ts
│   │   │       └── payment.module.ts
│   │   │
│   │   ├── store/                              # NGXS global state
│   │   │   ├── auth/
│   │   │   │   └── auth.state.ts              # State + Actions + Selectors in one file
│   │   │   ├── psychologists/
│   │   │   │   └── psychologists.state.ts
│   │   │   ├── appointments/
│   │   │   │   └── appointments.state.ts
│   │   │   └── app.state.ts                   # Root state — registers all states
│   │   │
│   │   ├── app-routing.module.ts
│   │   ├── app.component.ts
│   │   ├── app.component.html
│   │   └── app.module.ts
│   │
│   ├── environments/
│   │   ├── environment.ts                     # Dev config
│   │   └── environment.prod.ts                # Prod config
│   │
│   ├── assets/
│   │   ├── silent-check-sso.html              # Required for Keycloak SSO
│   │   ├── i18n/
│   │   │   ├── pl.json                        # Global Polish translations
│   │   │   ├── en.json                        # Global English translations
│   │   │   ├── booking/
│   │   │   │   ├── pl.json                    # Lazy-loaded per feature
│   │   │   │   └── en.json
│   │   │   ├── dashboard-client/
│   │   │   │   ├── pl.json
│   │   │   │   └── en.json
│   │   │   └── dashboard-psychologist/
│   │   │       ├── pl.json
│   │   │       └── en.json
│   │   └── images/
│   │       └── logo.svg
│   │
│   ├── styles.scss                            # Global styles
│   └── index.html
│
├── angular.json
├── tsconfig.json
├── package.json
├── Dockerfile
├── nginx.conf
└── .env.example
```

---

## Step 3: Environment Configuration

```typescript
// src/environments/environment.ts
export const environment = {
  production: false,
  apiGatewayUrl: 'http://localhost:8090',
  keycloak: {
    url: 'http://localhost:8080',
    realm: 'theralink',
    clientId: 'theralink-frontend',
  },
  googleMapsApiKey: '',             // Set via CI/CD secret injection
  stripePublishableKey: '',         // Set via CI/CD secret injection
};
```

```typescript
// src/environments/environment.prod.ts
export const environment = {
  production: true,
  apiGatewayUrl: 'https://api.theralink.com',
  keycloak: {
    url: 'https://auth.theralink.com',
    realm: 'theralink',
    clientId: 'theralink-frontend',
  },
  googleMapsApiKey: '#{GOOGLE_MAPS_KEY}#',    // Azure DevOps token replacement
  stripePublishableKey: '#{STRIPE_PUB_KEY}#',
};
```

---

## Step 3b: Internationalization (i18n) with Transloco

### Theory: Why Transloco Instead of Angular's Built-in i18n?

Angular ships with a built-in i18n system, but it has a major limitation: **it requires a separate build per language**. This means if you support Polish and English, you need to compile the app twice and serve two different bundles. For a platform like TheraLink with a primary Polish audience, this is unnecessary overhead.

**Transloco** solves this with **runtime translations** — translation files are loaded as JSON at runtime, and you can switch language without rebuilding the app. It also supports:
- Lazy-loading translations per feature module (keeps initial bundle small)
- Pluralization and parameterized messages
- Structural directives (`*transloco`) and pipes (`| transloco`)
- Falls back to a default language if a key is missing

### Installation

`ng add @jsverse/transloco` runs the schematics automatically and:
1. Installs the library
2. Creates `TranslocoHttpLoader` in your project
3. Creates initial `assets/i18n/pl.json` and `assets/i18n/en.json` files
4. Adds `TranslocoModule` to `AppModule`

### Translation Files Structure

```
src/assets/i18n/
├── pl.json        ← Primary language (Polish)
└── en.json        ← Secondary language (English)
```

For large apps, split translations per feature using Transloco's **scopes**:

```
src/assets/i18n/
├── pl.json                    ← Shared/global keys
├── en.json
├── psychologists/
│   ├── pl.json                ← Keys for psychologists feature only
│   └── en.json
├── booking/
│   ├── pl.json
│   └── en.json
├── dashboard-client/
│   ├── pl.json
│   └── en.json
└── dashboard-psychologist/
    ├── pl.json
    └── en.json
```

Scoped files are **lazy-loaded** — the `booking/pl.json` file is only downloaded when the user navigates to the booking feature. This keeps initial load fast.

### Global Translation File (`assets/i18n/pl.json`)

```json
{
  "nav": {
    "home": "Strona główna",
    "browse": "Znajdź psychologa",
    "login": "Zaloguj się",
    "logout": "Wyloguj się",
    "dashboard": "Panel"
  },
  "common": {
    "save": "Zapisz",
    "cancel": "Anuluj",
    "loading": "Ładowanie...",
    "error": "Wystąpił błąd. Spróbuj ponownie.",
    "success": "Operacja zakończona pomyślnie."
  },
  "auth": {
    "role": {
      "client": "Klient",
      "psychologist": "Psycholog"
    }
  },
  "status": {
    "PENDING": "Oczekujący",
    "APPROVED": "Zatwierdzony",
    "CANCELLED": "Anulowany",
    "COMPLETED": "Zakończony",
    "UNPAID": "Nieopłacony",
    "PAID": "Opłacony",
    "REFUNDED": "Zwrócony"
  }
}
```

```json
{
  "booking": {
    "title": "Potwierdź wizytę",
    "psychologist": "Psycholog",
    "date": "Data i godzina",
    "duration": "Czas trwania",
    "minutes": "{{ value }} minut",
    "price": "Cena",
    "confirm": "Potwierdź rezerwację",
    "success": "Wizyta zarezerwowana pomyślnie!",
    "error": "Nie udało się zarezerwować wizyty."
  }
}
```

### TranslocoModule Setup

```typescript
// src/app/app.module.ts
import { TranslocoModule, provideTransloco, translocoConfig } from '@jsverse/transloco';
import { TranslocoHttpLoader } from './transloco-loader';  // generated by schematic

@NgModule({
  providers: [
    provideTransloco({
      config: translocoConfig({
        availableLangs: ['pl', 'en'],
        defaultLang: 'pl',          // Polish is the primary language
        fallbackLang: 'pl',         // Fall back to Polish if a key is missing in English
        reRenderOnLangChange: true, // Re-render templates when language switches
        prodMode: environment.production,
      }),
      loader: TranslocoHttpLoader,  // Loads JSON files from /assets/i18n/
    }),
  ],
})
export class AppModule {}
```

### Using Translations in Templates (Pipe and Directive)

```html
<!-- OPTION 1: transloco pipe (for simple inline text) -->
<h1>{{ 'nav.home' | transloco }}</h1>

<!-- OPTION 2: transloco pipe with parameters -->
<p>{{ 'booking.minutes' | transloco: { value: 50 } }}</p>
<!-- Output: "50 minut" -->

<!-- OPTION 3: *transloco structural directive (recommended for blocks) -->
<!-- Loads the translation context once, all children read from it -->
<ng-container *transloco="let t">
  <h2>{{ t('booking.title') }}</h2>
  <p>{{ t('booking.psychologist') }}: {{ psychologist.name }}</p>
  <button>{{ t('booking.confirm') }}</button>
</ng-container>

<!-- OPTION 4: Scoped translations (lazy-loaded per feature) -->
<ng-container *transloco="let t; read: 'booking'">
  <h2>{{ t('title') }}</h2>           <!-- reads booking.title -->
  <button>{{ t('confirm') }}</button>  <!-- reads booking.confirm -->
</ng-container>
```

### Using Translations in TypeScript (Service)

```typescript
// features/booking/confirm-booking/confirm-booking.component.ts
import { TranslocoService } from '@jsverse/transloco';

@Component({ ... })
export class ConfirmBookingComponent {
  constructor(private transloco: TranslocoService) {}

  showSuccessMessage(): void {
    // Get translation imperatively (in TypeScript, not template)
    const message = this.transloco.translate('booking.success');
    this.snackBar.open(message, 'OK', { duration: 3000 });
  }

  // Translate with parameters
  formatDuration(minutes: number): string {
    return this.transloco.translate('booking.minutes', { value: minutes });
    // Output: "50 minut"
  }
}
```

### Language Switcher Component

```typescript
// shared/components/language-switcher/language-switcher.component.ts
import { TranslocoService } from '@jsverse/transloco';

@Component({
  selector: 'app-language-switcher',
  template: `
    <button mat-button (click)="setLang('pl')" [class.active]="activeLang === 'pl'">PL</button>
    <button mat-button (click)="setLang('en')" [class.active]="activeLang === 'en'">EN</button>
  `,
})
export class LanguageSwitcherComponent {
  activeLang = this.transloco.getActiveLang();

  constructor(private transloco: TranslocoService) {}

  setLang(lang: string): void {
    this.transloco.setActiveLang(lang);
    this.activeLang = lang;
    // Optionally persist to localStorage so the choice survives page refresh:
    localStorage.setItem('theralink-lang', lang);
  }
}
```

### Lazy-Loading Translations per Feature Module

Instead of loading all translations upfront, each feature module declares its own scope:

```typescript
// features/booking/booking.module.ts
import { TRANSLOCO_SCOPE } from '@jsverse/transloco';

@NgModule({
  providers: [
    {
      provide: TRANSLOCO_SCOPE,
      useValue: 'booking',   // loads assets/i18n/booking/pl.json on demand
    },
  ],
})
export class BookingModule {}
```

Then in templates inside this module:
```html
<ng-container *transloco="let t; read: 'booking'">
  <h1>{{ t('title') }}</h1>
</ng-container>
```

### Where to Replace Hardcoded Polish Strings

The current Next.js frontend has Polish text hardcoded in JSX (e.g., `"Zaloguj się"`, `"Znajdź psychologa"`). In Angular, **never hardcode visible text in templates**. Always use a translation key:

| Hardcoded (bad) | With Transloco (good) |
|---|---|
| `<h1>Zaloguj się</h1>` | `<h1>{{ 'nav.login' \| transloco }}</h1>` |
| `<button>Zapisz</button>` | `<button>{{ 'common.save' \| transloco }}</button>` |
| `<p>Oczekujący</p>` | `<p>{{ 'status.PENDING' \| transloco }}</p>` |

This also eliminates the need for the custom `status-label.pipe.ts` — just use translation keys for status values directly.

---

## Step 4: Keycloak Integration (Replaces AWS Amplify)

### Theory: How Keycloak Works with Angular
1. Angular app initializes → Keycloak checks if user is logged in (silent SSO)
2. If not logged in → redirect to Keycloak login page (your custom branded page)
3. After login → Keycloak redirects back to Angular with an authorization code
4. Angular exchanges the code for a JWT access token
5. JWT is stored in memory (not localStorage — safer)
6. `keycloak-angular`'s `KeycloakService.getToken()` always returns a valid, refreshed token

### Keycloak Init Factory
```typescript
// src/app/core/auth/keycloak-init.factory.ts
import { KeycloakService } from 'keycloak-angular';
import { environment } from '../../../environments/environment';

export function initializeKeycloak(keycloak: KeycloakService) {
  return () =>
    keycloak.init({
      config: {
        url: environment.keycloak.url,
        realm: environment.keycloak.realm,
        clientId: environment.keycloak.clientId,
      },
      initOptions: {
        onLoad: 'check-sso',
        // silent-check-sso.html in /assets prevents full page reload on every route
        silentCheckSsoRedirectUri:
          window.location.origin + '/assets/silent-check-sso.html',
        checkLoginIframe: false,
      },
      // These routes bypass the JWT interceptor
      bearerExcludedUrls: [
        '/assets',
        `${environment.apiGatewayUrl}/api/psychologists`,    // Public browse page
      ],
    });
}
```

### App Module Setup
```typescript
// src/app/app.module.ts
@NgModule({
  imports: [
    BrowserModule,
    BrowserAnimationsModule,
    KeycloakAngularModule,        // Registers KeycloakService globally
    HttpClientModule,
    AppRoutingModule,
    CoreModule,
    NgxsModule.forRoot([AuthState, PsychologistsState, AppointmentsState], {
      developmentMode: !environment.production,
    }),
    NgxsLoggerPluginModule.forRoot(),
    NgxsRouterPluginModule.forRoot(),
    NgxsFormPluginModule.forRoot(),
  ],
  providers: [
    {
      provide: APP_INITIALIZER,   // Runs BEFORE app bootstrap — ensures Keycloak ready
      useFactory: initializeKeycloak,
      deps: [KeycloakService],
      multi: true,
    },
    { provide: HTTP_INTERCEPTORS, useClass: JwtInterceptor, multi: true },
    { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor, multi: true },
  ],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

### Auth Service (Keycloak Wrapper)
```typescript
// src/app/core/auth/auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  constructor(private keycloak: KeycloakService) {}

  isLoggedIn(): boolean { return this.keycloak.isLoggedIn(); }

  async getUserProfile(): Promise<KeycloakProfile> {
    return this.keycloak.loadUserProfile();
  }

  async getToken(): Promise<string> { return this.keycloak.getToken(); }

  // Replaces: custom:role claim from Cognito
  getUserRoles(): string[] { return this.keycloak.getUserRoles(); }

  hasRole(role: string): boolean { return this.keycloak.isUserInRole(role); }

  logout(): void { this.keycloak.logout(window.location.origin); }

  login(): void { this.keycloak.login(); }
}
```

### JWT Interceptor (Replaces RTK Query `prepareHeaders`)
```typescript
// src/app/core/interceptors/jwt.interceptor.ts
@Injectable()
export class JwtInterceptor implements HttpInterceptor {
  constructor(private keycloak: KeycloakService) {}

  intercept(req: HttpRequest<unknown>, next: HttpHandler): Observable<any> {
    // getToken() auto-refreshes expired tokens — Keycloak handles this transparently
    return from(this.keycloak.getToken()).pipe(
      switchMap(token => {
        const authReq = token
          ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
          : req;
        return next.handle(authReq);
      })
    );
  }
}
```

### Auth Guard (Replaces AWS Amplify route protection)
```typescript
// src/app/core/auth/auth.guard.ts
@Injectable({ providedIn: 'root' })
export class AuthGuard extends KeycloakAuthGuard {
  constructor(protected router: Router, protected keycloak: KeycloakService) {
    super(router, keycloak);
  }

  async isAccessAllowed(route: ActivatedRouteSnapshot): Promise<boolean | UrlTree> {
    if (!this.authenticated) {
      await this.keycloak.login({ redirectUri: window.location.href });
      return false;
    }

    const requiredRoles: string[] = route.data['roles'] ?? [];
    if (requiredRoles.length === 0) return true;

    const hasRole = requiredRoles.some(role => this.roles.includes(role));
    return hasRole ? true : this.router.parseUrl('/unauthorized');
  }
}
```

---

## Step 5: Routing Configuration

```typescript
// src/app/app-routing.module.ts
const routes: Routes = [
  { path: '', component: HomeComponent },
  {
    path: 'psychologists',
    loadChildren: () =>
      import('./features/psychologists/psychologists.module').then(m => m.PsychologistsModule),
    // No guard — public browsing
  },
  {
    path: 'booking',
    loadChildren: () =>
      import('./features/booking/booking.module').then(m => m.BookingModule),
    canActivate: [AuthGuard],
    data: { roles: ['client'] },
  },
  {
    path: 'dashboard/client',
    loadChildren: () =>
      import('./features/dashboard-client/dashboard-client.module').then(m => m.DashboardClientModule),
    canActivate: [AuthGuard],
    data: { roles: ['client'] },
  },
  {
    path: 'dashboard/psychologist',
    loadChildren: () =>
      import('./features/dashboard-psychologist/dashboard-psychologist.module').then(m => m.DashboardPsychologistModule),
    canActivate: [AuthGuard],
    data: { roles: ['psychologist'] },
  },
  {
    path: 'payment',
    loadChildren: () =>
      import('./features/payment/payment.module').then(m => m.PaymentModule),
    canActivate: [AuthGuard],
    data: { roles: ['client'] },
  },
  { path: 'unauthorized', component: UnauthorizedComponent },
  { path: '**', redirectTo: '' },
];
```

---

## Step 6: NGXS State (Replaces Redux Toolkit)

### Theory: NGXS vs NgRx vs Redux Toolkit

NGXS is Angular's simplest Redux-style state management library. Compared to NgRx it has **much less boilerplate** — Actions, State, and Selectors all live in a single file per domain.

| Concept | Redux Toolkit | NgRx | NGXS |
|---|---|---|---|
| State definition | `createSlice` | separate reducer file | `@State` decorated class |
| Actions | `createAction` | separate actions file | plain classes with `type` property |
| Side effects | `createAsyncThunk` | `createEffect` in effects file | `@Action` methods in the state class |
| Selectors | `createSelector` | `createSelector` in selectors file | `@Selector` static methods in the state class |
| Boilerplate | medium | high | **low** |

**Key insight:** In NGXS, the state class **is** the reducer, effects, and selectors all in one. HTTP calls go directly inside `@Action` methods, not in separate effects files.

### AppModule Setup (NGXS Providers)

```typescript
// src/app/app.module.ts
import { NgxsModule } from '@ngxs/store';
import { NgxsLoggerPluginModule } from '@ngxs/logger-plugin';
import { NgxsRouterPluginModule } from '@ngxs/router-plugin';
import { NgxsFormPluginModule } from '@ngxs/form-plugin';
import { AuthState } from './store/auth/auth.state';
import { PsychologistsState } from './store/psychologists/psychologists.state';
import { AppointmentsState } from './store/appointments/appointments.state';

@NgModule({
  imports: [
    // ...
    NgxsModule.forRoot(
      [AuthState, PsychologistsState, AppointmentsState],
      { developmentMode: !environment.production }
    ),
    NgxsLoggerPluginModule.forRoot(),   // logs every dispatched action to the console
    NgxsRouterPluginModule.forRoot(),   // syncs router URL into the store
    NgxsFormPluginModule.forRoot(),     // two-way binding of forms to store state
  ],
})
export class AppModule {}
```

### Example: Psychologists State (Single File)

```typescript
// store/psychologists/psychologists.state.ts
import { Injectable } from '@angular/core';
import { Action, Selector, State, StateContext } from '@ngxs/store';
import { tap, catchError, of } from 'rxjs';
import { Psychologist } from '../../shared/models/psychologist.model';
import { PsychologistService } from '../../features/psychologists/services/psychologist.service';

// --- ACTIONS (defined in the same file, or a companion .actions.ts file) ---

export class LoadPsychologists {
  static readonly type = '[Psychologists] Load';
  constructor(public specialization?: string) {}
}

export class SelectPsychologist {
  static readonly type = '[Psychologists] Select';
  constructor(public id: string) {}
}

// --- STATE MODEL ---

export interface PsychologistsStateModel {
  items: Psychologist[];
  loading: boolean;
  error: string | null;
  selectedId: string | null;
}

// --- STATE CLASS (replaces reducer + effects + selectors) ---

@State<PsychologistsStateModel>({
  name: 'psychologists',
  defaults: {
    items: [],
    loading: false,
    error: null,
    selectedId: null,
  },
})
@Injectable()
export class PsychologistsState {

  constructor(private psychologistService: PsychologistService) {}

  // --- SELECTORS (replaces createSelector) ---

  @Selector()
  static items(state: PsychologistsStateModel): Psychologist[] {
    return state.items;
  }

  @Selector()
  static loading(state: PsychologistsStateModel): boolean {
    return state.loading;
  }

  @Selector()
  static selected(state: PsychologistsStateModel): Psychologist | undefined {
    return state.items.find(p => p.keycloakId === state.selectedId);
  }

  // --- ACTIONS / EFFECTS (replaces createEffect + reducer combined) ---

  // This replaces both: Redux Toolkit's createAsyncThunk AND the reducer cases
  @Action(LoadPsychologists)
  loadPsychologists(
    ctx: StateContext<PsychologistsStateModel>,
    action: LoadPsychologists
  ) {
    ctx.patchState({ loading: true, error: null });

    return this.psychologistService.getAll(action.specialization).pipe(
      tap(items => ctx.patchState({ items, loading: false })),
      catchError(err => {
        ctx.patchState({ loading: false, error: err.message });
        return of(null);
      })
    );
  }

  @Action(SelectPsychologist)
  selectPsychologist(
    ctx: StateContext<PsychologistsStateModel>,
    action: SelectPsychologist
  ) {
    ctx.patchState({ selectedId: action.id });
  }
}
```

### Using NGXS in a Component

```typescript
// features/psychologists/browse/browse.component.ts
import { Store, Select } from '@ngxs/store';
import { Observable } from 'rxjs';
import { PsychologistsState, LoadPsychologists, SelectPsychologist } from '../../../store/psychologists/psychologists.state';

@Component({
  selector: 'app-browse',
  templateUrl: './browse.component.html',
})
export class BrowseComponent implements OnInit {

  // @Select replaces useSelector() from Redux Toolkit
  @Select(PsychologistsState.items) psychologists$!: Observable<Psychologist[]>;
  @Select(PsychologistsState.loading) loading$!: Observable<boolean>;
  @Select(PsychologistsState.selected) selected$!: Observable<Psychologist | undefined>;

  constructor(private store: Store) {}

  ngOnInit(): void {
    // dispatch replaces useDispatch() + calling an action creator
    this.store.dispatch(new LoadPsychologists());
  }

  onCardClick(id: string): void {
    this.store.dispatch(new SelectPsychologist(id));
  }
}
```

```html
<!-- browse.component.html -->
<div *ngIf="loading$ | async">
  <mat-spinner></mat-spinner>
</div>

<div class="psychologist-grid">
  <app-psychologist-card
    *ngFor="let p of psychologists$ | async"
    [psychologist]="p"
    (click)="onCardClick(p.keycloakId)">
  </app-psychologist-card>
</div>
```

### NGXS Form Plugin (Two-Way Form Binding)

The `@ngxs/form-plugin` binds Angular reactive forms directly to NGXS state — no manual `setValue` or `patchValue` needed.

```typescript
// In state: declare a form model
@State<AccountSettingsStateModel>({
  name: 'accountSettings',
  defaults: {
    profileForm: {                        // matches your FormGroup control names
      model: { name: '', phoneNumber: '', location: '' },
      dirty: false,
      status: '',
      errors: {},
    },
  },
})
```

```html
<!-- In template: bind form to state path via [ngxsForm] directive -->
<form [formGroup]="profileForm"
      ngxsForm="accountSettings.profileForm"
      (ngSubmit)="save()">
  <mat-form-field>
    <input matInput formControlName="name" placeholder="Imię i nazwisko" />
  </mat-form-field>
  <mat-form-field>
    <input matInput formControlName="phoneNumber"
           [mask]="'+00 000 000 000'"       />  <!-- ngx-mask -->
  </mat-form-field>
  <button mat-raised-button type="submit">{{ 'common.save' | transloco }}</button>
</form>
```

The form state is automatically synced to NGXS store on every keystroke — no manual dispatch needed for form value changes.

---

## Step 7: HTTP Services (Replaces RTK Query `state/api.ts`)

```typescript
// features/psychologists/services/psychologist.service.ts
@Injectable({ providedIn: 'root' })
export class PsychologistService {
  private readonly baseUrl = `${environment.apiGatewayUrl}/api/psychologists`;

  constructor(private http: HttpClient) {}

  // Replaces: useGetAllPsychologistsQuery
  getAll(specialization?: string): Observable<Psychologist[]> {
    const params = specialization
      ? new HttpParams().set('specialization', specialization)
      : new HttpParams();
    return this.http.get<Psychologist[]>(this.baseUrl, { params });
  }

  // Replaces: useGetPsychologistQuery
  getById(keycloakId: string): Observable<Psychologist> {
    return this.http.get<Psychologist>(`${this.baseUrl}/${keycloakId}`);
  }

  // Replaces: useUpdatePsychologistMutation
  update(keycloakId: string, data: Partial<Psychologist>): Observable<Psychologist> {
    return this.http.put<Psychologist>(`${this.baseUrl}/${keycloakId}`, data);
  }
}
```

---

## Step 8: Shared Models (Replaces `prismaTypes.d.ts`)

```typescript
// shared/models/psychologist.model.ts
export interface Psychologist {
  id: string;
  keycloakId: string;
  name: string;
  email: string;
  phoneNumber: string;
  location: string;
  coordinates: { latitude: number; longitude: number; };
  hourlyRate: number;
  description: string;
  specializations: string[];   // Array (not single string like in Prisma)
  profileImageUrl?: string;
  isVerified: boolean;
  averageRating: number;
  totalRatings: number;
}

// shared/models/appointment.model.ts
export type AppointmentStatus = 'PENDING' | 'APPROVED' | 'CANCELLED' | 'COMPLETED';
export type PaymentStatus = 'UNPAID' | 'PAID' | 'REFUNDED';

export interface Appointment {
  id: string;
  clientKeycloakId: string;
  psychologistKeycloakId: string;
  clientName: string;
  psychologistName: string;
  scheduledAt: string;         // ISO-8601
  durationMinutes: number;
  meetingLink?: string;
  status: AppointmentStatus;
  paymentStatus: PaymentStatus;
}
```

---

## Step 9: Google Maps Integration

```typescript
// shared/components/map/map.component.ts
@Component({
  selector: 'app-map',
  template: '<div #mapContainer id="map" style="width:100%;height:500px;"></div>',
})
export class MapComponent implements OnInit {
  @Input() psychologists: Psychologist[] = [];
  @Output() psychologistSelected = new EventEmitter<Psychologist>();

  async ngOnInit(): Promise<void> {
    const loader = new Loader({
      apiKey: environment.googleMapsApiKey,
      version: 'weekly',
      libraries: ['places'],
    });

    const { Map, Marker } = await loader.importLibrary('maps');
    const map = new Map(document.getElementById('map')!, {
      center: { lat: 52.2297, lng: 21.0122 },  // Warsaw default center
      zoom: 12,
    });

    this.psychologists.forEach(p => {
      const marker = new Marker({
        map,
        position: { lat: p.coordinates.latitude, lng: p.coordinates.longitude },
        title: p.name,
      });
      marker.addListener('click', () => this.psychologistSelected.emit(p));
    });
  }
}
```

---

## Step 10: Stripe Payment Checkout

```typescript
// features/payment/checkout/checkout.component.ts
@Component({ selector: 'app-checkout', templateUrl: './checkout.component.html' })
export class CheckoutComponent implements OnInit {
  private stripe: Stripe | null = null;
  private elements: StripeElements | null = null;
  loading = false;
  errorMessage = '';

  @Input() appointmentId!: string;
  @Input() amountGroszy!: number;  // amount in Polish groszy (1 zł = 100 groszy)

  constructor(private paymentService: PaymentService) {}

  async ngOnInit(): Promise<void> {
    this.stripe = await loadStripe(environment.stripePublishableKey);

    // Step 1: Get clientSecret from Payment Service
    // Payment Service creates a Stripe PaymentIntent on its side
    const { clientSecret } = await firstValueFrom(
      this.paymentService.createPaymentIntent(this.appointmentId, this.amountGroszy)
    );

    // Step 2: Mount Stripe Elements — card data goes from browser directly to Stripe
    // Your server never sees the raw card number
    this.elements = this.stripe!.elements({ clientSecret });
    const paymentElement = this.elements.create('payment');
    paymentElement.mount('#payment-element');
  }

  async confirmPayment(): Promise<void> {
    this.loading = true;
    const { error } = await this.stripe!.confirmPayment({
      elements: this.elements!,
      confirmParams: {
        return_url: `${window.location.origin}/payment/success`,
      },
    });
    if (error) this.errorMessage = error.message ?? 'Błąd płatności';
    this.loading = false;
  }
}
```

```html
<!-- features/payment/checkout/checkout.component.html -->
<mat-card class="checkout-card">
  <mat-card-header>
    <mat-card-title>Płatność za wizytę</mat-card-title>
  </mat-card-header>
  <mat-card-content>
    <div id="payment-element"></div>
    <mat-error *ngIf="errorMessage">{{ errorMessage }}</mat-error>
  </mat-card-content>
  <mat-card-actions>
    <button mat-raised-button color="primary"
            (click)="confirmPayment()"
            [disabled]="loading">
      <mat-spinner diameter="20" *ngIf="loading"></mat-spinner>
      <span *ngIf="!loading">Zapłać</span>
    </button>
  </mat-card-actions>
</mat-card>
```

---

## Step 11: Docker Build

```dockerfile
# Dockerfile
# Stage 1: Build Angular app
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --legacy-peer-deps
COPY . .
RUN npm run build -- --configuration=production

# Stage 2: Serve with Nginx
FROM nginx:1.25-alpine
COPY --from=builder /app/dist/theralink-frontend/browser /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

```nginx
# nginx.conf — needed for Angular Router (HTML5 pushState navigation)
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # Angular handles all routing — unknown paths fall back to index.html
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|ico|svg|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

## Migration Execution Order (Feature by Feature)

| Phase | Features | Dependencies |
|---|---|---|
| 1 | Scaffold + Keycloak auth + JWT interceptor + auth guard | Keycloak running locally |
| 2 | Shared models (TypeScript interfaces matching Spring Boot DTOs) | User Service API contract |
| 3 | Browse psychologists (`/psychologists`) — public, no auth | User Service + Gateway |
| 4 | Booking flow (`/booking/confirm`) — first authenticated feature | Appointment Service |
| 5 | Client dashboard — history, billings, account settings | All client-facing services |
| 6 | Psychologist dashboard — availability, calendar, appointments | Psychologist Service |
| 7 | Payment checkout — Stripe Elements | Payment Service |
| 8 | Role upgrade flow — client selects "become psychologist" | Auth + User Service |

---

## Verification Checklist

| Check | How |
|---|---|
| Keycloak auth | Login button → Keycloak page → back to Angular with JWT |
| JWT in requests | Network tab: all `/api/*` requests have `Authorization: Bearer ...` |
| Route guard | `/dashboard/client` without login → redirected to Keycloak |
| Role guard | Psychologist at `/dashboard/client` → `/unauthorized` |
| Browse page | `GET /api/psychologists` returns list; map markers visible |
| Book appointment | `POST /api/appointments` → new record in MongoDB |
| NGXS DevTools | Redux DevTools extension shows dispatched actions + state (NgxsLoggerPlugin logs to console) |
| Payment | Stripe test card `4242 4242 4242 4242` → webhook → `paymentStatus = PAID` |
| Prod build | `ng build --configuration=production` — no errors |
| Docker | `docker build . && docker run -p 4200:80` — app served at :4200 |

---

## Powiązane

- [[index]] — Mapa dokumentacji
- [[backend-migration]] — Spring Boot API z którym łączy się Angular
- [[keycloak-setup]] — Keycloak auth (keycloak-angular, APP_INITIALIZER, guard)
- [[angular-style-guide]] — Konwencje kodu Angular dla TheraLink
- [[typescript-style-guide]] — Standardy TypeScript
