# DSA Tier 2 — The Next 25% That Makes You Confident

_Sorting · Binary Search · Stack · Recursion · Backtracking · Linked Lists · BFS/DFS · Trees_
_Language: JavaScript · Level: Teaching from zero_

---

# Chapter 6: Sorting

## Why Sorting Matters

Sorting isn't just "arrange numbers." Sorting is a **prerequisite** that unlocks faster algorithms. Once data is sorted, you can use binary search (O(log n) instead of O(n)), two pointers (O(n) instead of O(n²)), and merge intervals (adjacent overlaps).

**Interview rule:** If you're stuck, ask yourself "would sorting help here?" The answer is surprisingly often yes.

## The Sorting Algorithms You Need to Know

You'll never implement merge sort in an interview. But you need to explain them conceptually and know their complexities.

### Bubble Sort — The One You Should Know Is Bad

**How it works:** Walk through the array. Compare adjacent elements. If they're in the wrong order, swap them. Repeat until no swaps are needed.

**Why it's named "bubble":** The largest elements "bubble up" to the end with each pass, like bubbles rising in water.

```
Pass 1: [5, 3, 8, 1, 2]
  Compare 5,3 → swap: [3, 5, 8, 1, 2]
  Compare 5,8 → ok:   [3, 5, 8, 1, 2]
  Compare 8,1 → swap: [3, 5, 1, 8, 2]
  Compare 8,2 → swap: [3, 5, 1, 2, 8]  ← 8 bubbled to the end

Pass 2: [3, 5, 1, 2, 8]
  Compare 3,5 → ok
  Compare 5,1 → swap: [3, 1, 5, 2, 8]
  Compare 5,2 → swap: [3, 1, 2, 5, 8]  ← 5 bubbled

Pass 3: [3, 1, 2, 5, 8]
  Compare 3,1 → swap: [1, 3, 2, 5, 8]
  Compare 3,2 → swap: [1, 2, 3, 5, 8]  ← done!

Time: O(n²) — terrible for large data. NEVER use in production.
But: easy to understand and implement.
```

### Merge Sort — The One Interviewers Love to Ask About

**How it works:** Divide the array in half. Recursively sort each half. Merge the two sorted halves together.

**Why it's O(n log n):** You divide log n times (halving), and at each level you do O(n) work to merge.

```
Split phase (top-down):
                [5, 3, 8, 1, 2, 7, 4, 6]
               /                          \
        [5, 3, 8, 1]                [2, 7, 4, 6]
        /          \                /          \
    [5, 3]      [8, 1]        [2, 7]      [4, 6]
    /   \       /   \         /   \       /   \
  [5]   [3]  [8]   [1]     [2]   [7]  [4]   [6]

Merge phase (bottom-up):
  [5]   [3]  → compare 5,3 → [3, 5]
  [8]   [1]  → compare 8,1 → [1, 8]
  [2]   [7]  → compare 2,7 → [2, 7]
  [4]   [6]  → compare 4,6 → [4, 6]

  [3, 5] and [1, 8] → merge:
    Compare 3,1 → take 1: [1, ...]
    Compare 3,8 → take 3: [1, 3, ...]
    Compare 5,8 → take 5: [1, 3, 5, ...]
    Take 8: [1, 3, 5, 8]

  [2, 7] and [4, 6] → merge: [2, 4, 6, 7]

  [1, 3, 5, 8] and [2, 4, 6, 7] → merge:
    [1, 2, 3, 4, 5, 6, 7, 8] ✓

Time:  O(n log n) — always (best, worst, average)
Space: O(n) — needs extra array for merging
Stable: Yes — equal elements keep their original order
```

### Quick Sort — The Fastest in Practice

**How it works:** Pick a "pivot" element. Partition the array so everything smaller is on the left, everything larger is on the right. Recursively sort left and right.

```
Array: [5, 3, 8, 1, 2, 7, 4, 6]
Pivot: 5

Partition: everything < 5 goes left, everything > 5 goes right
  Left:  [3, 1, 2, 4]    Pivot: [5]    Right: [8, 7, 6]

Recursively sort left and right:
  [1, 2, 3, 4]  +  [5]  +  [6, 7, 8]
  = [1, 2, 3, 4, 5, 6, 7, 8] ✓

Time:  O(n log n) average, O(n²) worst case (bad pivot choice)
Space: O(log n) — recursive call stack
Stable: No
```

