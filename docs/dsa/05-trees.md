# Trees Fundamentals

## What You'll Learn
- Binary tree structure and properties
- Binary Search Trees (BST) and their advantages
- Tree traversal patterns (in-order, pre-order, post-order, level-order)
- How to recognize tree problems and which traversal to use
- Real-world applications in databases, file systems, and algorithms

## Why This Matters

Trees are **hierarchical data structures** that model relationships with parent-child semantics. They're everywhere: file systems, DOM trees, database indexes, expression parsing. Understanding trees deeply means understanding recursion, understanding optimal data organization, and being able to solve complex problems by recognizing patterns.

---

## Tree Fundamentals

### Basic Structure

```java
public class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
    
    TreeNode(int val) {
        this.val = val;
    }
}

// Key Properties of Trees:
// 1. Exactly one root node (entry point)
// 2. Each non-root node has exactly one parent
// 3. Acyclic (no cycles/loops)
// 4. Connected (path exists between any two nodes)
// 5. n nodes → n-1 edges
```

### Key Terminology

```
        A (root, height 0)
       / \
      B   C (height 1)
     / \   \
    D   E   F (height 2 - leaves at height 2)

Height:     Maximum distance from node to leaf
            A.height = 2, B.height = 1, D.height = 0

Depth:      Distance from root to node
            A.depth = 0, B.depth = 1, D.depth = 2

Balanced:   Height difference between subtrees ≤ 1
            This tree is balanced

Complete:   All levels full except possibly last
            Last level filled left-to-right

Full:       Every node has 0 or 2 children
            This tree is NOT full (C has 1 child)
```

### When Trees Are Perfect

Trees excel at:
- **Hierarchical data**: Organizational structures, file systems
- **Sorted data with fast search**: BSTs, balanced trees
- **Prefix problems**: Autocomplete (tries)
- **Expression parsing**: Compiler design
- **Decision-making**: Decision trees in ML

---

## Binary Search Trees (BST)

### Properties That Make BSTs Powerful

```java
public class BST {
    // Key property: For every node:
    // all left subtree values < node value
    // all right subtree values > node value
    
    //        50
    //       /  \
    //      30   70
    //     / \   / \
    //    20 40 60  80
    // This is a valid BST
    
    // The property enables:
    // - O(log n) search on average
    // - In-order traversal gives sorted sequence
    // - Efficient range queries
}
```

### BST Operations

```java
public class BST {
    private TreeNode root;
    
    // Search - O(log n) average, O(n) worst case
    public boolean search(int val) {
        return searchHelper(root, val);
    }
    
    private boolean searchHelper(TreeNode node, int val) {
        if (node == null) return false;
        
        if (val == node.val) return true;
        if (val < node.val) return searchHelper(node.left, val);
        return searchHelper(node.right, val);
    }
    
    // Insert - O(log n) average, O(n) worst case
    public void insert(int val) {
        root = insertHelper(root, val);
    }
    
    private TreeNode insertHelper(TreeNode node, int val) {
        if (node == null) return new TreeNode(val);
        
        if (val < node.val) {
            node.left = insertHelper(node.left, val);
        } else if (val > node.val) {
            node.right = insertHelper(node.right, val);
        }
        // Ignore duplicates
        
        return node;
    }
    
    // Delete - O(log n) average, O(n) worst case
    public void delete(int val) {
        root = deleteHelper(root, val);
    }
    
    private TreeNode deleteHelper(TreeNode node, int val) {
        if (node == null) return null;
        
        if (val < node.val) {
            node.left = deleteHelper(node.left, val);
        } else if (val > node.val) {
            node.right = deleteHelper(node.right, val);
        } else {
            // Found node to delete
            
            // Case 1: No children (leaf node)
            if (node.left == null && node.right == null) {
                return null;
            }
            
            // Case 2: One child
            if (node.left == null) return node.right;
            if (node.right == null) return node.left;
            
            // Case 3: Two children
            // Find in-order successor (smallest in right subtree)
            TreeNode minRight = findMin(node.right);
            node.val = minRight.val;
            node.right = deleteHelper(node.right, minRight.val);
        }
        
        return node;
    }
    
    private TreeNode findMin(TreeNode node) {
        while (node.left != null) {
            node = node.left;
        }
        return node;
    }
}
```

**Why in-order successor works for deletion**: The smallest value in the right subtree is greater than all left subtree values and smallest of right subtree, preserving BST property.

---

## Tree Traversal Patterns

### Pattern 1: In-Order Traversal (Left-Root-Right)

**When to use**: When you want sorted output from BST.

