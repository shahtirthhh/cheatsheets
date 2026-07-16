# TypeScript — Complete Language Mastery Cheatsheet
*Intern → Super Senior · Type system deep dive + TS-specific behavior*

---

## 1. TypeScript vs JavaScript — What TS Adds

TypeScript is a **structural type system** layered on top of JavaScript. All TS types are erased at compile time — the runtime is pure JavaScript. TS never changes runtime behavior.

```
What TS adds:                           What TS does NOT add:
─────────────                           ─────────────────────
Static types (compile-time checks)      Runtime type checking
Interfaces & type aliases               Runtime interfaces
Generics                                Runtime generics
Enums (transpiled to JS objects)        New runtime data structures
Access modifiers (public/private)       True private fields (use JS #private)
Decorators (same as JS proposal)        Magic — it's still JS underneath
```

---

## 2. Type Annotations

### Primitives

```typescript
let name: string = "Alice";
let age: number = 30;
let isActive: boolean = true;
let nothing: null = null;
let notDefined: undefined = undefined;
let id: symbol = Symbol("id");
let big: bigint = 100n;
```

### Arrays and Tuples

```typescript
// Arrays (two syntaxes, identical meaning)
let nums: number[] = [1, 2, 3];
let nums2: Array<number> = [1, 2, 3];

// Read-only arrays
let frozen: readonly number[] = [1, 2, 3];
let frozen2: ReadonlyArray<number> = [1, 2, 3];
// frozen.push(4);  // Error!

// Tuples (fixed-length, typed per position)
let pair: [string, number] = ["Alice", 30];
let triple: [string, number, boolean] = ["Alice", 30, true];

// Named tuples (for readability)
type UserTuple = [name: string, age: number];
const user: UserTuple = ["Alice", 30];

// Optional tuple elements
type Config = [string, number?];
const c1: Config = ["host"];
const c2: Config = ["host", 3000];

// Rest in tuples
type StringAndNumbers = [string, ...number[]];
const sn: StringAndNumbers = ["hello", 1, 2, 3, 4];

// Readonly tuples
type Point = readonly [number, number];
const p: Point = [3, 4];
// p[0] = 5;  // Error!

// as const (makes it a readonly tuple literal)
const colors = ["red", "green", "blue"] as const;
// type: readonly ["red", "green", "blue"]
// Not string[], but literally those 3 values
```

### Objects

```typescript
// Inline type
let user: { name: string; age: number; email?: string } = {
  name: "Alice",
  age: 30,
};

// Optional property (?)
user.email  // string | undefined

// Readonly
let config: { readonly host: string; readonly port: number } = {
  host: "localhost",
  port: 3000,
};
// config.port = 4000;  // Error!

// Index signature (dynamic keys)
let scores: { [key: string]: number } = {};
scores["math"] = 95;
scores["english"] = 88;

// Record shorthand
let scores2: Record<string, number> = { math: 95 };
```

---

## 3. Type Aliases vs Interfaces

```typescript
// Type alias — can represent ANY type
type ID = string | number;
type Point = { x: number; y: number };
type Callback = (data: string) => void;
type Pair<T> = [T, T];

// Interface — only for object shapes (but can be extended/merged)
interface User {
  name: string;
  age: number;
  greet(): string;
}

// Extending
interface Admin extends User {
  role: string;
  permissions: string[];
}

// Type intersection (same as extends but for type aliases)
type Admin2 = User & { role: string; permissions: string[] };

// Declaration merging (ONLY interfaces — NOT type aliases)
interface User {
  email: string;  // merged with the User interface above
}
// Now User has: name, age, greet, email

// When to use which:
// Interface: for objects that will be extended/implemented, or for public APIs (library code)
// Type alias: for unions, intersections, tuples, mapped types, utility types
```

### 🧩 TRICKY: Interface vs Type

```typescript
// Interfaces can be augmented (declaration merging)
interface Window {
  myCustomProp: string;
}
// Now window.myCustomProp is valid globally

// Type aliases CANNOT be merged
type Foo = { a: string };
type Foo = { b: string };  // Error! Duplicate identifier

// Interfaces extend interfaces; types use intersection
interface A { a: string }
interface B extends A { b: string }  // OK

type C = { a: string };
type D = C & { b: string };          // OK (intersection)
```

---

## 4. Union and Intersection Types

### Unions (OR)

```typescript
type Status = "pending" | "active" | "archived";  // string literal union
type Result = string | number;

function format(value: string | number): string {
  if (typeof value === "string") {
    return value.toUpperCase();   // TS knows it's string here (narrowing)
  }
  return value.toFixed(2);        // TS knows it's number here
}
```

### Intersections (AND)

```typescript
type HasName = { name: string };
type HasAge = { age: number };
type Person = HasName & HasAge;   // must have BOTH name and age

const person: Person = { name: "Alice", age: 30 };
```

### Discriminated Unions (Tagged Unions)

