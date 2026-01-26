# Heaps and Priority Queues

## What You'll Learn
- Heap properties and why they enable efficient priority operations
- Building intuition for when heaps outperform other structures
- Min-heap and max-heap implementations
- Heapify operations and building heaps efficiently
- Priority queue patterns (Top K, K-way merge, median finding)
- Real-world applications in task scheduling and stream processing
- Trade-offs: heaps vs balanced BSTs vs sorted arrays

## Why This Matters

Heaps are **specialized tree structures** optimized for **priority-based access**—always getting the minimum or maximum element in O(1) time. While arrays provide O(n) min/max and BSTs provide O(log n), heaps offer the best of both: O(1) access to extremum with O(log n) insertions. Understanding heaps means solving problems involving "top K elements," "merge K sorted lists," "task scheduling," and "streaming medians"—patterns that appear everywhere from operating system schedulers to real-time analytics.

---

## Building the Intuition

### The Problem Arrays and BSTs Solve Inefficiently

```java
// Problem: Maintain top 3 scores in a game, processing 1 million updates

// Approach 1: Sorted Array
List<Integer> scores = new ArrayList<>();
public void addScore(int score) {
    scores.add(score);
    Collections.sort(scores);  // O(n log n)
    if (scores.size() > 3) {
        scores.remove(0);  // O(n) due to shifting
    }
}
// Every insertion: O(n log n) - too slow!

// Approach 2: BST (TreeSet in Java)
TreeSet<Integer> scores = new TreeSet<>();
public void addScore(int score) {
    scores.add(score);  // O(log n)
    if (scores.size() > 3) {
        scores.pollFirst();  // O(log n)
    }
}
// Better, but still O(log n) per operation
// And we only need top 3, not full sorted order!

// Approach 3: Min-Heap (size 3)
PriorityQueue<Integer> topScores = new PriorityQueue<>(3);  // Min-heap
public void addScore(int score) {
    topScores.offer(score);  // O(log 3) = O(1)
    if (topScores.size() > 3) {
        topScores.poll();  // O(log 3) = O(1)
    }
}
// Peek at minimum: O(1)
// Fixed size heap → constant time operations!
```

**Key insight**: When you only need extremes (min/max) and don't need full sorted order, heaps are optimal.

---

## Heap Properties

### The Heap Invariant

```
Min-Heap Property:
For every node i (except root):
  heap[i] >= heap[parent(i)]

Parent is always smaller than or equal to children.
Result: Root contains minimum element.

Max-Heap Property:
For every node i (except root):
  heap[i] <= heap[parent(i)]

Parent is always larger than or equal to children.
Result: Root contains maximum element.

CRITICAL: No ordering between siblings!
```

### Visual Representation

```
Min-Heap:
        1              Array: [1, 3, 2, 7, 5, 4, 6]
       / \             
      3   2            Parent at i → children at 2i+1, 2i+2
     / \ / \           Child at i → parent at (i-1)/2
    7  5 4  6

Max-Heap:
        9              Array: [9, 7, 6, 3, 5, 2, 4]
       / \
      7   6
     / \ / \
    3  5 2  4

Key observations:
1. Complete binary tree (all levels filled except possibly last)
2. Last level filled left to right
3. Root = min (min-heap) or max (max-heap)
4. Efficient array representation (no pointers needed)
5. Height = O(log n)
```

### Why Array Representation is Genius

```java
// Binary tree stored as array - no Node objects!

class MinHeap {
    private int[] heap;
    private int size;
    private int capacity;
    
    // Index relationships:
    private int parent(int i) { return (i - 1) / 2; }
    private int leftChild(int i) { return 2 * i + 1; }
    private int rightChild(int i) { return 2 * i + 2; }
    
    // Example: heap = [1, 3, 2, 7, 5, 4, 6]
    // Index 0 (value 1): left=1, right=2 → values 3, 2 ✓
    // Index 1 (value 3): parent=0, left=3, right=4 → values 1, 7, 5 ✓
}

// Advantages:
// 1. No pointer overhead (Node objects)
// 2. Cache-friendly (contiguous memory)
// 3. Simple index arithmetic
// 4. No null checks
```

---

## Core Operations

### Pattern 1: Insert (Bubble Up)