### Summary Table

```
Algorithm       Time (Best)    Time (Avg)    Time (Worst)   Space    Stable
─────────       ──────────     ──────────    ────────────   ─────    ──────
Bubble Sort     O(n)           O(n²)         O(n²)          O(1)     Yes
Selection Sort  O(n²)          O(n²)         O(n²)          O(1)     No
Insertion Sort  O(n)           O(n²)         O(n²)          O(1)     Yes
Merge Sort      O(n log n)     O(n log n)    O(n log n)     O(n)     Yes
Quick Sort      O(n log n)     O(n log n)    O(n²)          O(log n) No
Heap Sort       O(n log n)     O(n log n)    O(n log n)     O(1)     No

JavaScript's Array.sort() uses TimSort (hybrid of merge + insertion): O(n log n)
```

### JavaScript Sorting

```javascript
// GOTCHA: sort() converts to strings by default!
[10, 9, 80].sort(); // [10, 80, 9] — WRONG! (lexicographic)
[10, 9, 80].sort((a, b) => a - b); // [9, 10, 80] — correct numeric sort

// How the comparator works:
// Return negative → a comes first
// Return positive → b comes first
// Return 0 → keep original order

// Sort objects by property
users.sort((a, b) => a.age - b.age); // ascending by age
users.sort((a, b) => b.age - a.age); // descending by age
users.sort((a, b) => a.name.localeCompare(b.name)); // alphabetical by name
```

---

# Chapter 7: Binary Search

## The Core Idea

Binary search works on **sorted** data. Instead of checking every element (O(n)), you check the middle, eliminate half, and repeat (O(log n)).

**Real-life analogy:** Looking up a word in a physical dictionary. You don't start at page 1. You open to the middle. Is your word before or after this page? Eliminate half the book. Repeat.

```
Searching for 7 in [1, 3, 5, 7, 9, 11, 13]:

Step 1: left=0, right=6, mid=3
  arr[3] = 7 === target ✓ Found at index 3!

Searching for 11 in [1, 3, 5, 7, 9, 11, 13]:

Step 1: left=0, right=6, mid=3
  [1, 3, 5, (7), 9, 11, 13]
             ↑ mid
  arr[3] = 7 < 11 → target is in the RIGHT half
  left = mid + 1 = 4

Step 2: left=4, right=6, mid=5
  [_, _, _, _, 9, (11), 13]
                    ↑ mid
  arr[5] = 11 === target ✓ Found at index 5!

Only 2 checks instead of 6. With 1 BILLION elements: ~30 checks.
```

```javascript
function binarySearch(arr, target) {
  let left = 0;
  let right = arr.length - 1;

  while (left <= right) {
    // note: <= not <
    const mid = Math.floor((left + right) / 2); // find middle index

    if (arr[mid] === target) {
      return mid; // found it!
    } else if (arr[mid] < target) {
      left = mid + 1; // target is in right half
    } else {
      right = mid - 1; // target is in left half
    }
  }

  return -1; // not found
}
// Time:  O(log n) — halve the search space each step
// Space: O(1)
```

## Problem: First Bad Version (LeetCode #278)

**Problem:** You have n versions [1, 2, ..., n]. One version is bad, and all after it are bad too. Find the FIRST bad version, calling isBadVersion() API as few times as possible.

```
Versions:  [1, 2, 3, 4, 5, 6, 7]
Bad?:      [✓, ✓, ✓, ✓, ✗, ✗, ✗]
                         ↑ first bad = 4

Binary search for the boundary between good and bad.

Step 1: mid=4, isBadVersion(4)=true → first bad is 4 or earlier → right=4
Step 2: mid=2, isBadVersion(2)=false → first bad is after 2 → left=3
Step 3: mid=3, isBadVersion(3)=false → first bad is after 3 → left=4
Step 4: left=4, right=4, left===right → answer is 4
```

```javascript
function firstBadVersion(n) {
  let left = 1,
    right = n;

  while (left < right) {
    const mid = Math.floor((left + right) / 2);

    if (isBadVersion(mid)) {
      right = mid; // mid might be the first bad, keep it in range
    } else {
      left = mid + 1; // mid is good, first bad must be after it
    }
  }

  return left; // left === right === first bad version
}
```

