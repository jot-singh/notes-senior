# Stacks and Queues

## What You'll Learn
- LIFO (Last-In-First-Out) stack semantics and patterns
- FIFO (First-In-First-Out) queue semantics and variations
- Monotonic stacks and deques for optimization
- Real-world use cases in system design
- Recognizing when a problem needs stack/queue structure

## Why This Matters

Stacks and queues are **restricted-access linear data structures** that enforce discipline in how you access elements. This constraint is powerful: it makes certain problems trivial (expression evaluation, task scheduling) and enables optimal solutions (stock span, sliding window max). Understanding when to apply them separates good solutions from great ones.

---

## Stack Fundamentals

### LIFO Principle: Last In, First Out

```java
public interface Stack<T> {
    void push(T item);      // O(1) - add to top
    T pop();                // O(1) - remove from top
    T peek();               // O(1) - view top without removing
    boolean isEmpty();      // O(1)
    int size();             // O(1)
}

// Array-based implementation
public class ArrayStack<T> {
    private List<T> items = new ArrayList<>();
    
    public void push(T item) { items.add(item); }
    public T pop() { return items.remove(items.size() - 1); }
    public T peek() { return items.get(items.size() - 1); }
    public boolean isEmpty() { return items.isEmpty(); }
}

// Linked list-based implementation
public class LinkedStack<T> {
    private class Node { T val; Node next; }
    private Node top = null;
    
    public void push(T item) {
        Node newNode = new Node();
        newNode.val = item;
        newNode.next = top;
        top = newNode;
    }
    
    public T pop() {
        T val = top.val;
        top = top.next;
        return val;
    }
}
```

### When Stacks Are Perfect

Stacks are ideal for problems requiring **backtracking or nested structure handling**:
- Expression evaluation (parentheses matching, postfix notation)
- Function call stack (recursion, execution context)
- Undo/redo operations
- DFS (depth-first search) in graphs
- Parsing and compilation

---

## Core Stack Patterns

### Pattern 1: Parentheses/Bracket Matching

**When to use**: Validating nested structures.

```java
// Problem: Validate balanced parentheses
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

// Time: O(n), Space: O(n) for stack
// Why: Each character is processed once; stack size ≤ n/2
```

**Intuition**: When you see opening bracket, push. When you see closing bracket, it must match most recent opening.

### Pattern 2: Expression Evaluation

**When to use**: Converting between infix, postfix, prefix notations or evaluating expressions.

```java
// Postfix notation evaluation: "3 4 + 2 *" = (3+4)*2 = 14
public int evaluatePostfix(String[] tokens) {
    Stack<Integer> stack = new Stack<>();
    Set<String> operators = Set.of("+", "-", "*", "/");
    
    for (String token : tokens) {
        if (operators.contains(token)) {
            int b = stack.pop();
            int a = stack.pop();
            
            int result = switch(token) {
                case "+" -> a + b;
                case "-" -> a - b;
                case "*" -> a * b;
                case "/" -> a / b;
                default -> 0;
            };
            
            stack.push(result);
        } else {
            stack.push(Integer.parseInt(token));
        }
    }
    
    return stack.pop();
}

// Time: O(n), Space: O(n)
```

**Why this works**: Postfix evaluation is natural for stacks - operands are on stack, operator pops 2 and pushes result.

### Pattern 3: Monotonic Stack

**When to use**: Finding next/previous greater/smaller element, tracking maximum in ranges.

**The Intuition**: Maintain stack in increasing/decreasing order. When you see an element:
- If it violates order, pop and process what you pop
- The element you can't pop is your "next greater/smaller"

```java
// Problem: Find next greater element for each element
// Input: [2, 1, 2, 4, 3]
// Output: [4, 2, 4, -1, -1]

public int[] nextGreaterElement(int[] nums) {
    int[] result = new int[nums.length];
    Arrays.fill(result, -1);
    Stack<Integer> stack = new Stack<>();
    
    // Traverse right to left
    for (int i = nums.length - 1; i >= 0; i--) {
        // Pop smaller elements - they won't be anyone's answer
        while (!stack.isEmpty() && stack.peek() <= nums[i]) {
            stack.pop();
        }
        
        // What's left on stack is next greater
        if (!stack.isEmpty()) {
            result[i] = stack.peek();
        }
        
        // Current becomes candidate for next greater
        stack.push(nums[i]);
    }
    
    return result;
}

// Time: O(n), Space: O(n)
// Why: Each element pushed once and popped once = O(n) total
```

