---
name: angular-architect
description: Angular architecture: standalone components, signals, NgRx/signal store, lazy loading, micro-frontends, testing with Jest/Cypress, and enterprise patterns.
---

# Angular Architect

Design, build, and scale Angular applications using modern APIs and enterprise patterns. Covers standalone components, signals reactivity, state management with NgRx and signal store, advanced routing, dependency injection, performance optimization, testing strategies, and monorepo architecture with Nx and Module Federation.

## Table of Contents

1. [Standalone Components](#1-standalone-components)
2. [Signals](#2-signals)
3. [State Management](#3-state-management)
4. [Routing](#4-routing)
5. [Dependency Injection](#5-dependency-injection)
6. [Performance](#6-performance)
7. [Testing](#7-testing)
8. [Enterprise Patterns](#8-enterprise-patterns)

---

## 1. Standalone Components

### When to Use

- Every new component, directive, and pipe should be standalone (default since Angular 17).
- Migrate existing NgModule-based code incrementally using the Angular CLI schematic.

### Bootstrapping a Standalone Application

```typescript
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig).catch(console.error);
```

```typescript
// app.config.ts
import { ApplicationConfig, provideZoneChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(routes),
    provideHttpClient(withInterceptors([authInterceptor])),
  ],
};
```

### Standalone Component Anatomy

```typescript
@Component({
  selector: 'app-header',
  standalone: true,
  imports: [RouterLink, MatButtonModule, UserAvatarComponent],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <nav class="header">
      <a routerLink="/">Home</a>
      <app-user-avatar [userId]="currentUserId" />
      <button mat-raised-button (click)="logout()">Logout</button>
    </nav>
  `,
})
export class HeaderComponent {
  currentUserId = 'u-123';
  logout(): void { /* ... */ }
}
```

### Migrating from NgModules

```bash
ng generate @angular/core:standalone --path src/app
```

Manual steps: add `standalone: true`, move NgModule `imports`/`declarations` into the component `imports` array, remove the component from the module, delete empty modules.

### Anti-Patterns

- **Importing an entire NgModule when only one component is needed.** Import the standalone component directly.
- **Creating barrel-file NgModules to group standalone components.** Use direct imports or a shared index re-export.
- **Forgetting `ChangeDetectionStrategy.OnPush`.** Always set it to avoid unnecessary change detection cycles.

---

## 2. Signals

### Core Primitives

```typescript
import { signal, computed, effect, untracked } from '@angular/core';

const count = signal(0);
count.set(5);
count.update((prev) => prev + 1);

const doubled = computed(() => count() * 2);

effect(() => {
  console.log(`Count changed to ${count()}`);
  const snapshot = untracked(() => someOtherSignal());
});
```

### Input Signals and Model Signals

```typescript
@Component({
  selector: 'app-slider',
  standalone: true,
  template: `
    <input type="range" [min]="min()" [max]="max()"
           [value]="value()" (input)="onInput($event)" />
    <span>{{ value() }}</span>
  `,
})
export class SliderComponent {
  min = input.required<number>();        // required input signal
  max = input<number>(100);              // optional with default
  value = model<number>(50);             // two-way binding model
  changed = output<number>();

  onInput(event: Event): void {
    const val = +(event.target as HTMLInputElement).value;
    this.value.set(val);
    this.changed.emit(val);
  }
}
```

Parent usage: `<app-slider [min]="0" [(value)]="brightness" />`

### Interop with RxJS

```typescript
import { toSignal, toObservable } from '@angular/core/rxjs-interop';

export class ProductDetailComponent {
  private route = inject(ActivatedRoute);
  private productService = inject(ProductService);

  private productId = toSignal(
    this.route.paramMap.pipe(map((p) => p.get('id')!)),
    { initialValue: '' }
  );

  product = toSignal(
    toObservable(this.productId).pipe(
      switchMap((id) => this.productService.getById(id))
    )
  );
}
```

### Anti-Patterns

- **Calling `.set()` inside `computed()`.** Computed signals are pure derivations; use `effect()` for side-effects.
- **Creating effects outside an injection context.** Pass `{ injector }` or run inside `constructor` / field initializer.
- **Over-converting between signals and observables.** Stay in one paradigm; convert only at boundaries.

---

## 3. State Management

### NgRx Store (Global State)

```typescript
// product.actions.ts
export const ProductActions = createActionGroup({
  source: 'Product',
  events: {
    'Load Products': emptyProps(),
    'Load Products Success': props<{ products: Product[] }>(),
    'Load Products Failure': props<{ error: string }>(),
  },
});

// product.feature.ts — createFeature bundles reducer + selectors
export const productFeature = createFeature({
  name: 'product',
  reducer: createReducer(
    initialState,
    on(ProductActions.loadProducts, (s) => ({ ...s, loading: true })),
    on(ProductActions.loadProductsSuccess, (s, { products }) => ({
      ...s, products, loading: false,
    })),
    on(ProductActions.loadProductsFailure, (s, { error }) => ({
      ...s, error, loading: false,
    }))
  ),
});
// Auto-generated selectors: selectProducts, selectLoading, etc.

// product.effects.ts — functional effect
export const loadProducts$ = createEffect(
  (actions$ = inject(Actions), svc = inject(ProductService)) =>
    actions$.pipe(
      ofType(ProductActions.loadProducts),
      mergeMap(() =>
        svc.getAll().pipe(
          map((products) => ProductActions.loadProductsSuccess({ products })),
          catchError((e) => of(ProductActions.loadProductsFailure({ error: e.message })))
        )
      )
    ),
  { functional: true }
);
```

### NgRx Signal Store (Lightweight)

```typescript
export const CartStore = signalStore(
  { providedIn: 'root' },
  withState<CartState>({ items: [], loading: false }),
  withComputed(({ items }) => ({
    totalItems: computed(() => items().reduce((sum, i) => sum + i.qty, 0)),
    totalPrice: computed(() => items().reduce((sum, i) => sum + i.qty * i.price, 0)),
  })),
  withMethods((store, cartApi = inject(CartApiService)) => ({
    addItem(item: CartItem): void {
      patchState(store, { items: [...store.items(), item] });
    },
    removeItem(productId: string): void {
      patchState(store, { items: store.items().filter((i) => i.productId !== productId) });
    },
    checkout: rxMethod<void>(
      pipe(
        tap(() => patchState(store, { loading: true })),
        switchMap(() => cartApi.checkout(store.items()).pipe(
          tapResponse({
            next: () => patchState(store, { items: [], loading: false }),
            error: () => patchState(store, { loading: false }),
          })
        ))
      )
    ),
  }))
);
```

### When to Use Which

| Pattern | Scope | Best For |
|---|---|---|
| Component signals | Single component | Local UI state (toggles, form fields) |
| Signal Store | Feature / shared | Medium complexity, fewer indirections |
| NgRx Store | App-wide | Complex flows, devtools, undo/redo, team scale |
| Component Store | Feature (legacy) | Existing codebases not yet on signal store |

### Anti-Patterns

- **Putting everything in global NgRx store.** Local UI state belongs in component signals.
- **Mutating state directly.** Always use `patchState` or return new objects.
- **Effects that dispatch actions synchronously in a loop.** Use `concatLatestFrom` for dependent state reads.

---

## 4. Routing

### Lazy Loading

```typescript
export const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./home/home.component').then((m) => m.HomeComponent),
  },
  {
    path: 'products',
    loadChildren: () => import('./products/product.routes').then((m) => m.PRODUCT_ROUTES),
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then((m) => m.ADMIN_ROUTES),
    canMatch: [isAdminGuard],
  },
  { path: '**', redirectTo: '' },
];
```

### Functional Guards and Resolvers

```typescript
export const isAdminGuard: CanMatchFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.hasRole('admin') || router.createUrlTree(['/login']);
};

export const productResolver: ResolveFn<Product> = (route) => {
  return inject(ProductService).getById(route.paramMap.get('id')!);
};
```

### Nested and Auxiliary Routes

```typescript
export const DASHBOARD_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./dashboard-shell.component').then((m) => m.DashboardShellComponent),
    children: [
      { path: '', redirectTo: 'overview', pathMatch: 'full' },
      { path: 'overview', loadComponent: () => import('./overview/overview.component').then((m) => m.OverviewComponent) },
      { path: 'analytics', loadComponent: () => import('./analytics/analytics.component').then((m) => m.AnalyticsComponent) },
      { path: 'notifications', outlet: 'sidebar', loadComponent: () => import('./notifications/notifications.component').then((m) => m.NotificationsComponent) },
    ],
  },
];
```

```html
<main><router-outlet /></main>
<aside><router-outlet name="sidebar" /></aside>
```

### Anti-Patterns

- **Eagerly importing components in route files.** Always use `loadComponent`/`loadChildren` with dynamic `import()`.
- **Using class-based guards.** Functional guards are simpler, tree-shakable, and the recommended API.
- **Deeply nested route configs in a single file.** Split child routes into co-located `*.routes.ts` files.

---

## 5. Dependency Injection

### Provider Configuration

```typescript
// Tree-shakable service
@Injectable({ providedIn: 'root' })
export class AuthService {
  private http = inject(HttpClient);
  login(creds: Credentials): Observable<AuthToken> {
    return this.http.post<AuthToken>('/api/v1/auth/login', creds);
  }
}

// InjectionToken with factory
export const APP_CONFIG = new InjectionToken<AppConfig>('AppConfig');

export function provideAppConfig(): Provider {
  return {
    provide: APP_CONFIG,
    useFactory: () => ({
      apiBaseUrl: environment.apiBaseUrl,
      featureFlags: environment.featureFlags,
    }),
  };
}
```

### Multi Providers

```typescript
export const ANALYTICS_ADAPTER = new InjectionToken<AnalyticsAdapter>('AnalyticsAdapter');

export function provideAnalytics(): Provider[] {
  return [
    { provide: ANALYTICS_ADAPTER, useClass: GoogleAnalyticsAdapter, multi: true },
    { provide: ANALYTICS_ADAPTER, useClass: MixpanelAdapter, multi: true },
  ];
}

@Injectable({ providedIn: 'root' })
export class AnalyticsService {
  private adapters = inject<AnalyticsAdapter[]>(ANALYTICS_ADAPTER);
  track(event: string, payload: Record<string, unknown>): void {
    this.adapters.forEach((a) => a.track(event, payload));
  }
}
```

### inject() vs Constructor Injection

```typescript
// Modern: inject() — works in field initializers, constructors, factory functions
export class OrderComponent {
  private orderService = inject(OrderService);
  private router = inject(Router);
  private destroyRef = inject(DestroyRef);
}
```

### Anti-Patterns

- **`providedIn: 'root'` for feature-specific stateful services.** Provide at route or component level so they are destroyed with the feature.
- **Constructor injection with many parameters.** Switch to `inject()` for cleaner field declarations.
- **Forgetting `multi: true` with multiple providers on one token.** The last registration silently wins.

---

## 6. Performance

### OnPush Change Detection

```typescript
@Component({
  selector: 'app-product-card',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <article class="card">
      <h3>{{ product().name }}</h3>
      <p>{{ product().price | currency }}</p>
    </article>
  `,
})
export class ProductCardComponent {
  product = input.required<Product>();
}
```

### trackBy with @for

```html
@for (item of items(); track item.id) {
  <app-product-card [product]="item" />
} @empty {
  <p>No products found.</p>
}
```

### Defer Blocks

```html
@defer (on viewport) {
  <app-heavy-chart [data]="chartData()" />
} @placeholder {
  <div class="skeleton-chart"></div>
} @loading (minimum 300ms) {
  <app-spinner />
} @error {
  <p>Failed to load chart.</p>
}

@defer (on interaction; prefetch on idle) {
  <app-comments [postId]="postId()" />
} @placeholder {
  <button>Show Comments</button>
}
```

### Preloading and Bundle Analysis

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withPreloading(PreloadAllModules)),
  ],
};
```

```bash
ng build --stats-json
npx source-map-explorer dist/my-app/browser/**/*.js
```

### Server-Side Rendering

```typescript
// app.config.server.ts
const serverConfig: ApplicationConfig = {
  providers: [
    provideServerRendering(),
    provideServerRouting(serverRoutes),
  ],
};
export const config = mergeApplicationConfig(appConfig, serverConfig);

