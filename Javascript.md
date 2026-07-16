# JavaScript — Complete Language Mastery Cheatsheet
*Intern → Super Senior · Every data type, method, gotcha, and engine internal*

---

## 1. Types and Type System

### The 8 Types

JavaScript has 7 primitive types and 1 non-primitive type:

```
Primitives (immutable, stored by value):
  string      "hello"
  number      42, 3.14, NaN, Infinity
  bigint      9007199254740991n
  boolean     true, false
  undefined   undefined
  null        null
  symbol      Symbol("id")

Non-primitive (mutable, stored by reference):
  object      {}, [], function(){}, new Date(), /regex/, etc.
```

### typeof Operator (and Its Lies)

```javascript
typeof "hello"       // "string"
typeof 42            // "number"
typeof 42n           // "bigint"
typeof true          // "boolean"
typeof undefined     // "undefined"
typeof Symbol("x")   // "symbol"
typeof {}            // "object"
typeof []            // "object"     ← array is an object
typeof function(){}  // "function"   ← special case (still an object)
typeof null          // "object"     ← BUG from JS v1, never fixed
typeof NaN           // "number"     ← NaN is a number!
```

**How to properly check types:**
```javascript
Array.isArray([1,2])              // true (only reliable array check)
value === null                    // null check (don't use typeof)
value === undefined               // undefined check
Number.isNaN(NaN)                 // true (don't use global isNaN)
value instanceof Date             // true for Date objects
Object.prototype.toString.call(value)  // "[object Array]", "[object Null]", etc. — most reliable
```

### Primitives vs Objects (Value vs Reference)

```javascript
// Primitives — copied by value
let a = 10;
let b = a;
b = 20;
console.log(a);  // 10 (unchanged)

// Objects — copied by reference
let obj1 = { name: "Alice" };
let obj2 = obj1;           // obj2 points to the SAME object
obj2.name = "Bob";
console.log(obj1.name);    // "Bob" (both point to same memory)

// How to actually copy an object
let shallow = { ...obj1 };                    // spread (shallow copy)
let shallow2 = Object.assign({}, obj1);       // Object.assign (shallow copy)
let deep = structuredClone(obj1);             // deep copy (modern, handles circular refs)
let deep2 = JSON.parse(JSON.stringify(obj1)); // deep copy (loses functions, dates, undefined)
```

---

## 2. Coercion and Equality

### Type Coercion Rules

```javascript
// String coercion (anything + string = string)
"5" + 3          // "53"       (number → string)
"5" + true       // "5true"
"5" + null       // "5null"
"5" + undefined  // "5undefined"
"5" + {}         // "5[object Object]"
"5" + []         // "5"        ([] → "")

// Number coercion (math operators except +)
"5" - 3          // 2          (string → number)
"5" * 2          // 10
"5" / 2          // 2.5
true + 1         // 2          (true → 1)
false + 1        // 1          (false → 0)
null + 1         // 1          (null → 0)
undefined + 1    // NaN        (undefined → NaN)
"" + 1           // "1"        (+ with string = concatenation)
"" - 1           // -1         (- forces number coercion)

// Boolean coercion
Boolean(0)           // false
Boolean("")          // false
Boolean(null)        // false
Boolean(undefined)   // false
Boolean(NaN)         // false
Boolean(false)       // false
// Everything else is truthy, including:
Boolean("0")         // true   ← common gotcha
Boolean(" ")         // true
Boolean([])          // true   ← empty array is truthy!
Boolean({})          // true   ← empty object is truthy!
```

### == vs === (Abstract vs Strict Equality)

```javascript
// === (strict): no coercion, types must match
5 === "5"        // false
null === undefined // false
NaN === NaN      // false  ← NaN is not equal to itself!

// == (abstract): coerces types, then compares
5 == "5"         // true   (string → number)
null == undefined // true  ← special rule: null only equals undefined
0 == false       // true   (false → 0)
"" == false      // true   (both → 0)
[] == false      // true   ([] → "" → 0 → false)
[] == ![]        // true   ← infamous gotcha

// How [] == ![] is true:
// ![] → false ([] is truthy, so ![] = false)
// [] == false
// [] → "" → 0, false → 0
// 0 == 0 → true

// Use Object.is() for edge cases
Object.is(NaN, NaN)     // true  (unlike ===)
Object.is(0, -0)        // false (unlike ===)
```

### 🧩 TRICKY OUTPUT: Coercion Madness

```javascript
console.log([] + []);       // ""        (both → "", "" + "" = "")
console.log([] + {});       // "[object Object]"
console.log({} + []);       // 0 or "[object Object]" depending on context
                            // In browser console: {} is treated as empty block, +[] → 0
                            // In expression: ({} + []) → "[object Object]"

console.log(true + true + true);  // 3
console.log(true - true);         // 0
console.log("b" + "a" + +"a" + "a");  // "baNaNa"  (+"a" → NaN)

console.log(!!"false");    // true  (non-empty string is truthy)
console.log(!!"");         // false
console.log(!!0);          // false
console.log(!!null);       // false
console.log(!!undefined);  // false
console.log(!!NaN);        // false
```

---

## 3. Numbers

### Number Type Internals

JavaScript uses 64-bit IEEE 754 floating point for all numbers. This means:

```javascript
// Floating point precision
0.1 + 0.2 === 0.3        // false!
0.1 + 0.2                // 0.30000000000000004

// Safe integer range
Number.MAX_SAFE_INTEGER   // 9007199254740991 (2^53 - 1)
Number.MIN_SAFE_INTEGER   // -9007199254740991
Number.isSafeInteger(9007199254740992)  // false

// Special values
Infinity                  // 1/0
-Infinity                 // -1/0
NaN                       // 0/0, parseInt("abc"), Math.sqrt(-1)

// NaN is the only value not equal to itself
NaN === NaN               // false
NaN == NaN                // false
Number.isNaN(NaN)         // true  (use this, not global isNaN)
isNaN("hello")            // true  ← BAD: coerces to number first
Number.isNaN("hello")     // false ← GOOD: no coercion
```

### Number Methods

```javascript
// Parsing
parseInt("42px")          // 42    (stops at first non-digit)
parseInt("0xFF", 16)      // 255   (hex)
parseInt("111", 2)        // 7     (binary)
parseFloat("3.14abc")     // 3.14
Number("42")              // 42    (strict: Number("42px") → NaN)
+"42"                     // 42    (unary + is shorthand for Number())

// Formatting
(3.14159).toFixed(2)      // "3.14"   (string! rounds)
(1234.5).toLocaleString() // "1,234.5" (locale-aware)
(255).toString(16)        // "ff"     (to hex string)
(7).toString(2)           // "111"    (to binary string)

// Checking
Number.isInteger(42)      // true
Number.isInteger(42.0)    // true  (42.0 === 42 in JS)
Number.isFinite(Infinity) // false
Number.isFinite(42)       // true
```

### BigInt

```javascript
const big = 9007199254740991n;       // n suffix
const big2 = BigInt("999999999999999999999999");

big + 1n          // 9007199254740992n
// big + 1        // TypeError! Can't mix BigInt and Number
Number(big)       // 9007199254740991 (may lose precision)
```

### Math Object

```javascript
Math.floor(4.7)      // 4    (round down)
Math.ceil(4.1)       // 5    (round up)
Math.round(4.5)      // 5    (round to nearest, .5 rounds up)
Math.trunc(4.7)      // 4    (remove decimal, no rounding)
Math.abs(-5)         // 5
Math.max(1, 2, 3)    // 3
Math.min(1, 2, 3)    // 1
Math.pow(2, 10)      // 1024  (or 2 ** 10)
Math.sqrt(16)        // 4
Math.random()        // 0 ≤ x < 1
Math.floor(Math.random() * 10)  // random int 0-9
Math.sign(-5)        // -1 (returns -1, 0, or 1)
Math.clamp(15, 0, 10) // not built-in, use: Math.min(Math.max(value, min), max)
```

