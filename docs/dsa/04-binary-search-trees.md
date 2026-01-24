# Binary Search Trees (BST)

## What You'll Learn
- BST properties and why they enable efficient operations
- Insert, search, delete operations with complexity analysis
- BST validation techniques
- Common BST patterns and problem-solving strategies
- Self-balancing trees (AVL, Red-Black) concepts
- When to use BST vs other data structures

## Why This Matters

Binary Search Trees are fundamental to understanding ordered data structures. They power database indexes (B-trees), in-memory caches, file systems, and language runtime internals. Mastering BSTs means understanding trade-offs between search efficiency and structural complexity, and recognizing when tree-based solutions outperform arrays or hash tables.

---

## BST Properties

### The Core Invariant

```java
public class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
    
    TreeNode(int val) {
        this.val = val;
    }
}

// BST Property (CRITICAL):
// For every node N:
//   - ALL values in left subtree < N.val
//   - ALL values in right subtree > N.val
//   - Both left and right subtrees are also BSTs

//        50
//       /  \
//      30   70      Valid BST
//     / \   / \
//    20 40 60  80

//        50
//       /  \
//      30   70      INVALID - 65 < 70 but in left of 60
//     / \   / \
//    20 40 65  80
```

### Why BST Property Matters

```java
// The property enables binary search on a tree:
public boolean search(TreeNode root, int target) {
    if (root == null) return false;
    
    if (root.val == target) return true;
    
    // Decision: go left or right based on value
    if (target < root.val) {
        return search(root.left, target);   // Only search left
    } else {
        return search(root.right, target);  // Only search right
    }
}

// Time: O(h) where h is height
// Best case (balanced): O(log n)
// Worst case (skewed): O(n)
```

**Key insight**: Each comparison eliminates half the remaining search space (in balanced tree), just like binary search on sorted array.

---

## BST Operations

### Pattern 1: Search

```java
// Recursive search
public TreeNode searchRecursive(TreeNode root, int val) {
    if (root == null || root.val == val) {
        return root;
    }
    
    if (val < root.val) {
        return searchRecursive(root.left, val);
    }
    return searchRecursive(root.right, val);
}

// Iterative search (preferred - no stack overflow risk)
public TreeNode searchIterative(TreeNode root, int val) {
    TreeNode current = root;
    
    while (current != null && current.val != val) {
        if (val < current.val) {
            current = current.left;
        } else {
            current = current.right;
        }
    }
    
    return current;  // null if not found
}

// Time: O(h), Space: O(1) for iterative, O(h) for recursive
```

**When to use**: Membership testing, finding specific values, range queries.

### Pattern 2: Insert

```java
// Recursive insert
public TreeNode insert(TreeNode root, int val) {
    if (root == null) {
        return new TreeNode(val);  // Base case: found insertion point
    }
    
    if (val < root.val) {
        root.left = insert(root.left, val);
    } else if (val > root.val) {
        root.right = insert(root.right, val);
    }
    // Ignore duplicates (or handle based on requirements)
    
    return root;
}

// Iterative insert
public TreeNode insertIterative(TreeNode root, int val) {
    if (root == null) {
        return new TreeNode(val);
    }
    
    TreeNode current = root;
    TreeNode parent = null;
    
    // Find insertion point
    while (current != null) {
        parent = current;
        if (val < current.val) {
            current = current.left;
        } else if (val > current.val) {
            current = current.right;
        } else {
            return root;  // Duplicate, no insertion
        }
    }
    
    // Insert as child of parent
    if (val < parent.val) {
        parent.left = new TreeNode(val);
    } else {
        parent.right = new TreeNode(val);
    }
    
    return root;
}

// Time: O(h), Space: O(1) for iterative, O(h) for recursive
```

**Critical insight**: New nodes always inserted as leaves. Path from root to insertion point is determined by BST property.

### Pattern 3: Delete (Most Complex)

