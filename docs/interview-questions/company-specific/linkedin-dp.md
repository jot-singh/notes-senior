# LinkedIn Interview Questions: Dynamic Programming

High-frequency DP problems that separate senior candidates from juniors. These test your ability to recognize patterns, define states, and optimize systematically.

---

## Must-Practice Problems

### 1. House Robber (Medium) ⭐⭐⭐⭐⭐
**Frequency**: Very High | **Pattern**: 1D DP

```java
/*
Problem: Maximum money you can rob without alerting police (can't rob adjacent).

Example:
Input: nums = [2,7,9,3,1]
Output: 12 (rob houses 0, 2, 4: 2+9+1=12)

LinkedIn Context: Resource allocation with constraints
*/

public int rob(int[] nums) {
    if (nums.length == 0) return 0;
    if (nums.length == 1) return nums[0];
    
    int prev2 = 0;
    int prev1 = nums[0];
    
    for (int i = 1; i < nums.length; i++) {
        int current = Math.max(prev1, nums[i] + prev2);
        prev2 = prev1;
        prev1 = current;
    }
    
    return prev1;
}

// Follow-up: Houses in circle (first and last are adjacent)
public int robCircular(int[] nums) {
    if (nums.length == 1) return nums[0];
    
    // Either rob first (can't rob last) or rob last (can't rob first)
    return Math.max(
        robRange(nums, 0, nums.length - 2),
        robRange(nums, 1, nums.length - 1)
    );
}

private int robRange(int[] nums, int start, int end) {
    int prev2 = 0, prev1 = 0;
    
    for (int i = start; i <= end; i++) {
        int current = Math.max(prev1, nums[i] + prev2);
        prev2 = prev1;
        prev1 = current;
    }
    
    return prev1;
}

// Time: O(n)
// Space: O(1)
```

---

### 2. Longest Increasing Subsequence (Medium) ⭐⭐⭐⭐⭐
**Frequency**: Very High | **Pattern**: 1D DP + Binary Search

```java
/*
Problem: Find length of longest strictly increasing subsequence.

Example:
Input: nums = [10,9,2,5,3,7,101,18]
Output: 4 (subsequence [2,3,7,101])

LinkedIn Context: Skill progression, career growth tracking
*/

// DP approach: O(n²)
public int lengthOfLIS(int[] nums) {
    int[] dp = new int[nums.length];
    Arrays.fill(dp, 1);  // Each element is subsequence of length 1
    
    int maxLen = 1;
    
    for (int i = 1; i < nums.length; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
        maxLen = Math.max(maxLen, dp[i]);
    }
    
    return maxLen;
}

// Optimized with Binary Search: O(n log n)
public int lengthOfLISOptimized(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    
    for (int num : nums) {
        int pos = binarySearch(tails, num);
        
        if (pos == tails.size()) {
            tails.add(num);
        } else {
            tails.set(pos, num);
        }
    }
    
    return tails.size();
}

private int binarySearch(List<Integer> tails, int target) {
    int left = 0, right = tails.size();
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        if (tails.get(mid) < target) {
            left = mid + 1;
        } else {
            right = mid;
        }
    }
    
    return left;
}
```

---

### 3. Coin Change (Medium) ⭐⭐⭐⭐⭐
**Frequency**: Very High | **Pattern**: Unbounded Knapsack

```java
/*
Problem: Minimum coins needed to make amount (unlimited coin supply).

Example:
Input: coins = [1,2,5], amount = 11
Output: 3 (5+5+1=11)

LinkedIn Context: Resource optimization, pricing strategies
*/

public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);  // Initialize with impossible value
    dp[0] = 0;
    
    for (int a = 1; a <= amount; a++) {
        for (int coin : coins) {
            if (coin <= a) {
                dp[a] = Math.min(dp[a], dp[a - coin] + 1);
            }
        }
    }
    
    return dp[amount] > amount ? -1 : dp[amount];
}

// Follow-up: Count number of ways to make amount
public int coinChangeWays(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    dp[0] = 1;
    
    // Important: Iterate coins first to avoid counting permutations
    for (int coin : coins) {
        for (int a = coin; a <= amount; a++) {
            dp[a] += dp[a - coin];
        }
    }
    
    return dp[amount];
}

// Time: O(amount × coins.length)
// Space: O(amount)
```