---

## 4. Strings

### String Internals

Strings are immutable sequences of UTF-16 code units. Every string method returns a new string.

```javascript
const str = "Hello, World!";

// Properties
str.length               // 13

// Access
str[0]                   // "H"
str.charAt(0)            // "H"
str.at(-1)               // "!"  (negative indexing, ES2022)
str.charCodeAt(0)        // 72   (UTF-16 code unit)
str.codePointAt(0)       // 72   (full Unicode code point — handles emoji)

// Search
str.indexOf("World")     // 7    (-1 if not found)
str.lastIndexOf("l")     // 10
str.includes("World")    // true
str.startsWith("Hello")  // true
str.endsWith("!")        // true
str.search(/world/i)     // 7    (regex search, returns index)

// Extract
str.slice(7, 12)         // "World"    (start, end — supports negative)
str.substring(7, 12)     // "World"    (start, end — no negative)
str.slice(-6)            // "orld!"    (last 6 chars)

// Transform
str.toUpperCase()        // "HELLO, WORLD!"
str.toLowerCase()        // "hello, world!"
str.trim()               // remove whitespace from both ends
str.trimStart()          // remove from start
str.trimEnd()            // remove from end
str.padStart(20, "*")    // "*******Hello, World!"
str.padEnd(20, "*")      // "Hello, World!*******"
str.repeat(3)            // "Hello, World!Hello, World!Hello, World!"
str.replace("World", "JS")       // "Hello, JS!"   (first occurrence only)
str.replaceAll("l", "L")         // "HeLLo, WorLd!" (all occurrences, ES2021)
str.replace(/l/g, "L")           // "HeLLo, WorLd!" (regex global — older way)

// Split / Join
"a,b,c".split(",")      // ["a", "b", "c"]
"hello".split("")        // ["h", "e", "l", "l", "o"]
["a", "b", "c"].join("-") // "a-b-c"

// Template literals
const name = "World";
const greeting = `Hello, ${name}!`;   // "Hello, World!"
const multiline = `line 1
line 2
line 3`;

// Tagged templates
function highlight(strings, ...values) {
  return strings.reduce((result, str, i) => {
    return result + str + (values[i] ? `<b>${values[i]}</b>` : '');
  }, '');
}
const msg = highlight`Hello ${name}, you have ${5} messages`;
// "Hello <b>World</b>, you have <b>5</b> messages"

// String.raw (no escape processing)
String.raw`\n\t`         // "\\n\\t" (literal backslashes)
```

### 🧩 TRICKY OUTPUT: Strings

```javascript
console.log("1" + 2 + 3);     // "123"  (left-to-right: "1"+2="12", "12"+3="123")
console.log(1 + 2 + "3");     // "33"   (left-to-right: 1+2=3, 3+"3"="33")
console.log("5" - 3);         // 2      (- forces number coercion)
console.log("5" + - + - "3"); // "53"   (- + - "3" → 3, "5"+3="53")... wait:
// Actually: +"3" → 3, -3, -(−3) → 3... but + in "5" + ... is string concat
// So: "5" + (-(+(-"3"))) → "5" + (-(+(-3))) → "5" + (-(3)) → "5" + (-3) → "5-3"
// Let me be precise: "5" + - + - "3" → evaluate right: -"3" = -3, +(-3) = -3, -(-3) = 3 → "5" + 3 = "53"
```

---

## 5. Arrays

### Array Internals

Arrays in JS are objects with numeric keys and a magic `length` property. They're not contiguous memory blocks (though V8 optimizes dense arrays).

### Creation

```javascript
const arr = [1, 2, 3];
const arr2 = new Array(5);        // [empty × 5] — sparse, length 5
const arr3 = Array.of(5);         // [5] — single element
const arr4 = Array.from("hello"); // ["h", "e", "l", "l", "o"]
const arr5 = Array.from({ length: 5 }, (_, i) => i * 2);  // [0, 2, 4, 6, 8]
const arr6 = Array(5).fill(0);    // [0, 0, 0, 0, 0]
```

### Mutating Methods (modify the original array)

```javascript
const a = [1, 2, 3, 4, 5];

// Add/Remove
a.push(6)           // [1,2,3,4,5,6]    returns new length (6)
a.pop()              // [1,2,3,4,5]      returns removed element (6)
a.unshift(0)         // [0,1,2,3,4,5]    returns new length (6)
a.shift()            // [1,2,3,4,5]      returns removed element (0)
a.splice(2, 1)       // [1,2,4,5]        remove 1 element at index 2, returns [3]
a.splice(2, 0, 3)    // [1,2,3,4,5]      insert 3 at index 2
a.splice(2, 1, 30)   // [1,2,30,4,5]     replace index 2 with 30

// Reorder
a.sort()             // sorts IN PLACE (lexicographic by default!)
[10, 9, 1].sort()    // [1, 10, 9] ← WRONG! Sorts as strings
[10, 9, 1].sort((a, b) => a - b)  // [1, 9, 10] ← correct numeric sort
a.reverse()          // reverses IN PLACE
a.fill(0, 2, 4)      // fill with 0 from index 2 to 4
a.copyWithin(0, 3)   // copy elements from index 3 to position 0
```

### Non-Mutating Methods (return new array/value)

```javascript
const a = [1, 2, 3, 4, 5];

// Iteration
a.forEach((val, idx, arr) => { ... })    // no return value, can't break
a.map(x => x * 2)                        // [2, 4, 6, 8, 10]
a.filter(x => x > 3)                     // [4, 5]
a.reduce((acc, x) => acc + x, 0)         // 15
a.reduceRight((acc, x) => acc + x, 0)    // 15 (right to left)

// Search
a.find(x => x > 3)          // 4      (first match, or undefined)
a.findIndex(x => x > 3)     // 3      (index of first match, or -1)
a.findLast(x => x > 3)      // 5      (ES2023)
a.findLastIndex(x => x > 3) // 4      (ES2023)
a.indexOf(3)                 // 2      (-1 if not found)
a.lastIndexOf(3)             // 2
a.includes(3)                // true

// Test
a.every(x => x > 0)         // true   (all elements pass test)
a.some(x => x > 4)          // true   (at least one passes)

// Transform
a.flat()                     // flatten 1 level: [[1,2],[3,[4]]].flat() → [1,2,3,[4]]
a.flat(Infinity)             // flatten all levels
a.flatMap(x => [x, x*2])    // [1,2,2,4,3,6,4,8,5,10]  (map then flat(1))
a.slice(1, 3)                // [2, 3]  (start, end — doesn't mutate)
a.concat([6, 7])             // [1,2,3,4,5,6,7]
a.join(", ")                 // "1, 2, 3, 4, 5"
a.toReversed()               // [5,4,3,2,1] (ES2023, non-mutating reverse)
a.toSorted((a,b) => a-b)    // non-mutating sort (ES2023)
a.toSpliced(2, 1)            // non-mutating splice (ES2023)
a.with(2, 99)                // [1,2,99,4,5] (non-mutating replacement, ES2023)

// Destructuring
const [first, second, ...rest] = a;   // first=1, second=2, rest=[3,4,5]
const [,, third] = a;                 // third=3 (skip first two)

// Spread
const combined = [...a, ...b];
const copy = [...a];                  // shallow copy
```

### 🧩 TRICKY OUTPUT: Arrays

