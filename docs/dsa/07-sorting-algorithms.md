# Sorting Algorithms

## What You'll Learn
- Comparison-based sorting: Merge Sort, Quick Sort, Heap Sort, Insertion Sort
- Linear-time sorting: Counting Sort, Radix Sort, Bucket Sort
- When to use each algorithm based on data characteristics
- Stability, space complexity, and real-world performance
- How sorting appears in system design and optimization

## Why This Matters

Sorting is **foundational to computer science**. Beyond just sorting arrays, understanding sorting teaches you algorithm design patterns: divide-and-conquer (merge sort), randomization (quick sort), heap properties (heap sort). It also teaches when to optimize - often the right approach is finding a smarter algorithm, not just implementing the obvious one faster.

---

## Sorting Landscape

### Comparison-Based vs Non-Comparison Sorting

```
Sorting Algorithms
├─ Comparison-Based (Compare elements to sort)
│  ├─ O(n log n) average/best: Merge Sort, Quick Sort, Heap Sort
│  ├─ O(n²) average/worst: Bubble Sort, Insertion Sort, Selection Sort
│  └─ Info Theory Lower Bound: Ω(n log n) for comparison-based
│
└─ Non-Comparison (Use element values directly)
   ├─ O(n + k) Counting Sort (k = range)
   ├─ O(n·d) Radix Sort (d = digits)
   └─ O(n + k) Bucket Sort (k = buckets)
```

**Key Insight**: If you know properties of your data (range, distribution), you can beat O(n log n) limit.

---

## Comparison-Based Sorting

### Pattern 1: Merge Sort (Divide and Conquer)

**When to use**: Guaranteed O(n log n), stable, predictable performance. Best for linked lists.

**The Intuition**: Divide array in half, recursively sort each half, then merge sorted halves.

```java
public class MergeSort {
    public void sort(int[] arr) {
        if (arr.length <= 1) return;
        
        mergeSort(arr, 0, arr.length - 1);
    }
    
    private void mergeSort(int[] arr, int left, int right) {
        if (left < right) {
            int mid = left + (right - left) / 2;
            
            mergeSort(arr, left, mid);        // Divide
            mergeSort(arr, mid + 1, right);   // Divide
            merge(arr, left, mid, right);     // Conquer
        }
    }
    
    private void merge(int[] arr, int left, int mid, int right) {
        int leftSize = mid - left + 1;
        int rightSize = right - mid;
        
        int[] leftArr = new int[leftSize];
        int[] rightArr = new int[rightSize];
        
        // Copy data to temp arrays
        System.arraycopy(arr, left, leftArr, 0, leftSize);
        System.arraycopy(arr, mid + 1, rightArr, 0, rightSize);
        
        int i = 0, j = 0, k = left;
        
        // Merge - always take smaller of two fronts
        while (i < leftSize && j < rightSize) {
            if (leftArr[i] <= rightArr[j]) {
                arr[k++] = leftArr[i++];
            } else {
                arr[k++] = rightArr[j++];
            }
        }
        
        // Copy remaining elements
        while (i < leftSize) arr[k++] = leftArr[i++];
        while (j < rightSize) arr[k++] = rightArr[j++];
    }
}

// Time: O(n log n) all cases
// Space: O(n) for temp arrays
// Stable: Yes (equal elements maintain relative order)
```

**Example**:
```
[38, 27, 43, 3, 9, 82, 10]

Split:
[38, 27, 43, 3] [9, 82, 10]

Recursively split and merge:
[38] [27] → [27, 38]
[43] [3] → [3, 43]
[27, 38] [3, 43] → [3, 27, 38, 43]

[9] [82, 10] → [9, 82] [10] → [9, 10, 82]

[3, 27, 38, 43] [9, 10, 82] → [3, 9, 10, 27, 38, 43, 82]
```

**Why stable**: When merging, we take from left array when equal (leftArr[i] <= rightArr[j]).

### Pattern 2: Quick Sort (Divide and Conquer with Randomization)

**When to use**: Average O(n log n) with low constants, in-place sorting, most practical choice.

**The Intuition**: Pick pivot, partition around it, recursively sort partitions. Random pivot avoids worst case.

