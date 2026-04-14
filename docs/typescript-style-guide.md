# Google TypeScript Style Guide — TheraLink Reference

> Source: https://google.github.io/styleguide/tsguide.html
> Applied to all TypeScript files in the TheraLink project (Angular frontend + Spring Boot is Java).

---

## 1. File Structure Order

Every `.ts` file must follow this order, each section separated by one blank line:

```
1. Copyright (if applicable)
2. @fileoverview JSDoc (if applicable)
3. Imports
4. Implementation
```

---

## 2. Imports & Exports

### Named exports — always preferred
```typescript
// GOOD
export class PsychologistCardComponent { }
export interface Psychologist { }
export const API_URL = '...';

// AVOID
export default class PsychologistCardComponent { }
```

### Import style
```typescript
// Named imports — preferred for specific symbols
import { Component, OnInit } from '@angular/core';
import { Store, Select } from '@ngxs/store';
import { format, parseISO } from 'date-fns';

// Namespace import — when accessing many symbols from a large API
import * as dateFns from 'date-fns';

// Type-only imports — when the import is only used as a type
import type { Psychologist } from '../models/psychologist.model';
import { Component } from '@angular/core';   // value import stays normal
```

### Relative paths for internal files
```typescript
// GOOD — relative path within the project
import { Psychologist } from '../../shared/models/psychologist.model';
import { environment } from '../../../environments/environment';

// AVOID — absolute path for own project files
// import { Psychologist } from 'src/app/shared/models/psychologist.model';
```

### No mutable exports
```typescript
// AVOID
export let currentUser: User;

// GOOD — use a getter or a BehaviorSubject
export function getCurrentUser(): User { return _user; }
```

---

## 3. Variables & Declarations

```typescript
// Use const by default
const psychologists: Psychologist[] = [];
const baseUrl = environment.apiGatewayUrl;

// Use let only when reassignment is needed
let selectedPsychologist: Psychologist | null = null;
let currentPage = 1;

// Never use var
// var count = 0;  // FORBIDDEN
```

One variable per declaration:
```typescript
// GOOD
const a = 1;
const b = 2;

// AVOID
const a = 1, b = 2;
```

---

## 4. Classes

### Class declarations — no trailing semicolon
```typescript
// GOOD
export class PsychologistService { }

// Class expression in statement — semicolon required
export const PsychologistService = class { };
```

### Constructors — always use `new`
```typescript
const service = new PsychologistService();   // always parentheses
```

### Parameter properties (shorthand)
```typescript
// GOOD — TypeScript shorthand
@Injectable({ providedIn: 'root' })
export class PsychologistService {
  constructor(private readonly http: HttpClient) {}
}

// AVOID — verbose equivalent
export class PsychologistService {
  private readonly http: HttpClient;
  constructor(http: HttpClient) { this.http = http; }
}
```

### Access modifiers
```typescript
// private — internal only
private destroy$ = new Subject<void>();

// protected — component template access
protected loading = false;
protected psychologists: Psychologist[] = [];

// readonly — never reassigned after construction
private readonly baseUrl = environment.apiGatewayUrl;
@Input() readonly psychologist!: Psychologist;

// NEVER use 'public' keyword (it is the default)
// public name: string;  // AVOID
```

### No private fields (`#`)
```typescript
// AVOID — ECMAScript private fields
#name = '';

// GOOD — TypeScript private
private name = '';
```

### Readonly for props never reassigned
```typescript
@Injectable()
export class AppointmentService {
  private readonly baseUrl = `${environment.apiGatewayUrl}/api/appointments`;
}
```

---

## 5. Functions

### Named functions — prefer declarations over expressions
```typescript
// GOOD — function declaration
function formatDate(date: string): string {
  return format(parseISO(date), 'dd.MM.yyyy');
}

// AVOID — function expression assigned to variable
// const formatDate = function(date: string): string { ... };
```

### Arrow functions — for inline callbacks and class methods needing `this`
```typescript
// GOOD — arrow for callbacks
const approved = appointments.filter(a => a.status === 'APPROVED');

// GOOD — arrow function body when return value is used
const names = psychologists.map(p => p.name);

// GOOD — block body when return value is NOT used
this.appointments$.subscribe(items => {
  this.displayItems = items;
});
```

### Rest parameters — prefer over `arguments`
```typescript
// GOOD
function logAll(...messages: string[]): void {
  messages.forEach(m => console.log(m));
}
```

### Default parameter values — no side effects
```typescript
// GOOD — simple default
function createSlot(date: string, duration = 60): TimeSlot { }

// AVOID — side effect in default
// function fetch(url: string, cache = loadCache()) { }
```

---

## 6. Control Flow

### Always use braces
```typescript
// GOOD
if (isLoggedIn) {
  navigateToDashboard();
}

// AVOID
if (isLoggedIn) navigateToDashboard();
```

### Prefer `for...of` for arrays
```typescript
// GOOD
for (const psychologist of psychologists) {
  process(psychologist);
}

// AVOID
for (let i = 0; i < psychologists.length; i++) { }
```

### Always `===` and `!==`
```typescript
// GOOD
if (status === 'APPROVED') { }
if (value !== null) { }

// Exception — null check covers both null and undefined
if (value == null) { }   // checks for null OR undefined
```

### Exception handling
```typescript
// Always throw Error instances
throw new Error('Appointment not found');

// Always use new
throw new Error('...');   // NOT: throw 'message';

// Type unknown in catch
try {
  await this.appointmentService.create(data);
} catch (e: unknown) {
  if (e instanceof Error) {
    this.errorMessage = e.message;
  }
}

// Never leave empty catch without a comment
try { ... } catch (e: unknown) {
  // Intentionally ignored — Keycloak silent SSO may fail silently
}
```

