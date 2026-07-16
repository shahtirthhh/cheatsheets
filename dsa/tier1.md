# DSA Tier 1 — The 60% That Matters Most

_Hash Maps · Two Pointers · Sliding Window · Strings · Array Patterns_
_Language: JavaScript · Level: Teaching from absolute zero_

---

# Chapter 1: Hash Maps (Dictionaries)

## What Is a Hash Map?

Imagine you have a box of 1,000 student files. Someone asks "Give me Alice's file." You have two choices:

**Option A (Array/List):** Start from the first file, check the name, move to the next. On average, you check 500 files before finding Alice. With 1 million files? Check 500,000.

**Option B (Hash Map):** The files are organized by a formula. Alice's name goes through a formula that says "slot 47." You walk straight to slot 47. Done. With 1 million files? Still walk to one slot.

That's a hash map. A key (like "Alice") is converted to an index by a hash function, giving you instant access.

```
How a hash map works step-by-step:

1. You want to store: "Alice" → "A+ student"

2. Hash function converts the key:
   hash("Alice") → 47

3. Store the value at index 47 in an internal array:
   internal[47] = { key: "Alice", value: "A+ student" }

4. Later, you want to look up "Alice":
   hash("Alice") → 47 → go to internal[47] → found!

This is why lookup is O(1) — the hash function tells you EXACTLY where to look.
```

### What Happens When Two Keys Hash to the Same Index? (Collision)

```
hash("Alice") → 47
hash("Aaron") → 47    ← collision!

Solution: Chaining — each slot holds a linked list
  internal[47] → ["Alice":"A+"] → ["Aaron":"B+"]

When looking up "Aaron":
  1. hash("Aaron") → 47
  2. Go to slot 47
  3. Walk the chain: "Alice"? No. "Aaron"? Yes!

This is why worst case is O(n) — if ALL keys collide, you're back to a list.
But with a good hash function, collisions are rare → average O(1).
```

## JavaScript Hash Map: Three Ways

```javascript
// WAY 1: Plain Object (most common in interviews)
const freq = {};
freq["apple"] = 3;
freq["banana"] = 5;
console.log(freq["apple"]);      // 3
console.log("apple" in freq);    // true (check if key exists)
delete freq["apple"];             // remove a key
Object.keys(freq);                // ["banana"] — all keys as array
Object.values(freq);              // [5] — all values
Object.entries(freq);             // [["banana", 5]] — key-value pairs

// WAY 2: Map Object (better features, less common in interviews)
const map = new Map();
map.set("name", "Alice");
map.set(42, "number key");       // ANY type as key (objects can't do this reliably)
map.get("name");                 // "Alice"
map.has("name");                 // true
map.delete("name");
map.size;                        // 1 (objects don't have .size)
for (const [key, value] of map) { ... }  // iterates in insertion order

// WAY 3: Set (only keys, no values — for "have I seen this?")
const seen = new Set();
seen.add(5);
seen.add(5);                     // ignored — already exists
seen.has(5);                     // true
seen.size;                       // 1
seen.delete(5);
```

### When to Reach for a Hash Map — Decision Guide

```
Ask yourself this about the problem:

"Do I need to COUNT how many times something appears?"
   → Yes → use an object as a frequency counter
     freq[item] = (freq[item] || 0) + 1;

"Do I need to check if I've SEEN something before?"
   → Yes → use a Set
     if (set.has(item)) found duplicate!

"Do I need to find a COMPLEMENT (like target - current)?"
   → Yes → use an object to store values you've seen
     Two Sum pattern: map[value] = index

"Do I need to GROUP things by a property?"
   → Yes → use an object with arrays as values
     map[key] = map[key] || []; map[key].push(item);

"Am I doing nested loops just to find matching pairs?"
   → Yes → a hash map can probably eliminate the inner loop
     O(n²) → O(n)
```

## The Frequency Counter Pattern (Most Common)

This pattern counts occurrences. You'll use it in anagram checking, finding duplicates, finding the most/least frequent element, and more.

```javascript
// THE TEMPLATE — memorize this rhythm
function countFrequency(arr) {
  const freq = {};

  for (const item of arr) {
    freq[item] = (freq[item] || 0) + 1;
    //           └─ if freq[item] is undefined, use 0, then add 1
  }

  return freq;
}
```

Let me walk through this line by line with an example:

```
Input: ["a", "b", "a", "c", "b", "a"]

Iteration 1: item = "a"
  freq["a"] exists? No → (undefined || 0) + 1 = 1
  freq = { a: 1 }

Iteration 2: item = "b"
  freq["b"] exists? No → (undefined || 0) + 1 = 1
  freq = { a: 1, b: 1 }

Iteration 3: item = "a"
  freq["a"] exists? Yes, it's 1 → (1 || 0) + 1 = 2
  freq = { a: 2, b: 1 }

Iteration 4: item = "c"
  freq["c"] exists? No → 0 + 1 = 1
  freq = { a: 2, b: 1, c: 1 }

Iteration 5: item = "b"
  freq["b"] = 1 → 1 + 1 = 2
  freq = { a: 2, b: 2, c: 1 }

Iteration 6: item = "a"
  freq["a"] = 2 → 2 + 1 = 3
  freq = { a: 3, b: 2, c: 1 }

Final: { a: 3, b: 2, c: 1 }
```