```java
public TreeNode delete(TreeNode root, int val) {
    if (root == null) return null;
    
    // Find node to delete
    if (val < root.val) {
        root.left = delete(root.left, val);
    } else if (val > root.val) {
        root.right = delete(root.right, val);
    } else {
        // Found node to delete - handle 3 cases
        
        // Case 1: Leaf node (no children)
        if (root.left == null && root.right == null) {
            return null;
        }
        
        // Case 2: One child
        if (root.left == null) {
            return root.right;  // Replace with right child
        }
        if (root.right == null) {
            return root.left;   // Replace with left child
        }
        
        // Case 3: Two children
        // Strategy: Replace with in-order successor (smallest in right subtree)
        // OR in-order predecessor (largest in left subtree)
        
        TreeNode successor = findMin(root.right);
        root.val = successor.val;  // Copy successor value
        root.right = delete(root.right, successor.val);  // Delete successor
    }
    
    return root;
}

private TreeNode findMin(TreeNode node) {
    while (node.left != null) {
        node = node.left;
    }
    return node;
}

private TreeNode findMax(TreeNode node) {
    while (node.right != null) {
        node = node.right;
    }
    return node;
}

// Alternative: Use predecessor instead
public TreeNode deleteWithPredecessor(TreeNode root, int val) {
    if (root == null) return null;
    
    if (val < root.val) {
        root.left = deleteWithPredecessor(root.left, val);
    } else if (val > root.val) {
        root.right = deleteWithPredecessor(root.right, val);
    } else {
        if (root.left == null) return root.right;
        if (root.right == null) return root.left;
        
        // Use predecessor (max of left subtree)
        TreeNode predecessor = findMax(root.left);
        root.val = predecessor.val;
        root.left = deleteWithPredecessor(root.left, predecessor.val);
    }
    
    return root;
}

// Time: O(h), Space: O(h)
```

**Why successor/predecessor works**:
- Successor is smallest value in right subtree → greater than all left values
- Predecessor is largest value in left subtree → smaller than all right values
- Either maintains BST property when replacing deleted node

---

## BST Validation

### Pattern: Validate BST

```java
// ❌ WRONG APPROACH - only checks immediate children
public boolean isValidBSTWrong(TreeNode root) {
    if (root == null) return true;
    
    if (root.left != null && root.left.val >= root.val) return false;
    if (root.right != null && root.right.val <= root.val) return false;
    
    return isValidBSTWrong(root.left) && isValidBSTWrong(root.right);
}

// Fails for:
//     5
//    / \
//   3   7
//  / \
// 1   6  <- 6 > 5 but in left subtree!

// ✅ CORRECT APPROACH - track valid range
public boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

private boolean validate(TreeNode node, long min, long max) {
    if (node == null) return true;
    
    // Check if current node violates bounds
    if (node.val <= min || node.val >= max) {
        return false;
    }
    
    // Left subtree: all values must be < node.val
    // Right subtree: all values must be > node.val
    return validate(node.left, min, node.val) &&
           validate(node.right, node.val, max);
}

// Alternative: In-order traversal should be strictly increasing
public boolean isValidBSTInOrder(TreeNode root) {
    List<Integer> inorder = new ArrayList<>();
    inOrderTraverse(root, inorder);
    
    for (int i = 1; i < inorder.size(); i++) {
        if (inorder.get(i) <= inorder.get(i - 1)) {
            return false;
        }
    }
    return true;
}

private void inOrderTraverse(TreeNode node, List<Integer> result) {
    if (node == null) return;
    inOrderTraverse(node.left, result);
    result.add(node.val);
    inOrderTraverse(node.right, result);
}

// Optimized in-order (no extra space for list)
private Integer prev = null;

public boolean isValidBSTOptimized(TreeNode root) {
    return inOrderValidate(root);
}

private boolean inOrderValidate(TreeNode node) {
    if (node == null) return true;
    
    if (!inOrderValidate(node.left)) return false;
    
    if (prev != null && node.val <= prev) return false;
    prev = node.val;
    
    return inOrderValidate(node.right);
}
```

**Key insight**: BST property is global, not local. Each node has valid range based on ancestors.

---

## Common BST Patterns

### Pattern 1: Find Kth Smallest/Largest

