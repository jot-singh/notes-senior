# Linked Lists

## What You'll Learn
- Singly and doubly linked list design and operations
- When linked lists are better than arrays
- Advanced pointer manipulation techniques (fast/slow pointers)
- Real-world applications in system design
- Problem patterns that naturally use linked lists

## Why This Matters

Linked lists solve problems that arrays struggle with: **dynamic sizing without pre-allocation, O(1) insertion/deletion at any known position, and breaking memory locality for specialized use cases**. Understanding linked lists deeply teaches pointer manipulation, which is foundational for graphs, complex data structures, and understanding system memory.

---

## Linked List Fundamentals

### Structure: Node-Based Sequential Storage

Unlike arrays with contiguous memory, linked lists use nodes pointing to each other:

```java
// Singly Linked List Node
public class ListNode {
    int val;
    ListNode next;
    
    ListNode(int val) {
        this.val = val;
        this.next = null;
    }
}

// Doubly Linked List Node - can traverse both directions
public class DoublyListNode {
    int val;
    DoublyListNode prev;
    DoublyListNode next;
    
    DoublyListNode(int val) {
        this.val = val;
    }
}
```

### When Linked Lists Win

| Operation | Array | Linked List | Trade-off |
|-----------|-------|-------------|-----------|
| Access element by index | O(1) | O(n) | Arrays much faster |
| Insert at position (with ref) | O(n) | O(1) | Linked lists much faster |
| Delete at position (with ref) | O(n) | O(1) | Linked lists much faster |
| Space for N elements | O(n) | O(n) | Linked list has more overhead |
| Memory efficiency | Very efficient | Extra pointers (8-16 bytes) | Arrays more efficient |
| Dynamic resizing | O(n) amortized | O(1) per insertion | Linked lists better |

**Use Linked Lists When:**
- You need frequent insertion/deletion at known positions
- Size is unknown and grows unpredictably
- You want to avoid dynamic array resizing overhead
- You're implementing other data structures (stacks, queues)

**Avoid Linked Lists When:**
- You need random access (use arrays or hash maps)
- Memory is tight (pointers add overhead)
- Cache locality matters (CPU caches favor sequential memory)

---

## Core Patterns

### Pattern 1: Fast and Slow Pointers (Tortoise and Hare)

**When to use**: Detecting cycles, finding middle, finding kth element from end.

**The Intuition**: Move one pointer 1 step, another 2 steps. If they meet, cycle exists. If slow reaches end, no cycle.

```java
// Problem: Detect cycle in linked list
public boolean hasCycle(ListNode head) {
    ListNode slow = head;
    ListNode fast = head;
    
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        
        if (slow == fast) {
            return true;  // Cycle detected
        }
    }
    
    return false;
}

// Why it works: In a cycle, fast pointer "laps" slow pointer eventually
// Time: O(n), Space: O(1) - much better than using HashSet
```

### Pattern 2: Find Middle and Kth Element From End

```java
// Find middle of linked list
public ListNode findMiddle(ListNode head) {
    ListNode slow = head;
    ListNode fast = head;
    
    // Fast moves 2 steps, slow moves 1
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    
    return slow;  // Slow is at middle
}

// Find kth element from end
public ListNode findKthFromEnd(ListNode head, int k) {
    ListNode fast = head;
    ListNode slow = head;
    
    // Move fast k steps ahead
    for (int i = 0; i < k; i++) {
        fast = fast.next;
    }
    
    // Move both until fast reaches end
    while (fast != null) {
        slow = slow.next;
        fast = fast.next;
    }
    
    return slow;
}

// Time: O(n), Space: O(1)
```

**Why these work**: The pointer distance naturally gives us the position we want.

### Pattern 3: Reversal and Re-ordering

**When to use**: Checking palindromes, reversing lists, rearranging elements.

```java
// Reverse entire linked list
public ListNode reverse(ListNode head) {
    ListNode prev = null;
    ListNode current = head;
    
    while (current != null) {
        // Save next before we change the link
        ListNode nextTemp = current.next;
        current.next = prev;  // Reverse the link
        prev = current;       // Move prev forward
        current = nextTemp;   // Move current forward
    }
    
    return prev;  // New head
}

// Problem: Check if palindrome using reversal
public boolean isPalindrome(ListNode head) {
    if (head == null || head.next == null) return true;
    
    // Find middle
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    
    // Reverse second half
    ListNode reversed = reverse(slow);
    
    // Compare first half with reversed second half
    ListNode node1 = head;
    ListNode node2 = reversed;
    while (node2 != null) {
        if (node1.val != node2.val) return false;
        node1 = node1.next;
        node2 = node2.next;
    }
    
    return true;
}

// Time: O(n), Space: O(1)
```