```java
public class QuickSort {
    private Random random = new Random();
    
    public void sort(int[] arr) {
        if (arr.length == 0) return;
        quickSort(arr, 0, arr.length - 1);
    }
    
    private void quickSort(int[] arr, int low, int high) {
        if (low < high) {
            int pivotIndex = partition(arr, low, high);
            
            quickSort(arr, low, pivotIndex - 1);   // Left of pivot
            quickSort(arr, pivotIndex + 1, high);  // Right of pivot
        }
    }
    
    // Hoare partition scheme (more efficient)
    private int partition(int[] arr, int low, int high) {
        // Random pivot avoids O(n²) on sorted arrays
        int randomIndex = low + random.nextInt(high - low + 1);
        swap(arr, randomIndex, high);
        
        int pivot = arr[high];
        int i = low - 1;
        
        for (int j = low; j < high; j++) {
            if (arr[j] < pivot) {
                i++;
                swap(arr, i, j);
            }
        }
        
        swap(arr, i + 1, high);
        return i + 1;
    }
    
    private void swap(int[] arr, int i, int j) {
        int temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }
}

// Time: O(n log n) average, O(n²) worst case (rare with random pivot)
// Space: O(log n) recursion depth
// Stable: No (partition breaks stability)
```

**Pivot Selection Strategies**:
```java
// 1. Random pivot (good for practical data)
int pivot = arr[random.nextInt(n)];

// 2. Median-of-three (good for mostly sorted data)
int mid = (low + high) / 2;
if (arr[low] > arr[mid]) swap(low, mid);
if (arr[mid] > arr[high]) swap(mid, high);
if (arr[low] > arr[mid]) swap(low, mid);
int pivot = arr[mid];

// 3. First/Last element (OK for random data, bad for sorted)
int pivot = arr[low];
```

### Pattern 3: Heap Sort (Using Heap Property)

**When to use**: Guaranteed O(n log n), minimal extra space, when stability doesn't matter.

**The Intuition**: Build max heap, repeatedly extract max and place at end.

```java
public class HeapSort {
    public void sort(int[] arr) {
        int n = arr.length;
        
        // Build max heap
        for (int i = n / 2 - 1; i >= 0; i--) {
            heapify(arr, n, i);
        }
        
        // Extract elements one by one
        for (int i = n - 1; i > 0; i--) {
            // Move root (max) to end
            swap(arr, 0, i);
            
            // Heapify reduced heap
            heapify(arr, i, 0);
        }
    }
    
    private void heapify(int[] arr, int n, int i) {
        int largest = i;
        int left = 2 * i + 1;
        int right = 2 * i + 2;
        
        // Find largest of root, left, right
        if (left < n && arr[left] > arr[largest]) {
            largest = left;
        }
        if (right < n && arr[right] > arr[largest]) {
            largest = right;
        }
        
        // If largest is not root, swap and recursively heapify
        if (largest != i) {
            swap(arr, i, largest);
            heapify(arr, n, largest);
        }
    }
    
    private void swap(int[] arr, int i, int j) {
        int temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }
}

// Time: O(n log n) all cases
// Space: O(1)
// Stable: No
```

**Example**:
```
[4, 10, 3, 5, 1]

Build heap:        10
                  /  \
                 5    3
                / \
               4   1

Extract max:
Move 10 to end, heapify with [1]:
           5            1         swap(1,5)     5
          / \          / \                     / \
         4   3  →     4   3                   4   3

Result: [1, 4, 3, 5, 10]
```

### Pattern 4: Insertion Sort (Small Arrays or Nearly Sorted)

**When to use**: Small arrays (< 10 elements), nearly sorted data, building blocks for other algorithms.

```java
public class InsertionSort {
    public void sort(int[] arr) {
        for (int i = 1; i < arr.length; i++) {
            int key = arr[i];
            int j = i - 1;
            
            // Shift larger elements right
            while (j >= 0 && arr[j] > key) {
                arr[j + 1] = arr[j];
                j--;
            }
            
            // Insert key at correct position
            arr[j + 1] = key;
        }
    }
}

// Time: O(n²) average/worst, O(n) best (nearly sorted)
// Space: O(1)
// Stable: Yes
// Used as: Final pass in hybrid sorts, base case for merge/quick sort
```

---

## Linear-Time Sorting (Non-Comparison)

### Pattern 1: Counting Sort

**When to use**: When element range is small (k ≤ n), need stable sort, distributed computing.

**The Intuition**: Count occurrences, then fill array based on counts.