```java
// Kth smallest = in-order traversal (sorted), return kth element
public int kthSmallest(TreeNode root, int k) {
    Stack<TreeNode> stack = new Stack<>();
    TreeNode current = root;
    int count = 0;
    
    while (current != null || !stack.isEmpty()) {
        while (current != null) {
            stack.push(current);
            current = current.left;
        }
        
        current = stack.pop();
        count++;
        
        if (count == k) {
            return current.val;  // Found kth smallest
        }
        
        current = current.right;
    }
    
    return -1;
}

// Kth largest = reverse in-order (right-root-left)
public int kthLargest(TreeNode root, int k) {
    Stack<TreeNode> stack = new Stack<>();
    TreeNode current = root;
    int count = 0;
    
    while (current != null || !stack.isEmpty()) {
        while (current != null) {
            stack.push(current);
            current = current.right;  // Go right first
        }
        
        current = stack.pop();
        count++;
        
        if (count == k) {
            return current.val;
        }
        
        current = current.left;  // Then left
    }
    
    return -1;
}

// Time: O(k + h), Space: O(h)
```

### Pattern 2: Lowest Common Ancestor (LCA)

```java
// BST property makes LCA simple
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    // Both in left subtree
    if (p.val < root.val && q.val < root.val) {
        return lowestCommonAncestor(root.left, p, q);
    }
    
    // Both in right subtree
    if (p.val > root.val && q.val > root.val) {
        return lowestCommonAncestor(root.right, p, q);
    }
    
    // Split point = LCA
    return root;
}

// Iterative version
public TreeNode lowestCommonAncestorIterative(TreeNode root, TreeNode p, TreeNode q) {
    TreeNode current = root;
    
    while (current != null) {
        if (p.val < current.val && q.val < current.val) {
            current = current.left;
        } else if (p.val > current.val && q.val > current.val) {
            current = current.right;
        } else {
            return current;  // Split point
        }
    }
    
    return null;
}

// Time: O(h), Space: O(1) iterative
```

**Key insight**: In BST, LCA is the split point where nodes diverge into different subtrees.

### Pattern 3: Range Sum Query

```java
// Sum all values in range [L, R]
public int rangeSumBST(TreeNode root, int L, int R) {
    if (root == null) return 0;
    
    int sum = 0;
    
    // Include current node if in range
    if (L <= root.val && root.val <= R) {
        sum += root.val;
    }
    
    // Prune: only explore left if root.val > L
    if (root.val > L) {
        sum += rangeSumBST(root.left, L, R);
    }
    
    // Prune: only explore right if root.val < R
    if (root.val < R) {
        sum += rangeSumBST(root.right, L, R);
    }
    
    return sum;
}

// Time: O(n) worst case, but O(k + log n) in balanced BST where k = nodes in range
```

### Pattern 4: Convert Sorted Array to BST

```java
// Create balanced BST from sorted array
public TreeNode sortedArrayToBST(int[] nums) {
    return buildBST(nums, 0, nums.length - 1);
}

private TreeNode buildBST(int[] nums, int left, int right) {
    if (left > right) return null;
    
    int mid = left + (right - left) / 2;  // Middle element as root
    TreeNode root = new TreeNode(nums[mid]);
    
    root.left = buildBST(nums, left, mid - 1);
    root.right = buildBST(nums, mid + 1, right);
    
    return root;
}

// Time: O(n), Space: O(log n) for recursion stack
```

**Why this works**: Middle element ensures equal distribution of nodes in left/right subtrees → balanced tree.

### Pattern 5: Two-Sum in BST

```java
// Find two nodes that sum to target
public boolean findTarget(TreeNode root, int k) {
    Set<Integer> seen = new HashSet<>();
    return findTargetHelper(root, k, seen);
}

private boolean findTargetHelper(TreeNode node, int k, Set<Integer> seen) {
    if (node == null) return false;
    
    if (seen.contains(k - node.val)) {
        return true;
    }
    
    seen.add(node.val);
    
    return findTargetHelper(node.left, k, seen) ||
           findTargetHelper(node.right, k, seen);
}

// Alternative: In-order traversal + two pointers
public boolean findTargetTwoPointers(TreeNode root, int k) {
    List<Integer> inorder = new ArrayList<>();
    inOrderTraverse(root, inorder);
    
    int left = 0, right = inorder.size() - 1;
    
    while (left < right) {
        int sum = inorder.get(left) + inorder.get(right);
        if (sum == k) return true;
        if (sum < k) left++;
        else right--;
    }
    
    return false;
}

// Time: O(n), Space: O(n) for set or O(h) for recursion
```

---

## Self-Balancing Trees

### Why Balance Matters