### Pattern 4: Merge Two Sorted Lists

**When to use**: Building merge sort for lists, combining data sources.

```java
// Merge two sorted linked lists
public ListNode mergeTwoLists(ListNode list1, ListNode list2) {
    // Dummy node makes the code simpler - no special case for head
    ListNode dummy = new ListNode(0);
    ListNode current = dummy;
    
    while (list1 != null && list2 != null) {
        if (list1.val <= list2.val) {
            current.next = list1;
            list1 = list1.next;
        } else {
            current.next = list2;
            list2 = list2.next;
        }
        current = current.next;
    }
    
    // Attach remaining elements
    current.next = (list1 != null) ? list1 : list2;
    
    return dummy.next;
}

// Time: O(n + m), Space: O(1)
```

### Pattern 5: Two-Pointer Pairs

**When to use**: Finding pairs with target sum, removing duplicates, sorting.

```java
// Remove duplicates from sorted list
public ListNode deleteDuplicates(ListNode head) {
    ListNode current = head;
    
    while (current != null && current.next != null) {
        if (current.val == current.next.val) {
            current.next = current.next.next;  // Skip duplicate
        } else {
            current = current.next;
        }
    }
    
    return head;
}

// Reorder list: 1->2->3->4 becomes 1->4->2->3
public void reorderList(ListNode head) {
    if (head == null || head.next == null) return;
    
    // Find middle
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    
    // Reverse second half
    ListNode reversed = reverse(slow);
    
    // Merge two halves alternately
    ListNode node1 = head;
    ListNode node2 = reversed;
    while (node2.next != null) {  // Important: node2.next, not node2
        ListNode temp1 = node1.next;
        ListNode temp2 = node2.next;
        
        node1.next = node2;
        node2.next = temp1;
        
        node1 = temp1;
        node2 = temp2;
    }
}

// Time: O(n), Space: O(1)
```

---

## Real-World Applications

### 1. **LRU Cache Implementation**

Most important application: caching with eviction policy.

```java
public class LRUCache {
    private class Node {
        int key, value;
        Node prev, next;
        Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }
    
    private int capacity;
    private Map<Integer, Node> cache;
    private Node head, tail;  // Dummy nodes for easier manipulation
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();
        this.head = new Node(0, 0);
        this.tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
    }
    
    public int get(int key) {
        if (!cache.containsKey(key)) return -1;
        
        Node node = cache.get(key);
        moveToHead(node);  // Mark as recently used
        return node.value;
    }
    
    public void put(int key, int value) {
        if (cache.containsKey(key)) {
            // Update existing
            Node node = cache.get(key);
            node.value = value;
            moveToHead(node);
        } else {
            // Add new
            if (cache.size() == capacity) {
                // Remove least recently used (just before tail)
                Node lru = tail.prev;
                removeNode(lru);
                cache.remove(lru.key);
            }
            
            Node newNode = new Node(key, value);
            addToHead(newNode);
            cache.put(key, newNode);
        }
    }
    
    private void moveToHead(Node node) {
        removeNode(node);
        addToHead(node);
    }
    
    private void removeNode(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
    
    private void addToHead(Node node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
}

// Get/Put: O(1) average
// Space: O(capacity)
// Used in: CPU caches, database buffering, web caches
```

### 2. **Undo/Redo Implementation**

Linked lists naturally support forward and backward navigation.

```java
public class UndoRedoSystem {
    private class Action {
        String content;
        Action prev, next;
        Action(String content) {
            this.content = content;
        }
    }
    
    private Action current;
    
    public void perform(String content) {
        Action newAction = new Action(content);
        
        // Forget any "redo" actions if we're not at the end
        newAction.prev = current;
        newAction.next = null;
        
        if (current != null) {
            current.next = null;  // Clear redo history
        }
        
        current = newAction;
    }
    
    public String undo() {
        if (current == null || current.prev == null) {
            return "Cannot undo";
        }
        current = current.prev;
        return current.content;
    }
    
    public String redo() {
        if (current == null || current.next == null) {
            return "Cannot redo";
        }
        current = current.next;
        return current.content;
    }
}

// Used in: Text editors, design software, git
```

### 3. **Browser History**