// app.routes.server.ts
export const serverRoutes: ServerRoute[] = [
  { path: '', renderMode: RenderMode.Prerender },
  { path: 'products/:id', renderMode: RenderMode.Server },
  { path: '**', renderMode: RenderMode.Client },
];
```

### Anti-Patterns

- **`Default` change detection everywhere.** Triggers checks on every event across the entire tree.
- **Missing `track` in `@for` blocks.** Angular destroys and recreates every DOM node on each change.
- **Loading heavy widgets eagerly.** Wrap in `@defer` blocks to reduce initial bundle size.
- **SSR without transfer state.** Use `provideClientHydration()` to avoid duplicate HTTP requests after hydration.

---

## 7. Testing

### Component Testing with TestBed

```typescript
describe('ProductListComponent', () => {
  let fixture: ComponentFixture<ProductListComponent>;
  let productService: jasmine.SpyObj<ProductService>;

  beforeEach(async () => {
    const spy = jasmine.createSpyObj('ProductService', ['getAll']);
    await TestBed.configureTestingModule({
      imports: [ProductListComponent],
      providers: [
        provideHttpClient(),
        provideHttpClientTesting(),
        { provide: ProductService, useValue: spy },
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(ProductListComponent);
    productService = TestBed.inject(ProductService) as jasmine.SpyObj<ProductService>;
  });

  it('should display products when loaded', () => {
    productService.getAll.and.returnValue(of([{ id: '1', name: 'Widget', price: 9.99 }]));
    fixture.detectChanges();
    const items = fixture.nativeElement.querySelectorAll('.product-item');
    expect(items.length).toBe(1);
    expect(items[0].textContent).toContain('Widget');
  });
});
```

### Component Harnesses (Angular CDK)

```typescript
describe('LoginComponent', () => {
  let loader: HarnessLoader;

  beforeEach(async () => {
    await TestBed.configureTestingModule({ imports: [LoginComponent] }).compileComponents();
    loader = TestbedHarnessEnvironment.loader(TestBed.createComponent(LoginComponent));
  });

  it('should submit credentials', async () => {
    const email = await loader.getHarness(MatInputHarness.with({ selector: '[data-testid="email"]' }));
    const password = await loader.getHarness(MatInputHarness.with({ selector: '[data-testid="password"]' }));
    const submit = await loader.getHarness(MatButtonHarness.with({ text: 'Login' }));

    await email.setValue('user@example.com');
    await password.setValue('secret');
    await submit.click();
    // Assert navigation or service call
  });
});
```

### Testing with Spectator

```typescript
describe('UserProfileComponent', () => {
  const createComponent = createComponentFactory({
    component: UserProfileComponent,
    providers: [
      mockProvider(UserService, {
        getProfile: () => of({ name: 'Jane', email: 'jane@example.com' }),
      }),
    ],
  });

  it('should display the user name', () => {
    const spectator = createComponent();
    expect(spectator.query('.user-name')).toHaveText('Jane');
  });
});
```

### Marble Testing for NgRx Effects

```typescript
describe('Product Effects', () => {
  let actions$: Observable<any>;
  let productService: jasmine.SpyObj<ProductService>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      providers: [
        provideMockActions(() => actions$),
        { provide: ProductService, useValue: jasmine.createSpyObj('ProductService', ['getAll']) },
      ],
    });
    productService = TestBed.inject(ProductService) as jasmine.SpyObj<ProductService>;
  });

  it('should load products successfully', () => {
    const products = [{ id: '1', name: 'Widget', price: 9.99 }];
    actions$ = hot('-a', { a: ProductActions.loadProducts() });
    productService.getAll.and.returnValue(cold('--b|', { b: products }));
    expect(fromEffects.loadProducts$).toBeObservable(
      cold('---c', { c: ProductActions.loadProductsSuccess({ products }) })
    );
  });
});
```

### Cypress Component Testing

```typescript
describe('ProductCardComponent', () => {
  it('should render product name and price', () => {
    cy.mount(ProductCardComponent, {
      componentProperties: { product: { id: '1', name: 'Gadget', price: 29.99 } },
    });
    cy.get('h3').should('contain.text', 'Gadget');
    cy.get('p').should('contain.text', '$29.99');
  });
});
```

### Anti-Patterns

- **Testing implementation details instead of behavior.** Assert what the user sees, not internal state.
- **Importing real services in unit tests.** Always mock external dependencies.
- **Not calling `fixture.detectChanges()` after signal updates.** Signal components still need a CD cycle in TestBed.
- **Skipping marble tests for complex effects.** Marble diagrams make async timing explicit and catch race conditions.

---

## 8. Enterprise Patterns

### Nx Monorepo Structure

```
my-org/
  apps/
    customer-portal/       # Angular app
    admin-dashboard/       # Angular app
    api/                   # NestJS backend
  libs/
    shared/ui/             # Shared UI components
    shared/util/           # Pure utility functions
    shared/models/         # TypeScript interfaces
    customer/feature-catalog/
    customer/data-access/  # NgRx state + API services
    admin/feature-users/
