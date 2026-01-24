# Arrays and Strings

## What You'll Learn
- Fundamental array operations and manipulation techniques
- String handling and pattern recognition
- Sliding window and two-pointer approaches for optimization
- When and why to use arrays vs other data structures
- How array patterns appear in real-world problems

## Why This Matters

Arrays are the foundation of all data structures. Understanding array manipulation deeply unlocks solutions to hundreds of problems. The patterns you develop here—sliding windows, two-pointers, prefix arrays—are building blocks for solving complex problems efficiently.

---

## Array Fundamentals

### What Makes Arrays Useful

Arrays provide **O(1) random access** to elements by index, making them ideal for:
- Cache-friendly sequential access (CPU cache locality)
- Quick lookups when you know the position
- Base data structure for many other data structures

```java
// Array characteristics
int[] arr = new int[10];

// ✅ Constant time access
int element = arr[5];  // O(1) - Direct memory access

// ❌ Linear time insertion/deletion (except at end)
// To insert at index 2, all elements 2+ must shift right: O(n)
```

### When Arrays Fit

- **Fixed-size data**: When you know the size upfront
- **Random access patterns**: When you need index-based lookups
- **Sequential processing**: When you iterate through all elements
- **Caching considerations**: When memory locality matters (databases, caches)

---

## Array Manipulation Patterns

### Pattern 1: Sliding Window

**When to use**: When you need to find a contiguous subarray with specific properties.

**The Intuition**: Instead of recalculating the window for each position, maintain a window and slide it, updating the calculation incrementally.

```java
// Problem: Find longest substring without repeating characters
// Pattern: Two-pointer window that expands and contracts

public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastIndex = new HashMap<>();
    int maxLength = 0;
    int left = 0;
    
    for (int right = 0; right < s.length(); right++) {
        char ch = s.charAt(right);
        
        // If character seen before and within current window, move left
        if (lastIndex.containsKey(ch) && lastIndex.get(ch) >= left) {
            left = lastIndex.get(ch) + 1;
        }
        
        lastIndex.put(ch, right);
        maxLength = Math.max(maxLength, right - left + 1);
    }
    
    return maxLength;
}
```

**Why this works**: Each element is visited at most twice (once by right pointer, once by left pointer), giving **O(n) time** instead of O(n²).

### Pattern 2: Two-Pointer Approach

**When to use**: When working with sorted arrays or when you need to find pairs/combinations.

**The Intuition**: Use pointers from opposite ends moving toward each other. This works because of the ordering guarantee.

```java
// Problem: Find two numbers that add up to target
// Pattern: One pointer from left, one from right

public int[] twoSum(int[] sorted, int target) {
    int left = 0;
    int right = sorted.length - 1;
    
    while (left < right) {
        int sum = sorted[left] + sorted[right];
        
        if (sum == target) {
            return new int[]{left, right};
        } else if (sum < target) {
            left++;  // Need larger sum, move left pointer right
        } else {
            right--;  // Need smaller sum, move right pointer left
        }
    }
    
    return new int[]{-1, -1};
}
```

**Why this works**: With sorted array, if sum is too small, all left-side elements are too small; if too large, all right-side elements are too large. This guarantees we'll converge correctly in **O(n) time**.

### Pattern 3: Prefix Arrays (Cumulative Sums)

**When to use**: When you need to answer range queries efficiently (sum/product of elements in range).

**The Intuition**: Precompute cumulative values so any range query becomes O(1) after O(n) preprocessing.

```java
// Problem: Answer multiple queries about sum of subarray
// Pattern: Prefix sum array for O(1) range queries

class RangeSum {
    private int[] prefix;
    
    public RangeSum(int[] nums) {
        prefix = new int[nums.length + 1];
        // prefix[i] = sum of all elements from 0 to i-1
        for (int i = 0; i < nums.length; i++) {
            prefix[i + 1] = prefix[i] + nums[i];
        }
    }
    
    // Sum from index left to right (inclusive)
    public int rangeSum(int left, int right) {
        return prefix[right + 1] - prefix[left];
    }
}

// Space-time trade-off: O(n) space for O(1) query vs O(n) time per query
```