## Problem: Two Sum (LeetCode #1)

**The most asked interview question in history.** If you learn ONE problem, learn this one.

**Problem:** Given an array of numbers and a target, find two numbers that add up to the target. Return their indices.

```
Input:  nums = [2, 7, 11, 15], target = 9
Output: [0, 1]
Why:    nums[0] + nums[1] = 2 + 7 = 9
```

### Brute Force: Try Every Pair

**Thinking process:** The simplest approach — pick a number, then look at every other number to see if they add up to the target. This means two nested loops.

**Why it's slow:** For an array of 10,000 elements, you'd check ~50 million pairs. The outer loop runs n times, and for EACH of those, the inner loop runs up to n times. That's n × n = n².

```javascript
function twoSumBrute(nums, target) {
  // For each number...
  for (let i = 0; i < nums.length; i++) {
    // ...check every number after it
    for (let j = i + 1; j < nums.length; j++) {
      if (nums[i] + nums[j] === target) {
        return [i, j];
      }
    }
  }
  return [];
}
// Time:  O(n²) — two nested loops
// Space: O(1)  — no extra data structures
```

```
Walking through [2, 7, 11, 15], target = 9:

i=0, j=1: nums[0] + nums[1] = 2 + 7 = 9 ✓ → return [0, 1]

(If not found that early, it would continue:)
i=0, j=2: 2 + 11 = 13 ✗
i=0, j=3: 2 + 15 = 17 ✗
i=1, j=2: 7 + 11 = 18 ✗
i=1, j=3: 7 + 15 = 22 ✗
i=2, j=3: 11 + 15 = 26 ✗
```

### Optimized: Hash Map (One Pass)

**The key insight that changes everything:** If I'm looking at the number 2 and the target is 9, then I need 9 - 2 = 7 to exist somewhere in the array. Instead of looping through the whole array to find 7, I can ask a hash map "have you seen 7?" — and the hash map answers in O(1).

**How it works:** Walk through the array once. For each number, calculate what its partner would be (target - current). Check if that partner is already in the map. If yes, we found our pair. If no, store the current number in the map for future lookups.

```
Walking through [2, 7, 11, 15], target = 9:

We maintain a hash map: { value → index }

Step 1: num = 2, index = 0
  I need: 9 - 2 = 7
  Is 7 in my map? Map is empty, so no.
  Store 2 in map: { 2: 0 }

  Think: "I haven't found 7 yet, but if I see it later,
          I'll know that 2 was at index 0."

Step 2: num = 7, index = 1
  I need: 9 - 7 = 2
  Is 2 in my map? YES! It's at index 0!
  Return [0, 1] ✓

  Think: "I need 2, and I remember seeing 2 at index 0.
          So indices 0 and 1 make the pair."
```

```javascript
function twoSum(nums, target) {
  const map = {}; // stores: number → its index

  for (let i = 0; i < nums.length; i++) {
    const complement = target - nums[i]; // what do I need?

    if (complement in map) {
      // have I seen it before?
      return [map[complement], i]; // yes! return both indices
    }

    map[nums[i]] = i; // no? store current for later
  }

  return [];
}
// Time:  O(n) — single loop, each hash map lookup is O(1)
// Space: O(n) — hash map stores up to n elements
```

**Why this is O(n) instead of O(n²):** The brute force uses the inner loop to search for the complement — that search is O(n). The hash map replaces that search with an O(1) lookup. So we go from O(n × n) to O(n × 1) = O(n).

## Problem: Group Anagrams (LeetCode #49)

**Problem:** Group strings that are anagrams of each other.

```
Input:  ["eat", "tea", "tan", "ate", "nat", "bat"]
Output: [["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

### Brute Force Thinking

Compare every pair of strings to check if they're anagrams. For each pair, count character frequencies and compare. With n strings of average length k, this is O(n² × k).

### Optimized: Sorted String as Map Key

**The insight:** All anagrams have the same characters, just in different order. If you sort the characters of each word, anagrams produce identical sorted strings. Use that sorted string as a hash map key to group them.

```
"eat" → sort letters → "aet"
"tea" → sort letters → "aet"    ← same key! They're anagrams.
"ate" → sort letters → "aet"    ← same key!

"tan" → sort letters → "ant"
"nat" → sort letters → "ant"    ← same key!

"bat" → sort letters → "abt"    ← unique key

Map after processing all words:
  "aet" → ["eat", "tea", "ate"]
  "ant" → ["tan", "nat"]
  "abt" → ["bat"]
