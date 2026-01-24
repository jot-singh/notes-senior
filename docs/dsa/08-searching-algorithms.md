# Searching Algorithms

## What You'll Learn
- Linear search and optimization techniques
- Binary search and its variations
- When to use each approach based on data structure and constraints
- Real-world search applications: databases, logs, version control
- How searching patterns combine with other data structures

## Why This Matters

Searching is **the complement to sorting**. While sorting takes O(n log n), it enables O(log n) searches for future queries. Understanding when to pay the sorting cost upfront versus searching unsorted data teaches optimization trade-offs. Binary search is a gateway to understanding greedy algorithms and pruning search spaces.

---

## Linear Search

### When Linear Search is Optimal

```java
public class LinearSearch {
    // Basic linear search
    public int linearSearch(int[] arr, int target) {
        for (int i = 0; i < arr.length; i++) {
            if (arr[i] == target) {
                return i;
            }
        }
        return -1;
    }
    
    // Time: O(n)
    // Space: O(1)
    // When better than binary search:
    // - Unsorted array
    // - Single search (binary needs sorting: O(n log n))
    // - Small array (binary search overhead not worth it)
    // - Searching through linked list
}
```

**Cost Analysis**:
```
Single search in unsorted array:
- Linear: O(n)
- Sort + Binary: O(n log n) + O(log n) = O(n log n)
- Linear is better unless doing many searches

Many searches in same array:
- Linear × k: O(k·n)
- Sort once + Binary × k: O(n log n) + O(k·log n)
- Binary is better if k·log n < k·n (almost always true)
```

---

## Binary Search

### Core Binary Search

**When to use**: Sorted array/list, searching for exact value or insertion point.

**The Intuition**: Compare middle element. If target is smaller, search left half. If larger, search right half.

```java
public class BinarySearch {
    // Standard binary search - find exact value
    public int binarySearch(int[] arr, int target) {
        int left = 0;
        int right = arr.length - 1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;  // Avoid overflow
            
            if (arr[mid] == target) {
                return mid;
            } else if (arr[mid] < target) {
                left = mid + 1;  // Search right
            } else {
                right = mid - 1;  // Search left
            }
        }
        
        return -1;  // Not found
    }
    
    // Time: O(log n)
    // Space: O(1)
}
```

**Why `left + (right - left) / 2` instead of `(left + right) / 2`**:
- Avoids integer overflow if left and right are large
- Same mathematical result but prevents bugs

### Pattern 1: Finding Insertion Point

**When to use**: Maintaining sorted arrays, implementing sorted maps.

```java
public class BinarySearchVariations {
    // Find where target should be inserted to keep array sorted
    public int searchInsertPosition(int[] arr, int target) {
        int left = 0;
        int right = arr.length;  // Note: length, not length-1
        
        while (left < right) {
            int mid = left + (right - left) / 2;
            
            if (arr[mid] < target) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
        
        return left;  // Position to insert
    }
    
    // Example: [1, 3, 5, 6], target = 5 → return 2
    // Example: [1, 3, 5, 6], target = 4 → return 2 (insert before 5)
    // Example: [1, 3, 5, 6], target = 7 → return 4
}
```

### Pattern 2: First and Last Occurrence

**When to use**: Finding range of equal elements, implementing range queries.

```java
public class FirstLastOccurrence {
    // Find first occurrence of target
    public int findFirst(int[] arr, int target) {
        int left = 0, right = arr.length - 1;
        int result = -1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            
            if (arr[mid] == target) {
                result = mid;
                right = mid - 1;  // Keep searching left
            } else if (arr[mid] < target) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
        
        return result;
    }
    
    // Find last occurrence of target
    public int findLast(int[] arr, int target) {
        int left = 0, right = arr.length - 1;
        int result = -1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            
            if (arr[mid] == target) {
                result = mid;
                left = mid + 1;  // Keep searching right
            } else if (arr[mid] < target) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
        
        return result;
    }
    
    // Example: [5, 7, 7, 8, 8, 10], target = 8
    // findFirst returns 3
    // findLast returns 4
    // Range: indices 3-4
}
```

### Pattern 3: Binary Search on Answer

**When to use**: Minimization/maximization with binary search condition, searching for optimal value.