```javascript
const arr = [1, 2, 3];
arr[10] = 99;
console.log(arr.length);    // 11    (sparse array, indices 3-9 are empty)
console.log(arr[5]);         // undefined (empty slot)

// forEach skips empty slots, map preserves them
[1, , 3].forEach(x => console.log(x));  // 1, 3 (skips empty)
[1, , 3].map(x => x * 2);               // [2, empty, 6]

// sort is lexicographic by default
console.log([10, 9, 80].sort());         // [10, 80, 9]  ← not numeric!

// Array comparison
console.log([1,2,3] === [1,2,3]);        // false (different references)
console.log([1,2,3] == [1,2,3]);         // false (still different references)

// delete creates holes (don't use it on arrays)
const x = [1, 2, 3];
delete x[1];
console.log(x);         // [1, empty, 3]
console.log(x.length);  // 3 (unchanged!)
```

---

## 6. Objects

### Creation and Access

```javascript
const obj = {
  name: "Alice",
  age: 30,
  "multi-word": true,     // quoted keys for special characters
  [Symbol("id")]: 123,    // computed property with symbol
  greet() { return `Hi, I'm ${this.name}`; },  // method shorthand
};

// Access
obj.name                  // "Alice"
obj["multi-word"]         // true (bracket notation for special keys)
obj?.address?.city        // undefined (optional chaining — no error if null/undefined)

// Destructuring
const { name, age, job = "unknown" } = obj;  // job defaults to "unknown"
const { name: userName } = obj;               // rename: userName = "Alice"
const { name, ...rest } = obj;               // rest = { age: 30, ... }

// Spread
const extended = { ...obj, job: "Engineer" };  // shallow copy + add/override
```

### Object Methods (Static)

```javascript
// Keys, values, entries
Object.keys(obj)              // ["name", "age", "multi-word", "greet"]
Object.values(obj)            // ["Alice", 30, true, ƒ]
Object.entries(obj)           // [["name","Alice"], ["age",30], ...]
Object.fromEntries([["a",1],["b",2]])  // { a: 1, b: 2 }

// Merge
Object.assign(target, source1, source2)  // merge into target (mutates target)
Object.assign({}, obj1, obj2)            // merge without mutation

// Property descriptors
Object.getOwnPropertyDescriptor(obj, "name")
// { value: "Alice", writable: true, enumerable: true, configurable: true }

Object.defineProperty(obj, "id", {
  value: 42,
  writable: false,      // can't change value
  enumerable: false,    // won't appear in for...in or Object.keys
  configurable: false,  // can't delete or reconfigure
});

// Freeze / Seal
Object.freeze(obj)      // can't add, remove, or modify properties (shallow!)
Object.seal(obj)        // can't add or remove, but CAN modify existing values
Object.isFrozen(obj)    // true
Object.isSealed(obj)    // true

// Prototype
Object.getPrototypeOf(obj)
Object.create(proto)              // create object with specified prototype
Object.hasOwn(obj, "name")       // true (ES2022, replaces obj.hasOwnProperty)
"name" in obj                     // true (checks own + prototype chain)
obj.hasOwnProperty("name")       // true (own only, older method)
```

### Property Shorthand and Computed Properties

```javascript
const x = 10, y = 20;
const point = { x, y };              // { x: 10, y: 20 }

const key = "dynamic";
const dynamic = { [key]: "value" };  // { dynamic: "value" }
const methods = { [`get${key}`]() { return this[key]; } };
```

### 🧩 TRICKY OUTPUT: Objects

```javascript
const a = { x: 1 };
const b = { x: 1 };
console.log(a === b);      // false (different references)
console.log(a == b);       // false (same — no coercion for objects vs objects)

const obj = { a: 1, b: 2, a: 3 };
console.log(obj);          // { a: 3, b: 2 } — last value wins

// Object keys are always strings (or symbols)
const obj2 = {};
obj2[1] = "one";
obj2["1"] = "ONE";
console.log(obj2[1]);      // "ONE" (1 is coerced to "1")
console.log(Object.keys(obj2));  // ["1"]
```

---

## 7. Functions

### Function Declarations vs Expressions

```javascript
// Declaration — hoisted (can be called before definition)
greet();  // works!
function greet() { return "hello"; }

// Expression — NOT hoisted
hello();  // TypeError: hello is not a function
var hello = function() { return "hello"; };

// Arrow function — NOT hoisted, no own `this`, no `arguments`
const add = (a, b) => a + b;
const square = x => x * x;       // single param: no parens needed
const getObj = () => ({ a: 1 });  // return object: wrap in parens
```

### `this` Binding Rules (Ranked by Priority)

```javascript
// 1. new binding (constructor)
function Person(name) { this.name = name; }
const p = new Person("Alice");   // this = new empty object

// 2. Explicit binding (call, apply, bind)
function greet() { return this.name; }
greet.call({ name: "Alice" });       // "Alice"
greet.apply({ name: "Alice" });      // "Alice" (same, but args as array)
const bound = greet.bind({ name: "Alice" });
bound();                              // "Alice"

// 3. Implicit binding (method call)
const obj = { name: "Alice", greet() { return this.name; } };
obj.greet();                          // "Alice" (this = obj)
const fn = obj.greet;
fn();                                 // undefined (this = global/undefined in strict mode)
                                      // ← common bug: losing `this` when extracting methods

// 4. Default binding
function whoAmI() { return this; }
whoAmI();                             // window (browser) or global (Node) in non-strict
                                      // undefined in strict mode

// Arrow functions: NO own `this` — inherits from enclosing scope
const obj2 = {
  name: "Alice",
  greet: () => this.name,             // `this` is NOT obj2, it's the outer scope
  greetCorrect() {
    const inner = () => this.name;    // arrow inherits `this` from greetCorrect
    return inner();
  },
};
obj2.greet();         // undefined (arrow's this = outer scope, not obj2)
obj2.greetCorrect();  // "Alice"
```

### call, apply, bind

```javascript
function introduce(greeting, punctuation) {
  return `${greeting}, I'm ${this.name}${punctuation}`;
}

const person = { name: "Alice" };

// call: comma-separated arguments
introduce.call(person, "Hello", "!");      // "Hello, I'm Alice!"

// apply: arguments as array
introduce.apply(person, ["Hi", "."]);      // "Hi, I'm Alice."

// bind: returns a new function with `this` permanently bound
const aliceIntro = introduce.bind(person);
aliceIntro("Hey", "!");                    // "Hey, I'm Alice!"

// bind with partial application
const aliceHello = introduce.bind(person, "Hello");
aliceHello("!");                           // "Hello, I'm Alice!"
```

### arguments Object

```javascript
function sum() {
  // `arguments` is array-LIKE (has length, indexable, but not a real array)
  console.log(arguments);        // { 0: 1, 1: 2, 2: 3, length: 3 }
  return [...arguments].reduce((a, b) => a + b, 0);
}
sum(1, 2, 3);  // 6

// Arrow functions do NOT have `arguments`
const arrowSum = () => arguments;  // ReferenceError (or captures outer arguments)

// Modern alternative: rest parameters
const modernSum = (...args) => args.reduce((a, b) => a + b, 0);
```

### Default Parameters, Rest, Spread

```javascript
// Default parameters
function greet(name = "World", greeting = `Hello, ${name}`) {
  return greeting;
}
greet();            // "Hello, World"
greet("Alice");     // "Hello, Alice"

// Rest parameters (must be last)
function sum(first, ...rest) {
  return rest.reduce((acc, n) => acc + n, first);
}

// Spread in function calls
const nums = [1, 2, 3];
Math.max(...nums);  // 3
```

### 🧩 TRICKY OUTPUT: Functions

```javascript
// Hoisting
console.log(a);     // undefined (var is hoisted, value is not)
console.log(b);     // ReferenceError (let is in TDZ)
console.log(c());   // "hello" (function declarations are fully hoisted)

var a = 1;
let b = 2;
function c() { return "hello"; }

// IIFE
(function() {
  var x = 10;
  console.log(x);   // 10
})();
// console.log(x);  // ReferenceError (x is scoped to the IIFE)