```

```javascript
function groupAnagrams(strs) {
  const map = {};

  for (const str of strs) {
    // Sort the characters to create a canonical key
    const key = str.split("").sort().join("");
    // "eat" → ["e","a","t"] → ["a","e","t"] → "aet"

    if (!map[key]) map[key] = []; // first time seeing this key? create array
    map[key].push(str); // add original string to its group
  }

  return Object.values(map); // extract just the grouped arrays
}
// Time:  O(n × k log k) where n = number of strings, k = max string length
//        (sorting each string is k log k, done n times)
// Space: O(n × k) — storing all strings in the map
```

---

# Chapter 2: Two Pointers

## What Is Two Pointers?

Imagine you have a sorted row of numbered cards on a table. You're asked to find two cards that add up to 10.

**Brute force:** Pick up card 1, compare it with every other card. Then pick up card 2, compare with every other card. That's O(n²).

**Two pointers:** Place your left hand on the smallest card. Place your right hand on the largest card. Look at the sum:

- Too big? Move your right hand one card left (smaller number).
- Too small? Move your left hand one card right (larger number).
- Just right? Done!

You never go backwards. Each hand only moves in one direction. This guarantees you find the answer in a single pass: O(n).

```
Why does this work? Because the array is SORTED:

Cards: [1, 2, 4, 6, 8, 9, 14]    target = 10
        L                    R

L points to smallest available number.
R points to largest available number.

sum = 1 + 14 = 15 > 10
  The right number is too big. Making L bigger would only increase the sum more.
  So we MUST decrease R.

Cards: [1, 2, 4, 6, 8, 9, 14]    target = 10
        L                 R

sum = 1 + 9 = 10 ✓   Found it!

At each step, we can PROVE that one direction is wrong, so we never need to backtrack.
This is what makes two pointers work — the sorted order gives us information to eliminate options.
```

### The Two Main Patterns

```
Pattern A: Opposite Direction (Converging)
  Used when: array is sorted, finding pairs, palindrome checking

  [□ □ □ □ □ □ □ □]
   L→            ←R      (move toward each other)


Pattern B: Same Direction (Fast & Slow)
  Used when: linked list cycle, removing elements, finding middle

  [□ □ □ □ □ □ □ □]
   S→  F→→             (fast moves 2× speed of slow)
```

## Problem: Valid Palindrome (LeetCode #125)

**Problem:** Is a string the same forwards and backwards? Ignore non-letters and case.

```
"A man, a plan, a canal: Panama" → true
"race a car" → false
```

### Brute Force: Clean and Reverse

Remove all non-alphanumeric characters, convert to lowercase, then reverse and compare. This requires creating two entirely new strings.

```javascript
function isPalindromeBrute(s) {
  const cleaned = s.replace(/[^a-zA-Z0-9]/g, "").toLowerCase();
  return cleaned === cleaned.split("").reverse().join("");
}
// Time:  O(n) — but makes 3 copies of the string
// Space: O(n) — three new strings in memory
```

### Optimized: Two Pointers

**The insight:** A palindrome reads the same from both ends. Instead of creating new strings, put one pointer at the start and one at the end. Compare characters, skipping non-alphanumeric ones. If all pairs match, it's a palindrome.

```
"A man, a plan, a canal: Panama"
 L                             R

Step 1: s[L]='A', s[R]='a' → both alpha → 'a'=='a' ✓ → move both inward
         L                           R
"A man, a plan, a canal: Panama"
  L                           R

Step 2: s[L]=' ' → not alphanumeric → skip L
"A man, a plan, a canal: Panama"
   L                          R

Step 3: s[L]='m', s[R]='m' → 'm'=='m' ✓ → move both inward

... continue until L crosses R → palindrome confirmed ✓
```

```javascript
function isPalindrome(s) {
  let left = 0;
  let right = s.length - 1;

  while (left < right) {
    // Skip non-alphanumeric from left
    while (left < right && !/[a-zA-Z0-9]/.test(s[left])) {
      left++;
    }
    // Skip non-alphanumeric from right
    while (left < right && !/[a-zA-Z0-9]/.test(s[right])) {
      right--;
    }

    // Compare (case-insensitive)
    if (s[left].toLowerCase() !== s[right].toLowerCase()) {
      return false; // mismatch → not a palindrome
    }

    left++;
    right--;
  }

  return true;
}
// Time:  O(n) — each pointer moves at most n times total
// Space: O(1) — only two integer variables, no new strings
```

## Problem: Two Sum II — Sorted Array (LeetCode #167)

**Problem:** Array is already sorted. Find two numbers that add up to target.

```
Input:  numbers = [2, 3, 7, 11, 15], target = 9
Output: [0, 2]   (because 2 + 7 = 9)
```

### Why This is Different from Two Sum #1

Two Sum #1 has an unsorted array — you need a hash map. This array is sorted — two pointers is more elegant and uses O(1) space instead of O(n).

### Step-by-Step Walkthrough

```
[2, 3, 7, 11, 15]    target = 9
 L              R

Step 1: sum = nums[L] + nums[R] = 2 + 15 = 17
  17 > 9 → sum is too LARGE
  How to make sum smaller? We need a smaller number on the right.
  Move R left.

[2, 3, 7, 11, 15]    target = 9
 L          R

Step 2: sum = 2 + 11 = 13
  13 > 9 → still too large → move R left

[2, 3, 7, 11, 15]    target = 9
 L      R

Step 3: sum = 2 + 7 = 9
  9 === 9 ✓ → FOUND! Return [0, 2]

