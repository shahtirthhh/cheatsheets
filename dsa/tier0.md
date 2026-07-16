# DSA Tier 0 — Time & Space Complexity

_The foundation everything else is built on_
_If you skip this, nothing else makes sense_

---

# Part 1: Why Complexity Matters

## The Real Question: "How Slow Will This Get?"

You wrote a function. It works perfectly with 10 items. But what happens with 10,000 items? 10 million? That's what complexity analysis answers — not "how fast is this right now" but "how does the speed change as the input grows?"

```
Imagine you're searching for a name in a phone book:

Approach 1: Start from page 1, read every name until you find it.
  10 names    → check ~10 names     (instant)
  1,000 names → check ~1,000 names  (a few seconds)
  1,000,000   → check ~1,000,000    (minutes)
  Input doubled → time doubled. This is LINEAR growth.

Approach 2: Open to the middle. Is the name before or after?
            Eliminate half. Repeat.
  10 names    → ~3 checks
  1,000 names → ~10 checks
  1,000,000   → ~20 checks
  Input doubled → only ONE more check. This is LOGARITHMIC growth.
```

The phone book didn't get faster. Your APPROACH determined how it scales.

---

# Part 2: Time Complexity (Big-O)

## What Big-O Notation Means

Big-O describes the **worst-case** growth rate of an algorithm as the input size (n) increases. We drop constants and lower-order terms because at massive scale, only the dominant term matters.

```
Actual operations:  3n² + 5n + 100
Big-O:              O(n²)

Why drop the rest?
  When n = 10:      3(100) + 50 + 100 = 450       (constants matter)
  When n = 1000:    3(1,000,000) + 5000 + 100      (n² dominates everything)
  When n = 1M:      3(1,000,000,000,000) + ...      (5n and 100 are irrelevant)
```

## The 7 Complexities You Must Know

### O(1) — Constant Time

**No matter how big the input is, it takes the same amount of time.**

Think of it like looking at your watch. Whether there are 10 people in the room or 10 million, checking the time takes the same effort.

```javascript
// Example 1: Array access by index
function getFirst(arr) {
  return arr[0]; // always 1 operation, regardless of array size
}

// Example 2: Hash map lookup
function getUser(map, id) {
  return map[id]; // always 1 operation (average)
}

// Example 3: Math calculation
function isEven(n) {
  return n % 2 === 0; // always 1 operation
}

// Example 4: Push/pop from end of array
arr.push(5); // O(1)
arr.pop(); // O(1)
```

```
Input Size:    10      100      1,000     1,000,000
Operations:     1        1          1             1
                ────────────────────────────────────
                flat line — doesn't grow
```

### O(log n) — Logarithmic Time

**Each step eliminates HALF of the remaining data.**

Think of it like guessing a number between 1 and 100. You don't guess 1, 2, 3... You guess 50 (too high?), then 25 (too low?), then 37... Each guess cuts the possibilities in half. That's logarithmic.

```
How many times can you halve n before reaching 1?
  n = 8:    8 → 4 → 2 → 1    = 3 steps  (log₂(8) = 3)
  n = 16:   16 → 8 → 4 → 2 → 1 = 4 steps (log₂(16) = 4)
  n = 1024: 10 steps           (log₂(1024) = 10)
  n = 1,000,000: ~20 steps     (log₂(1M) ≈ 20)

  A MILLION items and you only need ~20 steps!
```

```javascript
// Binary search — the classic O(log n) algorithm
function binarySearch(sortedArr, target) {
  let left = 0;
  let right = sortedArr.length - 1;

  while (left <= right) {
    const mid = Math.floor((left + right) / 2);

    if (sortedArr[mid] === target) return mid;
    else if (sortedArr[mid] < target)
      left = mid + 1; // eliminate left half
    else right = mid - 1; // eliminate right half
  }

  return -1;
}
```

```
Searching for 7 in [1, 3, 5, 7, 9, 11, 13]:

Step 1: [1, 3, 5, 7, 9, 11, 13]    mid=7, found! (3 elements checked)
                 ↑ mid

With 1 BILLION sorted items: only ~30 comparisons needed.
```

```
Input Size:    10      100      1,000     1,000,000
Operations:     3        7         10            20
                ────────────────────────────────────
                grows incredibly slowly
```

### O(n) — Linear Time

**You look at every element exactly once (or a fixed number of times).**

Think of it like counting people in a room. The only way to count them is to look at each person once. More people = proportionally more time.