```java
// Problem: Find minimum capacity of ship to transport packages within days
// Approach: Search for answer (capacity), not array indices

public class BinarySearchOnAnswer {
    public int shipWithinDays(int[] packages, int days) {
        int left = getMaxPackage(packages);  // Min capacity
        int right = getTotalWeight(packages);  // Max capacity
        
        while (left < right) {
            int mid = left + (right - left) / 2;
            
            // Can we ship with capacity mid in days?
            if (canShip(packages, days, mid)) {
                right = mid;  // Try smaller capacity
            } else {
                left = mid + 1;  // Need larger capacity
            }
        }
        
        return left;
    }
    
    private boolean canShip(int[] packages, int days, int capacity) {
        int currentDay = 1;
        int currentLoad = 0;
        
        for (int pkg : packages) {
            if (currentLoad + pkg > capacity) {
                currentDay++;
                currentLoad = pkg;
            } else {
                currentLoad += pkg;
            }
        }
        
        return currentDay <= days;
    }
    
    private int getMaxPackage(int[] packages) {
        return Arrays.stream(packages).max().orElse(0);
    }
    
    private int getTotalWeight(int[] packages) {
        return Arrays.stream(packages).sum();
    }
    
    // Time: O(n log S) where S = sum of packages
}
```

**Key Insight**: Binary search doesn't just work on arrays - it works on any **monotonic function**. If increasing X always makes the answer "better," binary search the answer space.

### Pattern 4: Two Pointer Binary Search (For Rotated Array)

**When to use**: Handling special cases like rotated sorted arrays.

```java
public class RotatedArraySearch {
    // Find target in rotated sorted array
    // Example: [4, 5, 6, 7, 0, 1, 2], target = 0
    
    public int search(int[] arr, int target) {
        int left = 0, right = arr.length - 1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            
            if (arr[mid] == target) {
                return mid;
            }
            
            // Determine which side is sorted
            if (arr[left] <= arr[mid]) {
                // Left side is sorted
                if (arr[left] <= target && target < arr[mid]) {
                    // Target is in sorted left side
                    right = mid - 1;
                } else {
                    // Target is in right side
                    left = mid + 1;
                }
            } else {
                // Right side is sorted
                if (arr[mid] < target && target <= arr[right]) {
                    // Target is in sorted right side
                    left = mid + 1;
                } else {
                    // Target is in left side
                    right = mid - 1;
                }
            }
        }
        
        return -1;
    }
    
    // Time: O(log n) average, O(n) worst with duplicates
    // Space: O(1)
}
```

---

## Real-World Search Applications

### 1. **Database Index Search**

```java
// B-Tree style search (databases use this for indexes)
public class DatabaseIndexSearch {
    private class BTreeNode {
        List<Integer> keys = new ArrayList<>();
        List<Integer> pointers = new ArrayList<>();
        boolean isLeaf = false;
    }
    
    // Search in B-Tree (used by databases for disk-efficient searching)
    public Integer search(BTreeNode node, int key) {
        int i = 0;
        
        // Find first key greater than search key
        while (i < node.keys.size() && node.keys.get(i) < key) {
            i++;
        }
        
        // Check if key found
        if (i < node.keys.size() && node.keys.get(i) == key) {
            return node.pointers.get(i);  // Data pointer
        }
        
        // If leaf, key doesn't exist
        if (node.isLeaf) {
            return null;
        }
        
        // Recursively search child
        BTreeNode child = getChild(i);  // Would load from disk
        return search(child, key);
    }
    
    // Used in: SQL indexes, file systems, document databases
    // Benefit: Minimizes disk I/O through optimal branching
}
```

### 2. **Log Searching**

```java
// Find log entries within date range efficiently
public class LogSearch {
    // Logs are time-sorted
    private List<LogEntry> logs;
    
    public List<LogEntry> searchByDateRange(long startTime, long endTime) {
        int start = binarySearchFirst(startTime);
        int end = binarySearchLast(endTime);
        
        if (start == -1 || end == -1 || start > end) {
            return new ArrayList<>();
        }
        
        return logs.subList(start, end + 1);
    }
    
    private int binarySearchFirst(long time) {
        int left = 0, right = logs.size() - 1;
        int result = -1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            if (logs.get(mid).timestamp >= time) {
                result = mid;
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        }
        
        return result;
    }
    
    private int binarySearchLast(long time) {
        int left = 0, right = logs.size() - 1;
        int result = -1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            if (logs.get(mid).timestamp <= time) {
                result = mid;
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
        
        return result;
    }
    
    // Time: O(log n + result size)
    // Used in: Log analysis tools, time-series databases
}
```