```

```bash
npx nx generate @nx/angular:library \
  --name=feature-catalog \
  --directory=libs/customer/feature-catalog \
  --standalone --lazy
```

### Module Boundary Enforcement

```jsonc
// .eslintrc.json
{
  "rules": {
    "@nx/enforce-module-boundaries": ["error", {
      "depConstraints": [
        { "sourceTag": "type:feature", "onlyDependOnLibsWithTags": ["type:data-access", "type:ui", "type:util"] },
        { "sourceTag": "type:data-access", "onlyDependOnLibsWithTags": ["type:util", "type:models"] },
        { "sourceTag": "scope:customer", "onlyDependOnLibsWithTags": ["scope:customer", "scope:shared"] }
      ]
    }]
  }
}
```

### Module Federation Micro-Frontends

```typescript
// Host — webpack.config.ts
export default withModuleFederationPlugin({
  remotes: {
    catalog: 'http://localhost:4201/remoteEntry.js',
    checkout: 'http://localhost:4202/remoteEntry.js',
  },
  shared: {
    '@angular/core': { singleton: true, strictVersion: true },
    '@angular/common': { singleton: true, strictVersion: true },
    '@angular/router': { singleton: true, strictVersion: true },
  },
});

// Host routes
export const routes: Routes = [
  { path: 'catalog', loadChildren: () => import('catalog/Routes').then((m) => m.CATALOG_ROUTES) },
  { path: 'checkout', loadChildren: () => import('checkout/Routes').then((m) => m.CHECKOUT_ROUTES) },
];