The most important pattern in TypeScript for type-safe state management.

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rectangle"; width: number; height: number }
  | { kind: "triangle"; base: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;    // TS knows radius exists
    case "rectangle":
      return shape.width * shape.height;      // TS knows width/height exist
    case "triangle":
      return 0.5 * shape.base * shape.height;
    default:
      const _exhaustive: never = shape;       // compile error if a case is missed
      throw new Error(`Unknown shape: ${_exhaustive}`);
  }
}
```

---

## 5. Type Narrowing

How TypeScript narrows a broad type to a specific one within a code branch.

```typescript
// typeof guard
function process(value: string | number) {
  if (typeof value === "string") {
    value.toUpperCase();     // string methods available
  } else {
    value.toFixed(2);        // number methods available
  }
}

// instanceof guard
function handle(err: Error | string) {
  if (err instanceof Error) {
    err.message;             // Error properties available
  } else {
    err.toUpperCase();       // string
  }
}

// in operator
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(animal: Fish | Bird) {
  if ("swim" in animal) {
    animal.swim();           // narrowed to Fish
  } else {
    animal.fly();            // narrowed to Bird
  }
}

// Truthiness narrowing
function greet(name: string | null) {
  if (name) {
    name.toUpperCase();      // string (null is excluded)
  }
}

// Equality narrowing
function compare(a: string | number, b: string | boolean) {
  if (a === b) {
    a.toUpperCase();         // both must be string (only common type)
  }
}

// Custom type guard (type predicate)
function isString(value: unknown): value is string {
  return typeof value === "string";
}

if (isString(input)) {
  input.toUpperCase();       // TS trusts the type guard
}

// Assertion function
function assertDefined<T>(value: T | null | undefined, name: string): asserts value is T {
  if (value == null) throw new Error(`${name} is not defined`);
}

assertDefined(user, "user");
user.name;  // TS knows user is not null after assertion
```

---

## 6. Generics

### Basic Generics

```typescript
// Generic function
function identity<T>(value: T): T {
  return value;
}
identity<string>("hello");   // explicit type
identity(42);                 // inferred as identity<number>

// Generic interface
interface Box<T> {
  value: T;
  map<U>(fn: (val: T) => U): Box<U>;
}

// Generic class
class Stack<T> {
  private items: T[] = [];

  push(item: T): void { this.items.push(item); }
  pop(): T | undefined { return this.items.pop(); }
  peek(): T | undefined { return this.items[this.items.length - 1]; }
}

const numStack = new Stack<number>();
numStack.push(42);
numStack.push("hello");  // Error! Expected number
```

### Generic Constraints

```typescript
// Constrain T to objects with a length property
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}
longest("hello", "hi");         // "hello"
longest([1, 2, 3], [1, 2]);    // [1, 2, 3]
longest(42, 100);               // Error! number doesn't have .length

// keyof constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
const user = { name: "Alice", age: 30 };
getProperty(user, "name");      // string
getProperty(user, "age");       // number
getProperty(user, "email");     // Error! "email" is not in keyof User

// Multiple type parameters
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b };
}
```

### Generic Defaults

```typescript
interface ApiResponse<T = unknown> {
  data: T;
  status: number;
  message: string;
}

const res: ApiResponse = { data: "anything", status: 200, message: "ok" };
const res2: ApiResponse<User[]> = { data: users, status: 200, message: "ok" };
```

---

## 7. Utility Types (Built-in)

```typescript
interface User {
  name: string;
  age: number;
  email: string;
  role: "admin" | "user";
}

// Partial — all properties optional
type UpdateUser = Partial<User>;
// { name?: string; age?: number; email?: string; role?: "admin" | "user" }

// Required — all properties required
type RequiredUser = Required<Partial<User>>;

// Readonly — all properties readonly
type FrozenUser = Readonly<User>;
// Cannot assign to 'name' because it is a read-only property

// Pick — select specific properties
type UserName = Pick<User, "name" | "email">;
// { name: string; email: string }

// Omit — remove specific properties
type UserWithoutRole = Omit<User, "role">;
// { name: string; age: number; email: string }

// Record — construct type with keys K and values T
type Roles = Record<"admin" | "user" | "guest", string[]>;
// { admin: string[]; user: string[]; guest: string[] }

// Exclude — remove types from a union
type Status = "active" | "inactive" | "deleted";
type ActiveStatus = Exclude<Status, "deleted">;
// "active" | "inactive"

// Extract — keep only matching types from a union
type StringStatus = Extract<string | number | boolean, string>;
// string

// NonNullable — remove null and undefined
type Name = NonNullable<string | null | undefined>;
// string

// ReturnType — extract return type of a function
function fetchUser() { return { name: "Alice", age: 30 }; }
type UserReturn = ReturnType<typeof fetchUser>;
// { name: string; age: number }

// Parameters — extract parameter types as a tuple
type FetchParams = Parameters<typeof fetchUser>;
// []

// Awaited — unwrap Promise type
type Data = Awaited<Promise<string>>;
// string
type DeepData = Awaited<Promise<Promise<number>>>;
// number

// ConstructorParameters, InstanceType — for classes
```

---

## 8. Mapped Types

```typescript
// Create new types by transforming properties of existing types