// this in setTimeout
const obj = {
  name: "Alice",
  delayed() {
    setTimeout(function() {
      console.log(this.name);  // undefined (this = global in non-strict)
    }, 100);
    setTimeout(() => {
      console.log(this.name);  // "Alice" (arrow inherits this from delayed)
    }, 100);
  }
};
```

---

## 8. Scope, Hoisting, and Closures

### Scope Types

```javascript
// Global scope
var globalVar = "I'm global";

// Function scope (var)
function example() {
  var functionVar = "I'm function-scoped";
  if (true) {
    var stillFunctionScoped = "I'm NOT block-scoped";
  }
  console.log(stillFunctionScoped);  // works! var ignores blocks
}

// Block scope (let, const)
if (true) {
  let blockLet = "I'm block-scoped";
  const blockConst = "Me too";
}
// console.log(blockLet);  // ReferenceError
```

### Hoisting Deep Dive

```javascript
// var: declaration hoisted, initialization NOT hoisted
console.log(x);    // undefined (not ReferenceError — x exists but has no value)
var x = 5;
console.log(x);    // 5

// Equivalent to:
var x;             // hoisted to top
console.log(x);    // undefined
x = 5;

// let/const: hoisted but in Temporal Dead Zone (TDZ)
console.log(y);    // ReferenceError: Cannot access 'y' before initialization
let y = 5;
// The variable exists (hoisted) but accessing it before the let statement throws.

// Function declarations: FULLY hoisted (name + body)
sayHi();           // "Hi!" — works
function sayHi() { console.log("Hi!"); }

// Function expressions: only the var is hoisted
sayBye();          // TypeError: sayBye is not a function
var sayBye = function() { console.log("Bye!"); };
// Because: var sayBye; is hoisted → sayBye = undefined → calling undefined() → TypeError
```

### Closures

A closure is a function that remembers the variables from its outer scope even after the outer function has finished executing.

```javascript
function createCounter() {
  let count = 0;  // enclosed in the closure
  return {
    increment() { return ++count; },
    decrement() { return --count; },
    getCount() { return count; },
  };
}
const counter = createCounter();
counter.increment();  // 1
counter.increment();  // 2
counter.getCount();   // 2
// `count` is private — can't be accessed directly

// Closure in a loop (classic gotcha)
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 3, 3, 3  (all callbacks share the same `i`, which is 3 after the loop)

// Fix 1: use `let` (block-scoped, creates new binding per iteration)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 0, 1, 2

// Fix 2: IIFE (creates a new scope per iteration)
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 100);
  })(i);
}
// Prints: 0, 1, 2
```

### 🧩 TRICKY OUTPUT: Closures

```javascript
function createFunctions() {
  const result = [];
  for (var i = 0; i < 3; i++) {
    result.push(function() { return i; });
  }
  return result;
}
const fns = createFunctions();
console.log(fns[0]());  // 3
console.log(fns[1]());  // 3
console.log(fns[2]());  // 3
// All return 3 because they all close over the same `i` (var is function-scoped)
```

---

## 9. Prototypes and Inheritance

### Prototype Chain

```javascript
const animal = {
  type: "animal",
  speak() { return `${this.name} makes a sound`; },
};

const dog = Object.create(animal);  // dog's prototype = animal
dog.name = "Rex";
dog.bark = function() { return "Woof!"; };

dog.bark();          // "Woof!"      (own method)
dog.speak();         // "Rex makes a sound" (inherited from animal)
dog.type;            // "animal"     (inherited)
dog.hasOwnProperty("name");   // true
dog.hasOwnProperty("type");   // false (it's on the prototype)

// Prototype chain: dog → animal → Object.prototype → null
```

### Classes (ES6 Syntax Sugar over Prototypes)

```javascript
class Animal {
  #sound;                    // private field (ES2022)
  static count = 0;          // static property

  constructor(name, sound) {
    this.name = name;
    this.#sound = sound;
    Animal.count++;
  }

  speak() {                  // prototype method
    return `${this.name} says ${this.#sound}`;
  }

  get info() {               // getter
    return `${this.name} (${this.constructor.name})`;
  }

  set nickname(value) {      // setter
    this._nickname = value.trim();
  }

  static create(name, sound) {  // static method
    return new Animal(name, sound);
  }
}

class Dog extends Animal {
  constructor(name) {
    super(name, "Woof");     // MUST call super before using `this`
  }

  speak() {                  // override
    return `${super.speak()}! (tail wagging)`;
  }
}

const rex = new Dog("Rex");
rex.speak();             // "Rex says Woof! (tail wagging)"
rex instanceof Dog;      // true
rex instanceof Animal;   // true
Animal.count;            // 1
```

---

## 10. Asynchronous JavaScript

### Event Loop

```
┌───────────────────────────────────────────────────────┐
│                     Call Stack                         │
│         (synchronous code executes here)               │
└───────────────┬───────────────────────────────────────┘
                │
    ┌───────────▼──────────────────────────────────┐
    │              Event Loop                       │
    │  Checks queues after call stack is empty      │
    │                                               │
    │  Priority: Microtasks FIRST, then Macrotasks  │
    └───────────┬──────────────────┬───────────────┘
                │                  │
    ┌───────────▼──────┐  ┌───────▼────────────┐
    │   Microtask      │  │   Macrotask        │
    │   Queue          │  │   Queue            │
    │                  │  │                    │
    │  • Promise.then  │  │  • setTimeout      │
    │  • async/await   │  │  • setInterval     │
    │  • queueMicro-   │  │  • setImmediate    │
    │    task()        │  │  • I/O callbacks   │
    │  • MutationObs   │  │  • UI rendering    │
    └──────────────────┘  └────────────────────┘

    After EVERY macrotask: drain ALL microtasks before next macrotask.
```

### 🧩 TRICKY OUTPUT: Event Loop

```javascript
console.log("1");                          // sync → runs first

setTimeout(() => console.log("2"), 0);     // macrotask

Promise.resolve().then(() => console.log("3"));  // microtask

console.log("4");                          // sync

// Output: 1, 4, 3, 2
// Why: sync (1, 4) → microtask (3) → macrotask (2)
```

```javascript
// Harder version
console.log("start");

setTimeout(() => {
  console.log("timeout 1");
  Promise.resolve().then(() => console.log("promise inside timeout"));
}, 0);

Promise.resolve().then(() => {
  console.log("promise 1");
  setTimeout(() => console.log("timeout inside promise"), 0);
});

Promise.resolve().then(() => console.log("promise 2"));

console.log("end");

// Output:
// start
// end
// promise 1
// promise 2
// timeout 1
// promise inside timeout
// timeout inside promise
```

### Callbacks → Promises → Async/Await

```javascript
// Callback (old pattern — callback hell)
fs.readFile("file.txt", (err, data) => {
  if (err) throw err;
  fs.writeFile("copy.txt", data, (err) => {
    if (err) throw err;
    console.log("Done");
  });
});

// Promise (chainable, better error handling)
const promise = new Promise((resolve, reject) => {
  setTimeout(() => resolve("data"), 1000);
});

promise
  .then(data => process(data))
  .then(result => save(result))
  .catch(err => console.error(err))
  .finally(() => cleanup());

// Async/Await (syntactic sugar over promises)
async function main() {
  try {
    const data = await readFile("file.txt");
    const result = await process(data);
    await save(result);
  } catch (err) {
    console.error(err);
  } finally {
    cleanup();
  }
}
```

### Promise Static Methods

```javascript
// Promise.all — all must succeed (fail-fast)
const [users, orders] = await Promise.all([
  fetchUsers(),
  fetchOrders(),
]);
// If ANY rejects, the whole thing rejects immediately

// Promise.allSettled — wait for all, regardless of success/failure
const results = await Promise.allSettled([fetchA(), fetchB()]);
// results = [
//   { status: "fulfilled", value: dataA },
//   { status: "rejected", reason: errorB },
// ]