---

### 4. Longest Common Subsequence (Medium) ⭐⭐⭐⭐⭐
**Frequency**: High | **Pattern**: 2D DP

```java
/*
Problem: Find length of longest subsequence common to both strings.

Example:
Input: text1 = "abcde", text2 = "ace"
Output: 3 (subsequence "ace")

LinkedIn Context: Text similarity, diff algorithms
*/

public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length();
    int n = text2.length();
    
    int[][] dp = new int[m + 1][n + 1];
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i - 1) == text2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    
    return dp[m][n];
}

// Space-optimized: O(n)
public int longestCommonSubsequenceOptimized(String text1, String text2) {
    int m = text1.length();
    int n = text2.length();
    
    int[] prev = new int[n + 1];
    int[] curr = new int[n + 1];
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i - 1) == text2.charAt(j - 1)) {
                curr[j] = prev[j - 1] + 1;
            } else {
                curr[j] = Math.max(prev[j], curr[j - 1]);
            }
        }
        int[] temp = prev;
        prev = curr;
        curr = temp;
    }
    
    return prev[n];
}

// Time: O(m × n)
// Space: O(n) optimized
```

---

### 5. Word Break (Medium) ⭐⭐⭐⭐⭐
**Frequency**: Very High | **Pattern**: 1D DP

```java
/*
Problem: Determine if string can be segmented into dictionary words.

Example:
Input: s = "leetcode", wordDict = ["leet","code"]
Output: true

LinkedIn Context: Message parsing, autocomplete validation
*/

public boolean wordBreak(String s, List<String> wordDict) {
    Set<String> wordSet = new HashSet<>(wordDict);
    boolean[] dp = new boolean[s.length() + 1];
    dp[0] = true;  // Empty string
    
    for (int i = 1; i <= s.length(); i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] && wordSet.contains(s.substring(j, i))) {
                dp[i] = true;
                break;
            }
        }
    }
    
    return dp[s.length()];
}

// Follow-up: Return all possible segmentations
public List<String> wordBreakII(String s, List<String> wordDict) {
    Set<String> wordSet = new HashSet<>(wordDict);
    Map<Integer, List<String>> memo = new HashMap<>();
    return backtrack(s, 0, wordSet, memo);
}

private List<String> backtrack(String s, int start, Set<String> wordSet,
                               Map<Integer, List<String>> memo) {
    if (memo.containsKey(start)) {
        return memo.get(start);
    }
    
    List<String> result = new ArrayList<>();
    
    if (start == s.length()) {
        result.add("");
        return result;
    }
    
    for (int end = start + 1; end <= s.length(); end++) {
        String word = s.substring(start, end);
        
        if (wordSet.contains(word)) {
            List<String> sublist = backtrack(s, end, wordSet, memo);
            
            for (String sub : sublist) {
                result.add(word + (sub.isEmpty() ? "" : " " + sub));
            }
        }
    }
    
    memo.put(start, result);
    return result;
}

// Time: O(n² + 2^n) for Word Break II
// Space: O(n)
```

---

### 6. Maximum Product Subarray (Medium) ⭐⭐⭐⭐
**Frequency**: High | **Pattern**: 1D DP with Min/Max Tracking

```java
/*
Problem: Find contiguous subarray with largest product.

Example:
Input: nums = [2,3,-2,4]
Output: 6 (subarray [2,3])

LinkedIn Context: Performance metrics, compound growth
*/

public int maxProduct(int[] nums) {
    int maxProduct = nums[0];
    int maxSoFar = nums[0];
    int minSoFar = nums[0];  // Track min for negative numbers
    
    for (int i = 1; i < nums.length; i++) {
        // Negative number flips max and min
        if (nums[i] < 0) {
            int temp = maxSoFar;
            maxSoFar = minSoFar;
            minSoFar = temp;
        }
        
        maxSoFar = Math.max(nums[i], maxSoFar * nums[i]);
        minSoFar = Math.min(nums[i], minSoFar * nums[i]);
        
        maxProduct = Math.max(maxProduct, maxSoFar);
    }
    
    return maxProduct;
}

// Time: O(n)
// Space: O(1)
```