---

## 7. Types

### Prefer type inference — omit trivial annotations
```typescript
// GOOD — type inferred
const loading = false;
const name = 'Jan Kowalski';

// GOOD — annotation needed for generics
const psychologists: Psychologist[] = [];
const subject = new BehaviorSubject<Psychologist | null>(null);
```

### Use `interface` for object shapes
```typescript
// GOOD
export interface Psychologist {
  keycloakId: string;
  name: string;
  specializations: string[];
}

// AVOID — type alias for objects
// type Psychologist = { keycloakId: string; ... };
```

### Array types — simple syntax for simple types
```typescript
// GOOD — simple type
let ids: string[];
let amounts: number[];

// GOOD — complex type uses generic form
let data: Array<{ id: string; amount: number }>;
```

### Avoid `any`
```typescript
// AVOID
const data: any = response;

// GOOD — use unknown and narrow
const data: unknown = response;
if (typeof data === 'object' && data !== null) {
  // safe to use
}

// GOOD — or use specific type
const data: Psychologist = response as Psychologist;
```

### Avoid `{}` type
```typescript
// AVOID
const options: {} = {};

// GOOD
const options: Record<string, string> = {};
const options: object = {};         // excludes primitives
const options: unknown = value;     // truly unknown
```

### Nullable fields — use optional `?` not `| undefined`
```typescript
// GOOD
export interface Psychologist {
  profileImageUrl?: string;
  notes?: string;
}

// AVOID
export interface Psychologist {
  profileImageUrl: string | undefined;
}
```

### Type assertions — use sparingly with comments
```typescript
// Add comment explaining why
const el = document.getElementById('map') as HTMLElement;  // guaranteed to exist at this point

// Double assertion when TypeScript disagrees
(value as unknown as SpecificType).method();
```

---

## 8. Naming Conventions

| Symbol | Style | Example |
|---|---|---|
| Class | `UpperCamelCase` | `PsychologistCardComponent` |
| Interface | `UpperCamelCase` | `Psychologist`, `Appointment` |
| Enum | `UpperCamelCase` | `AppointmentStatus` |
| Enum value | `UPPER_CASE` | `AppointmentStatus.APPROVED` |
| Function | `lowerCamelCase` | `formatDate()` |
| Method | `lowerCamelCase` | `loadPsychologists()` |
| Variable | `lowerCamelCase` | `selectedId` |
| Constant (global) | `UPPER_CASE` | `MAX_RETRY_COUNT` |
| Observable | `$` suffix | `psychologists$`, `loading$` |
| Type parameter | Single letter or `UpperCamelCase` | `T`, `TState` |

### Descriptive names — no ambiguous abbreviations
```typescript
// GOOD
const psychologistKeycloakId = jwt.sub;
const appointmentScheduledAt = parseISO(dateString);

// AVOID
const id = jwt.sub;
const dt = parseISO(dateString);
```

### Acronyms — treat as whole words
```typescript
// GOOD
loadHttpUrl()
parseXmlDocument()
getApiResponse()

// AVOID
loadHTTPURL()
parseXMLDocument()
getAPIResponse()
```

---

## 9. Comments & Documentation

### JSDoc — for public API (exported functions, classes, interfaces)
```typescript
/**
 * Formats a date string into Polish locale format.
 * @param isoDate - ISO-8601 date string (e.g. "2026-03-10T14:00:00Z")
 * @returns Formatted string (e.g. "10.03.2026 14:00")
 */
export function formatAppointmentDate(isoDate: string): string {
  return format(parseISO(isoDate), 'dd.MM.yyyy HH:mm');
}
```

### Line comments — for implementation details
```typescript
// Keycloak uses 'sub' claim as the unique user identifier (replaces cognitoId)
const keycloakId = jwt.getSubject();

// Denormalize psychologist name at booking time to avoid cross-service lookups
const snapshot = { psychologistName: psychologist.name };
```

### Multi-line — use multiple `//` lines, not `/* */`
```typescript
// This interceptor attaches the Keycloak JWT to every outgoing request.
// Keycloak-Angular's getToken() automatically refreshes expired tokens,
// so this works transparently even after the token expires.
```

---

## 10. Disallowed Patterns (Never Use)

```typescript
// No var
var x = 1;

// No wrapper objects
new String('hello')
new Boolean(false)
new Number(42)

// No eval
eval('console.log(x)');

// No debugger in committed code
debugger;

// No with
with (obj) { }

// No modifying prototypes
Array.prototype.myMethod = () => {};

// No @ts-ignore / @ts-nocheck
// @ts-ignore
someInvalidCode();
```

---

## 11. TheraLink-Specific TypeScript Rules

```typescript
// 1. Always type HTTP responses explicitly
this.http.get<Psychologist[]>(url)        // NOT .get<any>(url)

// 2. Observable suffix for all Observable properties
psychologists$: Observable<Psychologist[]>;
loading$: Observable<boolean>;

// 3. Enums for status values (not plain strings)
export enum AppointmentStatus {
  PENDING = 'PENDING',
  APPROVED = 'APPROVED',
  CANCELLED = 'CANCELLED',
  COMPLETED = 'COMPLETED',
}

// 4. keycloakId — always string, never number
keycloakId: string;        // from JWT sub claim

// 5. Dates — always ISO-8601 strings from API, parse with date-fns
scheduledAt: string;       // "2026-03-10T14:00:00Z"
format(parseISO(apt.scheduledAt), 'dd.MM.yyyy')

// 6. Money — always store in groszy (smallest unit), display divided by 100
amount: number;            // 20000 = 200.00 zł
`${(amount / 100).toFixed(2)} zł`
```
