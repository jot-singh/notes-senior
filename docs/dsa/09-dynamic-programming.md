# Dynamic Programming Fundamentals

## What You'll Learn
- Core DP concepts: overlapping subproblems and optimal substructure
- Memoization vs tabulation approaches
- Problem-solving methodology: recognizing DP problems
- State definition and transition strategies
- Real-world optimization problems solved with DP

## Why This Matters

Dynamic Programming is a **problem-solving paradigm** that transforms seemingly complex problems into elegant solutions by breaking them into overlapping subproblems. It's not about clever tricks - it's about **systematic thinking**. Understanding DP teaches you to think recursively, recognize patterns, and optimize systematically.

---

## What Makes a Problem Solvable by DP?

### Two Essential Properties

**1. Optimal Substructure**: Solution to problem contains optimal solutions to subproblems.

```
Optimal: Shortest path problem
- Shortest path from A to D passes through B
- A→B→D must be: (shortest A→B) + (shortest B→D)

Not optimal: Longest path problem (without cycle restrictions)
- Longest path from A to D might need to go through B, C, E
- Can't just combine longest A→B + longest B→D
```

**2. Overlapping Subproblems**: Subproblems are solved multiple times.

```
Fibonacci:
fib(5) = fib(4) + fib(3)
fib(4) = fib(3) + fib(2)
fib(3) appears twice! This is overlapping.

fib(5)
├── fib(4)
│   ├── fib(3) ←── computed again!
│   │   ├── fib(2)
│   │   └── fib(1)
│   └── fib(2)
└── fib(3) ←── duplicate work
```

---

## Core DP Concepts

### Pattern 1: Memoization (Top-Down)

**When to use**: Natural recursive problem, want to code recursively.

**The Idea**: Solve recursively, cache results to avoid recomputation.

```java
public class Fibonacci {
    // Without memoization: O(2^n) exponential
    public int fibExponential(int n) {
        if (n <= 1) return n;
        return fibExponential(n - 1) + fibExponential(n - 2);
    }
    
    // With memoization: O(n) linear
    private Map<Integer, Integer> memo = new HashMap<>();
    
    public int fibMemo(int n) {
        if (n <= 1) return n;
        
        if (memo.containsKey(n)) {
            return memo.get(n);  // Return cached result
        }
        
        int result = fibMemo(n - 1) + fibMemo(n - 2);
        memo.put(n, result);  // Cache before returning
        return result;
    }
}

// Analysis:
// Exponential: fib(5) = 15 calls to fibExponential
// Memoized: fib(5) = 5 calls to fibMemo (5, 4, 3, 2, 1)
```

**Why memoization works**:
```
fib(5)
├── fib(4)
│   ├── fib(3) [compute and cache]
│   │   ├── fib(2) [compute and cache]
│   │   └── fib(1)
│   └── fib(2) [return from cache!]
└── fib(3) [return from cache!]
```

```java
// Example: Coin change problem with memoization
public class CoinChange {
    public int minCoins(int[] coins, int amount, Map<Integer, Integer> memo) {
        if (amount == 0) return 0;
        if (amount < 0) return -1;
        
        if (memo.containsKey(amount)) {
            return memo.get(amount);
        }
        
        int min = Integer.MAX_VALUE;
        for (int coin : coins) {
            int result = minCoins(coins, amount - coin, memo);
            if (result >= 0) {
                min = Math.min(min, 1 + result);
            }
        }
        
        int result = min == Integer.MAX_VALUE ? -1 : min;
        memo.put(amount, result);
        return result;
    }
    
    // Time: O(amount × coins.length)
    // Space: O(amount) for memo + O(amount) recursion stack
}
```

### Pattern 2: Tabulation (Bottom-Up)

**When to use**: Want to avoid recursion, optimize space, or prefer iterative solutions.

**The Idea**: Build table from base cases up to final answer.