```java
public class MinHeap {
    private int[] heap;
    private int size;
    
    public MinHeap(int capacity) {
        heap = new int[capacity];
        size = 0;
    }
    
    // Insert new element at end, then bubble up
    public void insert(int value) {
        if (size == heap.length) {
            throw new IllegalStateException("Heap is full");
        }
        
        // Add at end
        heap[size] = value;
        size++;
        
        // Bubble up to maintain heap property
        bubbleUp(size - 1);
    }
    
    private void bubbleUp(int index) {
        int parentIndex = parent(index);
        
        // While not at root and smaller than parent
        while (index > 0 && heap[index] < heap[parentIndex]) {
            // Swap with parent
            swap(index, parentIndex);
            
            // Move up
            index = parentIndex;
            parentIndex = parent(index);
        }
    }
    
    private void swap(int i, int j) {
        int temp = heap[i];
        heap[i] = heap[j];
        heap[j] = temp;
    }
    
    private int parent(int i) { return (i - 1) / 2; }
    
    // Time: O(log n) - height of tree
    // Space: O(1)
}

// Visual example of insert:
// Initial heap: [1, 3, 2, 7, 5]
//        1
//       / \
//      3   2
//     / \
//    7   5

// Insert 0:
// Step 1: Add at end
//        1
//       / \
//      3   2
//     / \ /
//    7  5 0

// Step 2: Bubble up (0 < 2)
//        1
//       / \
//      3   0
//     / \ /
//    7  5 2

// Step 3: Bubble up (0 < 1)
//        0
//       / \
//      3   1
//     / \ /
//    7  5 2
```

### Pattern 2: Extract Min (Bubble Down)

```java
// Remove and return root (minimum element)
public int extractMin() {
    if (size == 0) {
        throw new IllegalStateException("Heap is empty");
    }
    
    // Save root value
    int min = heap[0];
    
    // Move last element to root
    heap[0] = heap[size - 1];
    size--;
    
    // Bubble down to maintain heap property
    if (size > 0) {
        bubbleDown(0);
    }
    
    return min;
}

private void bubbleDown(int index) {
    while (true) {
        int smallest = index;
        int left = leftChild(index);
        int right = rightChild(index);
        
        // Find smallest among node and its children
        if (left < size && heap[left] < heap[smallest]) {
            smallest = left;
        }
        if (right < size && heap[right] < heap[smallest]) {
            smallest = right;
        }
        
        // If current node is smallest, done
        if (smallest == index) {
            break;
        }
        
        // Swap with smaller child and continue
        swap(index, smallest);
        index = smallest;
    }
}

private int leftChild(int i) { return 2 * i + 1; }
private int rightChild(int i) { return 2 * i + 2; }

// Time: O(log n)
// Space: O(1)

// Visual example:
// Initial heap: [1, 3, 2, 7, 5, 4]
//        1
//       / \
//      3   2
//     / \ /
//    7  5 4

// Extract min (1):
// Step 1: Replace root with last element
//        4
//       / \
//      3   2
//     / \
//    7  5

// Step 2: Bubble down (4 > 2, swap with smaller child)
//        2
//       / \
//      3   4
//     / \
//    7  5

// Result: [2, 3, 4, 7, 5]
```

### Pattern 3: Peek (Get Min without Removing)

```java
public int peek() {
    if (size == 0) {
        throw new IllegalStateException("Heap is empty");
    }
    return heap[0];
}

// Time: O(1) - just return root
// Space: O(1)

// This is why heaps are powerful:
// Instant access to min/max, but maintaining it is only O(log n)
```

### Pattern 4: Heapify (Build Heap from Array)

```java
// Build heap from unsorted array
public MinHeap(int[] array) {
    heap = array.clone();
    size = array.length;
    
    // Start from last non-leaf node and bubble down
    for (int i = size / 2 - 1; i >= 0; i--) {
        bubbleDown(i);
    }
}

// Why start from size/2 - 1?
// Nodes at index >= size/2 are leaves (no children)
// Only internal nodes need heapify

// Time: O(n) - NOT O(n log n)!
// Proof: Most nodes are near leaves (height 0)
//        Only few nodes near root (height log n)
//        Weighted sum converges to O(n)

// Space: O(1)

// Example: heapify [4, 10, 3, 5, 1]
//
// Initial tree:
//        4
//       / \
//      10  3
//     / \
//    5   1
//
// Start at index 1 (value 10):
// Swap 10 with 1:
//        4
//       / \
//      1   3
//     / \
//    5  10
//
// Move to index 0 (value 4):
// Swap 4 with 1:
//        1
//       / \
//      4   3
//     / \
//    5  10
//
// Swap 4 with 5? No, 4 < 5
// Final heap: [1, 4, 3, 5, 10]
```