**Example walkthrough**:
```
nums = [2, 1, 2, 4, 3]

i=4 (nums[4]=3): stack empty, push 3 → result[4]=-1, stack=[3]
i=3 (nums[3]=4): 4 > 3, pop 3 → stack empty, push 4 → result[3]=-1, stack=[4]
i=2 (nums[2]=2): 2 < 4, push 2 → result[2]=4, stack=[4,2]
i=1 (nums[1]=1): 1 < 2, push 1 → result[1]=2, stack=[4,2,1]
i=0 (nums[0]=2): 2 > 1, pop 1 → 2 > 2? no, push 2 → result[0]=4, stack=[4,2,2]
```

### Pattern 4: Daily Temperatures (Monotonic Stack Application)

```java
// Problem: For each day, find how many days until warmer temperature
// Input: [73, 74, 75, 71, 69, 72, 76, 73]
// Output: [1, 1, 4, 2, 1, 1, 0, 0]

public int[] dailyTemperatures(int[] temps) {
    int[] result = new int[temps.length];
    Stack<Integer> stack = new Stack<>();  // Stores indices
    
    for (int i = 0; i < temps.length; i++) {
        // Pop all days with lower temperature
        while (!stack.isEmpty() && temps[i] > temps[stack.peek()]) {
            int idx = stack.pop();
            result[idx] = i - idx;  // Days until warmer
        }
        
        stack.push(i);
    }
    
    return result;
}

// Time: O(n), Space: O(n)
```

---

## Queue Fundamentals

### FIFO Principle: First In, First Out

```java
public interface Queue<T> {
    void enqueue(T item);   // O(1) - add to rear
    T dequeue();            // O(1) - remove from front
    T peek();               // O(1) - view front
    boolean isEmpty();
}

// Array-based (circular) - efficient
public class CircularQueue<T> {
    private T[] items;
    private int front = 0;
    private int rear = -1;
    private int size = 0;
    
    @SuppressWarnings("unchecked")
    public CircularQueue(int capacity) {
        items = new Object[capacity];
    }
    
    public void enqueue(T item) {
        rear = (rear + 1) % items.length;
        items[rear] = item;
        size++;
    }
    
    public T dequeue() {
        T item = items[front];
        front = (front + 1) % items.length;
        size--;
        return item;
    }
}

// Linked list-based - simpler but more memory
public class LinkedQueue<T> {
    private class Node { T val; Node next; }
    private Node front = null;
    private Node rear = null;
    
    public void enqueue(T item) {
        Node newNode = new Node();
        newNode.val = item;
        
        if (rear == null) {
            front = rear = newNode;
        } else {
            rear.next = newNode;
            rear = newNode;
        }
    }
    
    public T dequeue() {
        T val = front.val;
        front = front.next;
        if (front == null) rear = null;
        return val;
    }
}
```

### When Queues Are Perfect

Queues model **sequential processing and scheduling**:
- Task scheduling (job queues, thread pools)
- BFS (breadth-first search)
- Stream processing
- Message passing between components
- Resource allocation (printer queue, CPU scheduling)

---

## Core Queue Patterns

### Pattern 1: BFS (Breadth-First Search)

**When to use**: Finding shortest path, level-order traversal, connected components.

```java
// Problem: Level-order traversal of binary tree
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;
    
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    
    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> level = new ArrayList<>();
        
        // Process all nodes at current level
        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            level.add(node.val);
            
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        
        result.add(level);
    }
    
    return result;
}

// Time: O(n), Space: O(w) where w is max width
```

### Pattern 2: Producer-Consumer (Task Queue)