Why not move L instead of R in step 1?
  If 2 + 15 is already too big, then 3 + 15 would be EVEN bigger.
  So moving L right is guaranteed to be wrong.
  The sorted order tells us which direction to go.
```

```javascript
function twoSumSorted(numbers, target) {
  let left = 0;
  let right = numbers.length - 1;

  while (left < right) {
    const sum = numbers[left] + numbers[right];

    if (sum === target) {
      return [left, right]; // found the pair
    } else if (sum < target) {
      left++; // sum too small → need bigger number on left
    } else {
      right--; // sum too big → need smaller number on right
    }
  }

  return [];
}
// Time:  O(n) — each pointer moves at most n times
// Space: O(1) — just two variables
```

## The Two Pointer Template

```javascript
// Template A: Converging pointers (sorted array, palindrome)
function converging(arr) {
  let left = 0;
  let right = arr.length - 1;

  while (left < right) {
    // Compute something with arr[left] and arr[right]
    const result = compute(arr[left], arr[right]);

    if (result === target) {
      return [left, right]; // found what we need
    } else if (result < target) {
      left++; // need bigger → move left forward
    } else {
      right--; // need smaller → move right backward
    }
  }
}

// Template B: Fast and slow (linked lists, cycle detection)
function fastSlow(head) {
  let slow = head;
  let fast = head;

  while (fast && fast.next) {
    slow = slow.next; // move 1 step
    fast = fast.next.next; // move 2 steps

    if (slow === fast) {
      return true; // cycle detected (they met!)
    }
  }

  return false; // fast reached the end → no cycle
}
```

---

# Chapter 3: Sliding Window

## What Is a Sliding Window?

Imagine you're reading a book and you need to find the most interesting 3-page section. You could re-read every possible set of 3 pages from scratch — but that's wasteful. Instead, you read pages 1-3, then "slide" your view: forget page 1, read page 4. Now you're looking at pages 2-4. Slide again: forget page 2, read page 5. Pages 3-5.

At each step, you only process ONE new page instead of re-reading all three. That's the sliding window technique.

```
Why it's faster — a visual:

Array: [3, 1, 4, 1, 5, 9, 2, 6]     window size k = 3

Brute Force (re-sum every window):
  Window 1: [3, 1, 4]         → 3+1+4 = 8    (3 additions)
  Window 2: [1, 4, 1]         → 1+4+1 = 6    (3 additions)
  Window 3: [4, 1, 5]         → 4+1+5 = 10   (3 additions)
  Window 4: [1, 5, 9]         → 1+5+9 = 15   (3 additions)
  Total work: n × k additions

Sliding Window (adjust by 1 element):
  Window 1: [3, 1, 4]         → 3+1+4 = 8    (3 additions, first window only)
  Window 2: [_, 1, 4, 1]      → 8 - 3 + 1 = 6   (1 subtraction + 1 addition)
  Window 3: [_, _, 4, 1, 5]   → 6 - 1 + 5 = 10  (1 subtraction + 1 addition)
  Window 4: [_, _, _, 1, 5, 9] → 10 - 4 + 9 = 15 (1 subtraction + 1 addition)
  Total work: k + (n-k) × 2 = O(n)
```

### Two Types of Sliding Window

**Fixed-size window:** The window is always k elements wide. You just slide it one step at a time. Used when the problem says "K consecutive elements" or "window of size K."

**Variable-size window:** The window grows and shrinks dynamically. You expand it (move right pointer) to include more elements, and shrink it (move left pointer) when a condition is violated. Used when the problem says "longest/shortest substring/subarray satisfying some condition."

```
Fixed-size (k=3):
  [■ ■ ■] □ □ □ □       always exactly 3 elements
  □ [■ ■ ■] □ □ □
  □ □ [■ ■ ■] □ □
  □ □ □ [■ ■ ■] □

Variable-size:
  [■] □ □ □ □ □ □       start small
  [■ ■ ■ ■] □ □ □       grow until condition breaks
  □ [■ ■ ■] □ □ □       shrink from left to restore condition
  □ [■ ■ ■ ■ ■] □       grow again
  □ □ □ [■ ■] □ □       shrink again
```

## Problem: Maximum Sum Subarray of Size K (Fixed Window)

**Problem:** Find the maximum sum of any K consecutive elements.

```
Input:  [2, 1, 5, 1, 3, 2], k = 3
Output: 9   (subarray [5, 1, 3] has sum 5+1+3=9)
```

### Brute Force

For every starting position, sum the next k elements. Two nested loops.

```javascript
function maxSumBrute(arr, k) {
  let maxSum = -Infinity;

  for (let i = 0; i <= arr.length - k; i++) {
    let windowSum = 0;
    for (let j = i; j < i + k; j++) {
      windowSum += arr[j];
    }
    maxSum = Math.max(maxSum, windowSum);
  }

  return maxSum;
}
// Time:  O(n × k) — for each of n positions, sum k elements
// Space: O(1)
```

### Optimized: Fixed Sliding Window

**The insight:** When you slide the window right by one position, you add the new element on the right and remove the old element on the left. The sum changes by exactly (new element - old element). No need to re-sum everything.

```
Array: [2, 1, 5, 1, 3, 2], k = 3