```java
public class FibonacciTabulation {
    public int fibTab(int n) {
        if (n <= 1) return n;
        
        int[] dp = new int[n + 1];
        dp[0] = 0;
        dp[1] = 1;
        
        // Build from bottom-up
        for (int i = 2; i <= n; i++) {
            dp[i] = dp[i - 1] + dp[i - 2];
        }
        
        return dp[n];
    }
    
    // Time: O(n)
    // Space: O(n) for table
    
    // Space-optimized: Only need last two values
    public int fibOptimized(int n) {
        if (n <= 1) return n;
        
        int prev1 = 0, prev2 = 1;
        for (int i = 2; i <= n; i++) {
            int current = prev1 + prev2;
            prev1 = prev2;
            prev2 = current;
        }
        
        return prev2;
    }
    
    // Time: O(n)
    // Space: O(1)
}
```

**When to space-optimize**:
```
Full DP table:         Space-optimized:
dp[0] = 0              prev1 = 0
dp[1] = 1              prev2 = 1
dp[2] = dp[0] + dp[1]  current = prev1 + prev2
dp[3] = dp[1] + dp[2]  → update prev1, prev2
...

Only track values you need for next computation!
```

---

## DP Problem-Solving Methodology

### Step 1: Define State

State represents a **subproblem**. What information do you need to uniquely identify a subproblem?

```java
// Example: Climbing Stairs
// At step n, how many ways to reach it?
// State: dp[i] = number of ways to reach step i

// Example: House Robber
// After deciding about house i, what matters?
// State: dp[i] = max money robbing houses 0..i

// Example: 2D Grid
// Reaching cell (i, j), what path did you take?
// State: dp[i][j] = max value reaching cell (i, j)
```

### Step 2: Find Transitions

How do you compute current state from previous states?

```java
// Climbing Stairs: dp[i] = dp[i-1] + dp[i-2]
// Reason: Reach step i by either from step i-1 or i-2

// House Robber: dp[i] = max(dp[i-1], houses[i] + dp[i-2])
// Reason: Either rob house i (skip i-1) or don't rob house i

// 2D Grid: dp[i][j] = max(dp[i-1][j], dp[i][j-1]) + grid[i][j]
// Reason: Come from top or left, add current cell value
```

### Step 3: Base Cases

Handle edge cases where transitions don't apply.

```java
// Single element: dp[0] = base answer
// Empty: dp[-1] = 0 or -infinity depending on problem
// First row/column: depends on state definition

public int solve(int n) {
    int[] dp = new int[n + 1];
    
    // Base cases
    dp[0] = 0;      // Answer for 0 elements
    dp[1] = 1;      // Answer for 1 element
    
    // Transition
    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i-1] + dp[i-2];
    }
    
    return dp[n];
}
```

---

## Core DP Patterns

### Pattern 1: 1D Array DP (Linear Sequence Problems)

```java
// Problem: House Robber
// You can't rob adjacent houses. Maximize money.
// houses = [1, 2, 3, 1]

public int rob(int[] houses) {
    if (houses.length == 0) return 0;
    if (houses.length == 1) return houses[0];
    
    int[] dp = new int[houses.length];
    dp[0] = houses[0];
    dp[1] = Math.max(houses[0], houses[1]);
    
    for (int i = 2; i < houses.length; i++) {
        // Either rob house i or skip it
        // If rob: get houses[i] + money from dp[i-2]
        // If skip: get money from dp[i-1]
        dp[i] = Math.max(dp[i - 1], houses[i] + dp[i - 2]);
    }
    
    return dp[houses.length - 1];
}

// Example walkthrough:
// houses = [1, 2, 3, 1]
// dp[0] = 1           (rob house 0)
// dp[1] = 2           (rob house 1, skip 0)
// dp[2] = max(2, 3+1) = 4  (rob houses 0,2)
// dp[3] = max(4, 1+2) = 4  (best is houses 0,2)
// Answer: 4
```

### Pattern 2: 2D Array DP (Grid/2D Problems)

```java
// Problem: Maximum path sum in grid (move only right or down)
// grid = [[1,3,1],[2,2,1]]

public int maxPathSum(int[][] grid) {
    int m = grid.length;
    int n = grid[0].length;
    
    int[][] dp = new int[m][n];
    dp[0][0] = grid[0][0];
    
    // Initialize first column (can only come from above)
    for (int i = 1; i < m; i++) {
        dp[i][0] = dp[i - 1][0] + grid[i][0];
    }
    
    // Initialize first row (can only come from left)
    for (int j = 1; j < n; j++) {
        dp[0][j] = dp[0][j - 1] + grid[0][j];
    }
    
    // Fill rest of table
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            // Come from top or left (whichever is better)
            dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]) + grid[i][j];
        }
    }
    
    return dp[m - 1][n - 1];
}

// Example:
// grid:
// [1, 3, 1]
// [2, 2, 1]
//
// dp (building up):
// [1,  4,  5]
// [3,  6,  6]
```