```javascript
// Example 1: Find the maximum
function findMax(arr) {
  let max = arr[0];
  for (let i = 1; i < arr.length; i++) {
    // visits each element once
    if (arr[i] > max) max = arr[i];
  }
  return max;
}

// Example 2: Sum all elements
function sum(arr) {
  let total = 0;
  for (const num of arr) {
    // visits each element once
    total += num;
  }
  return total;
}

// Example 3: Linear search
function find(arr, target) {
  for (let i = 0; i < arr.length; i++) {
    if (arr[i] === target) return i; // might check all n elements
  }
  return -1;
}

// Two sequential loops is still O(n):
// O(n) + O(n) = O(2n) = O(n)  (drop the constant)
```

```
Input Size:    10      100      1,000     1,000,000
Operations:    10      100      1,000     1,000,000
               ────────────────────────────────────
               straight line — grows proportionally
```

### O(n log n) — Linearithmic Time

**For each of n elements, you do log n work. This is the speed of efficient sorting.**

Think of it like organizing a deck of cards using merge sort: you split the deck in half repeatedly (log n splits), and at each level you merge all n cards.

```javascript
// Built-in sort in most languages is O(n log n)
arr.sort((a, b) => a - b);

// Merge sort conceptually:
// Split:  [5,3,8,1] → [5,3] and [8,1] → [5],[3],[8],[1]   (log n levels)
// Merge:  [5],[3] → [3,5]   [8],[1] → [1,8]                (n work per level)
//         [3,5],[1,8] → [1,3,5,8]
```

```
Input Size:    10        100        1,000       1,000,000
Operations:    33        664        9,966       19,931,568
               ─────────────────────────────────────────
               slightly faster than quadratic, much slower than linear
```

### O(n²) — Quadratic Time

**For each of n elements, you look at all n elements again. Nested loops.**

Think of it like introducing everyone in a room to everyone else. With 10 people, that's 45 handshakes. With 100 people, it's 4,950. With 1,000 people, it's ~500,000.

```javascript
// Example 1: Check all pairs
function hasDuplicateBrute(arr) {
  for (let i = 0; i < arr.length; i++) {
    // n iterations
    for (let j = i + 1; j < arr.length; j++) {
      // n iterations (roughly)
      if (arr[i] === arr[j]) return true;
    }
  }
  return false;
}

// Example 2: Bubble sort
function bubbleSort(arr) {
  for (let i = 0; i < arr.length; i++) {
    // n
    for (let j = 0; j < arr.length - i - 1; j++) {
      // n
      if (arr[j] > arr[j + 1]) {
        [arr[j], arr[j + 1]] = [arr[j + 1], arr[j]];
      }
    }
  }
}
```

```
Input Size:    10      100       1,000      1,000,000
Operations:   100    10,000   1,000,000   1,000,000,000,000
              ─────────────────────────────────────────
              EXPLODES. 1M items = 1 trillion operations = hours/days
```

### O(2ⁿ) — Exponential Time

**Every element DOUBLES the work.** Used in brute-force solutions that explore every possible combination.

```javascript
// Fibonacci without memoization
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2); // each call spawns 2 more calls
}

// Call tree for fib(5):
//                    fib(5)
//                  /        \
//             fib(4)        fib(3)
//            /     \        /     \
//        fib(3)  fib(2)  fib(2)  fib(1)
//        /   \   /   \    /  \
//     fib(2) fib(1) ...  ...  ...
//
// Each level doubles. Total calls ≈ 2ⁿ
```

```
Input Size:    10        20         30          40
Operations:  1,024    1,048,576   ~1 billion   ~1 trillion
             ──────────────────────────────────────
             Unusable beyond ~25-30 items
```

### O(n!) — Factorial Time

**Try every possible ordering.** The traveling salesman problem (brute force), generating all permutations.

```
n = 3:     6 permutations
n = 5:     120
n = 10:    3,628,800
n = 15:    1,307,674,368,000 (1.3 trillion)
n = 20:    2,432,902,008,176,640,000 (impossible)
```

## The Visual Comparison

```
Operations
    ↑
    │                                              O(n!)
    │                                           O(2ⁿ)
    │                                        ╱
    │                                     ╱
    │                                  ╱
    │                     O(n²)    ╱
    │                    ╱      ╱
    │                 ╱      ╱
    │              ╱      ╱
    │           ╱     ╱
    │  O(n log n)  ╱
    │       ╱   ╱
    │  O(n)  ╱
    │   ╱ ╱
    │  ╱──── O(log n)
    │╱═══════════════════ O(1)
    └────────────────────────────────▶ Input Size (n)
```

## How to Determine Time Complexity

### Rule 1: Count the Loops