PHASE 1 — Build the first window:
  Add arr[0]=2, arr[1]=1, arr[2]=5
  windowSum = 2 + 1 + 5 = 8
  maxSum = 8

  [2, 1, 5] 1, 3, 2     sum=8

PHASE 2 — Slide one step at a time:
  i=3: Add arr[3]=1, Remove arr[0]=2
    windowSum = 8 + 1 - 2 = 7
    maxSum = max(8, 7) = 8

    2 [1, 5, 1] 3, 2     sum=7

  i=4: Add arr[4]=3, Remove arr[1]=1
    windowSum = 7 + 3 - 1 = 9
    maxSum = max(8, 9) = 9

    2, 1 [5, 1, 3] 2     sum=9 ← new max!

  i=5: Add arr[5]=2, Remove arr[2]=5
    windowSum = 9 + 2 - 5 = 6
    maxSum = max(9, 6) = 9

    2, 1, 5 [1, 3, 2]    sum=6

Answer: 9
```

```javascript
function maxSum(arr, k) {
  // Phase 1: Build the first window
  let windowSum = 0;
  for (let i = 0; i < k; i++) {
    windowSum += arr[i];
  }
  let maxSum = windowSum;

  // Phase 2: Slide the window
  for (let i = k; i < arr.length; i++) {
    windowSum += arr[i]; // add new element entering from right
    windowSum -= arr[i - k]; // remove element leaving from left
    maxSum = Math.max(maxSum, windowSum);
  }

  return maxSum;
}
// Time:  O(n) — single pass
// Space: O(1) — just two variables
```

## Problem: Longest Substring Without Repeating Characters (LeetCode #3)

**One of the most asked medium-difficulty problems.** Variable-size sliding window.

```
Input:  "abcabcbb"
Output: 3   (the longest substring without repeating characters is "abc")

Input:  "bbbbb"
Output: 1   (just "b")
```

### Brute Force Thinking

Check every possible substring. For each one, verify there are no duplicate characters. There are O(n²) substrings and checking each takes O(n), so this is O(n³).

### Optimized: Variable Sliding Window + Set

**The insight:** Maintain a window (defined by left and right pointers) that contains only unique characters. Expand the window by moving right. If a duplicate character enters, shrink from the left until the duplicate is removed.

**Why a Set?** We need to instantly know "is this character already in my window?" A Set answers that in O(1).

```
String: "a b c a b c b b"
         0 1 2 3 4 5 6 7

Step-by-step:

right=0: char='a', set={}, 'a' not in set
  Add 'a' to set. set={a}. Window: "a" (length 1). maxLen=1
  L=0, R=0

right=1: char='b', 'b' not in set
  Add 'b'. set={a,b}. Window: "ab" (length 2). maxLen=2
  L=0, R=1

right=2: char='c', 'c' not in set
  Add 'c'. set={a,b,c}. Window: "abc" (length 3). maxLen=3
  L=0, R=2

right=3: char='a', 'a' IS IN SET! ← duplicate detected!
  We must shrink from the left until 'a' is removed:
    Remove s[L]='a' from set. set={b,c}. L=1.
  Now 'a' is not in set anymore.
  Add 'a'. set={b,c,a}. Window: "bca" (length 3). maxLen=3
  L=1, R=3

right=4: char='b', 'b' IS IN SET!
  Shrink: remove s[1]='b'. set={c,a}. L=2.
  Now 'b' is gone.
  Add 'b'. set={c,a,b}. Window: "cab" (length 3). maxLen=3
  L=2, R=4

right=5: char='c', 'c' IS IN SET!
  Shrink: remove s[2]='c'. set={a,b}. L=3.
  Add 'c'. set={a,b,c}. Window: "abc" (length 3). maxLen=3
  L=3, R=5

right=6: char='b', 'b' IS IN SET!
  Shrink: remove s[3]='a'. set={b,c}. L=4. 'b' still in set!
  Shrink: remove s[4]='b'. set={c}. L=5. Now 'b' is gone.
  Add 'b'. set={c,b}. Window: "cb" (length 2). maxLen=3
  L=5, R=6

right=7: char='b', 'b' IS IN SET!
  Shrink: remove s[5]='c'. set={b}. L=6. 'b' still in set!
  Shrink: remove s[6]='b'. set={}. L=7.
  Add 'b'. set={b}. Window: "b" (length 1). maxLen=3
  L=7, R=7