```java
public class BrowserHistory {
    private class Page {
        String url;
        Page prev, next;
        Page(String url) { this.url = url; }
    }
    
    private Page current;
    
    public BrowserHistory(String homepage) {
        current = new Page(homepage);
    }
    
    public void visit(String url) {
        Page newPage = new Page(url);
        newPage.prev = current;
        current.next = null;  // Clear forward history
        current = newPage;
    }
    
    public String back(int steps) {
        while (steps > 0 && current.prev != null) {
            current = current.prev;
            steps--;
        }
        return current.url;
    }
    
    public String forward(int steps) {
        while (steps > 0 && current.next != null) {
            current = current.next;
            steps--;
        }
        return current.url;
    }
}
```

### 4. **Blockchain Implementation**

Linked list structure naturally fits blockchain where each block points to previous.

```java
public class Blockchain {
    private class Block {
        int index;
        long timestamp;
        String data;
        String prevHash;
        String hash;
        Block next;
        
        Block(int index, String data, String prevHash) {
            this.index = index;
            this.timestamp = System.currentTimeMillis();
            this.data = data;
            this.prevHash = prevHash;
            this.hash = calculateHash();
        }
        
        private String calculateHash() {
            return Integer.toHexString((index + timestamp + data + prevHash).hashCode());
        }
    }
    
    private Block genesis;
    private Block current;
    
    public Blockchain() {
        genesis = new Block(0, "Genesis", "0");
        current = genesis;
    }
    
    public void addBlock(String data) {
        Block newBlock = new Block(
            current.index + 1,
            data,
            current.hash
        );
        current.next = newBlock;
        current = newBlock;
    }
    
    public boolean isValid() {
        Block node = genesis;
        while (node.next != null) {
            if (!node.next.prevHash.equals(node.hash)) {
                return false;
            }
            node = node.next;
        }
        return true;
    }
}
```

---

## Doubly Linked Lists vs Singly

### When to use Doubly Linked Lists

```java
// Doubly linked lists enable O(1) removal from middle when you have the node
// Singly linked lists require finding previous node: O(n)

// Perfect for: LRU cache, music playlists (previous/next track)

public class DoublyLinkedList {
    private DoublyListNode removeAndReturnNext(DoublyListNode node) {
        // O(1) - we have both neighbors
        DoublyListNode prevNode = node.prev;
        DoublyListNode nextNode = node.next;
        
        prevNode.next = nextNode;
        nextNode.prev = prevNode;
        
        return nextNode;
    }
}
```

**Trade-off**: Doubly linked lists use more memory (extra pointer) but enable faster operations.

---

## Common Patterns Checklist

When you see a linked list problem:

1. **Cycle involved?** → Fast/slow pointers
2. **Need to find middle?** → Fast/slow pointers
3. **Merging/combining?** → Dummy node + two-pointer merge
4. **Reversing or rearranging?** → Pointer manipulation
5. **Frequent add/remove at ends?** → Deque or circular linked list
6. **Need O(1) access to middle?** → Doubly linked list

---

## Common Mistakes (Anti-Patterns)

### ❌ Losing Reference to Nodes
```java
// Wrong: losing reference
ListNode temp = node.next;
node.next = newNode;
// node.next.next is lost!

// ✅ Correct: save before modifying
ListNode nextTemp = node.next;
node.next = newNode;
newNode.next = nextTemp;
```

### ❌ Forgetting Null Checks
```java
// ❌ Will throw NPE
while (current.next != null) {
    current = current.next;
}
// Never check if current itself is null

// ✅ Always check
while (current != null && current.next != null) {
    current = current.next;
}
```

### ❌ Not Using Dummy Nodes
```java
// Complex code with special case for head
if (head.val == target) {
    head = head.next;
    return head;
}
ListNode current = head;
while (current.next != null) {
    if (current.next.val == target) {
        current.next = current.next.next;
    } else {
        current = current.next;
    }
}

// ✅ Simpler with dummy node
ListNode dummy = new ListNode(0);
dummy.next = head;
ListNode current = dummy;
while (current.next != null) {
    if (current.next.val == target) {
        current.next = current.next.next;
    } else {
        current = current.next;
    }
}
return dummy.next;
```

---

## Complexity Analysis

| Operation | Singly LL | Doubly LL | Notes |
|-----------|-----------|-----------|-------|
| Access by value | O(n) | O(n) | Must traverse |
| Insert at known position | O(1) | O(1) | If you have pointer |
| Delete at known position | O(1) | O(1) | If you have pointer |
| Find previous | O(n) | O(1) | Doubly has prev |
| Reverse | O(n) | O(n) | Must visit each node |
| Detect cycle | O(n) | O(n) | Fast/slow works both |

---

## Next Steps

- Practice fast/slow pointer problems extensively
- Build LRU cache from scratch multiple times
- Understand memory layout differences between arrays and linked lists
- Move to [Stacks and Queues](03-stacks-queues.md) - both are built on linked lists