```javascript
// No loop → O(1)
return arr[0];

// Single loop → O(n)
for (let i = 0; i < n; i++) { ... }

// Nested loops → multiply
for (let i = 0; i < n; i++) {          // O(n)
  for (let j = 0; j < n; j++) {        //   × O(n)
    // ...                              // = O(n²)
  }
}

// Loop that halves → O(log n)
while (n > 1) { n = Math.floor(n / 2); }

// Loop inside a halving loop → O(n log n)
for (let i = 1; i < n; i *= 2) {       // O(log n)
  for (let j = 0; j < n; j++) {        //   × O(n)
    // ...                              // = O(n log n)
  }
}
```

### Rule 2: Sequential Steps ADD, Nested Steps MULTIPLY

```javascript
// Sequential: O(n) + O(n) = O(2n) = O(n)
for (let i = 0; i < n; i++) { /* step 1 */ }
for (let j = 0; j < n; j++) { /* step 2 */ }

// Nested: O(n) × O(n) = O(n²)
for (let i = 0; i < n; i++) {
  for (let j = 0; j < n; j++) { /* step inside step */ }
}

// Sequential with different sizes: O(a + b)
for (let i = 0; i < a; i++) { ... }
for (let j = 0; j < b; j++) { ... }

// Nested with different sizes: O(a × b)
for (let i = 0; i < a; i++) {
  for (let j = 0; j < b; j++) { ... }
}
```

### Rule 3: Drop Constants and Lower-Order Terms

```
O(2n)         → O(n)         (drop the 2)
O(n + 10)     → O(n)         (drop the 10)
O(n² + n)     → O(n²)        (drop the n — it's dominated by n²)
O(500)        → O(1)         (any fixed amount = constant)
O(n/2)        → O(n)         (drop the /2)
O(3n² + 2n + 7) → O(n²)     (keep only dominant term)
```

### Rule 4: Recursive Functions — Draw the Call Tree

```javascript
function recursiveExample(n) {
  if (n <= 0) return; // base case
  recursiveExample(n - 1); // ONE recursive call → O(n)
}

function recursiveDouble(n) {
  if (n <= 0) return;
  recursiveDouble(n - 1); // TWO recursive calls → O(2ⁿ)
  recursiveDouble(n - 1);
}

function mergeSort(arr) {
  // Split in half, recurse on each half, merge
  // Depth: log n, Work per level: n
  // Total: O(n log n)
}
```

---

# Part 3: Space Complexity

## What Space Complexity Measures

Time complexity measures how many operations. Space complexity measures how much **extra memory** your algorithm uses (beyond the input itself).

```
The input array itself doesn't count.
What counts: new arrays, objects, maps, sets, recursive call stack, etc.
```

### O(1) Space — Constant

You use a fixed number of variables regardless of input size.

```javascript
function findMax(arr) {
  let max = arr[0];      // one variable
  for (let i = 1; i < arr.length; i++) {
    if (arr[i] > max) max = arr[i];
  }
  return max;
}
// Space: O(1) — only `max` and `i`, no matter how big arr is

// Two pointers pattern — always O(1) space
function isPalindrome(s) {
  let left = 0, right = s.length - 1;   // two variables
  while (left < right) { ... }
}
```

### O(n) Space — Linear

You create a data structure that grows proportionally with input.

```javascript
function removeDuplicates(arr) {
  const set = new Set(arr); // set grows with arr size
  return [...set]; // new array also grows with input
}
// Space: O(n)

function frequencies(arr) {
  const map = {}; // map grows with unique elements
  for (const item of arr) {
    map[item] = (map[item] || 0) + 1;
  }
  return map;
}
// Space: O(n) worst case (all unique)
```

### O(n) Space — Recursive Call Stack

**Every recursive call adds a frame to the call stack.** This uses memory!

```javascript
function factorial(n) {
  if (n <= 1) return 1;
  return n * factorial(n - 1);
}

// factorial(5) creates this call stack:
//   factorial(5)    ← waiting for factorial(4)
//     factorial(4)  ← waiting for factorial(3)
//       factorial(3)
//         factorial(2)
//           factorial(1)  ← returns 1
//         returns 2
//       returns 6
//     returns 24
//   returns 120
//
// Maximum stack depth: n → Space: O(n)
```

### O(log n) Space — Recursive with Halving

```javascript
function binarySearchRecursive(arr, target, left, right) {
  if (left > right) return -1;
  const mid = Math.floor((left + right) / 2);
  if (arr[mid] === target) return mid;
  if (arr[mid] < target)
    return binarySearchRecursive(arr, target, mid + 1, right);
  return binarySearchRecursive(arr, target, left, mid - 1);
}
// Each recursive call halves the range → max depth: log n → Space: O(log n)
```

### O(n²) Space