```java
// Returns elements in sorted order for BST
public void inOrder(TreeNode node) {
    if (node == null) return;
    
    inOrder(node.left);        // Left
    System.out.println(node.val);  // Root
    inOrder(node.right);       // Right
}

// Tree:     2          Output: 1 2 3
//          / \
//         1   3

// Iterative version using explicit stack:
public List<Integer> inOrderIterative(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Stack<TreeNode> stack = new Stack<>();
    TreeNode current = root;
    
    while (current != null || !stack.isEmpty()) {
        // Go to leftmost node
        while (current != null) {
            stack.push(current);
            current = current.left;
        }
        
        // Current is null, pop from stack
        current = stack.pop();
        result.add(current.val);
        
        // Visit right subtree
        current = current.right;
    }
    
    return result;
}
```

**Use case**: Validating BST, getting sorted sequence without explicit sort.

### Pattern 2: Pre-Order Traversal (Root-Left-Right)

**When to use**: When you want to process parent before children (copying tree, serialization).

```java
public void preOrder(TreeNode node) {
    if (node == null) return;
    
    System.out.println(node.val);  // Root
    preOrder(node.left);            // Left
    preOrder(node.right);           // Right
}

// Tree:     2          Output: 2 1 3
//          / \
//         1   3

// Practical: Create copy of tree
public TreeNode copyTree(TreeNode node) {
    if (node == null) return null;
    
    TreeNode copy = new TreeNode(node.val);  // Process root
    copy.left = copyTree(node.left);          // Copy left
    copy.right = copyTree(node.right);        // Copy right
    
    return copy;
}

// Used in: Serialization, copying structures, prefix expression evaluation
```

### Pattern 3: Post-Order Traversal (Left-Right-Root)

**When to use**: When you need to process children before parent (deletion, calculating tree properties).

```java
public void postOrder(TreeNode node) {
    if (node == null) return;
    
    postOrder(node.left);           // Left
    postOrder(node.right);          // Right
    System.out.println(node.val);   // Root
}

// Tree:     2          Output: 1 3 2
//          / \
//         1   3

// Practical: Calculate tree height
public int treeHeight(TreeNode node) {
    if (node == null) return 0;
    
    int leftHeight = treeHeight(node.left);    // Process left
    int rightHeight = treeHeight(node.right);  // Process right
    
    return 1 + Math.max(leftHeight, rightHeight);  // Use results
}

// Practical: Delete tree
public void deleteTree(TreeNode node) {
    if (node == null) return;
    
    deleteTree(node.left);    // Delete left
    deleteTree(node.right);   // Delete right
    // Now safe to delete node - no dangling pointers
}

// Used in: Calculation-based problems, deletion, postfix evaluation
```

### Pattern 4: Level-Order Traversal (BFS)

**When to use**: When you need to process nodes by depth, building level-by-level structure.

```java
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

// Tree:     1                Output: [[1], [2,3], [4,5]]
//          / \
//         2   3
//        /
//       4 5
```

---

## Core Tree Problem Patterns

### Pattern 1: Path Sum Problems

**When to use**: Finding paths with specific properties, validating paths.

```java
// Problem: Find root-to-leaf paths summing to target
public List<List<Integer>> pathSum(TreeNode root, int target) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(root, target, new ArrayList<>(), result);
    return result;
}

private void backtrack(TreeNode node, int remaining, 
                       List<Integer> path, List<List<Integer>> result) {
    if (node == null) return;
    
    // Add current node to path
    path.add(node.val);
    remaining -= node.val;
    
    // Check if leaf and path sum matches
    if (node.left == null && node.right == null && remaining == 0) {
        result.add(new ArrayList<>(path));
    }
    
    // Explore left and right
    backtrack(node.left, remaining, path, result);
    backtrack(node.right, remaining, path, result);
    
    // Backtrack: remove node for other paths
    path.remove(path.size() - 1);
}

// Time: O(n) to visit all, O(h) space for recursion
// Space: O(h) recursion depth + O(n) for result paths
```

### Pattern 2: Lowest Common Ancestor (LCA)

**When to use**: Finding relationships between nodes, tree comparison.