Final answer: 3
```

```javascript
function lengthOfLongestSubstring(s) {
  const set = new Set(); // characters currently in our window
  let left = 0; // left boundary of window
  let maxLength = 0;

  for (let right = 0; right < s.length; right++) {
    // If the new character is a duplicate, shrink window from left
    while (set.has(s[right])) {
      set.delete(s[left]); // remove leftmost character
      left++; // move left boundary right
    }

    // Add the new character (now guaranteed no duplicate)
    set.add(s[right]);

    // Update the maximum length
    maxLength = Math.max(maxLength, right - left + 1);
  }

  return maxLength;
}
// Time:  O(n) — each character is added to the set once and removed at most once
//               so the while loop runs at most n times TOTAL across all iterations
// Space: O(min(n, 26)) — set holds at most 26 unique lowercase letters
```

## Variable Sliding Window Template

```javascript
function variableSlidingWindow(s) {
  // Track window state (set for uniqueness, map for frequency, etc.)
  const windowState = new Set();   // or {} for frequency
  let left = 0;
  let result = 0;

  for (let right = 0; right < s.length; right++) {
    // STEP 1: Expand — add s[right] to window state

    // STEP 2: Shrink — while window is INVALID, remove s[left] and move left++
    while (/* window condition is violated */) {
      // remove s[left] from window state
      left++;
    }

    // STEP 3: Update — window is now valid, update the result
    result = Math.max(result, right - left + 1);  // for "longest"
    // or
    result = Math.min(result, right - left + 1);  // for "shortest"
  }

  return result;
}
```

---

# Chapter 4: String Manipulation

## Why Strings Come Up Constantly

Strings are interviewers' favorite for full-stack roles because they test language fluency without requiring advanced algorithms. They're easy to understand, easy to draw on a whiteboard, and reveal how well you know your tools.

## The Essential String Methods You Need

```javascript
const s = "Hello, World!";

// LENGTH — how many characters
s.length                         // 13

// ACCESS — get a character
s[0]                             // "H"
s[s.length - 1]                  // "!"
s.at(-1)                         // "!" (negative index, cleaner)

// SEARCH — find things
s.includes("World")              // true (is it anywhere?)
s.indexOf("World")               // 7 (WHERE is it? -1 if not found)
s.startsWith("Hello")            // true
s.endsWith("!")                  // true

// EXTRACT — get a piece
s.slice(7, 12)                   // "World" (start index, end index exclusive)
s.slice(-6)                      // "orld!" (last 6 characters)

// TRANSFORM — change it (returns NEW string, original unchanged)
s.toLowerCase()                  // "hello, world!"
s.toUpperCase()                  // "HELLO, WORLD!"
s.trim()                         // removes whitespace from both ends
s.replace("World", "JS")         // "Hello, JS!" (first occurrence only)
s.replaceAll("l", "L")           // "HeLLo, WorLd!" (all occurrences)

// SPLIT into array / JOIN from array
"a,b,c".split(",")               // ["a", "b", "c"]
"hello".split("")                 // ["h", "e", "l", "l", "o"]
["a", "b", "c"].join("-")        // "a-b-c"

// CONVERT string ↔ char codes
"A".charCodeAt(0)                 // 65
String.fromCharCode(65)           // "A"

// Spread into array of characters
[..."hello"]                      // ["h", "e", "l", "l", "o"]

// REPEAT
"ha".repeat(3)                    // "hahaha"

// PAD
"5".padStart(3, "0")              // "005"
```

## Problem: Valid Anagram (LeetCode #242)

**Problem:** Given two strings, determine if one is an anagram of the other. Anagrams use the exact same characters, just rearranged.

```
"anagram" and "nagaram" → true  (same letters: a×3, n×1, g×1, r×1, m×1)
"rat" and "car"         → false (different letters)
```

### Brute Force: Sort Both Strings

If you sort the characters of both strings, anagrams will produce the same sorted string. "eat" sorted → "aet", "tea" sorted → "aet". Same? Anagram!

```javascript
function isAnagramBrute(s, t) {
  if (s.length !== t.length) return false; // quick check
  return s.split("").sort().join("") === t.split("").sort().join("");
}
// Time:  O(n log n) — sorting dominates
// Space: O(n) — split creates arrays
```

### Optimized: Frequency Counter

**The insight:** Anagrams have identical character frequencies. Count every character in string s (increment), then go through string t (decrement). If all counts end at zero, they're anagrams.

```
s = "anagram"
  a:3, n:1, g:1, r:1, m:1    (count UP)

t = "nagaram"
  a:3→2→1→0, n:1→0, g:1→0, r:1→0, m:1→0    (count DOWN)

All zeros → anagram ✓
```

```javascript
function isAnagram(s, t) {
  if (s.length !== t.length) return false;

  const freq = {};

  // Count characters in s
  for (const c of s) {
    freq[c] = (freq[c] || 0) + 1;
  }

  // Subtract characters in t
  for (const c of t) {
    if (!freq[c]) return false; // character doesn't exist or count exhausted
    freq[c]--;
  }

  return true;
}
// Time:  O(n) — two passes through the strings
// Space: O(1) — at most 26 lowercase letters in the map
```

## Problem: Reverse String (LeetCode #344)

**Problem:** Reverse an array of characters in-place (no new array).

This is the classic two-pointer problem. Swap characters from the outside in.

```
["h", "e", "l", "l", "o"]

Step 1: Swap [0] and [4]:  ["o", "e", "l", "l", "h"]
                             L                   R

Step 2: Swap [1] and [3]:  ["o", "l", "l", "e", "h"]
                                 L           R

Step 3: L=2, R=2, L >= R → stop. Done!
        ["o", "l", "l", "e", "h"] ✓