```javascript
// Creating a 2D matrix
function createMatrix(n) {
  const matrix = [];
  for (let i = 0; i < n; i++) {
    matrix[i] = new Array(n).fill(0); // n rows × n columns
  }
  return matrix;
}
// Space: O(n²)
```

---

# Part 4: Analyzing Real Code — Practice

## Example 1

```javascript
function mystery(arr) {
  const result = []; // O(n) space in worst case
  for (let i = 0; i < arr.length; i++) {
    // O(n) time
    if (arr[i] % 2 === 0) {
      // O(1) per check
      result.push(arr[i]);
    }
  }
  return result;
}
// Time:  O(n)   — single loop through n elements
// Space: O(n)   — result array could hold all n elements
```

## Example 2

```javascript
function mystery2(arr) {
  for (let i = 0; i < arr.length; i++) {
    // O(n)
    for (let j = 0; j < arr.length; j++) {
      //   × O(n)
      if (i !== j && arr[i] === arr[j]) return true;
    }
  }
  return false;
}
// Time:  O(n²)  — nested loops
// Space: O(1)   — no extra data structures
```

## Example 3

```javascript
function mystery3(arr) {
  const sorted = [...arr].sort((a, b) => a - b); // O(n log n) time, O(n) space
  for (let i = 0; i < sorted.length - 1; i++) {
    // O(n)
    if (sorted[i] === sorted[i + 1]) return true;
  }
  return false;
}
// Time:  O(n log n)  — sort dominates
// Space: O(n)        — copied array
```

## Example 4

```javascript
function mystery4(n) {
  if (n <= 0) return 0;
  return mystery4(n - 1) + mystery4(n - 1);
}
// Draw the tree:
//        m4(3)
//       /     \
//    m4(2)    m4(2)
//    /   \    /   \
// m4(1) m4(1) m4(1) m4(1)
//  / \   / \   / \   / \
// m4(0)×2 m4(0)×2 m4(0)×2 m4(0)×2
//
// Each level doubles → 2ⁿ nodes
// Time:  O(2ⁿ)
// Space: O(n) — max depth of call stack is n (only one branch exists at a time)
```

---

# Part 5: The Tradeoff — Time vs Space

Almost every optimization trades space for time (or vice versa).

```
Algorithm               Time        Space       Tradeoff
──────────────────      ────        ─────       ────────
Has duplicate (brute)   O(n²)       O(1)        slow but no extra memory
Has duplicate (set)     O(n)        O(n)        fast but uses extra memory
Has duplicate (sort)    O(n log n)  O(1)*       medium speed, no extra memory

Two Sum (brute)         O(n²)       O(1)        slow
Two Sum (hash map)      O(n)        O(n)        fast but stores the map

Fibonacci (recursive)   O(2ⁿ)       O(n)        exponentially slow
Fibonacci (memoized)    O(n)        O(n)        fast, stores results
Fibonacci (iterative)   O(n)        O(1)        fast AND memory efficient

* if sorting in-place
```

**Interview rule:** When in doubt, trade space for time. Interviewers prefer faster solutions even if they use more memory. Memory is cheap; user patience is not.

---

# Part 6: Quick Reference Card

```
Notation     Name           Example                          Max n (~1sec)
────────     ─────────      ──────────────────────            ──────────
O(1)         Constant       Hash map lookup, array[i]         ∞
O(log n)     Logarithmic    Binary search                     ∞
O(n)         Linear         Single loop, two pointers         ~10⁷
O(n log n)   Linearithmic   Sorting, divide & conquer         ~10⁶
O(n²)        Quadratic      Nested loops, brute force pairs   ~10⁴
O(n³)        Cubic          Triple nested loops               ~500
O(2ⁿ)        Exponential    Subsets, recursive backtracking   ~25
O(n!)        Factorial      Permutations                      ~12

Common operations:
  Array access:       O(1)
  Array push/pop:     O(1)
  Array shift/unshift: O(n)    ← moves all elements!
  Array.includes():   O(n)    ← linear search
  Array.sort():       O(n log n)
  Object/Map get/set: O(1)
  Set.has():          O(1)
  String concat (+):  O(n)    ← creates new string
  String.slice():     O(n)    ← creates new string
```

---

# Part 7: Interview Cheat — What They Want to Hear

When the interviewer asks "What's the complexity?" answer like this:

```
"This solution runs in O(n) time because we iterate through the array once,
 and O(n) space because we store up to n elements in the hash map."

"The brute force would be O(n²) with nested loops, but by using a hash map
 we trade O(n) space to bring the time down to O(n)."

"Sorting takes O(n log n) which dominates the O(n) scan afterward,
 so the overall time complexity is O(n log n). Space is O(1) if we sort in-place."
```

Always mention: time complexity, space complexity, and the tradeoff.