```java
// Worst case: Inserting sorted data into BST
//     1           Time per operation:
//      \          Search: O(n)
//       2         Insert: O(n)
//        \        Delete: O(n)
//         3       
//          \      Tree degenerates to linked list!
//           4

// Balanced tree:
//        2        Time per operation:
//       / \       Search: O(log n)
//      1   3      Insert: O(log n)
//           \     Delete: O(log n)
//            4
```

### AVL Trees

**Property**: For every node, height difference between left and right subtrees ≤ 1.

```java
public class AVLNode {
    int val;
    int height;
    AVLNode left;
    AVLNode right;
    
    AVLNode(int val) {
        this.val = val;
        this.height = 1;
    }
}

public class AVLTree {
    // Get height (handles null)
    private int height(AVLNode node) {
        return node == null ? 0 : node.height;
    }
    
    // Balance factor
    private int getBalance(AVLNode node) {
        return node == null ? 0 : height(node.left) - height(node.right);
    }
    
    // Right rotation
    private AVLNode rotateRight(AVLNode y) {
        AVLNode x = y.left;
        AVLNode T2 = x.right;
        
        // Perform rotation
        x.right = y;
        y.left = T2;
        
        // Update heights
        y.height = Math.max(height(y.left), height(y.right)) + 1;
        x.height = Math.max(height(x.left), height(x.right)) + 1;
        
        return x;  // New root
    }
    
    // Left rotation
    private AVLNode rotateLeft(AVLNode x) {
        AVLNode y = x.right;
        AVLNode T2 = y.left;
        
        y.left = x;
        x.right = T2;
        
        x.height = Math.max(height(x.left), height(x.right)) + 1;
        y.height = Math.max(height(y.left), height(y.right)) + 1;
        
        return y;
    }
    
    // Insert with balancing
    public AVLNode insert(AVLNode node, int val) {
        // Standard BST insert
        if (node == null) {
            return new AVLNode(val);
        }
        
        if (val < node.val) {
            node.left = insert(node.left, val);
        } else if (val > node.val) {
            node.right = insert(node.right, val);
        } else {
            return node;  // Duplicates not allowed
        }
        
        // Update height
        node.height = 1 + Math.max(height(node.left), height(node.right));
        
        // Get balance factor
        int balance = getBalance(node);
        
        // Left-Left case
        if (balance > 1 && val < node.left.val) {
            return rotateRight(node);
        }
        
        // Right-Right case
        if (balance < -1 && val > node.right.val) {
            return rotateLeft(node);
        }
        
        // Left-Right case
        if (balance > 1 && val > node.left.val) {
            node.left = rotateLeft(node.left);
            return rotateRight(node);
        }
        
        // Right-Left case
        if (balance < -1 && val < node.right.val) {
            node.right = rotateRight(node.right);
            return rotateLeft(node);
        }
        
        return node;  // Already balanced
    }
}

// Guarantees: Height always O(log n)
// All operations: O(log n) guaranteed
```

### Red-Black Trees

**Properties**:
1. Each node is red or black
2. Root is black
3. All leaves (NIL) are black
4. Red node cannot have red children
5. All paths from node to leaves have same number of black nodes

```java
// Simpler to implement than AVL
// Slightly less balanced than AVL (max height = 2 log n)
// Faster insertion/deletion (fewer rotations)
// Used in: Java TreeMap/TreeSet, Linux kernel, C++ std::map

// Properties guarantee O(log n) height
// Insertion/deletion: O(log n) with at most 3 rotations
```

**When to use each**:
- **AVL**: More reads than writes, strict balance needed
- **Red-Black**: Mixed reads/writes, slightly relaxed balance acceptable
- **Standard BST**: Small datasets, random insertion order

---

## Real-World Applications

### Database Indexes (B-Trees)

```java
// B-trees are generalization of BST
// Each node has multiple keys and children
// Properties:
// - All leaves at same level
// - Node contains k keys → k+1 children
// - Minimum degree t: node has at least t-1 keys (except root)

// Why B-trees for databases:
// 1. Minimize disk I/O (one node = one disk block)
// 2. Wide trees (high branching factor) → shallow height
// 3. Bulk operations efficient
// 4. Good cache locality

// Example: MySQL InnoDB uses B+ tree
// - O(log n) search, insert, delete
// - Range queries efficient (leaf nodes linked)
```

### In-Memory Caches