```java
// Problem: Find LCA of two nodes in BST
public TreeNode lowestCommonAncestorBST(TreeNode root, int p, int q) {
    // In BST, LCA is the first node where p and q go different directions
    
    if (p < root.val && q < root.val) {
        // Both left
        return lowestCommonAncestorBST(root.left, p, q);
    }
    
    if (p > root.val && q > root.val) {
        // Both right
        return lowestCommonAncestorBST(root.right, p, q);
    }
    
    // One left, one right (or one is root)
    return root;
}

// For general binary tree (not necessarily BST):
public TreeNode lowestCommonAncestorBT(TreeNode root, int p, int q) {
    if (root == null) return null;
    
    // If current is one of targets
    if (root.val == p || root.val == q) {
        return root;
    }
    
    TreeNode left = lowestCommonAncestorBT(root.left, p, q);
    TreeNode right = lowestCommonAncestorBT(root.right, p, q);
    
    if (left != null && right != null) {
        // Found on both sides - root is LCA
        return root;
    }
    
    // Found on one side only
    return left != null ? left : right;
}

// Time: O(n), Space: O(h)
```

### Pattern 3: Validate BST

**When to use**: Checking tree properties, ensuring data structure integrity.

```java
public boolean isValidBST(TreeNode root) {
    return validate(root, Long.MIN_VALUE, Long.MAX_VALUE);
}

private boolean validate(TreeNode node, long minVal, long maxVal) {
    if (node == null) return true;
    
    // Check if current violates range
    if (node.val <= minVal || node.val >= maxVal) {
        return false;
    }
    
    // Left subtree must have values < node.val
    // Right subtree must have values > node.val
    return validate(node.left, minVal, node.val) &&
           validate(node.right, node.val, maxVal);
}

// Time: O(n), Space: O(h)
// Note: Using Long to handle Integer.MIN_VALUE/MAX_VALUE edge cases
```

### Pattern 4: Tree Construction from Traversals

**When to use**: Reconstructing tree from data, understanding relationships between traversals.

```java
// Build tree from preorder and inorder
// Preorder: [3,9,20,15,7]  (root first)
// Inorder:  [9,3,15,20,7]  (root middle)
// Result tree:  3
//              / \
//             9  20
//               /  \
//              15   7

public TreeNode buildTree(int[] preorder, int[] inorder) {
    Map<Integer, Integer> inMap = new HashMap<>();
    for (int i = 0; i < inorder.length; i++) {
        inMap.put(inorder[i], i);
    }
    
    return buildHelper(preorder, 0, 0, inorder.length - 1, inMap);
}

private TreeNode buildHelper(int[] preorder, int preStart, 
                             int inStart, int inEnd, Map<Integer, Integer> inMap) {
    if (preStart > preorder.length - 1 || inStart > inEnd) {
        return null;
    }
    
    TreeNode root = new TreeNode(preorder[preStart]);
    int inIndex = inMap.get(preorder[preStart]);
    
    // Left subtree has (inIndex - inStart) nodes
    int leftSize = inIndex - inStart;
    
    root.left = buildHelper(preorder, preStart + 1, 
                           inStart, inIndex - 1, inMap);
    root.right = buildHelper(preorder, preStart + leftSize + 1, 
                            inIndex + 1, inEnd, inMap);
    
    return root;
}

// Time: O(n), Space: O(n) for map and recursion
```

---

## Real-World Applications

### 1. **File System Hierarchy**

Trees naturally represent nested folder structures.

```java
public class FileSystem {
    private class File {
        String name;
        boolean isDirectory;
        Map<String, File> children;
        File parent;
        
        File(String name, boolean isDir) {
            this.name = name;
            this.isDirectory = isDir;
            this.children = new HashMap<>();
        }
    }
    
    private File root = new File("/", true);
    private File current = root;
    
    public void mkdir(String path) {
        File dir = navigateOrCreate(path, true);
    }
    
    public void touch(String path) {
        File file = navigateOrCreate(path, false);
    }
    
    public void ls(String path) {
        File file = navigate(path);
        if (file.isDirectory) {
            file.children.values().forEach(f -> System.out.println(f.name));
        }
    }
    
    // Tree traversal to calculate directory size
    public long getSize(File file) {
        if (!file.isDirectory) return 1;
        
        long size = 0;
        for (File child : file.children.values()) {
            size += getSize(child);
        }
        return size;
    }
}
```

### 2. **DOM Tree (Web Browsers)**

HTML structure is a tree; CSS selectors traverse it.

```java
public class DOM {
    public class Element {
        String tagName;
        Map<String, String> attributes;
        List<Element> children;
        Element parent;
        String textContent;
    }
    
    // Find element by ID
    public Element getElementById(Element root, String id) {
        if (root.attributes.get("id").equals(id)) {
            return root;
        }
        
        for (Element child : root.children) {
            Element result = getElementById(child, id);
            if (result != null) return result;
        }
        
        return null;
    }
    
    // Get all elements with class
    public List<Element> getElementsByClass(Element root, String className) {
        List<Element> results = new ArrayList<>();
        
        if (root.attributes.get("class", "").contains(className)) {
            results.add(root);
        }
        
        for (Element child : root.children) {
            results.addAll(getElementsByClass(child, className));
        }
        
        return results;
    }
}

// Used in: Browser rendering, CSS parsing, DOM manipulation
```