---

### 7. Unique Paths (Medium) ⭐⭐⭐⭐
**Frequency**: High | **Pattern**: 2D Grid DP

```java
/*
Problem: Count unique paths from top-left to bottom-right (move only right/down).

Example:
Input: m = 3, n = 7
Output: 28

LinkedIn Context: Route optimization, path counting
*/

public int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];
    
    // Initialize first row and column
    for (int i = 0; i < m; i++) dp[i][0] = 1;
    for (int j = 0; j < n; j++) dp[0][j] = 1;
    
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        }
    }
    
    return dp[m - 1][n - 1];
}

// Space-optimized: O(n)
public int uniquePathsOptimized(int m, int n) {
    int[] dp = new int[n];
    Arrays.fill(dp, 1);
    
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[j] += dp[j - 1];
        }
    }
    
    return dp[n - 1];
}

// Follow-up: With obstacles
public int uniquePathsWithObstacles(int[][] grid) {
    int m = grid.length;
    int n = grid[0].length;
    
    if (grid[0][0] == 1) return 0;
    
    int[][] dp = new int[m][n];
    dp[0][0] = 1;
    
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (grid[i][j] == 1) {
                dp[i][j] = 0;
            } else {
                if (i > 0) dp[i][j] += dp[i - 1][j];
                if (j > 0) dp[i][j] += dp[i][j - 1];
            }
        }
    }
    
    return dp[m - 1][n - 1];
}

// Time: O(m × n)
// Space: O(n) optimized
```

---

### 8. Best Time to Buy and Sell Stock (Multiple Transactions) (Hard) ⭐⭐⭐⭐⭐
**Frequency**: High | **Pattern**: State Machine DP

```java
/*
Problem: Maximum profit with at most k transactions.

LinkedIn Context: Trading strategies, investment optimization
*/

// At most 2 transactions
public int maxProfitTwo(int[] prices) {
    int buy1 = -prices[0], sell1 = 0;
    int buy2 = -prices[0], sell2 = 0;
    
    for (int i = 1; i < prices.length; i++) {
        buy1 = Math.max(buy1, -prices[i]);
        sell1 = Math.max(sell1, buy1 + prices[i]);
        buy2 = Math.max(buy2, sell1 - prices[i]);
        sell2 = Math.max(sell2, buy2 + prices[i]);
    }
    
    return sell2;
}

// At most k transactions
public int maxProfit(int k, int[] prices) {
    if (prices.length == 0) return 0;
    
    // If k >= n/2, unlimited transactions
    if (k >= prices.length / 2) {
        return maxProfitUnlimited(prices);
    }
    
    int[] buy = new int[k + 1];
    int[] sell = new int[k + 1];
    Arrays.fill(buy, -prices[0]);
    
    for (int i = 1; i < prices.length; i++) {
        for (int j = k; j >= 1; j--) {
            sell[j] = Math.max(sell[j], buy[j] + prices[i]);
            buy[j] = Math.max(buy[j], sell[j - 1] - prices[i]);
        }
    }
    
    return sell[k];
}

private int maxProfitUnlimited(int[] prices) {
    int profit = 0;
    for (int i = 1; i < prices.length; i++) {
        profit += Math.max(0, prices[i] - prices[i - 1]);
    }
    return profit;
}

// Time: O(n × k)
// Space: O(k)
```

---

### 9. Edit Distance (Hard) ⭐⭐⭐⭐⭐
**Frequency**: High | **Pattern**: 2D String DP

