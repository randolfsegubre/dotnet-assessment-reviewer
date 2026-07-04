# Topic 17 — Angular (Senior Full-Stack .NET)

> Angular is the dominant frontend framework in enterprise .NET shops. TypeScript-first, opinionated, and deeply integrated with Microsoft tooling.

---

## Angular vs React vs Vue

| Feature | Angular | React | Vue |
|---|---|---|---|
| Type | Framework (batteries included) | Library (UI only) | Progressive framework |
| Language | TypeScript-first | JS/TS | JS/TS |
| Learning curve | Steep | Moderate | Easy |
| State management | Services + RxJS | External (Redux/Zustand) | Pinia (built-in) |
| Forms | Template-driven + Reactive | react-hook-form | VeeValidate |
| HTTP | HttpClient (built-in) | fetch/axios | fetch/axios |
| DI | Built-in DI container | Context/props | Vue provide/inject |
| Routing | Router (built-in) | React Router | Vue Router |
| Best for | Enterprise, large teams | SPAs, flexibility | Smaller apps, learning |
| Common with .NET | ✅ Very common | ✅ Most popular | ⚠️ Less common |

---

## Component Architecture

```typescript
// app.component.ts — standalone component (Angular 14+):
import { Component, OnInit, Input, Output, EventEmitter, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterOutlet } from '@angular/router';

@Component({
    selector: 'app-transaction-card',
    standalone: true,
    imports: [CommonModule], // declare dependencies here
    template: `
        <div class="card" [class.highlighted]="isHighlighted">
            <h3>{{ transaction.description }}</h3>
            <p [ngClass]="{'income': transaction.type === 'income', 'expense': transaction.type === 'expense'}">
                {{ transaction.amount | currency:'PHP':'symbol':'1.2-2' }}
            </p>
            <span>{{ transaction.date | date:'mediumDate' }}</span>
            <button (click)="onDelete()">Delete</button>
        </div>
    `,
    styles: [`
        .card { border: 1px solid #ccc; padding: 16px; border-radius: 8px; }
        .income  { color: green; }
        .expense { color: red; }
    `]
})
export class TransactionCardComponent {
    // Inputs (props):
    @Input({ required: true }) transaction!: Transaction;
    @Input() isHighlighted = false;
    
    // Outputs (events):
    @Output() delete = new EventEmitter<string>();
    
    onDelete() {
        this.delete.emit(this.transaction.id);
    }
}
```

---

## Services & Dependency Injection

```typescript
// Angular has its own DI container — similar to ASP.NET Core DI

@Injectable({ providedIn: 'root' }) // Singleton — one instance for entire app
export class TransactionService {
    private readonly http = inject(HttpClient); // functional injection
    private readonly apiUrl = inject(API_URL);  // injection token
    
    getAll(userId: string): Observable<Transaction[]> {
        return this.http.get<Transaction[]>(`${this.apiUrl}/transactions`, {
            params: { userId }
        });
    }
    
    create(dto: CreateTransactionDto): Observable<Transaction> {
        return this.http.post<Transaction>(`${this.apiUrl}/transactions`, dto);
    }
    
    delete(id: string): Observable<void> {
        return this.http.delete<void>(`${this.apiUrl}/transactions/${id}`);
    }
}

// Scoped to component tree (new instance per component):
@Injectable()
export class FormStateService { /* ... */ }

@Component({
    providers: [FormStateService] // new instance for this component + children
})
export class TransactionFormComponent { }

// Using inject() in constructor:
@Component({ /* ... */ })
export class DashboardComponent implements OnInit {
    private readonly txService = inject(TransactionService);
    private readonly router = inject(Router);
    
    transactions: Transaction[] = [];
    
    ngOnInit(): void {
        this.txService.getAll(this.userId).subscribe({
            next: (txs) => this.transactions = txs,
            error: (err) => console.error(err)
        });
    }
}
```

---

## RxJS Observables

Angular is built on RxJS. Understanding Observables is critical.

```typescript
import { Observable, Subject, BehaviorSubject, combineLatest, of, from } from 'rxjs';
import { map, filter, switchMap, catchError, takeUntilDestroyed, debounceTime, distinctUntilChanged } from 'rxjs/operators';

// Observable vs Promise:
// Observable — lazy, cancellable, can emit multiple values over time
// Promise — eager, not cancellable, one value

// Common operators:
@Component({ /* ... */ })
export class SearchComponent {
    private destroyRef = inject(DestroyRef);
    
    searchControl = new FormControl('');
    results$ = this.searchControl.valueChanges.pipe(
        debounceTime(300),            // wait 300ms after typing stops
        distinctUntilChanged(),        // ignore if value didn't change
        filter(term => term.length >= 2),
        switchMap(term =>              // cancel previous request, start new
            this.searchService.search(term).pipe(
                catchError(() => of([]))  // handle errors gracefully
            )
        ),
        takeUntilDestroyed(this.destroyRef) // auto-unsubscribe on destroy
    );
}

// BehaviorSubject — stateful observable with current value:
@Injectable({ providedIn: 'root' })
export class AuthStore {
    private _user$ = new BehaviorSubject<User | null>(null);
    
    readonly user$ = this._user$.asObservable(); // expose as read-only
    readonly isLoggedIn$ = this._user$.pipe(map(u => u !== null));
    
    setUser(user: User) { this._user$.next(user); }
    clearUser() { this._user$.next(null); }
    get currentUser() { return this._user$.getValue(); }
}

// combineLatest — combine multiple streams:
readonly dashboard$ = combineLatest({
    user: this.authStore.user$,
    transactions: this.txService.transactions$,
    budgets: this.budgetService.budgets$
}).pipe(
    map(({ user, transactions, budgets }) => ({
        totalIncome: transactions.filter(t => t.type === 'income').reduce((s, t) => s + t.amount, 0),
        // ... compute dashboard
    }))
);
```

---

## Angular Routing

```typescript
// app.routes.ts:
export const routes: Routes = [
    { path: '', redirectTo: '/dashboard', pathMatch: 'full' },
    { path: 'login', component: LoginComponent, canActivate: [noAuthGuard] },
    {
        path: 'dashboard',
        component: DashboardComponent,
        canActivate: [authGuard],
        title: 'Dashboard — BudgetPH'
    },
    {
        path: 'transactions',
        loadComponent: () => import('./transactions/transactions.component')
            .then(m => m.TransactionsComponent), // lazy loading
    },
    {
        path: 'admin',
        canActivate: [authGuard, adminGuard],
        loadChildren: () => import('./admin/admin.routes') // lazy module
            .then(m => m.ADMIN_ROUTES)
    },
    { path: 'transactions/:id', component: TransactionDetailComponent },
    { path: '**', component: NotFoundComponent }
];

// Functional guards (Angular 14+):
export const authGuard: CanActivateFn = (route, state) => {
    const auth = inject(AuthStore);
    const router = inject(Router);
    
    if (auth.currentUser) return true;
    return router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};

// Route resolver — preload data:
export const transactionResolver: ResolveFn<Transaction> = (route) => {
    return inject(TransactionService).getById(route.paramMap.get('id')!);
};
```

---

## Angular Forms

```typescript
// Reactive forms (recommended for complex forms):
@Component({ /* ... */ })
export class TransactionFormComponent {
    private fb = inject(FormBuilder);
    private txService = inject(TransactionService);
    
    form = this.fb.group({
        amount: [0, [Validators.required, Validators.min(0.01), Validators.max(1_000_000)]],
        description: ['', [Validators.required, Validators.maxLength(500)]],
        type: ['expense' as 'income' | 'expense', Validators.required],
        date: [new Date().toISOString().split('T')[0], Validators.required],
        categoryId: [null as string | null]
    });
    
    // Typed form access:
    get amount() { return this.form.controls.amount; }
    get amountError() {
        if (this.amount.hasError('required')) return 'Amount is required';
        if (this.amount.hasError('min')) return 'Amount must be greater than 0';
        return null;
    }
    
    submit() {
        if (this.form.invalid) return;
        
        this.txService.create(this.form.value as CreateTransactionDto).subscribe({
            next: () => this.form.reset(),
            error: (err) => console.error(err)
        });
    }
}
```

---

## Angular Signals (Modern Angular 16+)

```typescript
// Signals — simpler reactivity alternative to RxJS for state:
@Component({ /* ... */ })
export class CounterComponent {
    count = signal(0);                        // writable signal
    doubled = computed(() => this.count() * 2); // derived signal
    
    increment() { this.count.update(c => c + 1); }
    reset() { this.count.set(0); }
    
    // Effect — runs when signals change:
    constructor() {
        effect(() => {
            console.log(`Count changed to: ${this.count()}`);
        });
    }
}

// Signal-based inputs (Angular 17+):
@Component({ /* ... */ })
export class CardComponent {
    title = input.required<string>();   // required input signal
    subtitle = input('');               // optional with default
    clicked = output<void>();           // output signal
}
```

---

## HTTP Interceptors (Angular)

```typescript
// JWT interceptor — adds Authorization header to all requests:
export const authInterceptor: HttpInterceptorFn = (req, next) => {
    const auth = inject(AuthStore);
    const token = auth.currentUser?.token;
    
    if (token) {
        const authReq = req.clone({
            headers: req.headers.set('Authorization', `Bearer ${token}`)
        });
        return next(authReq);
    }
    
    return next(req);
};

// Error interceptor — handle 401 globally:
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
    const router = inject(Router);
    const auth = inject(AuthStore);
    
    return next(req).pipe(
        catchError((error: HttpErrorResponse) => {
            if (error.status === 401) {
                auth.clearUser();
                router.navigate(['/login']);
            }
            return throwError(() => error);
        })
    );
};

// Register in app.config.ts:
export const appConfig: ApplicationConfig = {
    providers: [
        provideRouter(routes),
        provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))
    ]
};
```

---

## Angular Pipes

```typescript
// Built-in pipes:
// {{ amount | currency:'PHP':'symbol':'1.2-2' }}   → ₱5,000.50
// {{ date | date:'fullDate' }}                      → Saturday, July 5, 2026
// {{ text | uppercase }}                            → HELLO
// {{ 0.75 | percent }}                              → 75%
// {{ items | slice:0:5 }}                           → first 5 items
// {{ object | json }}                               → pretty-print object
// {{ observable$ | async }}                         → subscribes + auto-unsubscribes

// Custom pipe:
@Pipe({ name: 'relativeTime', standalone: true })
export class RelativeTimePipe implements PipeTransform {
    transform(date: Date | string): string {
        const d = new Date(date);
        const diff = Date.now() - d.getTime();
        const days = Math.floor(diff / 86_400_000);
        if (days === 0) return 'Today';
        if (days === 1) return 'Yesterday';
        if (days < 7) return `${days} days ago`;
        return d.toLocaleDateString('en-PH');
    }
}
// Usage: {{ transaction.date | relativeTime }}
```

---

## Interview Q&A

**Q1: What is the difference between `ngOnInit` and `constructor` in Angular?**
> `constructor` is a TypeScript feature, runs before Angular processes the component, should only be used for dependency injection. `ngOnInit` is an Angular lifecycle hook, runs after Angular initializes inputs and bindings — use for data fetching, subscriptions, and initialization logic that depends on inputs.

**Q2: What is `switchMap` and when would you use it over `mergeMap`?**
> `switchMap` cancels the previous inner Observable when a new value arrives — perfect for search-as-you-type where you want to discard stale requests. `mergeMap` runs all inner Observables concurrently — use when all emissions matter (e.g., parallel file uploads). `concatMap` queues them in order; `exhaustMap` ignores new values while the inner is active (e.g., form submit button).

**Q3: What is the difference between template-driven and reactive forms?**
> Template-driven forms use Angular directives in the template (`ngModel`, `ngForm`) — simpler for basic forms, less testable. Reactive forms define the model in the component class using `FormBuilder` — more explicit, testable, better for complex validation and dynamic forms. Reactive forms are the recommended approach for senior developers.

**Q4: How does Angular's change detection work?**
> Angular's default change detection checks the entire component tree on any async event (click, HTTP, setTimeout). `OnPush` strategy only checks when inputs change or a signal/observable emits — much more performant for large apps. With signals, Angular 17+ moves toward fine-grained reactivity without zone.js.

**Q5: What is lazy loading in Angular and why is it important?**
> Lazy loading splits the app into separate JavaScript bundles loaded on demand when the user navigates to a route. This reduces initial bundle size, improving Time to Interactive (TTI). Implemented with `loadComponent()` or `loadChildren()` in route definitions.