**Why this works**: sum[left to right] = prefix[right+1] - prefix[left]. This arithmetic allows instant computation.

### Pattern 4: In-Place Modifications

**When to use**: When you need to modify an array using O(1) space or can't allocate extra space.

**The Intuition**: Use clever pointer manipulation or value encoding to modify in-place.

```java
// Problem: Remove duplicates from sorted array
// Pattern: Use slow pointer for final array, fast pointer for scanning

public int removeDuplicates(int[] nums) {
    int slow = 0;
    
    for (int fast = 1; fast < nums.length; fast++) {
        if (nums[fast] != nums[slow]) {
            slow++;
            nums[slow] = nums[fast];
        }
    }
    
    return slow + 1;
}

// Another approach: Negative indexing for marking presence
// Problem: Mark presence without extra space
public void markPresence(int[] nums) {
    for (int num : nums) {
        int index = Math.abs(num);
        nums[index - 1] = -Math.abs(nums[index - 1]);
    }
}
```

---

## String Manipulation Patterns

### What Makes Strings Unique

Strings are arrays of characters but have special properties:
- Immutable in most languages (Java, Python) → create new string on modification
- Unicode/encoding considerations
- Pattern matching is a major use case
- Often memory-constrained in production

### Pattern 1: String Building (Avoiding Concatenation)

**The Problem**: String concatenation in a loop is O(n²) because strings are immutable.

```java
// ❌ O(n²) - Each concatenation creates new string
String result = "";
for (char c : chars) {
    result += c;  // Creates new string each time
}

// ✅ O(n) - StringBuilder accumulates
StringBuilder sb = new StringBuilder();
for (char c : chars) {
    sb.append(c);
}
String result = sb.toString();

// Real-world: Building JSON, SQL queries, logs
```

### Pattern 2: Anagram Detection

**When to use**: Comparing character frequencies or permutations.

```java
// Pattern: Character frequency comparison
public boolean isAnagram(String s1, String s2) {
    if (s1.length() != s2.length()) return false;
    
    // Method 1: Sorting - O(n log n)
    char[] arr1 = s1.toCharArray();
    char[] arr2 = s2.toCharArray();
    Arrays.sort(arr1);
    Arrays.sort(arr2);
    return Arrays.equals(arr1, arr2);
    
    // Method 2: Frequency map - O(n)
    int[] freq = new int[26];
    for (char c : s1.toCharArray()) freq[c - 'a']++;
    for (char c : s2.toCharArray()) freq[c - 'a']--;
    for (int f : freq) if (f != 0) return false;
    return true;
}
```

### Pattern 3: Palindrome Recognition

**When to use**: When checking symmetry in strings or understanding substring properties.

```java
// Pattern 1: Expand around center
public int expandAroundCenter(String s, int left, int right) {
    while (left >= 0 && right < s.length() && s.charAt(left) == s.charAt(right)) {
        left--;
        right++;
    }
    return right - left - 1; // Length of palindrome
}

public String longestPalindrome(String s) {
    int maxLen = 0;
    int maxStart = 0;
    
    for (int i = 0; i < s.length(); i++) {
        // Odd-length palindromes (center is a character)
        int len1 = expandAroundCenter(s, i, i);
        // Even-length palindromes (center is between characters)
        int len2 = expandAroundCenter(s, i, i + 1);
        
        int len = Math.max(len1, len2);
        if (len > maxLen) {
            maxLen = len;
            maxStart = i - (len - 1) / 2;
        }
    }
    
    return s.substring(maxStart, maxStart + maxLen);
}

// Time: O(n²), Space: O(1)
```

---

## Real-World Applications

### 1. **Caching and Buffer Management**
Arrays are the backbone of caches and memory buffers.