### 3. **Expression Trees (Compilers)**

Parse mathematical expressions into tree structure.

```java
public class ExpressionTree {
    public class Node {
        String op;  // "+", "-", "*", "/", or number
        Node left;
        Node right;
    }
    
    // Evaluate expression tree
    public double evaluate(Node node) {
        // Base case: leaf (number)
        if (node.left == null && node.right == null) {
            return Double.parseDouble(node.op);
        }
        
        double leftVal = evaluate(node.left);
        double rightVal = evaluate(node.right);
        
        return switch(node.op) {
            case "+" -> leftVal + rightVal;
            case "-" -> leftVal - rightVal;
            case "*" -> leftVal * rightVal;
            case "/" -> leftVal / rightVal;
            default -> 0;
        };
    }
    
    // Inorder gives infix notation
    // Preorder gives prefix notation
    // Postorder gives postfix notation
}

// Used in: Compilers, calculators, symbolic math
```

### 4. **Binary Indexed Tree (Fenwick Tree) for Range Queries**

A tree structure for efficient range sum queries.

```java
public class BIT {
    private int[] tree;
    private int n;
    
    public BIT(int[] nums) {
        n = nums.length;
        tree = new int[n + 1];
        
        for (int i = 0; i < n; i++) {
            update(i, nums[i]);
        }
    }
    
    public void update(int idx, int delta) {
        idx++;  // BIT is 1-indexed
        
        while (idx <= n) {
            tree[idx] += delta;
            idx += idx & -idx;  // Add rightmost set bit
        }
    }
    
    public int sumRange(int left, int right) {
        return rangeSum(right) - (left > 0 ? rangeSum(left - 1) : 0);
    }
    
    private int rangeSum(int idx) {
        idx++;  // 1-indexed
        int sum = 0;
        
        while (idx > 0) {
            sum += tree[idx];
            idx -= idx & -idx;  // Remove rightmost set bit
        }
        
        return sum;
    }
}

// Time: O(log n) per update and query
// Space: O(n)
// Used in: Competitive programming, database indexes
```

---

## Tree Traversal Decision Tree

```
What do I need to do?

├─ Visit parent before children?
│  └─ Pre-order (root-left-right)
│
├─ Want sorted output (from BST)?
│  └─ In-order (left-root-right)
│
├─ Use results of children?
│  └─ Post-order (left-right-root)
│
└─ Process by depth/level?
   └─ Level-order BFS (queue)
```

---

## Common Mistakes (Anti-Patterns)

### ❌ Forgetting Base Cases
```java
// ❌ Will cause stack overflow
public int height(TreeNode node) {
    return 1 + Math.max(height(node.left), height(node.right));
}

// ✅ Always include null check
public int height(TreeNode node) {
    if (node == null) return 0;
    return 1 + Math.max(height(node.left), height(node.right));
}
```

### ❌ Modifying Tree While Traversing
```java
// ❌ Dangerous - tree structure changes
public void deleteEvenNodes(TreeNode node) {
    if (node.left != null && node.left.val % 2 == 0) {
        node.left = null;  // May lose entire subtree
    }
}

// ✅ Mark nodes, then delete in separate pass
// Or reconstruct tree appropriately
```

### ❌ Not Considering Space Complexity of Recursion
```java
// ❌ O(n) space for skewed tree (like linked list)
public int height(TreeNode node) {
    if (node == null) return 0;
    return 1 + height(node.left) + height(node.right);  // O(n) call stack
}

// For skewed tree, this can cause stack overflow
// Iterative approach would use heap instead
```

---

## Complexity Reference

| Operation | Balanced BST | Unbalanced BST | Array |
|-----------|-------------|----------------|-------|
| Search | O(log n) | O(n) | O(n) |
| Insert | O(log n) | O(n) | O(n) |
| Delete | O(log n) | O(n) | O(n) |
| Space | O(n) | O(n) | O(n) |

**Tree Height**:
- Balanced: O(log n)
- Skewed: O(n)

---

## Next Steps

- Practice all four traversal patterns until they're automatic
- Build at least one self-balancing tree (AVL, Red-Black) structure
- Master LCA, path sum, and tree construction problems
- Move to [Graphs](06-graphs.md) - graphs are trees without the hierarchy constraint