### Pattern 3: String/Subsequence DP

```java
// Problem: Longest common subsequence (LCS)
// s1 = "abcde", s2 = "ace"
// LCS = "ace" (length 3)

public int longestCommonSubsequence(String s1, String s2) {
    int m = s1.length();
    int n = s2.length();
    
    int[][] dp = new int[m + 1][n + 1];
    // dp[i][j] = LCS of s1[0..i-1] and s2[0..j-1]
    
    // Base case: dp[0][j] = 0, dp[i][0] = 0 (initialized by default)
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                // Characters match: extend LCS from dp[i-1][j-1]
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                // Characters don't match: take max of skipping one
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    
    return dp[m][n];
}

// Table for s1="abcde", s2="ace":
//     ""  a  c  e
// ""   0  0  0  0
// a    0  1  1  1
// b    0  1  1  1
// c    0  1  2  2
// d    0  1  2  2
// e    0  1  2  3
//
// Answer: 3 (ace)
```

### Pattern 4: Knapsack DP (Multiple Dimensions)

```java
// Problem: 0/1 Knapsack - maximize value with weight limit
// items: value = [1,6,10,16], weight = [1,2,3,5]
// capacity = 7

public int knapsack(int[] values, int[] weights, int capacity) {
    int n = values.length;
    int[][] dp = new int[n + 1][capacity + 1];
    // dp[i][w] = max value using items 0..i-1 with weight limit w
    
    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= capacity; w++) {
            // Option 1: Don't take item i-1
            int skip = dp[i - 1][w];
            
            // Option 2: Take item i-1 (if it fits)
            int take = Integer.MIN_VALUE;
            if (weights[i - 1] <= w) {
                take = values[i - 1] + dp[i - 1][w - weights[i - 1]];
            }
            
            dp[i][w] = Math.max(skip, take);
        }
    }
    
    return dp[n][capacity];
}

// Space-optimized (1D array):
public int knapsackOptimized(int[] values, int[] weights, int capacity) {
    int[] dp = new int[capacity + 1];
    
    for (int i = 0; i < values.length; i++) {
        // IMPORTANT: Iterate backward to avoid using same item twice
        for (int w = capacity; w >= weights[i]; w--) {
            dp[w] = Math.max(dp[w], values[i] + dp[w - weights[i]]);
        }
    }
    
    return dp[capacity];
}

// Backward iteration ensures each item used at most once
```

---

## Real-World Applications

### 1. **Edit Distance (String Similarity)**

```java
// Minimum edits to transform s1 to s2
// Used in: spell checkers, diff algorithms, DNA sequence matching

public int editDistance(String s1, String s2) {
    int m = s1.length();
    int n = s2.length();
    
    int[][] dp = new int[m + 1][n + 1];
    
    // Base: transform empty string
    for (int i = 0; i <= m; i++) dp[i][0] = i;  // Delete i chars
    for (int j = 0; j <= n; j++) dp[0][j] = j;  // Insert j chars
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1];  // Match, no edit needed
            } else {
                dp[i][j] = 1 + Math.min({
                    dp[i - 1][j],      // Delete from s1
                    dp[i][j - 1],      // Insert into s1
                    dp[i - 1][j - 1]   // Replace
                });
            }
        }
    }
    
    return dp[m][n];
}

// Used in: Text editors, version control (git diff), spell checkers
```

### 2. **Coin Change (Unlimited Supply)**