---

## Priority Queue Implementation

### Java's PriorityQueue

```java
// Min-heap by default
PriorityQueue<Integer> minHeap = new PriorityQueue<>();

// Max-heap using comparator
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

// Custom comparator
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    (a, b) -> a.priority - b.priority
);

// Common operations:
minHeap.offer(5);      // Insert: O(log n)
minHeap.peek();        // Get min: O(1)
minHeap.poll();        // Extract min: O(log n)
minHeap.size();        // Size: O(1)
minHeap.isEmpty();     // Check empty: O(1)

// Remove arbitrary element: O(n) - must search + O(log n) removal
minHeap.remove(element);
```

### Custom Min-Heap with Generic Type

```java
public class MinHeap<T extends Comparable<T>> {
    private List<T> heap;
    
    public MinHeap() {
        heap = new ArrayList<>();
    }
    
    public void insert(T value) {
        heap.add(value);
        bubbleUp(heap.size() - 1);
    }
    
    public T extractMin() {
        if (heap.isEmpty()) {
            throw new NoSuchElementException();
        }
        
        T min = heap.get(0);
        T last = heap.remove(heap.size() - 1);
        
        if (!heap.isEmpty()) {
            heap.set(0, last);
            bubbleDown(0);
        }
        
        return min;
    }
    
    public T peek() {
        if (heap.isEmpty()) {
            throw new NoSuchElementException();
        }
        return heap.get(0);
    }
    
    private void bubbleUp(int index) {
        int parentIndex = (index - 1) / 2;
        
        while (index > 0 && heap.get(index).compareTo(heap.get(parentIndex)) < 0) {
            Collections.swap(heap, index, parentIndex);
            index = parentIndex;
            parentIndex = (index - 1) / 2;
        }
    }
    
    private void bubbleDown(int index) {
        int size = heap.size();
        
        while (true) {
            int smallest = index;
            int left = 2 * index + 1;
            int right = 2 * index + 2;
            
            if (left < size && heap.get(left).compareTo(heap.get(smallest)) < 0) {
                smallest = left;
            }
            if (right < size && heap.get(right).compareTo(heap.get(smallest)) < 0) {
                smallest = right;
            }
            
            if (smallest == index) break;
            
            Collections.swap(heap, index, smallest);
            index = smallest;
        }
    }
    
    public int size() {
        return heap.size();
    }
    
    public boolean isEmpty() {
        return heap.isEmpty();
    }
}
```

---

## Common Heap Patterns

### Pattern 1: Top K Elements

```java
// Find K largest elements in array
public List<Integer> findKLargest(int[] nums, int k) {
    // Use min-heap of size k
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(k);
    
    for (int num : nums) {
        minHeap.offer(num);
        
        // Keep heap size at k
        if (minHeap.size() > k) {
            minHeap.poll();  // Remove smallest
        }
    }
    
    return new ArrayList<>(minHeap);
}

// Why min-heap for LARGEST elements?
// Min-heap of size k keeps k largest seen so far
// Root is smallest of k largest → anything smaller gets rejected
// Time: O(n log k)
// Space: O(k)

// Example: nums = [3, 2, 1, 5, 6, 4], k = 2
// Step by step:
// [3]           size=1
// [2, 3]        size=2
// [2, 3] poll 1 (1<2, ignore)
// [3, 5] poll 2 (5>2, 2 removed)
// [5, 6] poll 3 (6>3, 3 removed)
// [5, 6] poll 4 (4<5, ignore)
// Result: [5, 6]

// Reverse: K smallest elements use max-heap
public List<Integer> findKSmallest(int[] nums, int k) {
    PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
    
    for (int num : nums) {
        maxHeap.offer(num);
        if (maxHeap.size() > k) {
            maxHeap.poll();  // Remove largest
        }
    }
    
    return new ArrayList<>(maxHeap);
}
```