---

# Chapter 8: Stack

## What Is a Stack?

A stack is Last-In-First-Out (LIFO). Think of a stack of plates: you can only add to the top and remove from the top. The last plate you put on is the first one you take off.

```
Operations:
  push(x)  → add to top        O(1)
  pop()    → remove from top    O(1)
  peek()   → look at top        O(1)

Visual:
  push(A)    push(B)    push(C)    pop()      pop()
  ┌───┐      ┌───┐      ┌───┐     ┌───┐      ┌───┐
  │ A │      │ B │      │ C │     │ B │      │ A │
  └───┘      │ A │      │ B │     │ A │      └───┘
             └───┘      │ A │     └───┘
                        └───┘     returns C   returns B
```

In JavaScript, use an array: `push()` and `pop()` give you a stack.

```javascript
const stack = [];
stack.push(1); // [1]
stack.push(2); // [1, 2]
stack.push(3); // [1, 2, 3]
stack.pop(); // 3 — returns and removes top
stack[stack.length - 1]; // 2 — peek at top without removing
```

## Problem: Valid Parentheses (LeetCode #20)

**The most classic stack problem. Asked everywhere.**

**Problem:** Given a string containing just `(){}[]`, determine if the input is valid. Every open bracket must be closed by the same type in the correct order.

```
"()"      → true
"()[]{}"  → true
"(]"      → false
"([)]"    → false
"{[]}"    → true
```

**The insight:** When you see an opening bracket, you expect the matching closing bracket to come BEFORE any other closing bracket. That's LIFO — exactly what a stack does.

```
Processing "{[]}" step by step:

char='{': opening bracket → push onto stack
  Stack: ['{']

char='[': opening bracket → push onto stack
  Stack: ['{', '[']

char=']': closing bracket → does it match the top of stack?
  Top of stack is '[' → '[' matches ']' ✓ → pop
  Stack: ['{']

char='}': closing bracket → does it match the top?
  Top of stack is '{' → '{' matches '}' ✓ → pop
  Stack: []

End: stack is empty → all brackets matched → true ✓
```

```
Processing "([)]" step by step:

char='(': push → Stack: ['(']
char='[': push → Stack: ['(', '[']
char=')': top is '[' → '[' does NOT match ')' → false ✗

The ')' expects '(' but finds '[' — wrong nesting.
```

```javascript
function isValid(s) {
  const stack = [];
  const pairs = {
    ")": "(",
    "]": "[",
    "}": "{",
  };

  for (const char of s) {
    if (char === "(" || char === "[" || char === "{") {
      stack.push(char); // opening bracket → push
    } else {
      // closing bracket → check if it matches the top
      if (stack.length === 0 || stack[stack.length - 1] !== pairs[char]) {
        return false; // nothing to match, or wrong match
      }
      stack.pop(); // correct match → remove the opening bracket
    }
  }

  return stack.length === 0; // all brackets matched?
}
// Time:  O(n) — single pass through the string
// Space: O(n) — stack could hold all characters (e.g., "((((((")
```

---

# Chapter 9: Recursion & Backtracking

## What Is Recursion?

A function that calls itself. Every recursive function needs two things:

1. **Base case** — when to STOP (without this, infinite loop → stack overflow)
2. **Recursive case** — the function calls itself with a SMALLER problem

**Real-life analogy:** Russian nesting dolls (matryoshka). To find the smallest doll, you open the outer doll, then open the next one inside, then the next... until you reach one that doesn't open. That's the base case.

```javascript
// Example: Factorial (5! = 5 × 4 × 3 × 2 × 1 = 120)
function factorial(n) {
  if (n <= 1) return 1;           // base case: stop here
  return n * factorial(n - 1);    // recursive case: n × (n-1)!
}

// How it executes — the call stack:
factorial(5)
  → 5 * factorial(4)
    → 4 * factorial(3)
      → 3 * factorial(2)
        → 2 * factorial(1)
          → returns 1              ← base case hit, start returning
        → returns 2 * 1 = 2
      → returns 3 * 2 = 6
    → returns 4 * 6 = 24
  → returns 5 * 24 = 120
```