```java
// Minimum coins to make amount
// Used in: Currency exchange, optimization problems

public int minCoins(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, Integer.MAX_VALUE);
    dp[0] = 0;
    
    for (int a = 1; a <= amount; a++) {
        for (int coin : coins) {
            if (coin <= a && dp[a - coin] != Integer.MAX_VALUE) {
                dp[a] = Math.min(dp[a], 1 + dp[a - coin]);
            }
        }
    }
    
    return dp[amount] == Integer.MAX_VALUE ? -1 : dp[amount];
}

// Table for coins=[1,2,5], amount=5:
// dp[0] = 0
// dp[1] = 1 (one 1-coin)
// dp[2] = 1 (one 2-coin)
// dp[3] = 2 (one 2-coin + one 1-coin)
// dp[4] = 2 (two 2-coins)
// dp[5] = 1 (one 5-coin)
```

### 3. **Stock Trading Problems**

```java
// Maximum profit with at most 2 transactions
// Used in: Financial modeling, trading algorithms

public int maxProfit(int[] prices) {
    if (prices.length < 2) return 0;
    
    int n = prices.length;
    // buy1, sell1, buy2, sell2 = states
    int buy1 = -prices[0], sell1 = 0;
    int buy2 = -prices[0], sell2 = 0;
    
    for (int i = 1; i < n; i++) {
        buy1 = Math.max(buy1, -prices[i]);
        sell1 = Math.max(sell1, buy1 + prices[i]);
        buy2 = Math.max(buy2, sell1 - prices[i]);
        sell2 = Math.max(sell2, buy2 + prices[i]);
    }
    
    return sell2;
}

// State:
// buy1: Max profit after first buy
// sell1: Max profit after first sell
// buy2: Max profit after second buy (using sell1 profit)
// sell2: Max profit after second sell
```

---

## DP Problem Recognition

When you see a problem, ask:

```
1. Can I break it into subproblems?
2. Do subproblems repeat?
3. Can I build solution from subproblem solutions?

If YES to all 3 → Consider DP

Question patterns that suggest DP:
- "Maximum/minimum..."
- "How many ways..."
- "Count distinct..."
- "Longest/shortest..."
- "Best choice..."
- "Optimal..."
```

---

## Common Mistakes (Anti-Patterns)

### ❌ Forgetting Base Cases
```java
// ❌ Stack overflow
int fib(int n) {
    return fib(n-1) + fib(n-2);  // No base case!
}

// ✅ Always define base cases
int fib(int n) {
    if (n <= 1) return n;  // Base cases
    return fib(n-1) + fib(n-2);
}
```

### ❌ Not Recognizing Overlapping Subproblems
```java
// ❌ Still exponential (memoization not helping)
Map<String, Integer> memo = new HashMap<>();

int solve(String s) {
    if (memo.containsKey(s)) return memo.get(s);
    // Actually creates new subproblems, doesn't reuse
    String s1 = s.substring(0, 1);
    String s2 = s.substring(1);
    // ...
}

// ✅ Problem must have overlapping subproblems
// Good DP: fib(n) reuses fib(n-1), fib(n-2)
// Bad: each string is unique, no overlap
```

### ❌ Wrong State Definition
```java
// ❌ State doesn't uniquely identify subproblem
// What does dp[i] mean if you don't know what's happened before?

// ✅ Clear state definition
// dp[i] = max money robbing houses 0..i
// dp[i][j] = min cost reaching cell (i,j)
```

### ❌ Backward Iteration Issue (1D Array Knapsack)
```java
// ❌ Using item twice (wrong!)
for (int w = 0; w <= capacity; w++) {
    dp[w] = Math.max(dp[w], values[i] + dp[w - weights[i]]);
}

// ✅ Backward iteration ensures each item once
for (int w = capacity; w >= weights[i]; w--) {
    dp[w] = Math.max(dp[w], values[i] + dp[w - weights[i]]);
}
```

---

## Complexity Analysis

| Approach | Time | Space | Best For |
|----------|------|-------|----------|
| Recursion | O(2^n) | O(n) | None - too slow |
| Memoization | O(states × transitions) | O(states) | Natural recursion |
| Tabulation | O(states × transitions) | O(states) | Complex dependencies |
| Space-opt Tab | O(states × transitions) | O(reduced) | Memory constraints |

---

## Next Steps

- Practice 1D DP (house robber, climbing stairs)
- Master 2D DP (grid problems, LCS, edit distance)
- Learn unbounded knapsack (coin change variant)
- Study state compression for advanced optimization
- Explore [Algorithm Patterns](12-algorithm-patterns.md) for other paradigms