// Promise.race — first to settle (resolve OR reject) wins
const result = await Promise.race([
  fetchData(),
  timeout(5000),  // rejects after 5s
]);

// Promise.any — first to RESOLVE wins (ignores rejections)
const fastest = await Promise.any([mirror1(), mirror2(), mirror3()]);
// If ALL reject, throws AggregateError
```

### 🧩 TRICKY OUTPUT: Async

```javascript
async function foo() {
  console.log("foo start");
  await bar();
  console.log("foo end");     // this runs as a microtask after bar
}

async function bar() {
  console.log("bar");
}

console.log("script start");
foo();
console.log("script end");

// Output:
// script start
// foo start
// bar
// script end      ← sync code finishes first
// foo end         ← then microtask (continuation after await)
```

---

## 11. Iterators, Generators, and Symbols

### Iterators

```javascript
// Any object with a [Symbol.iterator] method is iterable
const iterable = {
  [Symbol.iterator]() {
    let i = 0;
    return {
      next() {
        return i < 3 ? { value: i++, done: false } : { done: true };
      }
    };
  }
};

for (const val of iterable) console.log(val);  // 0, 1, 2
[...iterable];  // [0, 1, 2]
```

### Generators

```javascript
function* range(start, end) {
  for (let i = start; i < end; i++) {
    yield i;           // pauses execution, returns value
  }
}

const gen = range(0, 3);
gen.next();  // { value: 0, done: false }
gen.next();  // { value: 1, done: false }
gen.next();  // { value: 2, done: false }
gen.next();  // { value: undefined, done: true }

[...range(0, 5)];  // [0, 1, 2, 3, 4]

// Infinite generator
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

// Async generator (for streaming)
async function* streamTokens(prompt) {
  for await (const chunk of llm.stream(prompt)) {
    yield chunk.content;
  }
}
```

### Symbols

```javascript
const id = Symbol("id");           // unique, even if same description
const id2 = Symbol("id");
id === id2;                         // false (always unique)

// Use as private-ish object keys
const obj = { [id]: 42, name: "Alice" };
Object.keys(obj);                   // ["name"] — symbols not included
Object.getOwnPropertySymbols(obj);  // [Symbol(id)]

// Well-known symbols
Symbol.iterator     // makes objects iterable (for...of)
Symbol.toPrimitive  // custom type coercion
Symbol.hasInstance   // customize instanceof
Symbol.toStringTag   // customize Object.prototype.toString output
```

---

## 12. Maps, Sets, WeakMap, WeakSet

### Map (Key-Value, Any Key Type)

```javascript
const map = new Map();
map.set("name", "Alice");
map.set(42, "the answer");
map.set(true, "yes");
const objKey = { id: 1 };
map.set(objKey, "object as key!");   // objects can be keys (unlike plain objects)

map.get("name")       // "Alice"
map.has(42)            // true
map.size               // 4
map.delete(42)         // true
map.clear()            // removes all

// Iteration (insertion order preserved)
for (const [key, value] of map) { ... }
map.forEach((value, key) => { ... });
[...map.keys()]        // all keys
[...map.values()]      // all values
[...map.entries()]     // all [key, value] pairs

// Object → Map
const map2 = new Map(Object.entries({ a: 1, b: 2 }));
// Map → Object
const obj = Object.fromEntries(map2);
```

### Set (Unique Values)

```javascript
const set = new Set([1, 2, 3, 3, 2]);  // {1, 2, 3} — duplicates removed

set.add(4)         // {1, 2, 3, 4}
set.has(3)         // true
set.delete(3)      // true
set.size           // 3
set.clear()

// Common patterns
const unique = [...new Set([1,1,2,2,3])];  // [1, 2, 3] — deduplicate array
const intersection = [...setA].filter(x => setB.has(x));
const union = new Set([...setA, ...setB]);
const difference = [...setA].filter(x => !setB.has(x));

// Set methods (ES2025)
setA.intersection(setB)    // new Set with common elements
setA.union(setB)           // new Set with all elements
setA.difference(setB)      // in A but not in B
setA.symmetricDifference(setB)  // in A or B but not both
setA.isSubsetOf(setB)
setA.isSupersetOf(setB)
```

### WeakMap and WeakSet

```javascript
// WeakMap: keys must be objects, entries are garbage-collected when key is GC'd
const cache = new WeakMap();
let element = document.getElementById("app");
cache.set(element, expensiveComputation(element));
element = null;  // element is GC'd, WeakMap entry is automatically removed

// No .size, no iteration (entries are transient)
// Use cases: caching, private data, DOM node metadata

// WeakSet: same concept, values must be objects
const visited = new WeakSet();
visited.add(someObject);
visited.has(someObject);  // true
```

---

## 13. Error Handling

```javascript
// Error types
new Error("generic error")
new TypeError("wrong type")
new RangeError("out of range")
new ReferenceError("undefined variable")
new SyntaxError("invalid syntax")

// try/catch/finally
try {
  const data = JSON.parse(invalidJson);
} catch (err) {
  console.error(err.message);   // error message
  console.error(err.stack);     // stack trace
  console.error(err.name);      // "SyntaxError"
} finally {
  cleanup();  // always runs, even if return is in try/catch
}

// Custom error
class AppError extends Error {
  constructor(message, code, statusCode) {
    super(message);
    this.name = "AppError";
    this.code = code;
    this.statusCode = statusCode;
  }
}

// Error in promises
promise.catch(err => console.error(err));
// or
try { await promise; } catch (err) { ... }

// Unhandled rejection (Node.js)
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled:", reason);
});
```

---

## 14. Modules

```javascript
// ESM (ES Modules) — modern standard
// Named exports
export const PI = 3.14;
export function add(a, b) { return a + b; }
export class Calculator { ... }

// Default export
export default class App { ... }

// Import
import App from "./app.js";                    // default
import { PI, add } from "./math.js";          // named
import { add as sum } from "./math.js";       // rename
import * as math from "./math.js";            // namespace
import "./side-effects.js";                    // import for side effects only

// Dynamic import (code splitting, lazy loading)
const module = await import("./heavy-module.js");

// CommonJS (Node.js legacy)
const fs = require("fs");
module.exports = { add, subtract };
module.exports = App;  // default-like
```

---

## 15. Proxy and Reflect

```javascript
const handler = {
  get(target, prop, receiver) {
    console.log(`Accessing ${prop}`);
    return Reflect.get(target, prop, receiver);
  },
  set(target, prop, value, receiver) {
    if (prop === "age" && (typeof value !== "number" || value < 0)) {
      throw new TypeError("Age must be a positive number");
    }
    return Reflect.set(target, prop, value, receiver);
  },
  has(target, prop) {
    // customize `in` operator
    return prop !== "secret" && Reflect.has(target, prop);
  },
  deleteProperty(target, prop) {
    if (prop === "id") throw new Error("Cannot delete id");
    return Reflect.deleteProperty(target, prop);
  },
};

const user = new Proxy({ name: "Alice", age: 30, secret: "shh" }, handler);
user.name;           // logs "Accessing name", returns "Alice"
user.age = -5;       // throws TypeError
"secret" in user;    // false (hidden by has trap)
```

**Use cases:** validation, logging, reactive systems (Vue 3 uses Proxy), access control, default values.

---

## 16. Miscellaneous Advanced Concepts

### Structured Clone (Deep Copy)

```javascript
const original = { a: 1, nested: { b: 2 }, date: new Date(), arr: [1, [2, 3]] };
const deep = structuredClone(original);   // true deep copy
// Handles: Date, RegExp, Map, Set, ArrayBuffer, circular references
// Does NOT handle: functions, DOM nodes, Errors
```

### WeakRef and FinalizationRegistry

```javascript
let obj = { data: "important" };
const ref = new WeakRef(obj);
ref.deref();      // { data: "important" } (or undefined if GC'd)