```java
// Real-world: Thread pool executor
public class TaskQueue {
    private Queue<Runnable> tasks = new LinkedList<>();
    private final int maxWorkers = 4;
    
    public synchronized void submit(Runnable task) {
        tasks.offer(task);
        notifyAll();  // Wake up worker threads
    }
    
    public void processTask() throws InterruptedException {
        while (true) {
            Runnable task;
            
            synchronized (this) {
                while (tasks.isEmpty()) {
                    wait();  // Wait for task
                }
                task = tasks.poll();
            }
            
            task.run();  // Execute task
        }
    }
}

// Used in: ThreadPoolExecutor, message queues, pub/sub systems
```

### Pattern 3: Deque (Double-Ended Queue)

**When to use**: Problems needing access from both ends, sliding window maximum.

```java
// Deque allows efficient add/remove from both ends
public interface Deque<T> {
    void addFirst(T item);
    void addLast(T item);
    T removeFirst();
    T removeLast();
    T getFirst();
    T getLast();
}

// Problem: Maximum in sliding window
// Input: [1, 3, -1, -3, 5, 3, 6, 7], k=3
// Output: [3, 3, 5, 5, 6, 7]

public int[] maxSlidingWindow(int[] nums, int k) {
    int[] result = new int[nums.length - k + 1];
    Deque<Integer> deque = new LinkedList<>();
    
    for (int i = 0; i < nums.length; i++) {
        // Remove elements outside window
        if (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
            deque.pollFirst();
        }
        
        // Remove smaller elements from back (they won't be max)
        while (!deque.isEmpty() && nums[deque.peekLast()] <= nums[i]) {
            deque.pollLast();
        }
        
        // Add current element
        deque.addLast(i);
        
        // Once window is full, add to result
        if (i >= k - 1) {
            result[i - k + 1] = nums[deque.peekFirst()];
        }
    }
    
    return result;
}

// Time: O(n), Space: O(k) for deque
// Monotonic deque: indices in increasing order of their values
```

---

## Real-World Applications

### 1. **Function Call Stack (Recursion)**

The runtime uses a stack to track function calls.

```java
// Each function call gets pushed, returns cause pop
public class RecursionVisualizer {
    private static int depth = 0;
    
    public void recursiveFunction(int n) {
        depth++;
        String indent = "  ".repeat(depth);
        System.out.println(indent + "→ Call: n=" + n);
        
        if (n <= 0) {
            System.out.println(indent + "← Base case");
            depth--;
            return;
        }
        
        recursiveFunction(n - 1);
        
        System.out.println(indent + "← Return: n=" + n);
        depth--;
    }
    
    // Call stack at deepest point:
    // recursiveFunction(3)
    //   recursiveFunction(2)
    //     recursiveFunction(1)
    //       recursiveFunction(0)
}
```

### 2. **Browser History (Stack)**

```java
public class BrowserNavigation {
    private Stack<String> backStack = new Stack<>();
    private Stack<String> forwardStack = new Stack<>();
    private String currentPage = "home";
    
    public void visit(String page) {
        backStack.push(currentPage);
        forwardStack.clear();  // Clear forward history
        currentPage = page;
    }
    
    public void back() {
        if (!backStack.isEmpty()) {
            forwardStack.push(currentPage);
            currentPage = backStack.pop();
        }
    }
    
    public void forward() {
        if (!forwardStack.isEmpty()) {
            backStack.push(currentPage);
            currentPage = forwardStack.pop();
        }
    }
}
```

### 3. **Message Queue (Queue)**

```java
// Real-world: RabbitMQ, Kafka pattern
public class MessageQueue {
    private Queue<Message> messages = new LinkedList<>();
    private final int MAX_SIZE = 1000;
    
    public synchronized void publish(Message msg) throws InterruptedException {
        while (messages.size() >= MAX_SIZE) {
            wait();  // Back pressure: wait if queue full
        }
        
        messages.offer(msg);
        notifyAll();  // Wake subscribers
    }
    
    public synchronized Message consume() throws InterruptedException {
        while (messages.isEmpty()) {
            wait();  // Block until message available
        }
        
        Message msg = messages.poll();
        notifyAll();  // Wake publishers if they were waiting
        return msg;
    }
}

// Used in: Task distribution, event processing, decoupling components
```

