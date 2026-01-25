# LinkedIn Interview Questions: Arrays and Strings

High-frequency array and string problems asked in LinkedIn coding interviews. These problems test pattern recognition, optimization, and clean code implementation.

---

## Must-Practice Problems

### 1. Two Sum (Easy) ⭐⭐⭐⭐⭐
**Frequency**: Very High | **Pattern**: Hash Map

```java
/*
Problem: Given array of integers, return indices of two numbers that add up to target.
You may assume exactly one solution exists.

Example:
Input: nums = [2,7,11,15], target = 9
Output: [0,1] (nums[0] + nums[1] = 9)

LinkedIn Context: Finding matching users based on combined scores
*/

// Approach 1: Hash Map - O(n) time, O(n) space
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();
    
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        
        if (map.containsKey(complement)) {
            return new int[]{map.get(complement), i};
        }
        
        map.put(nums[i], i);
    }
    
    return new int[]{};
}

// Follow-up Questions (LinkedIn Asks These!):
// 1. What if array is sorted? → Two pointers O(n) time, O(1) space
// 2. What if we need all pairs? → Continue searching instead of returning
// 3. What if duplicates exist? → Store list of indices instead of single index
// 4. What if array is too large for memory? → External sorting + two pointers

// Sorted array variant
public int[] twoSumSorted(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) {
            return new int[]{left, right};
        } else if (sum < target) {
            left++;
        } else {
            right--;
        }
    }
    
    return new int[]{};
}
```

---

### 2. Three Sum (Medium) ⭐⭐⭐⭐⭐
**Frequency**: Very High | **Pattern**: Two Pointers + Sorting

```java
/*
Problem: Find all unique triplets that sum to zero.

Example:
Input: nums = [-1,0,1,2,-1,-4]
Output: [[-1,-1,2],[-1,0,1]]

LinkedIn Context: Finding groups of users with balanced attributes
*/

public List<List<Integer>> threeSum(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(nums);  // O(n log n)
    
    for (int i = 0; i < nums.length - 2; i++) {
        // Skip duplicates for first element
        if (i > 0 && nums[i] == nums[i - 1]) continue;
        
        // Two sum for target = -nums[i]
        int left = i + 1;
        int right = nums.length - 1;
        
        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            
            if (sum == 0) {
                result.add(Arrays.asList(nums[i], nums[left], nums[right]));
                
                // Skip duplicates
                while (left < right && nums[left] == nums[left + 1]) left++;
                while (left < right && nums[right] == nums[right - 1]) right--;
                
                left++;
                right--;
            } else if (sum < 0) {
                left++;
            } else {
                right--;
            }
        }
    }
    
    return result;
}

// Time: O(n²)
// Space: O(1) excluding output
```

---

### 3. Longest Substring Without Repeating Characters (Medium) ⭐⭐⭐⭐⭐
**Frequency**: Very High | **Pattern**: Sliding Window

```java
/*
Problem: Find length of longest substring without repeating characters.

Example:
Input: s = "abcabcbb"
Output: 3 (substring "abc")

LinkedIn Context: Finding longest unique skill sequence, message parsing
*/

public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastIndex = new HashMap<>();
    int maxLength = 0;
    int left = 0;
    
    for (int right = 0; right < s.length(); right++) {
        char ch = s.charAt(right);
        
        // If character seen and within current window
        if (lastIndex.containsKey(ch) && lastIndex.get(ch) >= left) {
            left = lastIndex.get(ch) + 1;
        }
        
        lastIndex.put(ch, right);
        maxLength = Math.max(maxLength, right - left + 1);
    }
    
    return maxLength;
}

// Time: O(n)
// Space: O(min(n, charset))

// Follow-up: What if only lowercase letters? Use int[26] instead of HashMap
public int lengthOfLongestSubstringOptimized(String s) {
    int[] lastIndex = new int[128];  // ASCII
    Arrays.fill(lastIndex, -1);
    
    int maxLength = 0;
    int left = 0;
    
    for (int right = 0; right < s.length(); right++) {
        char ch = s.charAt(right);
        
        if (lastIndex[ch] >= left) {
            left = lastIndex[ch] + 1;
        }
        
        lastIndex[ch] = right;
        maxLength = Math.max(maxLength, right - left + 1);
    }
    
    return maxLength;
}
```

---

### 4. Merge Intervals (Medium) ⭐⭐⭐⭐⭐
**Frequency**: Very High | **Pattern**: Sorting + Greedy