```
The Call Stack (visualized as literal stack of frames):

  ┌─────────────────┐
  │ factorial(1) = 1 │  ← base case, returns 1
  ├─────────────────┤
  │ factorial(2)     │  ← waiting, gets 1, returns 2×1=2
  ├─────────────────┤
  │ factorial(3)     │  ← waiting, gets 2, returns 3×2=6
  ├─────────────────┤
  │ factorial(4)     │  ← waiting, gets 6, returns 4×6=24
  ├─────────────────┤
  │ factorial(5)     │  ← waiting, gets 24, returns 5×24=120
  └─────────────────┘

  Each call adds a frame. Too many calls → Stack Overflow.
  That's why the base case is critical.
```

## What Is Backtracking?

Backtracking is recursion with "undo." You make a choice, explore it recursively, then UNDO the choice and try the next option. Think of it like navigating a maze: go forward, hit a dead end, back up, try a different path.

```
The backtracking template:

1. CHOOSE   — pick an option
2. EXPLORE  — recurse with that choice
3. UNCHOOSE — undo the choice (backtrack)

This explores ALL possible paths through the decision tree.
```

## Problem: Subsets (LeetCode #78)

**Problem:** Given a set of distinct integers, return all possible subsets.

```
Input:  [1, 2, 3]
Output: [[], [1], [2], [3], [1,2], [1,3], [2,3], [1,2,3]]
```

**How to think about it:** For each number, you have two choices: include it or exclude it. This creates a binary decision tree.

```
Decision tree for [1, 2, 3]:

                         []
                  /              \
              include 1        exclude 1
              [1]                []
            /      \           /      \
        inc 2    exc 2     inc 2    exc 2
        [1,2]    [1]       [2]      []
        / \      / \       / \      / \
     inc3 exc3 inc3 exc3 inc3 exc3 inc3 exc3
    [1,2,3][1,2][1,3][1] [2,3][2]  [3]  []

Every LEAF is a valid subset. Collect them all.
```

```javascript
function subsets(nums) {
  const result = [];

  function backtrack(index, current) {
    // Base case: we've considered all elements
    if (index === nums.length) {
      result.push([...current]); // save a COPY of current subset
      return;
    }

    // Choice 1: INCLUDE nums[index]
    current.push(nums[index]); // choose
    backtrack(index + 1, current); // explore
    current.pop(); // unchoose (backtrack!)

    // Choice 2: EXCLUDE nums[index]
    backtrack(index + 1, current); // explore without it
  }

  backtrack(0, []);
  return result;
}
// Time:  O(2ⁿ) — 2 choices per element, n elements
// Space: O(n) — recursion depth + current subset
```

```
Tracing through [1,2,3]:

backtrack(0, [])
  include 1: backtrack(1, [1])
    include 2: backtrack(2, [1,2])
      include 3: backtrack(3, [1,2,3]) → SAVE [1,2,3], return
      exclude 3: backtrack(3, [1,2])   → SAVE [1,2], return
    exclude 2: backtrack(2, [1])
      include 3: backtrack(3, [1,3])   → SAVE [1,3], return
      exclude 3: backtrack(3, [1])     → SAVE [1], return
  exclude 1: backtrack(1, [])
    include 2: backtrack(2, [2])
      include 3: backtrack(3, [2,3])   → SAVE [2,3], return
      exclude 3: backtrack(3, [2])     → SAVE [2], return
    exclude 2: backtrack(2, [])
      include 3: backtrack(3, [3])     → SAVE [3], return
      exclude 3: backtrack(3, [])      → SAVE [], return

Result: [[1,2,3],[1,2],[1,3],[1],[2,3],[2],[3],[]]
```

---

# Chapter 10: Linked Lists

## What Is a Linked List?

An array stores elements in contiguous memory (side by side). A linked list stores elements scattered in memory, connected by pointers.

```
Array:          [10, 20, 30, 40]
Memory:         |10|20|30|40|     (contiguous block)

Linked List:    10 → 20 → 30 → 40 → null
Memory:         10 is at address 0x1A
                20 is at address 0x7F    (not next to 10!)
                30 is at address 0x3C
                Each node stores: { value, pointer to next node }
```

```
Node structure:
  ┌───────┬───────┐     ┌───────┬───────┐     ┌───────┬───────┐
  │ val:1 │ next:─┼────▶│ val:2 │ next:─┼────▶│ val:3 │ next: │──▶ null
  └───────┴───────┘     └───────┴───────┘     └───────┴───────┘
     head                                        tail
```