### 4. **Undo/Redo with Two Stacks**

```java
public class TextEditor {
    private Stack<String> undoStack = new Stack<>();
    private Stack<String> redoStack = new Stack<>();
    private StringBuilder text = new StringBuilder();
    
    public void type(String s) {
        undoStack.push(text.toString());
        redoStack.clear();  // Clear redo on new action
        text.append(s);
    }
    
    public void undo() {
        if (!undoStack.isEmpty()) {
            redoStack.push(text.toString());
            text = new StringBuilder(undoStack.pop());
        }
    }
    
    public void redo() {
        if (!redoStack.isEmpty()) {
            undoStack.push(text.toString());
            text = new StringBuilder(redoStack.pop());
        }
    }
}

// Time: O(n) where n is text length
// Used in: Any editor, design tool, versioning system
```

---

## Stack vs Queue Decision Tree

```
Do I need LIFO (last-in-first-out)?
├─ Yes → Stack
│   ├─ Matching pairs? → Simple stack
│   ├─ Finding next greater/smaller? → Monotonic stack
│   └─ Evaluating expression? → Stack
│
└─ No, I need FIFO (first-in-first-out) → Queue
    ├─ Shortest path problem? → BFS + queue
    ├─ Task scheduling? → Regular queue
    ├─ Access from both ends? → Deque
    └─ Need max/min in window? → Monotonic deque
```

---

## Common Patterns Checklist

When you see a stack/queue problem:

1. **Nested structures?** → Stack for matching/validation
2. **Next greater/smaller?** → Monotonic stack
3. **Shortest path?** → BFS with queue
4. **Sequential processing?** → Queue
5. **Need to track many levels?** → Stack for recursion
6. **Both ends access?** → Deque

---

## Common Mistakes (Anti-Patterns)

### ❌ Using Array.remove(0) for Queue Operations
```java
// ❌ O(n) because it shifts all elements
List<Integer> queue = new ArrayList<>();
queue.add(1);
queue.remove(0);  // Expensive!

// ✅ O(1) amortized
Queue<Integer> queue = new LinkedList<>();
queue.offer(1);
queue.poll();
```

### ❌ Forgetting to Check Size Before Level-Order Processing
```java
// ❌ Processes elements at wrong level
while (!queue.isEmpty()) {
    TreeNode node = queue.poll();
    // Miss the level information!
}

// ✅ Correct: capture size at start of level
while (!queue.isEmpty()) {
    int levelSize = queue.size();
    for (int i = 0; i < levelSize; i++) {
        TreeNode node = queue.poll();
        // Now at correct level
    }
}
```

### ❌ Stack Not Clear About Purpose
```java
// ❌ Unclear why stack is used
Stack<Integer> stack = new Stack<>();
for (int i = 0; i < arr.length; i++) {
    stack.push(i);
    // What pattern is this solving?
}

// ✅ Clear naming and commenting
Stack<Integer> indexStack = new Stack<>();  // Monotonic stack for next greater
for (int i = 0; i < arr.length; i++) {
    while (!indexStack.isEmpty() && arr[indexStack.peek()] < arr[i]) {
        result[indexStack.pop()] = arr[i];
    }
    indexStack.push(i);
}
```

---

## Complexity Reference

| Operation | Stack | Queue | Deque |
|-----------|-------|-------|-------|
| Push/Enqueue/AddFirst | O(1) | O(1) | O(1) |
| Pop/Dequeue/RemoveFirst | O(1) | O(1) | O(1) |
| Peek | O(1) | O(1) | O(1) |
| isEmpty | O(1) | O(1) | O(1) |
| Search | O(n) | O(n) | O(n) |

---

## Next Steps

- Master monotonic stacks - they appear frequently in optimal solutions
- Practice BFS extensively - fundamental for graph problems
- Understand deques deeply - they solve sliding window problems elegantly
- Move to [Trees](05-trees.md) where stacks/queues are essential for traversals