### Pattern 2: K-Way Merge

```java
// Merge K sorted arrays
public List<Integer> mergeKSortedArrays(List<List<Integer>> arrays) {
    List<Integer> result = new ArrayList<>();
    
    // Min-heap stores (value, arrayIndex, elementIndex)
    PriorityQueue<int[]> minHeap = new PriorityQueue<>(
        (a, b) -> a[0] - b[0]  // Compare by value
    );
    
    // Initialize: add first element from each array
    for (int i = 0; i < arrays.size(); i++) {
        if (!arrays.get(i).isEmpty()) {
            minHeap.offer(new int[]{arrays.get(i).get(0), i, 0});
        }
    }
    
    // Extract min and add next from same array
    while (!minHeap.isEmpty()) {
        int[] current = minHeap.poll();
        int value = current[0];
        int arrayIdx = current[1];
        int elemIdx = current[2];
        
        result.add(value);
        
        // Add next element from same array
        if (elemIdx + 1 < arrays.get(arrayIdx).size()) {
            minHeap.offer(new int[]{
                arrays.get(arrayIdx).get(elemIdx + 1),
                arrayIdx,
                elemIdx + 1
            });
        }
    }
    
    return result;
}

// Time: O(N log k) where N = total elements, k = number of arrays
// Space: O(k) for heap

// Why this works:
// Heap always contains smallest unprocessed element from each array
// Extract min → add next from same array → repeat
// Result is merged sorted list
```

### Pattern 3: Median of Data Stream

```java
// Maintain median of running stream of numbers
public class MedianFinder {
    // Max-heap for smaller half
    private PriorityQueue<Integer> maxHeap;
    
    // Min-heap for larger half
    private PriorityQueue<Integer> minHeap;
    
    public MedianFinder() {
        maxHeap = new PriorityQueue<>(Collections.reverseOrder());
        minHeap = new PriorityQueue<>();
    }
    
    public void addNum(int num) {
        // Add to max-heap first
        maxHeap.offer(num);
        
        // Balance: ensure maxHeap.peek() <= minHeap.peek()
        minHeap.offer(maxHeap.poll());
        
        // Balance sizes: maxHeap.size() >= minHeap.size()
        if (maxHeap.size() < minHeap.size()) {
            maxHeap.offer(minHeap.poll());
        }
    }
    
    public double findMedian() {
        if (maxHeap.size() > minHeap.size()) {
            return maxHeap.peek();  // Odd total, max-heap has extra
        }
        return (maxHeap.peek() + minHeap.peek()) / 2.0;  // Even total
    }
    
    // Time: O(log n) per insertion
    // Space: O(n)
}

// Visual:
// Stream: [1, 2, 3, 4, 5]
//
// After 1: maxHeap=[1], minHeap=[], median=1
// After 2: maxHeap=[1], minHeap=[2], median=1.5
// After 3: maxHeap=[2,1], minHeap=[3], median=2
// After 4: maxHeap=[2,1], minHeap=[3,4], median=2.5
// After 5: maxHeap=[3,2,1], minHeap=[4,5], median=3

// Key insight: 
// Max-heap stores smaller half (can access max easily)
// Min-heap stores larger half (can access min easily)
// Median is at boundary between heaps
```

### Pattern 4: Meeting Rooms / Interval Scheduling

```java
// Minimum meeting rooms needed for intervals
public int minMeetingRooms(int[][] intervals) {
    if (intervals.length == 0) return 0;
    
    // Sort by start time
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
    
    // Min-heap stores end times of ongoing meetings
    PriorityQueue<Integer> endTimes = new PriorityQueue<>();
    
    for (int[] interval : intervals) {
        int start = interval[0];
        int end = interval[1];
        
        // If earliest ending meeting finished, reuse that room
        if (!endTimes.isEmpty() && endTimes.peek() <= start) {
            endTimes.poll();
        }
        
        // Add current meeting's end time
        endTimes.offer(end);
    }
    
    // Heap size = number of concurrent meetings = rooms needed
    return endTimes.size();
}

// Example: [[0,30],[5,10],[15,20]]
// Sort: [[0,30],[5,10],[15,20]]
//
// Process [0,30]: endTimes=[30], rooms=1
// Process [5,10]: 5<30, can't reuse, endTimes=[10,30], rooms=2
// Process [15,20]: 10<=15, reuse room, endTimes=[20,30], rooms=2
//
// Result: 2 rooms needed

// Time: O(n log n) for sorting + O(n log n) for heap = O(n log n)
// Space: O(n) for heap
```