```javascript
// Node definition
class ListNode {
  constructor(val, next = null) {
    this.val = val;
    this.next = next;
  }
}

// Building: 1 → 2 → 3
const head = new ListNode(1, new ListNode(2, new ListNode(3)));

// Traversing
let current = head;
while (current !== null) {
  console.log(current.val); // 1, 2, 3
  current = current.next;
}
```

### Array vs Linked List

```
Operation          Array       Linked List
─────────          ─────       ───────────
Access by index    O(1)        O(n) — must walk from head
Insert at end      O(1)*       O(n) — must find the end first (O(1) with tail pointer)
Insert at start    O(n)        O(1) — just update head pointer
Insert at middle   O(n)        O(1) — if you have the node reference
Delete at start    O(n)        O(1)
Search             O(n)        O(n)

* amortized — array may need to resize
```

## Problem: Reverse Linked List (LeetCode #206)

**The most classic linked list problem.**

```
Input:  1 → 2 → 3 → 4 → 5 → null
Output: 5 → 4 → 3 → 2 → 1 → null
```

**The idea:** Walk through the list. At each node, flip its `next` pointer to point backward instead of forward. You need to save the next node BEFORE flipping (otherwise you lose it).

```
Step-by-step:

Start:    null  ←  1 → 2 → 3 → null
          prev    curr

Step 1: Save next (2). Point curr.next to prev. Move forward.
          null ← 1    2 → 3 → null
          prev  curr

Step 2: Save next (3). Point curr.next to prev. Move forward.
          null ← 1 ← 2    3 → null
                prev curr

Step 3: Save next (null). Point curr.next to prev. Move forward.
          null ← 1 ← 2 ← 3    null
                     prev curr

curr is null → stop. prev is the new head.
Result: 3 → 2 → 1 → null ✓
```

```javascript
function reverseList(head) {
  let prev = null;
  let curr = head;

  while (curr !== null) {
    const next = curr.next; // save next node before we break the link
    curr.next = prev; // flip the pointer backward
    prev = curr; // move prev forward
    curr = next; // move curr forward
  }

  return prev; // prev is now the new head
}
// Time:  O(n) — visit each node once
// Space: O(1) — just three pointers
```

## Problem: Detect Cycle (LeetCode #141)

**Problem:** Does this linked list have a cycle (a node that points back to a previous node)?

**Floyd's Tortoise and Hare:** Use two pointers — slow (moves 1 step) and fast (moves 2 steps). If there's a cycle, fast will eventually catch up to slow (like two runners on a circular track). If there's no cycle, fast reaches the end.

```
No cycle:
  slow: 1 → 2 → 3 → 4 → null
  fast: 1 → 3 → null (reaches end → no cycle)

With cycle:
  1 → 2 → 3 → 4 → 5
              ↑         ↓
              └─────────┘

  slow: 1, 2, 3, 4, 5, 3, 4, 5, 3...
  fast: 1, 3, 5, 4, 3, 5, 4, 3...
  They meet at some point → cycle detected!
```

```javascript
function hasCycle(head) {
  let slow = head;
  let fast = head;

  while (fast !== null && fast.next !== null) {
    slow = slow.next; // move 1 step
    fast = fast.next.next; // move 2 steps

    if (slow === fast) {
      return true; // they met → cycle exists
    }
  }

  return false; // fast reached the end → no cycle
}
// Time:  O(n)
// Space: O(1) — just two pointers
```

---

# Chapter 11: Trees and BFS/DFS

## What Is a Tree?

A tree is a hierarchical data structure. It has a root node at the top, and each node can have child nodes. A **binary tree** means each node has at most 2 children (left and right).

```
        Binary Tree:                    Not a binary tree:
            1                               1
           / \                           / | \
          2   3                         2  3  4
         / \   \                       /|\    |
        4   5   6                     5 6 7   8

Node structure:
  ┌────────┬──────┬───────┐
  │ val: 1 │ left │ right │
  └────────┴──┬───┴───┬───┘
              ↓       ↓
           node 2   node 3
```

```javascript
class TreeNode {
  constructor(val, left = null, right = null) {
    this.val = val;
    this.left = left;
    this.right = right;
  }
}

// Build this tree:
//       1
//      / \
//     2   3
//    / \
//   4   5
const root = new TreeNode(
  1,
  new TreeNode(2, new TreeNode(4), new TreeNode(5)),
  new TreeNode(3),
);
```