```java
/*
Problem: Merge overlapping intervals.

Example:
Input: [[1,3],[2,6],[8,10],[15,18]]
Output: [[1,6],[8,10],[15,18]]

LinkedIn Context: Meeting scheduler, event calendars, availability tracking
*/

public int[][] merge(int[][] intervals) {
    if (intervals.length <= 1) return intervals;
    
    // Sort by start time
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));
    
    List<int[]> merged = new ArrayList<>();
    int[] current = intervals[0];
    merged.add(current);
    
    for (int[] interval : intervals) {
        if (interval[0] <= current[1]) {
            // Overlapping - extend current
            current[1] = Math.max(current[1], interval[1]);
        } else {
            // Non-overlapping - add new interval
            current = interval;
            merged.add(current);
        }
    }
    
    return merged.toArray(new int[merged.size()][]);
}

// Time: O(n log n)
// Space: O(n)

// Follow-up: Insert new interval into sorted intervals
public int[][] insert(int[][] intervals, int[] newInterval) {
    List<int[]> result = new ArrayList<>();
    int i = 0;
    
    // Add all intervals before newInterval
    while (i < intervals.length && intervals[i][1] < newInterval[0]) {
        result.add(intervals[i]);
        i++;
    }
    
    // Merge overlapping intervals
    while (i < intervals.length && intervals[i][0] <= newInterval[1]) {
        newInterval[0] = Math.min(newInterval[0], intervals[i][0]);
        newInterval[1] = Math.max(newInterval[1], intervals[i][1]);
        i++;
    }
    result.add(newInterval);
    
    // Add remaining intervals
    while (i < intervals.length) {
        result.add(intervals[i]);
        i++;
    }
    
    return result.toArray(new int[result.size()][]);
}
```

---

### 5. Maximum Subarray (Kadane's Algorithm) (Medium) ⭐⭐⭐⭐
**Frequency**: High | **Pattern**: Dynamic Programming / Greedy

```java
/*
Problem: Find contiguous subarray with largest sum.

Example:
Input: nums = [-2,1,-3,4,-1,2,1,-5,4]
Output: 6 (subarray [4,-1,2,1])

LinkedIn Context: Analyzing engagement trends, revenue optimization
*/

public int maxSubArray(int[] nums) {
    int maxSum = nums[0];
    int currentSum = nums[0];
    
    for (int i = 1; i < nums.length; i++) {
        // Either extend current subarray or start new one
        currentSum = Math.max(nums[i], currentSum + nums[i]);
        maxSum = Math.max(maxSum, currentSum);
    }
    
    return maxSum;
}

// Time: O(n)
// Space: O(1)

// Follow-up: Return the actual subarray
public int[] maxSubArrayWithIndices(int[] nums) {
    int maxSum = nums[0];
    int currentSum = nums[0];
    int start = 0, end = 0, tempStart = 0;
    
    for (int i = 1; i < nums.length; i++) {
        if (currentSum < 0) {
            currentSum = nums[i];
            tempStart = i;
        } else {
            currentSum += nums[i];
        }
        
        if (currentSum > maxSum) {
            maxSum = currentSum;
            start = tempStart;
            end = i;
        }
    }
    
    return Arrays.copyOfRange(nums, start, end + 1);
}
```

---

### 6. Minimum Window Substring (Hard) ⭐⭐⭐⭐⭐
**Frequency**: High | **Pattern**: Sliding Window

```java
/*
Problem: Find minimum window in s that contains all characters of t.

Example:
Input: s = "ADOBECODEBANC", t = "ABC"
Output: "BANC"

LinkedIn Context: Search result matching, skill matching
*/

public String minWindow(String s, String t) {
    if (s.length() < t.length()) return "";
    
    Map<Character, Integer> target = new HashMap<>();
    for (char c : t.toCharArray()) {
        target.put(c, target.getOrDefault(c, 0) + 1);
    }
    
    int required = target.size();
    int formed = 0;
    
    Map<Character, Integer> window = new HashMap<>();
    int left = 0, right = 0;
    int minLen = Integer.MAX_VALUE;
    int minLeft = 0;
    
    while (right < s.length()) {
        // Expand window
        char c = s.charAt(right);
        window.put(c, window.getOrDefault(c, 0) + 1);
        
        if (target.containsKey(c) && 
            window.get(c).intValue() == target.get(c).intValue()) {
            formed++;
        }
        
        // Contract window
        while (left <= right && formed == required) {
            // Update result
            if (right - left + 1 < minLen) {
                minLen = right - left + 1;
                minLeft = left;
            }
            
            char leftChar = s.charAt(left);
            window.put(leftChar, window.get(leftChar) - 1);
            
            if (target.containsKey(leftChar) && 
                window.get(leftChar) < target.get(leftChar)) {
                formed--;
            }
            
            left++;
        }
        
        right++;
    }
    
    return minLen == Integer.MAX_VALUE ? "" : s.substring(minLeft, minLeft + minLen);
}

// Time: O(m + n) where m = s.length, n = t.length
// Space: O(m + n)
```

---

### 7. Group Anagrams (Medium) ⭐⭐⭐⭐
**Frequency**: High | **Pattern**: Hash Map