```java
public class CountingSort {
    public void sort(int[] arr) {
        if (arr.length == 0) return;
        
        int max = arr[0];
        for (int num : arr) max = Math.max(max, num);
        
        // Count occurrences
        int[] count = new int[max + 1];
        for (int num : arr) {
            count[num]++;
        }
        
        // Convert to cumulative count (for stability)
        for (int i = 1; i < count.length; i++) {
            count[i] += count[i - 1];
        }
        
        // Place elements in correct positions (backward to maintain stability)
        int[] sorted = new int[arr.length];
        for (int i = arr.length - 1; i >= 0; i--) {
            int num = arr[i];
            sorted[count[num] - 1] = num;
            count[num]--;
        }
        
        // Copy back
        System.arraycopy(sorted, 0, arr, 0, arr.length);
    }
}

// Time: O(n + k) where k = range of elements
// Space: O(n + k)
// Stable: Yes (if implemented with cumulative counts)
// When k ≤ n: Better than O(n log n)
```

**Example**:
```
Input: [4, 2, 2, 8, 3, 3, 1]

Count: [0, 1, 2, 2, 1, 0, 0, 0, 1]  (index = value)
       [0, 1, 2, 2, 1] (non-zero entries)

Cumulative: [0, 1, 3, 5, 6, 6, 6, 6, 7]
            (index 1 goes to positions 1, index 2 goes to 2-3, etc)

Place backward:
- arr[6]=1: count[1]=1 → sorted[0]=1, count[1]=0
- arr[5]=3: count[3]=5 → sorted[4]=3, count[3]=4
- arr[4]=3: count[3]=4 → sorted[3]=3, count[3]=3
- arr[3]=8: count[8]=7 → sorted[6]=8, count[8]=6
- arr[2]=2: count[2]=3 → sorted[2]=2, count[2]=2
- arr[1]=2: count[2]=2 → sorted[1]=2, count[2]=1
- arr[0]=4: count[4]=6 → sorted[5]=4, count[4]=5

Output: [1, 2, 2, 3, 3, 4, 8]
```

### Pattern 2: Radix Sort

**When to use**: Sorting integers/strings with fixed-length digits, parallel processing.

**The Intuition**: Sort by least significant digit first, maintaining stability, repeat for each digit.

```java
public class RadixSort {
    public void sort(int[] arr) {
        if (arr.length == 0) return;
        
        int max = arr[0];
        for (int num : arr) max = Math.max(max, num);
        
        // Sort by each digit from least significant to most
        for (int exponent = 1; max / exponent > 0; exponent *= 10) {
            countingSortByDigit(arr, exponent);
        }
    }
    
    private void countingSortByDigit(int[] arr, int exponent) {
        int[] count = new int[10];  // Digits 0-9
        
        // Count occurrences of each digit
        for (int num : arr) {
            count[(num / exponent) % 10]++;
        }
        
        // Convert to cumulative
        for (int i = 1; i < 10; i++) {
            count[i] += count[i - 1];
        }
        
        // Place elements
        int[] sorted = new int[arr.length];
        for (int i = arr.length - 1; i >= 0; i--) {
            int digit = (arr[i] / exponent) % 10;
            sorted[count[digit] - 1] = arr[i];
            count[digit]--;
        }
        
        System.arraycopy(sorted, 0, arr, 0, arr.length);
    }
}

// Time: O(n·d) where d = number of digits
// Space: O(n + 10) = O(n)
// Stable: Yes
// Better than O(n log n) when d is small
```

**Example**:
```
Input: [170, 45, 75, 90, 2, 2, 66, 802, 640]

Sort by ones place:
[170, 90, 45, 75, 2, 2, 640, 802, 66]

Sort by tens place (stable):
[2, 2, 45, 640, 66, 170, 75, 90, 802]

Sort by hundreds place (stable):
[2, 2, 45, 66, 75, 90, 170, 640, 802]
```

---

## Sorting Algorithm Decision Tree

```
Which sorting algorithm?

├─ Is stability required?
│  ├─ Yes (elements equal keys must keep order)
│  │  ├─ Small array?  → Insertion Sort
│  │  ├─ Large array?  → Merge Sort
│  │  └─ Integer range small? → Counting Sort
│  │
│  └─ No (stability not required)
│     ├─ Small array?  → Insertion Sort
│     ├─ Large array?  → Quick Sort (fastest average)
│     └─ Memory tight?  → Heap Sort (O(1) space)
│
├─ Is data nearly sorted?
│  ├─ Yes → Insertion Sort
│  └─ No  → See above
│
└─ Integer range small (k ≤ n)?
   ├─ Yes → Counting Sort or Radix Sort
   └─ No  → Comparison-based (Merge/Quick/Heap)
```