### Binary Search Tree (BST)

A BST has a special rule: for every node, ALL values in the left subtree are SMALLER, and ALL values in the right subtree are LARGER.

```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13

  Left of 8: all < 8 ✓ (3, 1, 6, 4, 7)
  Right of 8: all > 8 ✓ (10, 14, 13)

  This makes search O(log n) — like binary search on an array.
  Go left if target < current, right if target > current.
```

## Tree Traversals — The Four Ways to Visit Every Node

### DFS: Depth-First Search (Goes Deep Before Wide)

Three flavors, depending on WHEN you process the current node:

```
Tree:       1
           / \
          2   3
         / \
        4   5

Preorder  (Node, Left, Right):   1, 2, 4, 5, 3
  "Visit myself first, then my children"
  Use: copy/serialize a tree, prefix expression

Inorder   (Left, Node, Right):   4, 2, 5, 1, 3
  "Visit left child, then myself, then right child"
  Use: BST → gives elements in SORTED ORDER

Postorder (Left, Right, Node):   4, 5, 2, 3, 1
  "Visit both children first, then myself"
  Use: delete a tree, calculate directory size
```

```javascript
// Preorder: Node → Left → Right
function preorder(node) {
  if (node === null) return;
  console.log(node.val); // process FIRST
  preorder(node.left);
  preorder(node.right);
}

// Inorder: Left → Node → Right
function inorder(node) {
  if (node === null) return;
  inorder(node.left);
  console.log(node.val); // process in MIDDLE
  inorder(node.right);
}

// Postorder: Left → Right → Node
function postorder(node) {
  if (node === null) return;
  postorder(node.left);
  postorder(node.right);
  console.log(node.val); // process LAST
}
```

### BFS: Breadth-First Search (Level Order — Goes Wide Before Deep)

Visit all nodes at depth 0, then depth 1, then depth 2, etc. Uses a **queue**.

```
Tree:       1          Level 0
           / \
          2   3        Level 1
         / \   \
        4   5   6      Level 2

BFS visits: 1, 2, 3, 4, 5, 6 (level by level)
```

```javascript
function bfs(root) {
  if (!root) return [];

  const result = [];
  const queue = [root]; // start with root in the queue

  while (queue.length > 0) {
    const node = queue.shift(); // remove from front (FIFO)
    result.push(node.val);

    if (node.left) queue.push(node.left); // add children to back
    if (node.right) queue.push(node.right);
  }

  return result;
}
// For the tree above: [1, 2, 3, 4, 5, 6]
```

**Level-order with levels separated:**

```javascript
function levelOrder(root) {
  if (!root) return [];

  const result = [];
  const queue = [root];

  while (queue.length > 0) {
    const levelSize = queue.length; // how many nodes in this level
    const currentLevel = [];

    for (let i = 0; i < levelSize; i++) {
      const node = queue.shift();
      currentLevel.push(node.val);
      if (node.left) queue.push(node.left);
      if (node.right) queue.push(node.right);
    }

    result.push(currentLevel);
  }

  return result;
}
// [[1], [2, 3], [4, 5, 6]]
```

## Problem: Maximum Depth of Binary Tree (LeetCode #104)

```
Tree:       3
           / \
          9  20
            /  \
           15   7

Max depth = 3 (path: 3 → 20 → 15 or 3 → 20 → 7)
```

**The insight:** The depth of a tree = 1 + max(depth of left subtree, depth of right subtree). This is naturally recursive.

```
depth(3)
  = 1 + max(depth(9), depth(20))
  = 1 + max(1 + max(depth(null), depth(null)),
             1 + max(depth(15), depth(7)))
  = 1 + max(1 + max(0, 0),
             1 + max(1, 1))
  = 1 + max(1, 2)
  = 1 + 2
  = 3
```

```javascript
function maxDepth(root) {
  if (root === null) return 0; // base case: empty tree has depth 0

  const leftDepth = maxDepth(root.left);
  const rightDepth = maxDepth(root.right);

  return 1 + Math.max(leftDepth, rightDepth);
}
// Time:  O(n) — visit every node once
// Space: O(h) — h = height of tree (recursion stack depth)
//        O(log n) for balanced tree, O(n) for skewed tree
```