```java
// TreeMap in Java (Red-Black tree)
public class CacheWithOrdering {
    private TreeMap<Integer, String> cache = new TreeMap<>();
    
    // Get entries in sorted order
    public List<String> getRange(int low, int high) {
        return new ArrayList<>(
            cache.subMap(low, high + 1).values()
        );
    }
    
    // Get minimum/maximum key
    public int getMinKey() {
        return cache.firstKey();
    }
    
    // All operations: O(log n)
}

// Use when:
// - Need sorted iteration
// - Range queries
// - Floor/ceiling operations
```

### Expression Trees

```java
// Represent arithmetic expressions
//       +
//      / \
//     *   3
//    / \
//   2   5

// Post-order traversal evaluates expression
public int evaluate(TreeNode node) {
    if (node == null) return 0;
    
    if (node.left == null && node.right == null) {
        return Integer.parseInt(node.val);  // Leaf = operand
    }
    
    int left = evaluate(node.left);
    int right = evaluate(node.right);
    
    switch (node.val) {
        case "+": return left + right;
        case "-": return left - right;
        case "*": return left * right;
        case "/": return left / right;
    }
    
    return 0;
}
```

---

## BST vs Other Structures

| Operation | BST (Balanced) | Hash Table | Sorted Array |
|-----------|----------------|------------|--------------|
| Search | O(log n) | O(1) avg | O(log n) |
| Insert | O(log n) | O(1) avg | O(n) |
| Delete | O(log n) | O(1) avg | O(n) |
| Min/Max | O(log n) | O(n) | O(1) |
| Range Query | O(k + log n) | O(n) | O(k + log n) |
| Sorted Iteration | O(n) | O(n log n) | O(n) |
| Space | O(n) | O(n) | O(n) |

**Use BST when**:
- Need sorted order
- Frequent range queries
- Floor/ceiling operations
- Dynamic dataset (frequent insertions/deletions)

**Use Hash Table when**:
- Only need exact lookups
- No ordering required
- Best average case performance

**Use Sorted Array when**:
- Static or rarely changing data
- Need constant-time min/max
- Space efficiency critical

---

## Common Pitfalls

### ❌ Anti-Pattern 1: Ignoring Balance

```java
// Inserting sorted data without balancing
BST tree = new BST();
for (int i = 1; i <= 1000; i++) {
    tree.insert(i);  // Creates linked list, O(n) operations
}

// ✅ Solution: Use self-balancing tree or shuffle data
List<Integer> numbers = IntStream.range(1, 1001)
    .boxed()
    .collect(Collectors.toList());
Collections.shuffle(numbers);
numbers.forEach(tree::insert);  // Better average case
```

### ❌ Anti-Pattern 2: Incorrect Validation

```java
// Only checking immediate children
public boolean isValidBST(TreeNode root) {
    if (root == null) return true;
    if (root.left != null && root.left.val >= root.val) return false;
    if (root.right != null && root.right.val <= root.val) return false;
    return isValidBST(root.left) && isValidBST(root.right);
}
// WRONG: Misses violations in deeper subtrees

// ✅ Use range validation or in-order traversal
```

### ❌ Anti-Pattern 3: Duplicate Handling

```java
// Silently ignoring duplicates without documenting
public void insert(int val) {
    // What happens with duplicates? Undefined!
}

// ✅ Be explicit
public void insert(int val) {
    // Duplicates not allowed, insertion ignored
    // OR: Store count, allow duplicates on right, etc.
}
```

---

## Interview Tips

1. **Always ask about duplicates**: How should BST handle duplicate values?
2. **Consider balance**: If insertion order unknown, assume worst-case O(n) for operations
3. **In-order traversal = sorted**: Most BST problems leverage this property
4. **Range validation**: BST validation requires tracking valid range, not just comparing with children
5. **Iterative vs recursive**: Know both; iterative avoids stack overflow for deep trees
6. **Use BST properties to prune**: In range queries, skip entire subtrees based on values

## Quick Reference

```java
// Create from sorted array → balanced BST
TreeNode balanced = sortedArrayToBST(sortedArray);

// Validate BST
boolean valid = validate(root, Long.MIN_VALUE, Long.MAX_VALUE);

// In-order → sorted sequence
List<Integer> sorted = inOrder(root);

// Kth smallest
int kth = kthSmallest(root, k);

// Range sum
int sum = rangeSumBST(root, low, high);

// LCA in BST
TreeNode lca = lowestCommonAncestor(root, p, q);
```