```

```javascript
function reverseString(s) {
  let left = 0,
    right = s.length - 1;

  while (left < right) {
    // Swap using destructuring
    [s[left], s[right]] = [s[right], s[left]];
    left++;
    right--;
  }
}
// Time:  O(n) — n/2 swaps
// Space: O(1) — in-place, no new array
```

---

# Chapter 5: Array Patterns

## Prefix Sum

**The idea:** Pre-compute a running total so any range sum is instant.

**The problem it solves:** "What's the sum of elements from index 2 to index 5?" Normally you'd loop through indices 2, 3, 4, 5 and add them up (O(n) per query). With a prefix sum array, it's one subtraction (O(1) per query).

```
Original:    [2, 4, 1, 3, 5]
              0  1  2  3  4

Prefix Sum:  [2, 6, 7, 10, 15]
              0  1  2   3   4

prefix[i] = sum of all elements from index 0 to index i

How to build it:
  prefix[0] = 2                    (just the first element)
  prefix[1] = 2 + 4 = 6           (previous prefix + current element)
  prefix[2] = 6 + 1 = 7
  prefix[3] = 7 + 3 = 10
  prefix[4] = 10 + 5 = 15

How to query "sum from index 2 to index 4":
  prefix[4] - prefix[1] = 15 - 6 = 9
  Check: arr[2] + arr[3] + arr[4] = 1 + 3 + 5 = 9 ✓

Formula: sum(left, right) = prefix[right] - prefix[left - 1]
         (if left = 0, just use prefix[right])
```

```javascript
function buildPrefixSum(arr) {
  const prefix = [arr[0]];
  for (let i = 1; i < arr.length; i++) {
    prefix[i] = prefix[i - 1] + arr[i];
  }
  return prefix;
}

function rangeSum(prefix, left, right) {
  if (left === 0) return prefix[right];
  return prefix[right] - prefix[left - 1];
}
// Build: O(n) one-time cost
// Each query: O(1)
```

## Kadane's Algorithm — Maximum Subarray (LeetCode #53)

**Problem:** Find the contiguous subarray with the largest sum.

```
Input:  [-2, 1, -3, 4, -1, 2, 1, -5, 4]
Output: 6   (the subarray [4, -1, 2, 1] has the largest sum)
```

### Brute Force

Check every possible subarray (every start × every end), calculate its sum. O(n²) with prefix sums, O(n³) without.

### Optimized: Kadane's Algorithm

**The insight is simple but powerful:** As you walk through the array, you're building a running sum. At each position, you ask yourself ONE question:

**"Is it better to EXTEND the previous subarray by including this number, or START FRESH from this number?"**

If the running sum has gone negative, it can only drag down whatever comes next. So you throw it away and start over from the current number.

```
Array: [-2, 1, -3, 4, -1, 2, 1, -5, 4]

i=0: num = -2
  Extend previous (0) + (-2) = -2
  Start fresh = -2
  max(-2, -2) = -2.  currentMax = -2.  globalMax = -2.

i=1: num = 1
  Extend previous (-2) + 1 = -1
  Start fresh = 1
  max(-1, 1) = 1.  currentMax = 1.  globalMax = 1.

  Think: "Starting fresh at 1 is better than dragging -2 along."

i=2: num = -3
  Extend previous (1) + (-3) = -2
  Start fresh = -3
  max(-2, -3) = -2.  currentMax = -2.  globalMax = 1.

  Think: "Extending is slightly less bad than starting fresh."

i=3: num = 4
  Extend previous (-2) + 4 = 2
  Start fresh = 4
  max(2, 4) = 4.  currentMax = 4.  globalMax = 4.

  Think: "Previous sum was negative — starting fresh at 4 is better."

i=4: num = -1
  Extend: 4 + (-1) = 3
  Fresh: -1
  max(3, -1) = 3.  currentMax = 3.  globalMax = 4.

i=5: num = 2
  Extend: 3 + 2 = 5
  Fresh: 2
  max(5, 2) = 5.  currentMax = 5.  globalMax = 5.

i=6: num = 1
  Extend: 5 + 1 = 6
  Fresh: 1
  max(6, 1) = 6.  currentMax = 6.  globalMax = 6. ← new max!

i=7: num = -5
  Extend: 6 + (-5) = 1
  Fresh: -5
  max(1, -5) = 1.  currentMax = 1.  globalMax = 6.

i=8: num = 4
  Extend: 1 + 4 = 5
  Fresh: 4
  max(5, 4) = 5.  currentMax = 5.  globalMax = 6.

Answer: 6  (subarray [4, -1, 2, 1] at indices 3-6)
```

```javascript
function maxSubArray(nums) {
  let currentMax = nums[0]; // best sum ending at current position
  let globalMax = nums[0]; // best sum seen anywhere

  for (let i = 1; i < nums.length; i++) {
    // Extend previous subarray or start new one — pick the better option
    currentMax = Math.max(nums[i], currentMax + nums[i]);

    // Update global best if current is better
    globalMax = Math.max(globalMax, currentMax);
  }

  return globalMax;
}
// Time:  O(n) — single pass
// Space: O(1) — two variables
```

## Merge Intervals (LeetCode #56)

**Problem:** Given a list of intervals, merge all overlapping ones.

```
Input:  [[1,3], [2,6], [8,10], [15,18]]
Output: [[1,6], [8,10], [15,18]]
```

**Visualized on a number line:**

```
Input:
  1---3
    2------6
                8--10
                         15--18