const registry = new FinalizationRegistry((heldValue) => {
  console.log(`${heldValue} was garbage collected`);
});
registry.register(obj, "my-object");
obj = null;       // eventually triggers the callback
```

### Intl (Internationalization)

```javascript
// Number formatting
new Intl.NumberFormat("en-US", { style: "currency", currency: "USD" }).format(1234.5);
// "$1,234.50"

// Date formatting
new Intl.DateTimeFormat("en-US", { dateStyle: "long" }).format(new Date());
// "July 16, 2026"

// Relative time
new Intl.RelativeTimeFormat("en", { numeric: "auto" }).format(-1, "day");
// "yesterday"

// Plural rules
new Intl.PluralRules("en").select(1);   // "one"
new Intl.PluralRules("en").select(2);   // "other"

// Collation (locale-aware sorting)
["ä", "a", "z"].sort(new Intl.Collator("de").compare);  // ["a", "ä", "z"]
```

### Optional Chaining and Nullish Coalescing

```javascript
// Optional chaining (?.) — short-circuits to undefined if null/undefined
user?.address?.city          // undefined if any part is null/undefined
user?.friends?.[0]?.name     // array access
user?.greet?.()              // function call

// Nullish coalescing (??) — default only for null/undefined (NOT for 0, "", false)
const port = config.port ?? 3000;      // 3000 if port is null or undefined
const port2 = config.port || 3000;     // 3000 if port is null, undefined, 0, "", false
// ?? is stricter than ||

0 ?? 42        // 0   (0 is not null/undefined)
0 || 42        // 42  (0 is falsy)
"" ?? "default" // "" (empty string is not null/undefined)
"" || "default" // "default" (empty string is falsy)
```

---

## 17. var vs let vs const (Complete Comparison)

```
Feature              var              let              const
─────────────────    ──────────       ──────────       ──────────
Scope                Function         Block            Block
Hoisting             Yes (undefined)  Yes (TDZ)        Yes (TDZ)
Re-declaration       Allowed          Not allowed      Not allowed
Re-assignment        Allowed          Allowed          Not allowed
Global object prop   Yes (window.x)   No               No
```

```javascript
// const with objects — the binding is constant, not the value
const arr = [1, 2, 3];
arr.push(4);       // OK — modifying the array
arr = [5, 6];      // TypeError — can't reassign

const obj = { a: 1 };
obj.b = 2;         // OK — adding properties
obj = {};          // TypeError — can't reassign
```

---

## 18. for...in vs for...of

```javascript
const arr = ["a", "b", "c"];

// for...in — iterates over KEYS (enumerable properties) — for objects
for (const key in arr) console.log(key);       // "0", "1", "2" (strings!)
// DON'T use for...in on arrays — it iterates prototype chain too

// for...of — iterates over VALUES (iterable protocol) — for arrays, strings, maps, sets
for (const val of arr) console.log(val);       // "a", "b", "c"

// for...in on objects
const obj = { a: 1, b: 2 };
for (const key in obj) console.log(key);       // "a", "b"
// for...of does NOT work on plain objects (they're not iterable)
// Use: for (const [key, val] of Object.entries(obj))
```

---
 
## 19. Regular Expressions
 
### Creation
 
```javascript
const re1 = /hello/gi;                        // literal (compiled at parse time)
const re2 = new RegExp("hello", "gi");        // constructor (runtime, dynamic patterns)
const re3 = new RegExp(`user_${id}`, "i");    // template with variable
```
 
### Flags
 
```
g    global — find all matches (not just first)
i    case-insensitive
m    multiline — ^ and $ match line boundaries
s    dotAll — . matches newlines too (ES2018)
u    unicode — treat as Unicode code points
v    unicodeSets — extended Unicode support (ES2024)
y    sticky — match only at lastIndex position
d    hasIndices — include start/end indices in results (ES2022)
```
 
### Pattern Syntax
 
```javascript
// Character classes
/[abc]/          // a, b, or c
/[^abc]/         // NOT a, b, or c
/[a-z]/          // a through z
/\d/             // digit [0-9]
/\D/             // non-digit
/\w/             // word char [a-zA-Z0-9_]
/\W/             // non-word char
/\s/             // whitespace
/\S/             // non-whitespace
/./              // any char except newline (unless /s flag)
/\b/             // word boundary
 
// Quantifiers
/a*/             // 0 or more
/a+/             // 1 or more
/a?/             // 0 or 1
/a{3}/           // exactly 3
/a{2,5}/         // 2 to 5
/a{2,}/          // 2 or more
/a+?/            // non-greedy (match as few as possible)
 
// Groups and references
/(abc)/          // capturing group
/(?:abc)/        // non-capturing group
/(?<name>abc)/   // named capturing group (ES2018)
/\1/             // backreference to group 1
/\k<name>/       // backreference to named group
 
// Assertions
/^abc/           // start of string
/abc$/           // end of string
/(?=abc)/        // positive lookahead (followed by abc)
/(?!abc)/        // negative lookahead (NOT followed by abc)
/(?<=abc)/       // positive lookbehind (preceded by abc) (ES2018)
/(?<!abc)/       // negative lookbehind (NOT preceded by abc) (ES2018)
 
// Alternation
/cat|dog/        // cat or dog
```
 
### Methods
 
```javascript
// RegExp methods
const re = /(\d{4})-(\d{2})-(\d{2})/;
re.test("2024-01-15")            // true (boolean match check)
re.exec("2024-01-15")            // ["2024-01-15", "2024", "01", "15", index: 0, ...]
 
// String methods with regex
"hello world".match(/\w+/g)      // ["hello", "world"]
"hello world".match(/(?<word>\w+)/)  // { groups: { word: "hello" }, ... }
"hello world".matchAll(/\w+/g)   // iterator of all match objects
"hello world".search(/world/)    // 6 (index of first match)
"hello world".replace(/world/, "JS")  // "hello JS"
"hello world".replace(/(\w+) (\w+)/, "$2 $1")  // "world hello"
"a-b-c".split(/-/)              // ["a", "b", "c"]
 
// Replace with function
"hello 123 world 456".replace(/\d+/g, (match) => {
  return String(Number(match) * 2);
});
// "hello 246 world 912"
 
// Named groups in replace
"2024-01-15".replace(
  /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/,
  "$<day>/$<month>/$<year>"
);
// "15/01/2024"
```
 
### 🧩 TRICKY: Regex
 
```javascript
const re = /a/g;
re.test("abc");    // true
re.test("abc");    // false!  (lastIndex moved to 1, no match at position 1+)
re.test("abc");    // true   (lastIndex reset to 0 after failure)
// /g flag makes test() stateful — it remembers lastIndex
// Fix: don't reuse /g regexes with test(), or reset re.lastIndex = 0
```
 
---
 
## 20. Date and Temporal
 
### Date Object (Legacy but Still Asked)
 
```javascript
const now = new Date();                         // current date/time
const specific = new Date(2024, 0, 15);         // Jan 15, 2024 (months are 0-indexed!)
const fromStr = new Date("2024-01-15T10:30:00Z"); // ISO string
const fromMs = new Date(1705312200000);         // milliseconds since epoch
 
// Getters
now.getFullYear()       // 2026 (NOT getYear() — that's deprecated and broken)
now.getMonth()          // 0-11 (January = 0!)
now.getDate()           // 1-31 (day of month)
now.getDay()            // 0-6 (Sunday = 0)
now.getHours()          // 0-23
now.getMinutes()        // 0-59
now.getSeconds()        // 0-59
now.getTime()           // milliseconds since epoch
now.toISOString()       // "2026-07-16T..." (UTC)
now.toLocaleDateString("en-US")  // "7/16/2026"
 