### Pattern 5: Task Scheduler

```java
// Schedule tasks with cooldown period
public int leastInterval(char[] tasks, int n) {
    // Count frequency of each task
    Map<Character, Integer> freq = new HashMap<>();
    for (char task : tasks) {
        freq.put(task, freq.getOrDefault(task, 0) + 1);
    }
    
    // Max-heap by frequency
    PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
    maxHeap.addAll(freq.values());
    
    int cycles = 0;
    
    while (!maxHeap.isEmpty()) {
        List<Integer> temp = new ArrayList<>();
        
        // Try to schedule n+1 tasks (one slot per unique task)
        for (int i = 0; i <= n; i++) {
            if (!maxHeap.isEmpty()) {
                int count = maxHeap.poll();
                if (count > 1) {
                    temp.add(count - 1);
                }
            }
        }
        
        // Put remaining tasks back
        maxHeap.addAll(temp);
        
        // If heap empty, last cycle can be shorter
        cycles += maxHeap.isEmpty() ? temp.size() + 1 : n + 1;
    }
    
    return cycles;
}

// Example: tasks = ['A','A','A','B','B','B'], n = 2
// Frequency: A=3, B=3
// Heap: [3, 3]
//
// Cycle 1: Schedule A, B, idle → [2, 2], cycles=3
// Cycle 2: Schedule A, B, idle → [1, 1], cycles=6
// Cycle 3: Schedule A, B → [], cycles=8
//
// Result: 8 intervals needed
```

---

## Real-World Applications

### Application 1: Operating System Task Scheduler

```java
// CPU schedules processes by priority
public class CPUScheduler {
    private PriorityQueue<Process> readyQueue;
    
    class Process {
        int pid;
        int priority;
        int burstTime;
        
        Process(int pid, int priority, int burstTime) {
            this.pid = pid;
            this.priority = priority;
            this.burstTime = burstTime;
        }
    }
    
    public CPUScheduler() {
        // Higher priority = smaller number (Unix convention)
        readyQueue = new PriorityQueue<>((a, b) -> a.priority - b.priority);
    }
    
    public void addProcess(Process p) {
        readyQueue.offer(p);  // O(log n)
    }
    
    public Process getNextProcess() {
        return readyQueue.poll();  // O(log n), always highest priority
    }
}

// Real usage: Linux kernel uses heaps for process scheduling
```

### Application 2: Dijkstra's Shortest Path

```java
// Find shortest path in weighted graph using heap
public int[] dijkstra(int[][] graph, int start) {
    int n = graph.length;
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[start] = 0;
    
    // Min-heap: (distance, node)
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
    pq.offer(new int[]{0, start});
    
    while (!pq.isEmpty()) {
        int[] current = pq.poll();
        int d = current[0];
        int u = current[1];
        
        if (d > dist[u]) continue;  // Already processed with shorter path
        
        // Relax edges
        for (int v = 0; v < n; v++) {
            if (graph[u][v] > 0) {  // Edge exists
                int newDist = dist[u] + graph[u][v];
                if (newDist < dist[v]) {
                    dist[v] = newDist;
                    pq.offer(new int[]{newDist, v});
                }
            }
        }
    }
    
    return dist;
}

// Heap ensures we always process closest unvisited node
// Time: O((V + E) log V) instead of O(V²) with array
```

### Application 3: Huffman Coding (Data Compression)

```java
// Build optimal prefix-free code
public Node buildHuffmanTree(Map<Character, Integer> freq) {
    // Min-heap by frequency
    PriorityQueue<Node> pq = new PriorityQueue<>(
        (a, b) -> a.freq - b.freq
    );
    
    // Add all characters as leaf nodes
    for (Map.Entry<Character, Integer> entry : freq.entrySet()) {
        pq.offer(new Node(entry.getKey(), entry.getValue()));
    }
    
    // Build tree bottom-up
    while (pq.size() > 1) {
        Node left = pq.poll();
        Node right = pq.poll();
        
        Node parent = new Node('\0', left.freq + right.freq);
        parent.left = left;
        parent.right = right;
        
        pq.offer(parent);
    }
    
    return pq.poll();  // Root of Huffman tree
}

// Heap ensures we always merge lowest-frequency nodes
// Result: Optimal compression (most frequent = shortest code)
```