// Make all properties optional
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};

// Make all properties readonly
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Make all properties nullable
type Nullable<T> = {
  [K in keyof T]: T[K] | null;
};

// Key remapping (as clause)
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number; getEmail: () => string; getRole: () => "admin" | "user" }

// Remove specific properties via remapping
type RemoveReadonly<T> = {
  [K in keyof T as K extends "id" ? never : K]: T[K];
};
```

---

## 9. Conditional Types

```typescript
// T extends U ? X : Y — "if T is assignable to U, then X, else Y"

type IsString<T> = T extends string ? true : false;
type A = IsString<"hello">;   // true
type B = IsString<42>;         // false

// Distributive conditional types (distributes over unions)
type ToArray<T> = T extends any ? T[] : never;
type Result = ToArray<string | number>;
// string[] | number[]  (NOT (string | number)[])

// Prevent distribution with []
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
type Result2 = ToArrayNonDist<string | number>;
// (string | number)[]

// infer keyword (extract types from other types)
type UnwrapPromise<T> = T extends Promise<infer U> ? U : T;
type A2 = UnwrapPromise<Promise<string>>;  // string
type B2 = UnwrapPromise<number>;            // number

type GetReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type R = GetReturnType<(x: number) => string>;  // string

// Extract array element type
type ElementOf<T> = T extends (infer E)[] ? E : never;
type El = ElementOf<string[]>;  // string

// Real-world: Extract event payload type
type EventPayload<T> = T extends { type: string; payload: infer P } ? P : never;
```

---

## 10. Template Literal Types

```typescript
type Color = "red" | "green" | "blue";
type Size = "small" | "large";

type ClassName = `${Size}-${Color}`;
// "small-red" | "small-green" | "small-blue" | "large-red" | "large-green" | "large-blue"

// Event handler names
type EventName = "click" | "scroll" | "keypress";
type Handler = `on${Capitalize<EventName>}`;
// "onClick" | "onScroll" | "onKeypress"

// Intrinsic string manipulation types
Uppercase<"hello">        // "HELLO"
Lowercase<"HELLO">        // "hello"
Capitalize<"hello">       // "Hello"
Uncapitalize<"Hello">     // "hello"

// Pattern matching with template literals
type IsGetter<T> = T extends `get${infer _}` ? true : false;
type A = IsGetter<"getName">;    // true
type B = IsGetter<"setName">;    // false

// Extract event name from handler
type ExtractEvent<T> = T extends `on${infer E}` ? Uncapitalize<E> : never;
type E = ExtractEvent<"onClick">;  // "click"
```

---

## 11. Enums

```typescript
// Numeric enum (default, auto-increments from 0)
enum Direction {
  Up,      // 0
  Down,    // 1
  Left,    // 2
  Right,   // 3
}
Direction.Up          // 0
Direction[0]          // "Up" (reverse mapping — only for numeric enums)

// String enum (no reverse mapping)
enum Status {
  Active = "ACTIVE",
  Inactive = "INACTIVE",
  Deleted = "DELETED",
}
Status.Active         // "ACTIVE"

// const enum (inlined at compile time — no runtime object)
const enum Color {
  Red = "RED",
  Green = "GREEN",
  Blue = "BLUE",
}
// Color.Red compiles to just "RED" (no Color object at runtime)

// Enums vs Union Types (prefer unions for most cases)
type StatusUnion = "active" | "inactive" | "deleted";
// Advantages of unions: no runtime overhead, better tree-shaking, simpler
// Advantages of enums: reverse mapping (numeric), namespace, const enum inlining
```

### 🧩 TRICKY: Enum Behavior

```typescript
enum Fruit { Apple, Banana, Cherry }

// Numeric enums allow ANY number (TypeScript doesn't check at runtime)
const mystery: Fruit = 99;  // no error! Enums are just numbers
// This is why string enums or union types are generally safer
```

---

## 12. Type Assertions and Non-Null Assertion

```typescript
// Type assertion (you know better than TS)
const input = document.getElementById("name") as HTMLInputElement;
input.value = "Alice";

// Alternative syntax (not in JSX)
const input2 = <HTMLInputElement>document.getElementById("name");

// const assertion (narrows to literal types)
const config = { host: "localhost", port: 3000 } as const;
// type: { readonly host: "localhost"; readonly port: 3000 }
// Without as const: { host: string; port: number }

// Non-null assertion (! postfix — "trust me, this is not null")
const element = document.getElementById("app")!;  // HTMLElement (not null)
// Use sparingly — you're removing a safety check

// satisfies (TS 4.9) — validates without widening
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: [0, 0, 255],
} satisfies Record<string, string | number[]>;
// palette.red is still [number, number, number], not string | number[]
// satisfies checks the type without changing the inferred type
```

---

## 13. Special Types

```typescript
// any — disables type checking (escape hatch)
let anything: any = 42;
anything.nonexistent.method();  // no error! (but will crash at runtime)