Date.now()              // current timestamp in ms (no new needed)
 
// Gotchas
new Date("2024-01-15")           // parsed as UTC
new Date("01/15/2024")           // parsed as local time! (inconsistent)
new Date(2024, 0, 32)            // auto-rolls to Feb 1 (no error)
```
 
### 🧩 TRICKY: Date
 
```javascript
console.log(new Date(2024, 0, 1).getMonth());   // 0 (not 1!)
console.log(typeof new Date());                   // "object" (not "date")
console.log(new Date() - new Date(2024, 0, 1));  // milliseconds difference (auto-coerced to number)
console.log(new Date() + new Date());             // string concatenation! (toString() called)
```
 
---
 
## 21. JSON Methods
 
```javascript
// Stringify
JSON.stringify({ a: 1, b: "hello", c: true })   // '{"a":1,"b":"hello","c":true}'
 
// What gets stripped:
JSON.stringify({ fn: () => {}, sym: Symbol(), undef: undefined })
// '{}'  — functions, symbols, and undefined are silently removed
 
JSON.stringify({ a: 1 }, null, 2)  // pretty-printed with 2-space indent
 
// Replacer function (filter/transform)
JSON.stringify({ name: "Alice", password: "secret", age: 30 }, (key, value) => {
  if (key === "password") return undefined;  // exclude
  return value;
});
// '{"name":"Alice","age":30}'
 
// Replacer array (whitelist keys)
JSON.stringify({ a: 1, b: 2, c: 3 }, ["a", "c"])
// '{"a":1,"c":3}'
 
// toJSON method (custom serialization)
class User {
  constructor(name, password) {
    this.name = name;
    this.password = password;
  }
  toJSON() {
    return { name: this.name };  // exclude password
  }
}
JSON.stringify(new User("Alice", "secret"))  // '{"name":"Alice"}'
 
// Parse
JSON.parse('{"a":1,"b":"hello"}')   // { a: 1, b: "hello" }
 
// Reviver function (transform during parse)
JSON.parse('{"date":"2024-01-15T10:30:00Z"}', (key, value) => {
  if (key === "date") return new Date(value);
  return value;
});
// { date: Date object }
 
// Circular reference
const obj = {};
obj.self = obj;
JSON.stringify(obj);  // TypeError: Converting circular structure to JSON
// Fix: use structuredClone() or a replacer that tracks visited objects
```
 
---
 
## 22. Bitwise Operators
 
```javascript
// Operate on 32-bit integers
5 & 3        // 1   AND   (0101 & 0011 = 0001)
5 | 3        // 7   OR    (0101 | 0011 = 0111)
5 ^ 3        // 6   XOR   (0101 ^ 0011 = 0110)
~5           // -6  NOT   (~0101 = ...11111010 → -6 in two's complement)
5 << 1       // 10  left shift  (multiply by 2)
5 >> 1       // 2   right shift (divide by 2, preserve sign)
-5 >>> 1     // 2147483645  unsigned right shift (fills with 0s)
 
// Practical uses
// Fast floor for positive numbers
~~3.7          // 3  (double NOT — truncates to integer)
3.7 | 0        // 3  (OR with 0 — same effect)
3.7 >> 0       // 3
 
// Check if number is even/odd
n & 1          // 0 = even, 1 = odd (faster than n % 2)
 
// Swap without temp variable
a ^= b; b ^= a; a ^= b;
 
// Bit flags
const READ  = 1;    // 001
const WRITE = 2;    // 010
const EXEC  = 4;    // 100
let perms = READ | WRITE;   // 011
perms & READ;                // 1 (truthy — has READ)
perms & EXEC;                // 0 (falsy — no EXEC)
perms |= EXEC;               // add EXEC
perms &= ~WRITE;             // remove WRITE
```
 
---
 
## 23. Logical Assignment Operators (ES2021)
 
```javascript
// ||= (assign if falsy)
let a = 0;
a ||= 10;        // a = 10 (0 is falsy)
 
let b = "hello";
b ||= "default";  // b = "hello" (non-empty string is truthy)
 
// &&= (assign if truthy)
let c = 1;
c &&= 2;          // c = 2 (1 is truthy, so assign)
 
let d = 0;
d &&= 2;          // d = 0 (0 is falsy, so don't assign)
 
// ??= (assign if null/undefined — nullish)
let e = null;
e ??= "default";   // e = "default"
 
let f = 0;
f ??= 42;           // f = 0 (0 is NOT null/undefined — ??= doesn't trigger)
 
// Common pattern: initialize if missing
user.scores ??= [];
user.scores.push(95);
 
options.timeout ??= 5000;
```
 
---
 
## 24. Short-Circuit Evaluation
 
```javascript
// && returns first falsy value, or last value if all truthy
0 && "hello"        // 0 (stops at falsy)
1 && "hello"        // "hello" (all truthy, returns last)
"a" && "b" && "c"   // "c"
"" && "anything"    // "" (stops at empty string)
 
// || returns first truthy value, or last value if all falsy
0 || "hello"        // "hello" (skips falsy, returns first truthy)
"" || null || "ok"  // "ok"
0 || "" || false    // false (all falsy, returns last)
 
// Practical uses
const name = user.name || "Anonymous";         // default (but 0 and "" are treated as missing)
const name2 = user.name ?? "Anonymous";        // better: only null/undefined
const debug = isDebug && console.log("debug"); // conditional execution
callback && callback();                         // call only if defined
```
 
---
 
## 25. Strict Mode
 
```javascript
"use strict";  // at top of file or function
 
// What it changes:
// 1. Assigning to undeclared variables throws ReferenceError (no accidental globals)
x = 10;  // ReferenceError (without strict mode, creates global var)
 
// 2. Assigning to read-only properties throws TypeError
const obj = {};
Object.defineProperty(obj, "x", { writable: false, value: 42 });
obj.x = 100;  // TypeError in strict, silently fails in sloppy
 
// 3. Deleting undeletable properties throws TypeError
delete Object.prototype;  // TypeError
 
// 4. Duplicate parameter names not allowed
function f(a, a) {}  // SyntaxError in strict
 
// 5. Octal literals not allowed
const n = 010;  // SyntaxError (use 0o10 instead)
 
// 6. `this` in functions is undefined (not global object)
function f() { return this; }
f();  // undefined in strict, window/global in sloppy
 
// 7. eval doesn't introduce variables into surrounding scope
eval("var x = 10");
console.log(x);  // ReferenceError in strict
 
// ES modules and classes are always in strict mode automatically
```
 
---
 
## 26. Private Class Fields (#)
 
```javascript
class BankAccount {
  #balance = 0;          // truly private (not accessible outside the class)
  #owner;
 
  constructor(owner, initial) {
    this.#owner = owner;
    this.#balance = initial;
  }
 
  get balance() { return this.#balance; }
 
  deposit(amount) {
    if (amount <= 0) throw new Error("Invalid amount");
    this.#balance += amount;
  }
 
  #validate(amount) {    // private method
    return amount > 0 && amount <= this.#balance;
  }
 