---

## Heap vs Other Structures

| Operation | Min-Heap | Balanced BST | Sorted Array |
|-----------|----------|--------------|--------------|
| Insert | O(log n) | O(log n) | O(n) |
| Get Min | O(1) | O(log n) | O(1) |
| Extract Min | O(log n) | O(log n) | O(n) |
| Delete Arbitrary | O(n) | O(log n) | O(n) |
| Search | O(n) | O(log n) | O(log n) |
| Build from Array | O(n) | O(n log n) | O(n log n) |
| Space | O(n) | O(n) | O(n) |

**Use Heap when**:
- Need quick access to min/max
- Don't need full sorted order
- Priority-based processing
- Top K problems
- Streaming data

**Use BST when**:
- Need sorted iteration
- Range queries
- Frequent arbitrary deletions
- Need to find any element quickly

**Use Sorted Array when**:
- Static data (no modifications)
- Need binary search
- Space efficiency critical
- Small datasets

---

## Complexity Analysis

### Time Complexity

```java
// Core operations:
insert()       // O(log n) - bubble up height of tree
extractMin()   // O(log n) - bubble down height of tree
peek()         // O(1) - just return root
heapify()      // O(n) - build heap from array
remove(elem)   // O(n) - search + O(log n) removal

// Common patterns:
Top K          // O(n log k) - k-sized heap for n elements
K-way merge    // O(N log k) - N elements, k arrays
Median stream  // O(log n) per insert - two heaps
```

### Space Complexity

```java
// Array-based heap: O(n)
// No extra space for pointers
// Just the array itself

// Priority queue with custom objects: O(n)
// n objects + heap structure

// Heap sort: O(1) extra space
// In-place using input array as heap
```

---

## Common Pitfalls

### ❌ Anti-Pattern 1: Using Heap for Searching

```java
// WRONG: Heap is not for searching arbitrary elements
public boolean contains(int value) {
    for (int i = 0; i < size; i++) {
        if (heap[i] == value) return true;  // O(n) - no better than array!
    }
    return false;
}

// ✅ Use HashMap or BST for searching
```

### ❌ Anti-Pattern 2: Forgetting Heap Only Guarantees Root

```java
// WRONG: Assuming heap is sorted
int secondSmallest = heap[1];  // NOT guaranteed to be 2nd smallest!
// heap[1] could be larger than heap[2]

// ✅ Extract min twice
int min1 = heap.poll();
int min2 = heap.poll();
```

### ❌ Anti-Pattern 3: Max-Heap for Top K Largest

```java
// WRONG: Using max-heap for K largest
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
// This keeps K largest at top, but we lose smallest of K largest

// ✅ Use min-heap for K largest (counterintuitive but correct)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
// Min at top = smallest of K largest = threshold for inclusion
```

---

## Interview Tips

1. **Ask about duplicates**: Can heap contain duplicate values?
2. **Clarify min vs max**: Which extremum is needed?
3. **Consider heap size**: Top K problems use K-sized heap, not N-sized
4. **Remember O(n) heapify**: Building heap is O(n), not O(n log n)
5. **Two-heap pattern**: Median finding, sliding window median
6. **Custom comparators**: Know how to define priority for objects

## Quick Reference

```java
// Java PriorityQueue
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());

// Operations
minHeap.offer(value);      // Insert
minHeap.peek();            // Get min (don't remove)
minHeap.poll();            // Extract min
minHeap.size();            // Size
minHeap.isEmpty();         // Check empty

// Top K largest: use min-heap of size K
// Top K smallest: use max-heap of size K
// Median: use two heaps (max-heap for lower half, min-heap for upper half)
// K-way merge: use min-heap with (value, sourceIndex)

// Time: Insert O(log n), Extract O(log n), Peek O(1)
// Space: O(n)
```