```java
/*
Problem: Minimum edits to convert word1 to word2.
Operations: insert, delete, replace

Example:
Input: word1 = "horse", word2 = "ros"
Output: 3 (horse -> rorse -> rose -> ros)

LinkedIn Context: Text similarity, spell checking
*/

public int minDistance(String word1, String word2) {
    int m = word1.length();
    int n = word2.length();
    
    int[][] dp = new int[m + 1][n + 1];
    
    // Base cases
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i - 1) == word2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = 1 + Math.min(
                    dp[i - 1][j],      // Delete
                    Math.min(
                        dp[i][j - 1],  // Insert
                        dp[i - 1][j - 1]  // Replace
                    )
                );
            }
        }
    }
    
    return dp[m][n];
}

// Space-optimized: O(n)
public int minDistanceOptimized(String word1, String word2) {
    int m = word1.length();
    int n = word2.length();
    
    int[] prev = new int[n + 1];
    int[] curr = new int[n + 1];
    
    for (int j = 0; j <= n; j++) prev[j] = j;
    
    for (int i = 1; i <= m; i++) {
        curr[0] = i;
        
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i - 1) == word2.charAt(j - 1)) {
                curr[j] = prev[j - 1];
            } else {
                curr[j] = 1 + Math.min(prev[j], Math.min(curr[j - 1], prev[j - 1]));
            }
        }
        
        int[] temp = prev;
        prev = curr;
        curr = temp;
    }
    
    return prev[n];
}

// Time: O(m × n)
// Space: O(n) optimized
```

---

### 10. Regular Expression Matching (Hard) ⭐⭐⭐⭐
**Frequency**: Medium | **Pattern**: 2D DP

```java
/*
Problem: Implement regex matching with '.' and '*'.
'.' matches any single character
'*' matches zero or more of the preceding element

Example:
Input: s = "aa", p = "a*"
Output: true

LinkedIn Context: Search queries, pattern matching
*/

public boolean isMatch(String s, String p) {
    int m = s.length();
    int n = p.length();
    
    boolean[][] dp = new boolean[m + 1][n + 1];
    dp[0][0] = true;
    
    // Handle patterns like a*, a*b*, a*b*c*
    for (int j = 2; j <= n; j++) {
        if (p.charAt(j - 1) == '*') {
            dp[0][j] = dp[0][j - 2];
        }
    }
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            char sc = s.charAt(i - 1);
            char pc = p.charAt(j - 1);
            
            if (pc == '.' || pc == sc) {
                dp[i][j] = dp[i - 1][j - 1];
            } else if (pc == '*') {
                char prevChar = p.charAt(j - 2);
                
                // Two choices: use * as zero occurrences or more
                dp[i][j] = dp[i][j - 2];  // Zero occurrences
                
                if (prevChar == '.' || prevChar == sc) {
                    dp[i][j] = dp[i][j] || dp[i - 1][j];  // More occurrences
                }
            }
        }
    }
    
    return dp[m][n];
}

// Time: O(m × n)
// Space: O(m × n)
```

---

## DP Pattern Recognition

### 1D DP Patterns
- **Single sequence optimization**: House Robber, Climbing Stairs
- **Subsequence problems**: Longest Increasing Subsequence
- **Partitioning**: Word Break, Palindrome Partitioning

### 2D DP Patterns
- **Two sequences**: LCS, Edit Distance
- **Grid problems**: Unique Paths, Minimum Path Sum
- **Range queries**: Burst Balloons, Matrix Chain

### State Machine DP
- **Multiple states**: Stock trading with cooldown
- **Constraint tracking**: At most K transactions

---

## Interview Tips

### Problem-Solving Approach
1. **Identify if DP applies**: Overlapping subproblems + optimal substructure
2. **Define state**: What information uniquely identifies a subproblem?
3. **Find recurrence**: How to compute state from previous states?
4. **Handle base cases**: What are the trivial cases?
5. **Optimize space**: Can you reduce dimensions?

### Common Mistakes
❌ Not defining state clearly  
❌ Wrong base cases  
❌ Forgetting to initialize DP table  
❌ Off-by-one errors in indices  
❌ Not considering all transitions  

---

## Next: [Design Problems](linkedin-hashmap-design.md)