  withdraw(amount) {
    if (!this.#validate(amount)) throw new Error("Invalid withdrawal");
    this.#balance -= amount;
  }
 
  // Static private
  static #instances = 0;
  static getCount() { return BankAccount.#instances; }
}
 
const acc = new BankAccount("Alice", 100);
acc.#balance;       // SyntaxError! (truly private — unlike TS private which is only compile-time)
acc.balance;        // 100 (via getter)
"#balance" in acc;  // true (can check existence, but not access value)
```
 
---
 
## 27. Static Initialization Blocks (ES2022)
 
```javascript
class Database {
  static connection;
  static ready;
 
  // Static block — runs once when the class is evaluated
  static {
    try {
      this.connection = initializeDB();
      this.ready = true;
    } catch (e) {
      this.connection = null;
      this.ready = false;
    }
  }
}
// Can access private static fields, unlike static methods called from outside
```
 
---
 
## 28. Error.cause (ES2022)
 
```javascript
try {
  try {
    JSON.parse(invalidData);
  } catch (parseError) {
    throw new Error("Failed to process config", { cause: parseError });
  }
} catch (err) {
  console.log(err.message);        // "Failed to process config"
  console.log(err.cause.message);  // "Unexpected token..." (original error)
}
```
 
---
 
## 29. Object.groupBy and Map.groupBy (ES2024)
 
```javascript
const people = [
  { name: "Alice", age: 30 },
  { name: "Bob", age: 25 },
  { name: "Charlie", age: 30 },
  { name: "Diana", age: 25 },
];
 
const byAge = Object.groupBy(people, person => person.age);
// { 25: [{name:"Bob",...}, {name:"Diana",...}], 30: [{name:"Alice",...}, {name:"Charlie",...}] }
 
// Map.groupBy for non-string keys
const byAgeMap = Map.groupBy(people, person => person.age);
// Map { 30 => [...], 25 => [...] }
```
 
---
 
## 30. Promise.withResolvers (ES2024)
 
```javascript
// Old way
let resolve, reject;
const promise = new Promise((res, rej) => { resolve = res; reject = rej; });
 
// New way
const { promise, resolve, reject } = Promise.withResolvers();
 
// Use case: deferred patterns
function createDeferred() {
  const { promise, resolve, reject } = Promise.withResolvers();
  return { promise, resolve, reject };
}
 
const deferred = createDeferred();
setTimeout(() => deferred.resolve("done"), 1000);
await deferred.promise;  // "done"
```
 
---
 
## 31. ArrayBuffer, TypedArrays, and DataView
 
```javascript
// ArrayBuffer — raw binary data (fixed size, not directly readable)
const buffer = new ArrayBuffer(16);   // 16 bytes
 
// TypedArrays — views into an ArrayBuffer
const int32View = new Int32Array(buffer);    // 4 elements (16 bytes / 4 bytes each)
const uint8View = new Uint8Array(buffer);    // 16 elements (16 bytes / 1 byte each)
const float64View = new Float64Array(buffer); // 2 elements (16 bytes / 8 bytes each)
 
int32View[0] = 42;
uint8View[0];   // 42 (same underlying memory!)
 
// Available TypedArrays:
// Int8Array, Uint8Array, Uint8ClampedArray, Int16Array, Uint16Array,
// Int32Array, Uint32Array, Float32Array, Float64Array, BigInt64Array, BigUint64Array
 
// DataView — for mixed types or specific byte order
const view = new DataView(buffer);
view.setInt32(0, 42, true);       // little-endian
view.getInt32(0, true);           // 42
 
// Use case: WebSocket binary protocols, file parsing, WebGL, SharedArrayBuffer
```
 
---
 
## 32. globalThis (ES2020)
 
```javascript
// The universal way to access the global object in ANY environment
globalThis.setTimeout     // works in browser, Node.js, Web Workers, Deno
 
// Before globalThis:
// Browser: window
// Node.js: global
// Web Worker: self
// Any: Function("return this")() — hacky
```
 
---
 
## 33. Labels and Labeled Statements
 
```javascript
// Label a loop for breaking/continuing outer loops
outer: for (let i = 0; i < 3; i++) {
  inner: for (let j = 0; j < 3; j++) {
    if (i === 1 && j === 1) break outer;   // breaks the outer loop
    console.log(i, j);
  }
}
// 0 0, 0 1, 0 2, 1 0 — then stops
 
// continue with label
outer: for (let i = 0; i < 3; i++) {
  for (let j = 0; j < 3; j++) {
    if (j === 1) continue outer;  // skips to next iteration of outer
    console.log(i, j);
  }
}
// 0 0, 1 0, 2 0
```
 
---
 
## 34. The void and delete Operators
 
```javascript
// void — evaluates expression, returns undefined
void 0               // undefined
void "hello"         // undefined
void doSomething()   // call function, discard return value, return undefined
 
// Common use: ensure arrow function doesn't accidentally return
const onClick = () => void doSomething();  // always returns undefined
 
// delete — removes a property from an object
const obj = { a: 1, b: 2 };
delete obj.a;
console.log(obj);    // { b: 2 }
delete obj.a;        // true (even if property doesn't exist)
 
// Cannot delete: variables, function declarations, non-configurable properties
var x = 10;
delete x;            // false (no effect in strict mode, silently fails in sloppy)
```
 
---
 
## 35. The Comma Operator
 
```javascript
// Evaluates all expressions left to right, returns the last one
const result = (1, 2, 3);    // 3
 
// Common in for loops
for (let i = 0, j = 10; i < j; i++, j--) {
  console.log(i, j);
}
 
// Can cause subtle bugs
const x = (a = 1, b = 2, a + b);  // x = 3
```
 
---
 
## 36. Top-Level Await (ES2022)
 
```javascript
// In ES modules, you can await at the top level (no async function wrapper needed)
const response = await fetch("https://api.example.com/data");
const data = await response.json();
export default data;
 
// The module that imports this will wait for it to resolve
// This makes the module itself an async module
```
 
---
 
## 37. Destructuring Deep Patterns
 
```javascript
// Nested destructuring
const { address: { city, zip } } = user;    // extracts city and zip (address is NOT created)
 
// Array destructuring with skip
const [, second, , fourth] = [1, 2, 3, 4];  // second=2, fourth=4
 
// Mixed array + object
const { data: [first, ...rest] } = apiResponse;
 
// Default values in nested destructuring
const { settings: { theme = "dark", fontSize = 14 } = {} } = user;
// Even if user.settings is undefined, defaults apply
 
// Rename + default
const { name: userName = "Anonymous" } = {};  // userName = "Anonymous"
 
// Destructuring function parameters
function render({ title, body, footer = "default" } = {}) {
  // works even if no argument is passed (= {} default)
}
 
// Swapping
[a, b] = [b, a];
[a, b, c] = [c, a, b];  // rotate
```
 
---
 
## 38. Property Enumeration Order
 
```javascript
// Since ES2020, property order is:
// 1. Integer indices (0, 1, 2, ...) in ascending order
// 2. String keys in insertion order
// 3. Symbol keys in insertion order
 
const obj = {};
obj.b = 2;
obj[1] = "one";
obj.a = 1;
obj[0] = "zero";
obj[Symbol("s")] = "sym";
 
Object.keys(obj);  // ["0", "1", "b", "a"]
// Integer keys first (sorted), then strings (insertion order)
```
 
---
 
## 39. 🧩 TRICKY OUTPUT: Grand Finale
 
```javascript
// What does this return?
console.log(typeof typeof 42);
// typeof 42 → "number" (string)
// typeof "number" → "string"
// Answer: "string"
 
// Comparison chain
console.log(1 < 2 < 3);  // true   (1 < 2 → true, true < 3 → 1 < 3 → true)
console.log(3 > 2 > 1);  // false  (3 > 2 → true, true > 1 → 1 > 1 → false)
 
// Addition with arrays
console.log([1] + [2]);     // "12" (both → string "1" + "2")
console.log([1, 2] + [3]);  // "1,23"
 
// null arithmetic
console.log(null == false);     // false (null only == undefined)
console.log(null == 0);         // false
console.log(null >= 0);         // true!  (comparison coerces null → 0)
console.log(null > 0);          // false
console.log(null == undefined); // true
 
// Empty comparison
console.log([] == 0);       // true  ([] → "" → 0)
console.log("" == 0);       // true
console.log("" == false);   // true
console.log([] == false);   // true
 
// typeof undeclared
console.log(typeof undeclaredVar);  // "undefined" (no ReferenceError!)
// This is intentional — allows safe feature detection
```