### 3. **Version Control (Git-like)**

```java
// Binary search to find commit where bug was introduced
public class GitBisect {
    public Commit findBugIntroduction(List<Commit> commits, TestRunner tester) {
        int left = 0;
        int right = commits.size() - 1;
        int bugCommit = right;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            
            if (tester.testCommit(commits.get(mid))) {
                // Test passes - bug is after this commit
                left = mid + 1;
            } else {
                // Test fails - bug is this commit or earlier
                bugCommit = mid;
                right = mid - 1;
            }
        }
        
        return commits.get(bugCommit);
    }
    
    // Used in: Git bisect, finding which commit broke code
}
```

### 4. **HTTP Caching with Binary Search**

```java
// Cache stores URLs with TTL; find if cache valid
public class HTTPCache {
    private class CacheEntry {
        String url;
        long expiryTime;
    }
    
    private List<CacheEntry> cache;  // Sorted by expiryTime
    
    // Find first entry that hasn't expired
    public CacheEntry getValidEntry(String url) {
        long now = System.currentTimeMillis();
        
        // Find first entry with expiry >= now
        int idx = binarySearchFirstValid(now);
        
        if (idx != -1 && cache.get(idx).url.equals(url)) {
            return cache.get(idx);
        }
        
        return null;  // Expired or not found
    }
    
    private int binarySearchFirstValid(long now) {
        int left = 0, right = cache.size() - 1;
        int result = -1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2;
            if (cache.get(mid).expiryTime >= now) {
                result = mid;
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        }
        
        return result;
    }
}
```

---

## Search Decision Tree

```
What's the best search strategy?

├─ Array is sorted?
│  ├─ Yes
│  │  ├─ Single search? → Binary Search O(log n)
│  │  ├─ Range search? → Binary Search boundaries O(log n)
│  │  └─ Searching answer space? → Binary Search on Answer
│  │
│  └─ No
│     ├─ Single search? → Linear Search O(n)
│     ├─ Many searches? → Sort first O(n log n) then binary O(log n per search)
│     └─ Unknown data size? → Linear Search (can't assume sorted)
│
├─ Data is in database?
│  └─ Use Index if available (B-Tree binary search)
│
└─ Stream/sequential data?
   └─ Linear Search (can't index)
```

---

## Common Mistakes (Anti-Patterns)

### ❌ Binary Search on Unsorted Array
```java
// ❌ Dangerous - results undefined
int[] arr = [5, 2, 8, 1, 9];
int idx = binarySearch(arr, 8);  // Wrong!

// ✅ Sort first or use linear search
Arrays.sort(arr);
int idx = binarySearch(arr, 8);
```

### ❌ Off-by-One Errors
```java
// ❌ Will miss last element
int right = arr.length - 2;  // Should be length - 1

// ✅ Correct boundaries
int right = arr.length - 1;

// ✅ For insertion point
int right = arr.length;  // Can insert at end
```

### ❌ Not Handling Edge Cases
```java
// ❌ Crashes on empty array
int[] arr = {};
int idx = binarySearch(arr, 5);  // ArrayIndexOutOfBoundsException

// ✅ Check length
if (arr.length == 0) return -1;
```

### ❌ Forgetting Mid Calculation Bug
```java
// ❌ Can overflow if left and right are large
int mid = (left + right) / 2;

// ✅ Avoid overflow
int mid = left + (right - left) / 2;
```

---

## Complexity Comparison

| Scenario | Linear | Binary | Cost Analysis |
|----------|--------|--------|---------------|
| Single search, unsorted | O(n) | O(n log n) + O(log n) | Linear wins |
| Single search, sorted | O(n) | O(log n) | Binary wins |
| k searches, unsorted | O(k·n) | O(n log n) + O(k·log n) | Binary if k > log n |
| k searches, sorted | O(k·n) | O(k·log n) | Binary wins always |

---

## Next Steps

- Master binary search boundaries (< vs ≤, first vs last occurrence)
- Practice binary search on answer problems
- Understand trade-off between sorting cost and search benefits
- Move to [Dynamic Programming Fundamentals](09-dynamic-programming.md)