// unknown — type-safe version of any (must narrow before using)
let value: unknown = 42;
// value.toString();  // Error! unknown can't be used directly
if (typeof value === "number") {
  value.toFixed(2);  // OK after narrowing
}

// never — represents values that never occur
function throwError(msg: string): never {
  throw new Error(msg);   // function never returns
}

function infinite(): never {
  while (true) {}         // function never returns
}

// Exhaustiveness checking with never
type Animal = "cat" | "dog" | "bird";
function speak(animal: Animal): string {
  switch (animal) {
    case "cat": return "meow";
    case "dog": return "woof";
    case "bird": return "tweet";
    default:
      const _check: never = animal;  // error if a case is missed
      return _check;
  }
}

// void — function returns nothing (but may return undefined)
function log(msg: string): void {
  console.log(msg);
}

// object — any non-primitive value
function processObj(obj: object): void { ... }
processObj({});       // OK
processObj([]);       // OK
processObj(() => {}); // OK
processObj(42);       // Error! number is a primitive

// {} — any non-null, non-undefined value (almost everything)
let val: {} = 42;      // OK
let val2: {} = "hello"; // OK
let val3: {} = null;    // Error
```

---

## 14. Decorators (Stage 3 / TS 5.0+)

```typescript
// Method decorator
function log(originalMethod: any, context: ClassMethodDecoratorContext) {
  return function (this: any, ...args: any[]) {
    console.log(`Calling ${String(context.name)} with`, args);
    const result = originalMethod.call(this, ...args);
    console.log(`${String(context.name)} returned`, result);
    return result;
  };
}

class Calculator {
  @log
  add(a: number, b: number): number {
    return a + b;
  }
}

// Legacy decorators (experimentalDecorators flag — used by NestJS, Angular)
function Controller(path: string) {
  return function (target: Function) {
    Reflect.defineMetadata("path", path, target);
  };
}

@Controller("/users")
class UsersController { ... }
```

---

## 15. Module Augmentation and Declaration Merging

```typescript
// Augment a library's types
declare module "express" {
  interface Request {
    user?: {
      id: string;
      role: string;
    };
  }
}
// Now req.user is typed in all Express route handlers

// Global augmentation
declare global {
  interface Window {
    analytics: AnalyticsSDK;
  }
}

// .d.ts files (type declarations for JS libraries)
// These files contain ONLY type information, no runtime code
declare module "untyped-library" {
  export function doSomething(input: string): number;
  export interface Config {
    verbose: boolean;
  }
}
```

---

## 16. Advanced Patterns

### Branded Types (Nominal Typing Hack)

TypeScript is structural, but sometimes you want nominal types:

```typescript
type UserId = string & { readonly __brand: unique symbol };
type OrderId = string & { readonly __brand: unique symbol };

function createUserId(id: string): UserId {
  return id as UserId;
}

function getUser(id: UserId): User { ... }

const userId = createUserId("user_123");
const orderId = "order_456" as OrderId;

getUser(userId);    // OK
getUser(orderId);   // Error! OrderId is not assignable to UserId
getUser("raw");     // Error! string is not assignable to UserId
```

### Builder Pattern with Types

```typescript
class QueryBuilder<T extends object = {}> {
  private query: T;

  constructor(query: T = {} as T) {
    this.query = query;
  }

  where<K extends string, V>(key: K, value: V): QueryBuilder<T & Record<K, V>> {
    return new QueryBuilder({ ...this.query, [key]: value } as T & Record<K, V>);
  }

  build(): T {
    return this.query;
  }
}

const query = new QueryBuilder()
  .where("name", "Alice")       // type: { name: string }
  .where("age", 30)             // type: { name: string; age: number }
  .build();
// query: { name: string; age: number }
```

### Type-Safe Event Emitter

```typescript
type EventMap = {
  "user:login": { userId: string; timestamp: number };
  "user:logout": { userId: string };
  "error": { message: string; code: number };
};

class TypedEmitter<Events extends Record<string, any>> {
  private listeners = new Map<string, Set<Function>>();

  on<K extends keyof Events>(event: K, handler: (data: Events[K]) => void): void {
    if (!this.listeners.has(event as string)) {
      this.listeners.set(event as string, new Set());
    }
    this.listeners.get(event as string)!.add(handler);
  }

  emit<K extends keyof Events>(event: K, data: Events[K]): void {
    this.listeners.get(event as string)?.forEach(fn => fn(data));
  }
}