After merging:
  1--------6    8--10    15--18

[1,3] and [2,6] overlap (2 ≤ 3) → merge into [1,6]
[8,10] doesn't overlap with [1,6] (8 > 6) → keep separate
[15,18] doesn't overlap with [8,10] (15 > 10) → keep separate
```

**The approach:**

1. Sort intervals by start time (so overlapping ones are adjacent).
2. Walk through. Compare each interval with the last merged one.
   - If they overlap (current start ≤ last merged end), extend the merged interval.
   - If they don't overlap, add a new interval to the result.

**How to detect overlap:**

```
Interval A: [1, 6]    (ends at 6)
Interval B: [2, 8]    (starts at 2)

B starts at 2, which is ≤ A's end (6) → they overlap!
Merged: [1, max(6, 8)] = [1, 8]

Interval A: [1, 6]
Interval C: [8, 10]

C starts at 8, which is > A's end (6) → no overlap.
```

```javascript
function merge(intervals) {
  if (intervals.length <= 1) return intervals;

  // Step 1: Sort by start time
  intervals.sort((a, b) => a[0] - b[0]);

  const result = [intervals[0]]; // start with the first interval

  for (let i = 1; i < intervals.length; i++) {
    const current = intervals[i];
    const lastMerged = result[result.length - 1];

    if (current[0] <= lastMerged[1]) {
      // Overlapping: extend the end of the last merged interval
      lastMerged[1] = Math.max(lastMerged[1], current[1]);
    } else {
      // Not overlapping: add as a new interval
      result.push(current);
    }
  }

  return result;
}
// Time:  O(n log n) — sorting dominates the O(n) merge pass
// Space: O(n) — result array
```

## Contains Duplicate (LeetCode #217)

**Problem:** Return true if any value appears more than once.

```javascript
// Brute Force: O(n²) — compare every pair
function hasDuplicateBrute(nums) {
  for (let i = 0; i < nums.length; i++) {
    for (let j = i + 1; j < nums.length; j++) {
      if (nums[i] === nums[j]) return true;
    }
  }
  return false;
}

// Optimized: O(n) — use a Set to track what you've seen
function containsDuplicate(nums) {
  const seen = new Set();

  for (const num of nums) {
    if (seen.has(num)) return true; // seen this before? duplicate!
    seen.add(num); // first time seeing it — remember it
  }

  return false;
}
// Time:  O(n) — single pass, Set.has() is O(1)
// Space: O(n) — Set stores up to n elements
```

## Move Zeroes (LeetCode #283)

**Problem:** Move all zeroes to the end of the array while keeping the relative order of non-zero elements. Do it in-place.

```
Input:  [0, 1, 0, 3, 12]
Output: [1, 3, 12, 0, 0]
```

**The approach:** Use a pointer (`insertPos`) that tracks where the next non-zero element should go. Walk through the array. Every time you find a non-zero, place it at `insertPos` and increment. After the loop, fill the remaining positions with zeroes.

```
[0, 1, 0, 3, 12]   insertPos = 0

i=0: nums[0]=0 → skip (it's zero)
i=1: nums[1]=1 → place at insertPos=0 → [1, 1, 0, 3, 12], insertPos=1
i=2: nums[2]=0 → skip
i=3: nums[3]=3 → place at insertPos=1 → [1, 3, 0, 3, 12], insertPos=2
i=4: nums[4]=12 → place at insertPos=2 → [1, 3, 12, 3, 12], insertPos=3

Fill remaining with 0:
  insertPos=3 → [1, 3, 12, 0, 12]
  insertPos=4 → [1, 3, 12, 0, 0] ✓
```

```javascript
function moveZeroes(nums) {
  let insertPos = 0;

  // Move all non-zero elements to the front
  for (let i = 0; i < nums.length; i++) {
    if (nums[i] !== 0) {
      nums[insertPos] = nums[i];
      insertPos++;
    }
  }

  // Fill the rest with zeroes
  while (insertPos < nums.length) {
    nums[insertPos] = 0;
    insertPos++;
  }
}
// Time:  O(n) — single pass + fill
// Space: O(1) — in-place
```

---

# Big-O Reference Card

```
Notation     Name           What It Looks Like In Code              Speed
────────     ─────────      ───────────────────────────              ─────
O(1)         Constant       map[key], arr[i], push/pop              Instant
O(log n)     Logarithmic    Binary search, halving                  Fast
O(n)         Linear         Single loop, two pointers               Good
O(n log n)   Linearithmic   Sorting (built-in .sort())              Acceptable
O(n²)        Quadratic      Nested loops                            Slow
O(2ⁿ)        Exponential    Recursive without memoization           Unusable

Interview rule of thumb (~1 second time limit):
  n ≤ 10        → O(n!) fine
  n ≤ 1,000     → O(n²) fine
  n ≤ 100,000   → O(n log n) needed
  n ≤ 1,000,000 → O(n) needed
```