```java
// Ring buffer pattern - used in message queues, audio processing
class RingBuffer<T> {
    private T[] data;
    private int head = 0;
    private int tail = 0;
    private int size = 0;
    
    @SuppressWarnings("unchecked")
    public RingBuffer(int capacity) {
        data = new Object[capacity];
    }
    
    public void add(T item) {
        data[tail] = item;
        tail = (tail + 1) % data.length;
        size++;
    }
    
    public T remove() {
        T item = data[head];
        head = (head + 1) % data.length;
        size--;
        return item;
    }
    
    // Used in: Kafka topics, CircularBuffer in databases, audio streaming
}
```

### 2. **Image Processing**
2D arrays represent pixel grids; sliding windows are used for filters.

```java
// Gaussian blur: sliding window over 2D array
public int[][] applyBlur(int[][] image, int radius) {
    int rows = image.length;
    int cols = image[0].length;
    int[][] result = new int[rows][cols];
    
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            int sum = 0, count = 0;
            
            // Sliding window
            for (int x = Math.max(0, i - radius); x <= Math.min(rows - 1, i + radius); x++) {
                for (int y = Math.max(0, j - radius); y <= Math.min(cols - 1, j + radius); y++) {
                    sum += image[x][y];
                    count++;
                }
            }
            
            result[i][j] = sum / count;
        }
    }
    
    return result;
}
```

### 3. **Time Series and Metrics**
Arrays store historical data; sliding windows calculate rolling statistics.

```java
// Calculate 7-day moving average
public double[] movingAverage(double[] prices, int window) {
    double[] avg = new double[prices.length - window + 1];
    double sum = 0;
    
    // Initial window
    for (int i = 0; i < window; i++) sum += prices[i];
    avg[0] = sum / window;
    
    // Slide window
    for (int i = window; i < prices.length; i++) {
        sum = sum - prices[i - window] + prices[i];
        avg[i - window + 1] = sum / window;
    }
    
    return avg;
}

// Used in: Stock analysis, metrics dashboards, anomaly detection
```

---

## Problem-Solving Patterns Checklist

When you see an array problem:

1. **Can I sort it?** → Two-pointer patterns become available
2. **Is it about subarrays/substrings?** → Try sliding window
3. **Do I need range queries?** → Consider prefix arrays
4. **Is it about rearrangement?** → Two-pointer in-place modification
5. **Pattern matching or frequency?** → Hash map or frequency array

---

## Common Mistakes (Anti-Patterns)

### ❌ Off-by-One Errors
```java
// Wrong: accessing arr[n] which doesn't exist
for (int i = 0; i <= arr.length; i++) {
    System.out.println(arr[i]);
}

// ✅ Correct:
for (int i = 0; i < arr.length; i++) {
    System.out.println(arr[i]);
}
```

### ❌ Not Considering Space Optimization
```java
// O(n) space when O(1) is possible
public int removeDuplicates(int[] nums) {
    List<Integer> list = new ArrayList<>();
    for (int num : nums) {
        if (!list.contains(num)) {
            list.add(num);
        }
    }
    return list.size();
}

// ✅ O(1) space with two-pointer
public int removeDuplicates(int[] nums) {
    int slow = 0;
    for (int fast = 1; fast < nums.length; fast++) {
        if (nums[fast] != nums[slow]) {
            nums[++slow] = nums[fast];
        }
    }
    return slow + 1;
}
```

### ❌ String Concatenation Loop
Already covered above - always use StringBuilder.

---

## Complexity Analysis Summary

| Operation | Time | Space | Notes |
|-----------|------|-------|-------|
| Access element | O(1) | - | Direct index lookup |
| Linear search | O(n) | O(1) | Without sorting |
| Binary search | O(log n) | O(1) | Sorted array required |
| Insertion (not at end) | O(n) | O(n) | May need new array |
| Deletion (not at end) | O(n) | - | Elements shift |
| Sliding window | O(n) | O(k) | For window of size k |
| Prefix sum | O(n) | O(n) | Setup + O(1) queries |

---

## Next Steps

- Master these patterns before moving to more complex structures
- Practice problems that combine multiple patterns
- Recognize when arrays with secondary structures (hash maps, heaps) solve problems
- Move to [Linked Lists](02-linked-lists.md) to understand alternative sequential structures