// Remote — webpack.config.ts
export default withModuleFederationPlugin({
  name: 'catalog',
  exposes: { './Routes': './src/app/catalog.routes.ts' },
  shared: {
    '@angular/core': { singleton: true, strictVersion: true },
    '@angular/router': { singleton: true, strictVersion: true },
  },
});
```

### Shared API Layer

```typescript
@Injectable({ providedIn: 'root' })
export class ApiService {
  private http = inject(HttpClient);
  private config = inject(APP_CONFIG);

  get<T>(path: string, params?: QueryParams): Observable<T> {
    return this.http.get<T>(`${this.config.apiBaseUrl}${path}`, {
      params: this.buildParams(params),
    });
  }

  getPage<T>(path: string, params?: QueryParams): Observable<PaginatedResponse<T>> {
    return this.http.get<PaginatedResponse<T>>(`${this.config.apiBaseUrl}${path}`, {
      params: this.buildParams(params),
    });
  }

  post<T>(path: string, body: unknown): Observable<T> {
    return this.http.post<T>(`${this.config.apiBaseUrl}${path}`, body);
  }

  put<T>(path: string, body: unknown): Observable<T> {
    return this.http.put<T>(`${this.config.apiBaseUrl}${path}`, body);
  }

  delete<T>(path: string): Observable<T> {
    return this.http.delete<T>(`${this.config.apiBaseUrl}${path}`);
  }

  private buildParams(params?: QueryParams): HttpParams {
    let hp = new HttpParams();
    if (!params) return hp;
    if (params.page != null) hp = hp.set('page', params.page);
    if (params.pageSize != null) hp = hp.set('pageSize', params.pageSize);
    if (params.sort) hp = hp.set('sort', params.sort);
    if (params.filter) {
      Object.entries(params.filter).forEach(([k, v]) => { hp = hp.set(`filter[${k}]`, v); });
    }
    return hp;
  }
}
```

### Anti-Patterns

- **Sharing state between micro-frontends via global variables.** Use a shared signal store or message bus with well-defined contracts.
- **No module boundary enforcement.** Without Nx constraints, feature libraries will import each other and create circular dependencies.
- **One giant `shared` library.** Split into `shared/ui`, `shared/util`, `shared/models` so consumers only import what they need.
- **Skipping the API abstraction layer.** Direct `HttpClient` calls scattered across components make endpoint changes expensive and testing harder.
