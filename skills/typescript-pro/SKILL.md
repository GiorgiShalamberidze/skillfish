---
name: typescript-pro
description: Advanced TypeScript patterns: generics, conditional types, mapped types, module augmentation, declaration files, and strict-mode migration guides.
---

# TypeScript Pro

Advanced TypeScript patterns and techniques for senior developers. Covers the type system in depth -- generics, conditional types, mapped types, template literal types, module augmentation, declaration file authoring, and strict-mode migration strategies. Use this skill when solving complex type-level problems, hardening a codebase with stricter checks, or building type-safe library APIs.

## Table of Contents

- [Advanced Type System](#advanced-type-system)
- [Utility Types Deep Dive](#utility-types-deep-dive)
- [Type Guards and Narrowing](#type-guards-and-narrowing)
- [Module System and Declaration Files](#module-system-and-declaration-files)
- [Strict Mode Migration](#strict-mode-migration)
- [Performance Patterns](#performance-patterns)
- [Common Patterns Quick Reference](#common-patterns-quick-reference)

---

## Advanced Type System

### Generics with Constraints

```typescript
function mergeById<T extends { id: string }>(items: T[], patch: Partial<T> & { id: string }): T[] {
  return items.map(item => (item.id === patch.id ? { ...item, ...patch } : item));
}

function cloneAndStamp<T extends object>(obj: T): T & { clonedAt: Date } {
  return { ...obj, clonedAt: new Date() };
}
```

### Conditional Types

Conditional types branch at the type level. They distribute over unions when the checked type is a naked type parameter.

```typescript
type IsString<T> = T extends string ? true : false;
type Result = IsString<string | number>; // true | false (distributive)

// Prevent distribution by wrapping in a tuple
type IsStringStrict<T> = [T] extends [string] ? true : false;
type StrictResult = IsStringStrict<string | number>; // false
```

### The `infer` Keyword

```typescript
type ReturnOf<T> = T extends (...args: any[]) => infer R ? R : never;
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;
type ElementOf<T> = T extends readonly (infer E)[] ? E : never;
type SecondParam<T> = T extends (a: any, b: infer B, ...rest: any[]) => any ? B : never;
```

### Template Literal Types

```typescript
type Route = `/${"users" | "posts"}/${string}`;
type EventName<T extends string> = `on${Capitalize<T>}`;
type CSSUnit = "px" | "rem" | "em" | "vh" | "vw" | "%";
type CSSLength = `${number}${CSSUnit}`;
```

#### Type-Safe Routing with Template Literals

```typescript
type ExtractParams<T extends string> =
  T extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof ExtractParams<Rest>]: string }
    : T extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

type Params = ExtractParams<"/users/:userId/posts/:postId">;
// { userId: string; postId: string }

function navigate<T extends string>(pattern: T, params: ExtractParams<T>): string {
  return Object.entries(params).reduce(
    (path, [key, val]) => path.replace(`:${key}`, val as string), pattern as string
  );
}
```

### Recursive Conditional Types

```typescript
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> } : T;

type DeepReadonly<T> = T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> } : T;

// Dot-notation paths for nested objects
type DotPaths<T, Prefix extends string = ""> = T extends object
  ? { [K in keyof T & string]: T[K] extends object
        ? DotPaths<T[K], `${Prefix}${K}.`> : `${Prefix}${K}`;
    }[keyof T & string]
  : never;

interface Config { db: { host: string; port: number }; cache: { ttl: number } }
type ConfigPaths = DotPaths<Config>; // "db.host" | "db.port" | "cache.ttl"
```

### Variadic Tuple Types

```typescript
type Prepend<E, T extends unknown[]> = [E, ...T];
type Concat<A extends unknown[], B extends unknown[]> = [...A, ...B];
type Last<T extends unknown[]> = T extends [...infer _, infer L] ? L : never;

function pipe<Fns extends ((...args: any[]) => any)[]>(
  input: Parameters<Fns[0]>[0], ...fns: Fns
): ReturnType<Last<Fns>> {
  return fns.reduce((val, fn) => fn(val), input);
}

const result = pipe("hello", (s: string) => s.length, (n: number) => n > 3); // boolean
```

### Mapped Types with Key Remapping

```typescript
type Getters<T> = {
  [K in keyof T as `get${Capitalize<K & string>}`]: () => T[K];
};
interface User { name: string; age: number }
type UserGetters = Getters<User>; // { getName: () => string; getAge: () => number }

type StringKeysOnly<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};
```

---

## Utility Types Deep Dive

### Beyond the Built-Ins

| Utility Type | Purpose |
|---|---|
| `DeepPartial<T>` | Recursively make all properties optional |
| `DeepReadonly<T>` | Recursively make all properties readonly |
| `Mutable<T>` | Remove `readonly` from all properties |
| `RequireAtLeastOne<T>` | At least one property must be provided |
| `RequireExactlyOne<T, K>` | Exactly one of the specified keys must exist |
| `Prettify<T>` | Flatten intersection types for readable hover info |
| `StrictOmit<T, K>` | `Omit` that errors when `K` is not a key of `T` |
| `Branded<T, Brand>` | Nominal/branded type wrapper |

### Implementations

```typescript
type Mutable<T> = { -readonly [K in keyof T]: T[K] };

type RequireAtLeastOne<T, Keys extends keyof T = keyof T> =
  Pick<T, Exclude<keyof T, Keys>> &
  { [K in Keys]-?: Required<Pick<T, K>> & Partial<Pick<T, Exclude<Keys, K>>> }[Keys];

type Prettify<T> = { [K in keyof T]: T[K] } & {};

type StrictOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;

type RequireExactlyOne<T, Keys extends keyof T = keyof T> =
  Pick<T, Exclude<keyof T, Keys>> &
  { [K in Keys]-?: Required<Pick<T, K>> & Partial<Record<Exclude<Keys, K>, never>> }[Keys];
```

### Type-Level String Manipulation

```typescript
type KebabToCamel<S extends string> = S extends `${infer H}-${infer T}`
  ? `${H}${KebabToCamel<Capitalize<T>>}` : S;
type Converted = KebabToCamel<"my-component-name">; // "myComponentName"
```

---

## Type Guards and Narrowing

### Discriminated Unions

```typescript
type ApiResponse =
  | { status: "success"; data: unknown }
  | { status: "error"; error: string; retryable: boolean }
  | { status: "loading" };

function handleResponse(res: ApiResponse) {
  switch (res.status) {
    case "success": console.log(res.data); break;
    case "error":   console.log(res.error); break;
    case "loading": break;
    default: const _exhaustive: never = res; // compile error if variant unhandled
  }
}
```

### Custom Type Guard Functions

```typescript
interface Cat { kind: "cat"; purr(): void }
interface Dog { kind: "dog"; bark(): void }

function isCat(animal: Cat | Dog): animal is Cat {
  return animal.kind === "cat";
}

function isNonNullable<T>(value: T): value is NonNullable<T> {
  return value !== null && value !== undefined;
}
const clean: number[] = [1, null, 2, undefined, 3].filter(isNonNullable);
```

### Assertion Functions

Assertion functions throw on failure and narrow the type for subsequent code.

```typescript
function assertDefined<T>(value: T, msg?: string): asserts value is NonNullable<T> {
  if (value == null) throw new Error(msg ?? "Expected value to be defined");
}

function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") throw new TypeError(`Expected string, got ${typeof value}`);
}

function processInput(input: unknown) {
  assertIsString(input);
  console.log(input.toUpperCase()); // input is now `string`
}
```

### The `satisfies` Operator

`satisfies` validates that a value matches a type without widening its inferred type. Preserves literal types and autocomplete while enforcing structural constraints.

```typescript
type ColorMap = Record<string, [number, number, number] | string>;

// Without satisfies: literals lost, type is ColorMap
const colors1: ColorMap = { red: [255, 0, 0], green: "#00ff00" };

// With satisfies: inferred type keeps literals
const colors2 = { red: [255, 0, 0], green: "#00ff00" } satisfies ColorMap;
// colors2.red  => [number, number, number]  (tuple methods available)
// colors2.green => string
```

#### `satisfies` with `as const`

```typescript
const routes = {
  home: "/", users: "/users", user: "/users/:id",
} as const satisfies Record<string, string>;

type UserRoute2 = typeof routes.user; // "/users/:id" (literal, not just string)
```

---

## Module System and Declaration Files

### Writing `.d.ts` Declaration Files

```typescript
declare module "analytics-lib" {
  interface AnalyticsConfig { trackingId: string; debug?: boolean }
  export function init(config: AnalyticsConfig): AnalyticsInstance;
  export interface AnalyticsInstance {
    track(event: string, props?: Record<string, unknown>): void;
    identify(userId: string, traits?: Record<string, unknown>): void;
  }
}
```

### Module Augmentation

Extend existing library types without forking them.

```typescript
// Extend Express Request
import "express";
declare module "express" {
  interface Request { userId?: string; permissions?: string[] }
}

// Extend React CSSProperties to allow CSS custom properties
import "react";
declare module "react" {
  interface CSSProperties { [key: `--${string}`]: string | number }
}
```

### Ambient Declarations and Triple-Slash Directives

```typescript
// globals.d.ts
declare const __APP_VERSION__: string;
declare namespace NodeJS {
  interface ProcessEnv { NODE_ENV: "development" | "production" | "test"; DATABASE_URL: string }
}
```

```typescript
/// <reference types="vite/client" />
/// <reference path="./custom-types.d.ts" />
```

| Scenario | Use |
|---|---|
| Global type augmentation in `.d.ts` | `/// <reference types="..." />` |
| Importing types in `.ts` files | `import type { Foo } from "bar"` |
| Referencing sibling declaration files | `/// <reference path="..." />` |

---

## Strict Mode Migration

Step-by-step workflow for enabling `strict: true` in an existing codebase.

### Step 1: Audit

```bash
npx tsc --strict --noEmit 2>&1 | grep "error TS" | wc -l
```

### Step 2: Enable Flags Incrementally

| Order | Flag | Common Fix Patterns |
|---|---|---|
| 1 | `noImplicitAny` | Add explicit type annotations to parameters and variables |
| 2 | `strictNullChecks` | Add null guards, optional chaining, update return types |
| 3 | `strictFunctionTypes` | Fix contravariant parameter mismatches in callbacks |
| 4 | `strictBindCallApply` | Type `bind`/`call`/`apply` arguments correctly |
| 5 | `strictPropertyInitialization` | Add initializers or definite assignment assertion (`!`) |
| 6 | `noImplicitThis` | Add `this` parameter to functions, use arrow functions |
| 7 | `useUnknownInCatchVariables` | Replace `(e: any)` with `unknown` and narrow |
| 8 | `strict: true` | Enable once all individual flags pass |

### Step 3: Common Fix Patterns

```typescript
// noImplicitAny: annotate parameters
function process(data: { name: string }): string { return data.name; }

// strictNullChecks: guard against null
const el = document.getElementById("root");
if (!el) throw new Error("Missing #root");
el.innerHTML = "hello";

// useUnknownInCatchVariables: narrow error types
try { await fetch("/api"); }
catch (err) {
  if (err instanceof Error) console.error(err.message);
  else console.error("Unknown error", err);
}
```

### Step 4: Escape Hatches

Use `@ts-expect-error` (not `@ts-ignore`) for temporary suppression. It errors when the suppressed issue is fixed, signaling cleanup.

```typescript
// @ts-expect-error -- legacy code, tracked in JIRA-1234
const result = legacyFunction(untypedArg);
```

---

## Performance Patterns

### `const` Assertions

```typescript
const config = { endpoint: "/api", retries: 3 } as const;
// type: { readonly endpoint: "/api"; readonly retries: 3 }

const STATUS = ["idle", "loading", "success", "error"] as const;
type Status = (typeof STATUS)[number]; // "idle" | "loading" | "success" | "error"
```

### Branded / Nominal Types

```typescript
type Brand<T, B extends string> = T & { readonly __brand: B };
type UserId = Brand<string, "UserId">;
type OrderId = Brand<string, "OrderId">;

function createUserId(id: string): UserId { return id as UserId; }
function createOrderId(id: string): OrderId { return id as OrderId; }
function getUser(id: UserId) { /* ... */ }

getUser(createUserId("u-123"));   // OK
// getUser(createOrderId("o-456")); // Error: OrderId not assignable to UserId
```

### Avoiding Type Explosion

Deep generic nesting causes enormous type representations. Prefer `interface` over `type` (lazily evaluated), limit recursion depth with a counter tuple, and cache intermediate types.

```typescript
type DeepPartialWithLimit<T, Depth extends unknown[] = []> =
  Depth["length"] extends 5 ? T
    : T extends object
      ? { [K in keyof T]?: DeepPartialWithLimit<T[K], [...Depth, unknown]> }
      : T;
```

### Compile-Time Optimization Tips

| Tip | Why |
|---|---|
| Prefer `interface` over `type` for objects | Cached by name; type aliases are re-expanded |
| Use `import type` / `export type` | Erased at compile time, reduces dependency graph |
| Avoid `enum`, use `as const` objects | Enums emit runtime code; `as const` is zero-cost |
| Enable `incremental` in tsconfig | Caches previous build; massive re-compile speedup |
| Use project references for monorepos | Parallel builds, incremental per-package |
| Set `skipLibCheck: true` | Skip checking `.d.ts` in `node_modules` |

### Type-Safe Event Emitter

```typescript
type EventMap = {
  connect: [host: string, port: number];
  disconnect: [reason: string];
  data: [payload: Buffer];
};

class TypedEmitter<Events extends Record<string, unknown[]>> {
  private listeners = new Map<keyof Events, Set<Function>>();

  on<E extends keyof Events>(event: E, fn: (...args: Events[E]) => void): this {
    if (!this.listeners.has(event)) this.listeners.set(event, new Set());
    this.listeners.get(event)!.add(fn);
    return this;
  }

  emit<E extends keyof Events>(event: E, ...args: Events[E]): void {
    this.listeners.get(event)?.forEach(fn => fn(...args));
  }
}

const emitter = new TypedEmitter<EventMap>();
emitter.on("connect", (host, port) => console.log(`${host}:${port}`));
// emitter.emit("connect", "localhost"); // Error: expected 2 args
```

### Builder Pattern with Generics

Accumulate type information across chained method calls.

```typescript
class QueryBuilder<T extends Record<string, unknown> = {}> {
  private filters: Record<string, unknown> = {};

  where<K extends string, V>(key: K, value: V): QueryBuilder<T & Record<K, V>> {
    this.filters[key] = value;
    return this as any;
  }

  select<K extends keyof T>(...fields: K[]): Pick<T, K>[] {
    return [] as any; // execute query
  }
}

const results = new QueryBuilder()
  .where("name", "Alice")
  .where("age", 30)
  .select("name", "age");
// Pick<{ name: string; age: number }, "name" | "age">[]
```

---

## Common Patterns Quick Reference

| Pattern | When to Use | Example |
|---|---|---|
| Discriminated union | Model finite states | `{ status: "ok"; data: T } \| { status: "err"; error: E }` |
| Branded type | Prevent ID mixups | `type UserId = string & { __brand: "UserId" }` |
| `satisfies` | Validate without widening | `const x = { ... } satisfies Schema` |
| Builder + generics | Accumulate type info across calls | `new Builder().add("a", 1).build()` |
| `as const` array | Derive union from values | `const a = ["x","y"] as const; type T = (typeof a)[number]` |
| Template literal type | Type-safe strings | `` type Route = `/api/${string}` `` |
| Module augmentation | Extend third-party types | `declare module "lib" { interface X { extra: string } }` |
| Assertion function | Narrow + throw in one call | `asserts value is string` |
| Recursive conditional | Deep transformations | `DeepPartial<T>`, `DeepReadonly<T>` |
| `infer` in conditionals | Extract nested types | `T extends Promise<infer U> ? U : T` |
| Mapped type with `as` | Rename/filter keys | `` { [K in keyof T as `get${Capitalize<K>}`]: () => T[K] } `` |
| Variadic tuple | Typed function composition | `type Concat<A, B> = [...A, ...B]` |
| Depth limiter | Prevent type explosion | Counter tuple to cap recursion |
| `import type` | Compile-time only imports | `import type { Config } from "./config"` |
| `@ts-expect-error` | Temporary suppression | Errors when suppressed issue is fixed |