```java
/*
Problem: Group strings that are anagrams.

Example:
Input: strs = ["eat","tea","tan","ate","nat","bat"]
Output: [["bat"],["nat","tan"],["ate","eat","tea"]]

LinkedIn Context: Grouping similar profiles, skill clustering
*/

public List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> groups = new HashMap<>();
    
    for (String str : strs) {
        // Create key from sorted characters
        char[] chars = str.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars);
        
        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(str);
    }
    
    return new ArrayList<>(groups.values());
}

// Time: O(n * k log k) where n = number of strings, k = max length
// Space: O(n * k)

// Optimized: Use character count as key (no sorting needed)
public List<List<String>> groupAnagramsOptimized(String[] strs) {
    Map<String, List<String>> groups = new HashMap<>();
    
    for (String str : strs) {
        int[] count = new int[26];
        for (char c : str.toCharArray()) {
            count[c - 'a']++;
        }
        
        // Create key from count array
        StringBuilder keyBuilder = new StringBuilder();
        for (int i = 0; i < 26; i++) {
            if (count[i] > 0) {
                keyBuilder.append((char)('a' + i));
                keyBuilder.append(count[i]);
            }
        }
        String key = keyBuilder.toString();
        
        groups.computeIfAbsent(key, k -> new ArrayList<>()).add(str);
    }
    
    return new ArrayList<>(groups.values());
}

// Time: O(n * k) - better!
// Space: O(n * k)
```

---

### 8. Product of Array Except Self (Medium) ⭐⭐⭐⭐
**Frequency**: Medium | **Pattern**: Prefix/Suffix Arrays

```java
/*
Problem: Return array where output[i] = product of all elements except nums[i].
Must run in O(n) without division.

Example:
Input: nums = [1,2,3,4]
Output: [24,12,8,6]

LinkedIn Context: Computing relative metrics, normalized scores
*/

public int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    
    // result[i] = product of all elements to the left
    result[0] = 1;
    for (int i = 1; i < n; i++) {
        result[i] = result[i - 1] * nums[i - 1];
    }
    
    // Multiply by product of all elements to the right
    int rightProduct = 1;
    for (int i = n - 1; i >= 0; i--) {
        result[i] *= rightProduct;
        rightProduct *= nums[i];
    }
    
    return result;
}

// Time: O(n)
// Space: O(1) excluding output array
```

---

### 9. Valid Parentheses (Easy) ⭐⭐⭐⭐
**Frequency**: Medium | **Pattern**: Stack

```java
/*
Problem: Determine if string of brackets is valid.

Example:
Input: s = "()[]{}"
Output: true

LinkedIn Context: Expression validation, nested structure parsing
*/

public boolean isValid(String s) {
    Stack<Character> stack = new Stack<>();
    Map<Character, Character> pairs = Map.of(
        ')', '(',
        ']', '[',
        '}', '{'
    );
    
    for (char c : s.toCharArray()) {
        if (pairs.containsKey(c)) {
            // Closing bracket
            if (stack.isEmpty() || stack.pop() != pairs.get(c)) {
                return false;
            }
        } else {
            // Opening bracket
            stack.push(c);
        }
    }
    
    return stack.isEmpty();
}

// Time: O(n)
// Space: O(n)
```

---

### 10. Longest Palindromic Substring (Medium) ⭐⭐⭐
**Frequency**: Medium | **Pattern**: Expand Around Center

```java
/*
Problem: Find longest palindromic substring.

Example:
Input: s = "babad"
Output: "bab" or "aba"

LinkedIn Context: Finding symmetric patterns in data
*/

public String longestPalindrome(String s) {
    if (s.length() < 2) return s;
    
    int maxLen = 0;
    int start = 0;
    
    for (int i = 0; i < s.length(); i++) {
        // Odd-length palindromes
        int len1 = expandAroundCenter(s, i, i);
        // Even-length palindromes
        int len2 = expandAroundCenter(s, i, i + 1);
        
        int len = Math.max(len1, len2);
        if (len > maxLen) {
            maxLen = len;
            start = i - (len - 1) / 2;
        }
    }
    
    return s.substring(start, start + maxLen);
}

private int expandAroundCenter(String s, int left, int right) {
    while (left >= 0 && right < s.length() && s.charAt(left) == s.charAt(right)) {
        left--;
        right++;
    }
    return right - left - 1;
}

// Time: O(n²)
// Space: O(1)
```

---

## Interview Tips for Arrays/Strings

### Common Patterns
1. **Hash Map**: Two Sum, Group Anagrams
2. **Sliding Window**: Longest substring, minimum window
3. **Two Pointers**: Three Sum, merge intervals
4. **Sorting**: Merge intervals, group anagrams
5. **Stack**: Valid parentheses, next greater element

### Optimization Techniques
- Use hash map to trade space for time
- Sort if order doesn't matter
- Two pointers for sorted arrays
- Sliding window for contiguous subarrays

### Edge Cases to Consider
- Empty array/string
- Single element
- All same elements
- Negative numbers (for sum problems)
- Unicode vs ASCII (for character problems)

---

## Next: [Linked Lists and Two-Pointers](linkedin-linked-lists.md)
