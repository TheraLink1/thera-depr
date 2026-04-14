# Angular Style Guide — TheraLink Reference

> Source: https://angular.dev/style-guide
> This document captures the Angular team's official style conventions applied to TheraLink.

---

## Core Principle

> "Whenever you encounter a situation in which these rules contradict the style of a particular file, **prioritize maintaining consistency within a file**."
>
> "When in doubt, go with the approach that leads to **smaller files**."

---

## 1. File Naming

| What | Convention | Example |
|---|---|---|
| Separate words | hyphens | `user-profile.ts` |
| Component | `feature.component.ts` | `psychologist-card.component.ts` |
| Service | `feature.service.ts` | `psychologist.service.ts` |
| State (NGXS) | `feature.state.ts` | `psychologists.state.ts` |
| Guard | `feature.guard.ts` | `auth.guard.ts` |
| Interceptor | `feature.interceptor.ts` | `jwt.interceptor.ts` |
| Module | `feature.module.ts` | `dashboard-client.module.ts` |
| Template | same base as `.ts` | `psychologist-card.component.html` |
| Style | same base as `.ts` | `psychologist-card.component.scss` |
| Test | `.spec.ts` suffix | `auth.service.spec.ts` |
| Avoid | generic names | ~~`helpers.ts`~~, ~~`utils.ts`~~ |

---

## 2. Project Organization

```
src/app/
├── core/          ← Singleton services, guards, interceptors (imported ONCE in AppModule)
├── shared/        ← Reusable components, models, pipes (imported in every feature module)
└── features/      ← Lazy-loaded feature modules (one folder per domain)
    ├── psychologists/
    ├── booking/
    ├── dashboard-client/
    └── dashboard-psychologist/
```

**Rules:**
- Group **by feature area**, not by file type (avoid `components/`, `services/`, `directives/` top-level folders)
- Each file focuses on a **single concept** (one component, one service, one state class per file)
- Feature modules are **lazy-loaded** via `loadChildren`

---

## 3. Naming Conventions

### Classes
```typescript
// Components
export class PsychologistCardComponent { }
export class DashboardClientComponent { }

// Services
export class PsychologistService { }
export class AuthService { }

// NGXS State
export class PsychologistsState { }
export class AppointmentsState { }

// Guards
export class AuthGuard { }

// Models (interfaces, not classes)
export interface Psychologist { }
export interface Appointment { }
```

### Files match class names (kebab-case)
```
PsychologistCardComponent  →  psychologist-card.component.ts
PsychologistService        →  psychologist.service.ts
PsychologistsState         →  psychologists.state.ts
AuthGuard                  →  auth.guard.ts
```

### CSS Selectors
```typescript
// Components — use 'app-' prefix
@Component({ selector: 'app-psychologist-card' })

// Directives — use 'app' prefix + camelCase attribute
@Directive({ selector: '[appHighlight]' })
```

---

## 4. Dependency Injection

Prefer `inject()` over constructor injection when many dependencies exist:

```typescript
// PREFERRED (Angular 14+ inject() function)
@Component({ ... })
export class BrowseComponent {
  private store = inject(Store);
  private router = inject(Router);
  private transloco = inject(TranslocoService);
}

// ALSO FINE for simpler classes
@Injectable({ providedIn: 'root' })
export class PsychologistService {
  constructor(private http: HttpClient) {}
}
```

---

## 5. Components

### Access Modifiers for Templates
```typescript
@Component({ ... })
export class PsychologistCardComponent {
  // Public: accessed from parent via @Input / @Output
  @Input() readonly psychologist!: Psychologist;
  @Output() readonly selected = new EventEmitter<Psychologist>();

  // Protected: used in THIS component's template only (not public API)
  protected formatRate(rate: number): string {
    return `${rate} zł/h`;
  }

  // Private: pure internal logic
  private calculateRating(): number { ... }
}
```

### Property Organization (top to bottom)
```typescript
@Component({ ... })
export class ExampleComponent implements OnInit {
  // 1. Injected dependencies (inject() or constructor)
  private store = inject(Store);

  // 2. Inputs
  @Input() readonly data!: SomeModel;

  // 3. Outputs
  @Output() readonly action = new EventEmitter<void>();

  // 4. ViewChild / ContentChild queries
  @ViewChild('container') containerRef!: ElementRef;

  // 5. NGXS Selects
  @Select(SomeState.items) items$!: Observable<SomeModel[]>;

  // 6. Component state (signals or properties)
  protected loading = false;
  protected errorMessage = '';

  // 7. Lifecycle methods
  ngOnInit(): void { }

  // 8. Event handlers (named for WHAT they do, not what triggered them)
  protected saveProfile(): void { }      // NOT: onSaveButtonClick()
  protected loadData(): void { }         // NOT: handleFetchData()

  // 9. Private helpers
  private formatDate(date: string): string { ... }
}
```

### Event Handler Naming
```typescript
// CORRECT — named for the action performed
protected saveUserData(): void { }
protected navigateToDashboard(): void { }
protected confirmBooking(): void { }

// AVOID — named for the event
// protected handleClick(): void { }
// protected onButtonPress(): void { }
```

### Template Expressions
```html
<!-- GOOD — simple, readable expression -->
<p>{{ psychologist.hourlyRate | currency:'PLN' }}</p>

<!-- GOOD — use class binding instead of NgClass for simple cases -->
<div [class.active]="isSelected">

<!-- AVOID — complex logic in templates; move to TypeScript -->
<!-- <p>{{ items.filter(i => i.status === 'APPROVED').length }}</p> -->
```

### Lifecycle Hooks
```typescript
// GOOD — implement the interface to catch typos at compile time
export class CalendarComponent implements OnInit, OnDestroy {
  ngOnInit(): void {
    this.loadData();    // delegate to named method
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }

  private loadData(): void {
    this.store.dispatch(new LoadAppointments());
  }
}
```

---

## 6. Services

```typescript
// GOOD — providedIn: 'root' for app-wide singletons
@Injectable({ providedIn: 'root' })
export class PsychologistService {
  private readonly baseUrl = `${environment.apiGatewayUrl}/api/psychologists`;

  constructor(private http: HttpClient) {}

  getAll(): Observable<Psychologist[]> {
    return this.http.get<Psychologist[]>(this.baseUrl);
  }
}
```

---

## 7. Interfaces for Models

```typescript
// GOOD — use interface, not class, for data shapes
export interface Psychologist {
  keycloakId: string;
  name: string;
  specializations: string[];
}

// AVOID
// export class Psychologist { constructor(public name: string) {} }
```

---

## 8. Modules

```typescript
// Feature module — lazy-loaded
@NgModule({
  declarations: [BrowseComponent, PsychologistCardComponent],
  imports: [
    CommonModule,
    SharedModule,
    PsychologistsRoutingModule,
    TranslocoModule,
  ],
  providers: [
    { provide: TRANSLOCO_SCOPE, useValue: 'psychologists' },
  ],
})
export class PsychologistsModule {}
```

---

## 9. TheraLink-Specific Rules

| Rule | Detail |
|---|---|
| Selector prefix | `app-` for all components |
| State file | NGXS state, actions, and selectors in **one file** per domain |
| Translation | **Never hardcode Polish strings** in templates — use `'key' \| transloco` |
| API calls | Only in `*.service.ts` files — never directly in components or state |
| `keycloakId` | Use everywhere instead of `cognitoId` (migration) |
| Colors | Primary `#2b6369`, secondary `#224f54`, accent `#00bfa5` — define as SCSS vars |
| Date formatting | Always use `date-fns` functions — never manual string slicing |