## Problem: Invert Binary Tree (LeetCode #226)

```
Input:       4              Output:      4
           /   \                       /   \
          2     7                     7     2
         / \   / \                   / \   / \
        1   3 6   9                 9   6 3   1
```

**The insight:** At every node, swap its left and right children. Then recursively invert both subtrees.

```javascript
function invertTree(root) {
  if (root === null) return null;

  // Swap left and right children
  [root.left, root.right] = [root.right, root.left];

  // Recursively invert both subtrees
  invertTree(root.left);
  invertTree(root.right);

  return root;
}
// Time:  O(n) — visit every node
// Space: O(h) — recursion depth = tree height
```

## BFS/DFS on Grids (2D Arrays)

Many interview problems present a 2D grid and ask you to explore it. This is the same as tree BFS/DFS, but instead of left/right children, you have 4 neighbors (up, down, left, right).

## Problem: Number of Islands (LeetCode #200)

**Problem:** Given a 2D grid of '1's (land) and '0's (water), count the number of islands. An island is surrounded by water and connected horizontally/vertically.

```
Input:
  1 1 0 0 0
  1 1 0 0 0
  0 0 1 0 0
  0 0 0 1 1

Output: 3

Island 1: top-left (four 1's connected)
Island 2: middle (one 1)
Island 3: bottom-right (two 1's connected)
```

**The approach:** Scan the grid. When you find a '1' you haven't visited, that's a new island. Use DFS/BFS to "flood fill" all connected '1's (mark them as visited so you don't count them again).

```javascript
function numIslands(grid) {
  if (!grid.length) return 0;

  let count = 0;
  const rows = grid.length;
  const cols = grid[0].length;

  function dfs(r, c) {
    // Out of bounds or water or already visited → stop
    if (r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] === "0") {
      return;
    }

    grid[r][c] = "0"; // mark as visited (sink the land)

    // Explore all 4 directions
    dfs(r + 1, c); // down
    dfs(r - 1, c); // up
    dfs(r, c + 1); // right
    dfs(r, c - 1); // left
  }

  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === "1") {
        count++; // found a new island!
        dfs(r, c); // sink the entire island (mark all connected 1's)
      }
    }
  }

  return count;
}
// Time:  O(rows × cols) — visit each cell at most once
// Space: O(rows × cols) — worst case recursion depth (all land)
```

```
Walking through the grid:

1 1 0 0 0         After DFS from (0,0):    After DFS from (2,2):    After DFS from (3,3):
1 1 0 0 0         0 0 0 0 0                0 0 0 0 0                0 0 0 0 0
0 0 1 0 0         0 0 0 0 0                0 0 0 0 0                0 0 0 0 0
0 0 0 1 1         0 0 1 0 0                0 0 0 0 0                0 0 0 0 0
                  0 0 0 1 1                0 0 0 1 1                0 0 0 0 0
                  count=1                  count=2                  count=3
```

---

# DFS vs BFS — When to Use Which

```
BFS (Breadth-First):
  Data structure: Queue (FIFO)
  Explores: Level by level (nearest first)
  Best for:
    • Shortest path in unweighted graph
    • Level-order traversal
    • "Find nearest X" problems

DFS (Depth-First):
  Data structure: Stack (or recursion)
  Explores: As deep as possible first
  Best for:
    • Detecting cycles
    • Path existence ("is there a path from A to B?")
    • Topological sorting
    • Backtracking (subsets, permutations)
    • Tree problems (most are naturally DFS)
```

---

# Tier 2 Summary — Pattern Recognition Guide

```
"The array is sorted"                      → Binary Search or Two Pointers
"Find the shortest path"                   → BFS
"Explore all possibilities"                → Backtracking (DFS + undo)
"Check if brackets/tags are balanced"      → Stack
"Traverse a tree"                          → DFS (recursive) or BFS (queue)
"Connected components in a grid"           → DFS/BFS flood fill
"Reverse a linked list"                    → Three-pointer walk (prev, curr, next)
"Detect cycle in linked list"              → Fast & slow pointers
"Sort then do something"                   → Sort first, then apply Tier 1 pattern
"Find kth largest/smallest"               → Sort, or Heap (Tier 3)
```