const emitter = new TypedEmitter<EventMap>();
emitter.on("user:login", (data) => {
  data.userId;     // string (autocomplete works!)
  data.timestamp;  // number
});
emitter.emit("user:login", { userId: "123", timestamp: Date.now() });
emitter.emit("user:login", { userId: "123" });  // Error! missing timestamp
```

---

## 17. tsconfig.json Key Options

```jsonc
{
  "compilerOptions": {
    // Strictness (ALWAYS enable all of these)
    "strict": true,                    // enables all strict checks below
    "noUncheckedIndexedAccess": true,  // arr[0] is T | undefined (not just T)
    "exactOptionalProperties": true,   // optional props can't be set to undefined
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,

    // Module system
    "module": "ESNext",                // or "NodeNext" for Node.js
    "moduleResolution": "bundler",     // or "NodeNext"
    "esModuleInterop": true,           // import fs from 'fs' (instead of import * as fs)

    // Output
    "target": "ES2022",                // which JS version to emit
    "outDir": "./dist",
    "declaration": true,               // emit .d.ts files
    "sourceMap": true,

    // Path aliases
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

---

## 18. 🧩 TRICKY OUTPUT: TypeScript Gotchas

```typescript
// Object literal excess property checking
interface User { name: string; age: number; }

// Direct assignment: excess properties are REJECTED
const user: User = { name: "Alice", age: 30, email: "a@b.com" };
// Error! 'email' does not exist in type 'User'

// Variable assignment: excess properties are ALLOWED
const data = { name: "Alice", age: 30, email: "a@b.com" };
const user2: User = data;  // OK! (structural typing — data has name + age)

// Type widening
let x = "hello";         // type: string (widened)
const y = "hello";       // type: "hello" (literal type, because const)

let z = "hello" as const; // type: "hello" (literal even with let)

// null vs undefined in strict mode
let a: string;
console.log(a);           // Error: used before being assigned
let b: string | undefined;
console.log(b);           // OK: undefined

// typeof on class
class Foo { static bar = 42; }
type FooInstance = Foo;           // the instance type
type FooClass = typeof Foo;      // the class/constructor type
const f: FooClass = Foo;         // OK
const instance: FooInstance = new Foo();  // OK
```

## 19. Function Overloads
 
```typescript
// Multiple signatures, single implementation
function createElement(tag: "div"): HTMLDivElement;
function createElement(tag: "a"): HTMLAnchorElement;
function createElement(tag: "canvas"): HTMLCanvasElement;
function createElement(tag: string): HTMLElement;
function createElement(tag: string): HTMLElement {
  return document.createElement(tag);
}
 
const div = createElement("div");       // HTMLDivElement (not just HTMLElement)
const link = createElement("a");        // HTMLAnchorElement
const unknown = createElement("span");  // HTMLElement
 
// Overloads with different param counts
function padding(all: number): string;
function padding(vertical: number, horizontal: number): string;
function padding(top: number, right: number, bottom: number, left: number): string;
function padding(a: number, b?: number, c?: number, d?: number): string {
  if (b === undefined) return `${a}px`;
  if (c === undefined) return `${a}px ${b}px`;
  return `${a}px ${b}px ${c}px ${d}px`;
}
 
padding(10);              // "10px"
padding(10, 20);          // "10px 20px"
padding(10, 20, 30, 40);  // "10px 20px 30px 40px"
padding(10, 20, 30);      // Error! No matching overload (3 args)
```
 
---
 
## 20. Abstract Classes
 
```typescript
abstract class Shape {
  abstract area(): number;            // must be implemented by subclasses
  abstract perimeter(): number;
 
  // Concrete method — inherited as-is
  describe(): string {
    return `${this.constructor.name}: area=${this.area().toFixed(2)}`;
  }
}
 
// new Shape();  // Error! Cannot instantiate abstract class
 
class Circle extends Shape {
  constructor(private radius: number) { super(); }
 
  area(): number { return Math.PI * this.radius ** 2; }
  perimeter(): number { return 2 * Math.PI * this.radius; }
}
 
// Can be used as a type
function printArea(shape: Shape): void {
  console.log(shape.area());
}
```
 
---
 
## 21. Access Modifiers and Parameter Properties
 
```typescript
class User {
  // Parameter properties — shorthand for declaring + assigning in constructor
  constructor(
    public name: string,              // creates this.name
    private password: string,         // creates this.password (compile-time only!)
    protected role: string,           // creates this.role (accessible in subclasses)
    readonly id: number,              // creates this.id (can't reassign after construction)
  ) {}
 
  // public:    accessible everywhere (default)
  // private:   accessible only within the class (TS-only, erased at runtime)
  // protected: accessible within class and subclasses
  // readonly:  can only be set in constructor
 
  // JS #private is TRULY private at runtime:
  #secret = "truly private";          // not accessible outside, even at runtime
}
 
const u = new User("Alice", "pass", "admin", 1);
u.name;         // "Alice" (public)
u.password;     // Error in TS, but accessible in JS runtime! (TS private is only compile-time)
u.role;         // Error (protected)
u.id = 2;       // Error (readonly)
```
 
---
 
## 22. The override Keyword (TS 4.3+)
 
```typescript
class Base {
  greet() { return "hello"; }
}
 
class Child extends Base {
  override greet() { return "hi"; }       // OK — overrides Base.greet
 
  override missing() { return "oops"; }   // Error! Base doesn't have 'missing'
  // Catches typos in method names when you THINK you're overriding
}
 
// Enable in tsconfig: "noImplicitOverride": true
// Then ALL overrides must be marked with `override` keyword
```
 
---
 
## 23. Recursive Types
 
```typescript
// JSON value
type JSONValue = string | number | boolean | null | JSONValue[] | { [key: string]: JSONValue };
 
// Nested tree
type TreeNode<T> = {
  value: T;
  children: TreeNode<T>[];
};
 
const tree: TreeNode<string> = {
  value: "root",
  children: [
    { value: "child1", children: [] },
    { value: "child2", children: [
      { value: "grandchild", children: [] },
    ]},
  ],
};
 
// Deep Partial (recursively makes all properties optional)
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};
 
// Deep Readonly
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};
 
// Recursive path type (dot-notation paths into an object)
type Path<T, Prefix extends string = ""> = T extends object
  ? { [K in keyof T & string]:
      | `${Prefix}${K}`
      | Path<T[K], `${Prefix}${K}.`>
    }[keyof T & string]
  : never;
 
type UserPaths = Path<{ name: string; address: { city: string; zip: string } }>;
// "name" | "address" | "address.city" | "address.zip"
```
 
---
 
## 24. Variadic Tuple Types (TS 4.0+)
 
```typescript
// Spread in tuple types
type Concat<A extends unknown[], B extends unknown[]> = [...A, ...B];
type Result = Concat<[1, 2], [3, 4]>;  // [1, 2, 3, 4]
 
// Typed function: first arg + rest
type Head<T extends unknown[]> = T extends [infer H, ...unknown[]] ? H : never;
type Tail<T extends unknown[]> = T extends [unknown, ...infer R] ? R : never;
 
type H = Head<[1, 2, 3]>;  // 1
type T = Tail<[1, 2, 3]>;  // [2, 3]
 
// Last element
type Last<T extends unknown[]> = T extends [...unknown[], infer L] ? L : never;
type L = Last<[1, 2, 3]>;  // 3
 
// Real-world: typed pipe function
function pipe<A, B>(fn1: (a: A) => B): (a: A) => B;
function pipe<A, B, C>(fn1: (a: A) => B, fn2: (b: B) => C): (a: A) => C;
function pipe<A, B, C, D>(fn1: (a: A) => B, fn2: (b: B) => C, fn3: (c: C) => D): (a: A) => D;
function pipe(...fns: Function[]) {
  return (x: any) => fns.reduce((v, fn) => fn(v), x);
}
 
const transform = pipe(
  (x: string) => x.length,          // string → number
  (x: number) => x > 5,             // number → boolean
);
// transform: (a: string) => boolean
```
 
---
 
## 25. Variance Annotations (TS 4.7+)
 
```typescript
// out = covariant (producer/output position)
// in  = contravariant (consumer/input position)
 
interface Producer<out T> {
  get(): T;              // T is only in output position
}
 
interface Consumer<in T> {
  accept(value: T): void; // T is only in input position
}
 
interface Transformer<in I, out O> {
  transform(input: I): O;
}
 
// Why it matters: variance affects assignability
// Producer<Dog> is assignable to Producer<Animal> (covariant — out)
// Consumer<Animal> is assignable to Consumer<Dog> (contravariant — in)
 
// Without annotations, TS infers variance but can be wrong for complex types
// Annotations make variance explicit and catch errors
```
 
---
 
## 26. const Type Parameters (TS 5.0+)
 
```typescript
// Without const: inferred as general types
function routes<T extends readonly string[]>(paths: T): T {
  return paths;
}
const r1 = routes(["home", "about"]);
// type: string[]  ← too wide
 
// With const: inferred as literal types
function routes2<const T extends readonly string[]>(paths: T): T {
  return paths;
}
const r2 = routes2(["home", "about"]);
// type: readonly ["home", "about"]  ← exact literal tuple
 
// Replaces the need for `as const` at call sites
```
 
---
 
## 27. Type-Only Imports and Exports
 
```typescript
// Import only the type (stripped at compile time — no runtime import)
import type { User } from "./types";
import { type User, createUser } from "./users";  // inline type import
 
// Export only the type
export type { User };
export { type User, createUser };
 
// Why it matters:
// 1. Makes it clear what's a type vs a value
// 2. Can prevent circular dependency issues (type-only imports have no runtime effect)
// 3. Bundlers can tree-shake more aggressively
// 4. isolatedModules mode requires explicit type-only imports for re-exports of types
```
 
---
 
## 28. Index Signatures vs Record (Deep)
 
```typescript
// Index signature — allow any string key
interface Dictionary {
  [key: string]: number;
}
const d: Dictionary = { a: 1, b: 2, anything: 3 };  // any key works
 
// BUT: all properties must match the index signature type
interface Mixed {
  [key: string]: number;
  name: string;           // Error! 'string' not assignable to 'number'
}
 
// Fix: union the index signature
interface Mixed {
  [key: string]: number | string;
  name: string;           // OK (string is in the union)
  count: number;          // OK
}
 
// Record — same thing but as a type alias
type Dict = Record<string, number>;
 
// Record with specific keys
type Scores = Record<"math" | "english" | "science", number>;
// { math: number; english: number; science: number }
 
// noUncheckedIndexedAccess (tsconfig)
// When enabled:
const dict: Dictionary = { a: 1 };
dict["b"];  // number | undefined (not just number)
// This is critical for safety — highly recommended
```
 
---
 
## 29. ThisType Utility
 
```typescript
// ThisType<T> controls what `this` is inside an object literal
 
type ObjectWithThis<T> = T & ThisType<T>;
 
function makeStore<S>(config: { state: S; actions: Record<string, (this: S) => void> & ThisType<S> }): S {
  const store = config.state;
  for (const [name, action] of Object.entries(config.actions)) {
    (store as any)[name] = action.bind(store);
  }
  return store;
}
 
const store = makeStore({
  state: { count: 0 },
  actions: {
    increment() {
      this.count++;     // `this` is typed as { count: number } — not `any`!
    },
    reset() {
      this.count = 0;
    },
  },
});
```
 
---
 
## 30. Mixins Pattern
 
```typescript
// TypeScript doesn't support multiple inheritance, but mixins simulate it
 
type Constructor<T = {}> = new (...args: any[]) => T;
 
// Mixin 1: Timestamped
function Timestamped<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    createdAt = new Date();
    updatedAt = new Date();
 
    touch() { this.updatedAt = new Date(); }
  };
}
 
// Mixin 2: Serializable
function Serializable<TBase extends Constructor>(Base: TBase) {
  return class extends Base {
    serialize(): string {
      return JSON.stringify(this);
    }
  };
}
 
// Compose mixins
class BaseUser {
  constructor(public name: string) {}
}
 
const TimestampedSerializableUser = Serializable(Timestamped(BaseUser));
const user = new TimestampedSerializableUser("Alice");
user.name;         // "Alice" (from BaseUser)
user.createdAt;    // Date (from Timestamped)
user.touch();      // (from Timestamped)
user.serialize();  // JSON string (from Serializable)
```
 
---
 
## 31. Using Declarations (Explicit Resource Management — TS 5.2+)
 
```typescript
// Like Python's `with` statement or C#'s `using`
// Object must implement Symbol.dispose or Symbol.asyncDispose
 
class DatabaseConnection implements Disposable {
  constructor() { console.log("Connected"); }
 
  [Symbol.dispose]() {
    console.log("Disconnected");    // cleanup when scope exits
  }
}
 
function doWork() {
  using conn = new DatabaseConnection();  // auto-disposed at end of scope
  // do work with conn...
}  // conn[Symbol.dispose]() called here automatically
 
// Async version
class AsyncResource implements AsyncDisposable {
  async [Symbol.asyncDispose]() {
    await cleanup();
  }
}
 
async function doAsyncWork() {
  await using resource = new AsyncResource();
  // ...
}  // resource[Symbol.asyncDispose]() called here
```
 
---
 
## 32. typeof in Type Context
 
```typescript
// typeof in expressions (JS) vs typeof in type positions (TS)
 
const config = {
  host: "localhost",
  port: 3000,
  debug: true,
};
 
// typeof in type position — extracts the TYPE from a value
type Config = typeof config;
// { host: string; port: number; debug: boolean }
 
// Useful for functions
function createUser(name: string, age: number) {
  return { name, age, createdAt: new Date() };
}
 
type User = ReturnType<typeof createUser>;
// { name: string; age: number; createdAt: Date }
 
// typeof with as const
const DIRECTIONS = ["north", "south", "east", "west"] as const;
type Direction = typeof DIRECTIONS[number];
// "north" | "south" | "east" | "west"
 
// typeof with enum
enum Color { Red, Green, Blue }
type ColorEnum = typeof Color;
// typeof Color is the enum OBJECT type (with reverse mapping etc.)
// Color is the enum VALUE type ("Red" | "Green" | "Blue" in this case, 0 | 1 | 2 for numeric)
 
// keyof typeof pattern — get enum keys as union
type ColorKey = keyof typeof Color;
// "Red" | "Green" | "Blue"
```
 
---
 
## 33. Covariance and Contravariance (Interview Favorite)
 
```typescript
class Animal { name = "animal"; }
class Dog extends Animal { breed = "labrador"; }
class GoldenRetriever extends Dog { color = "golden"; }
 
// Covariance (out): subtype can be used where supertype is expected
// "Producer position" — return types are covariant
type Producer<T> = () => T;
const dogProducer: Producer<Dog> = () => new Dog();
const animalProducer: Producer<Animal> = dogProducer;  // OK: Dog is a subtype of Animal
// Reading from a Producer<Dog> always gives you at least an Animal
 
// Contravariance (in): supertype can be used where subtype is expected
// "Consumer position" — parameter types are contravariant
type Consumer<T> = (value: T) => void;
const animalConsumer: Consumer<Animal> = (a: Animal) => console.log(a.name);
const dogConsumer: Consumer<Dog> = animalConsumer;  // OK: can handle any Dog (because it handles any Animal)
// A function that accepts Animal can safely accept Dog
 
// Invariance: neither covariant nor contravariant
// Mutable containers are invariant (both producer and consumer)
// Array<Dog> is NOT safely assignable to Array<Animal> (you could push a Cat)
// TS is lenient here by design (allows it, unlike Java)
 
// strictFunctionTypes (tsconfig) — enforces correct contravariance for function parameters
// Without it, TS allows bivariant function params (unsafe but convenient)
```
 
---
 
## 34. NoInfer Utility Type (TS 5.4+)
 
```typescript
// Prevents a type parameter from being inferred from a specific position
 
function createFSM<S extends string>(config: {
  initial: NoInfer<S>;              // don't infer S from here
  states: Record<S, { on: Record<string, NoInfer<S>> }>;
}) { ... }
 
createFSM({
  initial: "idle",                  // without NoInfer: S inferred from here too, weakening the type
  states: {
    idle: { on: { start: "running" } },
    running: { on: { stop: "idle" } },
  },
  // With NoInfer: "idle" must be one of the keys in states
  // Without NoInfer: "idle" just inferred as string, no validation
});
```
 
---
 
## 35. Advanced Conditional Type Patterns
 
```typescript
// Flatten nested arrays to any depth
type Flatten<T> = T extends (infer U)[]
  ? Flatten<U>    // recursively flatten
  : T;
 
type A = Flatten<number[][][]>;  // number
 
// Check if types are equal (not just assignable)
type IsExact<A, B> = [A] extends [B]
  ? [B] extends [A]
    ? true
    : false
  : false;
 
type Test1 = IsExact<string, string>;          // true
type Test2 = IsExact<string, string | number>; // false
 
// Get all required keys
type RequiredKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? never : K;
}[keyof T];
 
interface User { name: string; age: number; email?: string; }
type RK = RequiredKeys<User>;  // "name" | "age"
 
// Get all optional keys
type OptionalKeys<T> = {
  [K in keyof T]-?: {} extends Pick<T, K> ? K : never;
}[keyof T];
 
type OK = OptionalKeys<User>;  // "email"
 
// Make specific properties required
type RequireSome<T, K extends keyof T> = T & Required<Pick<T, K>>;
type UserWithEmail = RequireSome<User, "email">;
// User & { email: string }
 
// String to Union
type StringToUnion<S extends string> = S extends `${infer C}${infer Rest}`
  ? C | StringToUnion<Rest>
  : never;
 
type Chars = StringToUnion<"abc">;  // "a" | "b" | "c"
```
 
---
 
## 36. Common Error Patterns and Fixes
 
```typescript
// Error: "Type 'string' is not assignable to type 'specific string'"
const status = "active";                     // type: string (widened)
const config: { status: "active" | "inactive" } = { status };  // Error!
// Fix 1:
const status2 = "active" as const;           // type: "active"
// Fix 2:
const config2 = { status: "active" as const };
 
// Error: "Property does not exist on type 'never'"
// Usually means: unreachable code (e.g., you narrowed all possibilities)
// Or: you have a type that reduced to never via bad intersection
 
// Error: "Object is possibly 'undefined'"
// Fix: optional chaining, nullish coalescing, or type guard
const name = user?.profile?.name ?? "Unknown";
 
// Error: "Type 'X' is not assignable to type 'Y'" with complex generics
// Common fix: add explicit type arguments
const result = fn<SpecificType>(args);
 
// Error: "Excessive stack depth comparing types"
// Your recursive type is too deep — add a depth limiter
type DeepPartial<T, Depth extends number[] = []> =
  Depth["length"] extends 10
    ? T
    : T extends object
      ? { [K in keyof T]?: DeepPartial<T[K], [...Depth, 0]> }
      : T;
 
// Error: "Declaration emit for this file requires type that is not accessible"
// Fix: export the type, or use `satisfies` instead of a return type annotation
```
 
---
 
## 37. 🧩 TRICKY: TypeScript Interview Questions
 
```typescript
// Q: What's the difference between these?
type A = { a: string } & { b: number };
interface B extends C { b: number }
interface C { a: string }
// A and B have the same SHAPE, but:
// - Intersection (A): conflicting properties → never. { a: string } & { a: number } → { a: never }
// - Interface extends (B): conflicting properties → compile error
// In interview: "Intersections silently merge to never; extends catches conflicts."
 
// Q: What's the type of `x`?
const x = [1, "hello", true];
// Answer: (string | number | boolean)[]
// NOT [number, string, boolean] — without `as const`, it's a general array
 
const y = [1, "hello", true] as const;
// Answer: readonly [1, "hello", true] — literal tuple
 
// Q: Will this compile?
function test(x: string): void;
function test(x: number): void;
function test(x: string | number): void {}
test(true);
// Answer: NO — boolean doesn't match any overload signature
// Even though the implementation accepts string | number, overloads are what callers see
 
// Q: Structural typing gotcha
class Celsius { constructor(public value: number) {} }
class Fahrenheit { constructor(public value: number) {} }
 
const temp: Celsius = new Fahrenheit(100);  // NO ERROR! Same structure = compatible
// This is why branded types exist (see original cheatsheet #16)
```