---

## Real-World Applications

### 1. **Database Query Optimization**

Databases choose sorting algorithm based on:
- Sort size vs memory
- Existing data patterns
- Whether additional memory is available

```java
// Database might use 2-pass sort:
// 1. Multiple merge sorts on chunks that fit in memory
// 2. Merge sorted chunks

public void externalMergeSort(File unsorted) {
    int chunkSize = 100_000;  // Fit in memory
    List<File> sortedChunks = new ArrayList<>();
    
    // Pass 1: Sort chunks
    try (BufferedReader reader = new BufferedReader(new FileReader(unsorted))) {
        List<Integer> chunk = new ArrayList<>();
        String line;
        
        while ((line = reader.readLine()) != null) {
            chunk.add(Integer.parseInt(line));
            
            if (chunk.size() == chunkSize) {
                Collections.sort(chunk);  // Merge Sort
                File chunkFile = saveChunk(chunk);
                sortedChunks.add(chunkFile);
                chunk.clear();
            }
        }
        
        if (!chunk.isEmpty()) {
            Collections.sort(chunk);
            sortedChunks.add(saveChunk(chunk));
        }
    }
    
    // Pass 2: Merge sorted chunks
    mergeChunks(sortedChunks);
}

// Used in: SQL ORDER BY, data warehouses
```

### 2. **Search Engine Ranking**

```java
// Sorting documents by relevance score
public class SearchRanking {
    public List<Document> rankResults(List<Document> docs, Query query) {
        // Custom comparator for stability and relevance
        docs.sort((d1, d2) -> {
            double score1 = d1.calculateRelevance(query);
            double score2 = d2.calculateRelevance(query);
            
            int cmp = Double.compare(score2, score1);  // Descending
            if (cmp != 0) return cmp;
            
            // Tie-breaker: Stable sort maintains original position
            return Integer.compare(d1.originalIndex, d2.originalIndex);
        });
        
        return docs;
    }
}

// Used in: Google, DuckDuckGo, search engines
```

### 3. **Real-Time Data Processing (k-largest elements)**

```java
// Find top-k without fully sorting (more efficient)
public List<Integer> topKElements(int[] nums, int k) {
    // Min heap of size k (O(n log k) instead of O(n log n))
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    
    for (int num : nums) {
        if (minHeap.size() < k) {
            minHeap.offer(num);
        } else if (num > minHeap.peek()) {
            minHeap.poll();
            minHeap.offer(num);
        }
    }
    
    return new ArrayList<>(minHeap);
}

// Used in: Real-time leaderboards, top-k queries, streaming analytics
```

---

## Common Mistakes (Anti-Patterns)

### ❌ Using Bubble Sort in Production
```java
// ❌ O(n²) - terrible for any real data
for (int i = 0; i < n; i++) {
    for (int j = 0; j < n - 1; j++) {
        if (arr[j] > arr[j + 1]) {
            swap(arr, j, j + 1);
        }
    }
}

// ✅ Use Java's built-in (Dual Pivot Quick Sort / Merge Sort)
Arrays.sort(arr);
Collections.sort(list);
```

### ❌ Not Considering Stability When Required
```java
// ❌ Quick Sort breaks stability - wrong for sorting by multiple keys
person[] people = {...};
quickSort(people);  // Lost original relative order

// ✅ Use Merge Sort when stability matters
mergeSort(people);
```

### ❌ Counting Sort with Wrong Range
```java
// ❌ If you don't know max value, this fails
int[] arr = [10, 5000, 200];  // Max is 5000
int[] count = new int[100];  // Too small!

// ✅ Find max first
int max = Arrays.stream(arr).max().orElse(0);
int[] count = new int[max + 1];
```

---

## Complexity Comparison Table

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|------|---------|-------|-------|--------|
| Insertion | O(n) | O(n²) | O(n²) | O(1) | ✅ |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ |
| Quick | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ |
| Heap | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ |
| Counting | O(n+k) | O(n+k) | O(n+k) | O(n+k) | ✅ |
| Radix | O(n·d) | O(n·d) | O(n·d) | O(n+k) | ✅ |

---

## Next Steps

- Understand that Java's `Arrays.sort()` uses Dual Pivot Quick Sort
- Learn when to use TimSort (merge sort for nearly sorted data) in Python
- Practice choosing algorithms based on data characteristics
- Move to [Searching Algorithms](08-searching-algorithms.md)
