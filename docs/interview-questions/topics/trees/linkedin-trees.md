# Interesting Trees Interview Problems

## Problem 1: Connect Nodes at Same Level (Populating Next Right Pointers)

**Difficulty**: Hard  
**Companies**: Google, Amazon  
**Key Concept**: Level-order traversal with O(1) space using next pointers

### Problem Statement

Given a binary tree where each node has three pointers:
- `left`: pointer to left child
- `right`: pointer to right child  
- `next`: pointer to the next node at the same level (initially `null`)

**Task**: Populate the `next` pointer of each node to point to its next right node at the same level. If there is no next right node, the `next` pointer should remain `null`.

**Constraint**: You must solve this using **O(1) extra space** (excluding recursion stack if using recursion).

### Node Structure

```java
class Node {
    int val;
    Node left;
    Node right;
    Node next;
    
    Node(int val) {
        this.val = val;
        this.left = null;
        this.right = null;
        this.next = null;
    }
}
```

### Examples

#### Example 1: Perfect Binary Tree

```
Input:
        1
       / \
      2   3
     / \ / \
    4  5 6  7

Output:
        1 → null
       / \
      2 → 3 → null
     / \ / \
    4→5→6→7 → null

Explanation:
- Level 0: 1 → null
- Level 1: 2 → 3 → null
- Level 2: 4 → 5 → 6 → 7 → null
```

#### Example 2: Non-Perfect Binary Tree

```
Input:
        1
       / \
      2   3
     / \   \
    4   5   7

Output:
        1 → null
       / \
      2 → 3 → null
     / \   \
    4→5 → 7 → null

Explanation:
- Level 0: 1 → null
- Level 1: 2 → 3 → null  
- Level 2: 4 → 5 → 7 → null (5 connects to 7 across parent boundary)
```

#### Example 3: Single Node

```
Input:
    1

Output:
    1 → null
```

### Constraints

- The number of nodes in the tree is in the range `[0, 6000]`
- `-100 <= Node.val <= 100`
- The tree is a **binary tree** (not necessarily complete or perfect)
- **You must use O(1) extra space** (constant space, not counting recursion stack)

### Follow-up Variations

1. **Perfect Binary Tree**: If the tree is guaranteed to be perfect (all levels completely filled), can you optimize further?
2. **Return Modified Tree**: Should the function return the root of the modified tree?
3. **Reverse Next Pointers**: Can you populate `next` pointers to point to the left neighbor instead?

### Why This Problem is Mind-Blowing

The challenge lies in:
1. **Level-order traversal** typically requires a queue → O(n) space
2. **O(1) space constraint** means we cannot use a queue
3. We must leverage the **next pointers we're building** to traverse the next level
4. Handling **non-perfect trees** where nodes at same level may not have direct parent relationships

### Key Insights

- Use the `next` pointers of the **current level** to traverse and connect the **next level**
- The current level acts as a "linked list" via `next` pointers
- For each node, connect its children before moving to the next node at the same level

---

## Problem 2: [To be added]

### Problem Statement

[Problem description coming soon]

---

## Solutions

### Solution 1: Connect Nodes at Same Level - O(1) Space

```java
public Node connect(Node root) {
    if (root == null) return null;
    
    // Start with the root level
    Node levelStart = root;
    
    // Process each level
    while (levelStart != null) {
        Node current = levelStart;
        Node nextLevelStart = null;  // First node of next level
        Node prev = null;             // Previous node in next level
        
        // Traverse current level using next pointers
        while (current != null) {
            // Process left child
            if (current.left != null) {
                if (prev != null) {
                    prev.next = current.left;
                } else {
                    nextLevelStart = current.left;  // First node of next level
                }
                prev = current.left;
            }
            
            // Process right child
            if (current.right != null) {
                if (prev != null) {
                    prev.next = current.right;
                } else {
                    nextLevelStart = current.right;
                }
                prev = current.right;
            }
            
            // Move to next node at current level
            current = current.next;
        }
        
        // Move to next level
        levelStart = nextLevelStart;
    }
    
    return root;
}
```

#### How It Works

```
Step-by-step for tree:
        1
       / \
      2   3
     / \   \
    4   5   7

Iteration 1 (Level 0: node 1):
- current = 1
- Connect 1.left (2) and 1.right (3)
- After: 2 → 3 → null
- nextLevelStart = 2

Iteration 2 (Level 1: nodes 2, 3):
- current = 2
  - Connect 2.left (4) and 2.right (5)
  - prev = 5
- current = 3 (via next pointer)
  - Connect prev (5) to 3.right (7)
- After: 4 → 5 → 7 → null
- nextLevelStart = 4

Iteration 3 (Level 2: nodes 4, 5, 7):
- All leaf nodes, no children to connect
- nextLevelStart = null
- Loop exits
```

#### Complexity Analysis

- **Time**: O(n) - visit each node exactly once
- **Space**: O(1) - only using pointers (levelStart, current, prev, nextLevelStart)

### Alternative: Perfect Binary Tree Optimization

If the tree is **guaranteed to be perfect** (all levels filled), we can simplify:

```java
public Node connectPerfect(Node root) {
    if (root == null) return null;
    
    Node levelStart = root;
    
    while (levelStart.left != null) {  // Has children
        Node current = levelStart;
        
        while (current != null) {
            // Connect left child to right child
            current.left.next = current.right;
            
            // Connect right child to next node's left child
            if (current.next != null) {
                current.right.next = current.next.left;
            }
            
            current = current.next;
        }
        
        levelStart = levelStart.left;  // Move to next level
    }
    
    return root;
}
```

**Why this works for perfect trees**:
- Every node has either 0 or 2 children
- Left child always connects to right child
- Right child connects to next parent's left child
- Simpler logic, easier to understand

#### Complexity Analysis

- **Time**: O(n)
- **Space**: O(1)

---

## Pattern Recognition

**When you see**:
- "Connect nodes at same level"
- "Level-order traversal with O(1) space"
- "Use previously built structure to traverse next level"

**Think**: 
- Leverage the data structure you're building
- Current level as linked list for traversing next level
- Process level by level using pointers

**Similar Problems**:
1. Flatten Binary Tree to Linked List (in-place)
2. Construct Binary Tree from Traversal (using pointers)
3. Morris Traversal (threading for O(1) space traversal)

---

## Key Takeaways

1. **O(1) space doesn't mean no tracking** - we can use a constant number of pointers
2. **Use what you build** - next pointers enable level traversal without queue
3. **Handle edge cases** - non-perfect trees require careful null checks
4. **Level-by-level processing** - outer loop for levels, inner loop for nodes in level
5. **Track level boundaries** - nextLevelStart marks beginning of next level

---

## Problem 2: Count Complete Tree Nodes

**Difficulty**: Hard  
**Companies**: Google  
**Key Concept**: Leverage complete tree properties to achieve better than O(n) complexity

### Problem Statement

Given the root of a **complete binary tree**, return the number of nodes in the tree.

**Complete Binary Tree Definition**:
- Every level, except possibly the last, is completely filled
- All nodes in the last level are as far left as possible
- Last level can have between 1 and 2^h nodes (where h is height)

**Constraint**: Your algorithm should run in **better than O(n)** time complexity.

### Node Structure

```java
class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
    
    TreeNode(int val) {
        this.val = val;
    }
}
```

### Examples

#### Example 1: Complete Binary Tree

```
Input:
        1
       / \
      2   3
     / \ /
    4  5 6

Output: 6

Explanation: All levels filled except last, which is left-aligned.
```

#### Example 2: Perfect Binary Tree

```
Input:
        1
       / \
      2   3
     / \ / \
    4  5 6  7

Output: 7

Explanation: All levels completely filled (special case of complete tree).
```

#### Example 3: Single Node

```
Input:
    1

Output: 1
```

#### Example 4: Left-Skewed at Last Level

```
Input:
        1
       / \
      2   3
     /
    4

Output: 4
```

### Constraints

- The number of nodes in the tree is in the range `[0, 50000]`
- `0 <= Node.val <= 50000`
- The tree is **guaranteed to be complete**

### Approach Comparison

| Approach | Time Complexity | Space Complexity | Key Idea |
|----------|----------------|------------------|----------|
| Any Traversal (DFS/BFS) | O(n) | O(h) or O(w) | Visit every node |
| Optimal (Height-based) | O(log² n) | O(log n) | Use complete tree property |

### Why This Problem is Mind-Blowing

The challenge lies in:
1. **Standard traversal is O(n)** - visits every node
2. **Complete tree property** - enables mathematical counting
3. **Height comparison trick** - detect perfect subtrees instantly
4. **Recursive elimination** - skip entire subtrees using formula
5. **O(log² n) solution** - much faster for large trees

---

## Solutions

### Solution 1: Naive Approach - Any Traversal O(n)

```java
// Simple DFS counting - visits every node
public int countNodes(TreeNode root) {
    if (root == null) return 0;
    return 1 + countNodes(root.left) + countNodes(root.right);
}

// BFS variant
public int countNodesBFS(TreeNode root) {
    if (root == null) return 0;
    
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    int count = 0;
    
    while (!queue.isEmpty()) {
        TreeNode node = queue.poll();
        count++;
        
        if (node.left != null) queue.offer(node.left);
        if (node.right != null) queue.offer(node.right);
    }
    
    return count;
}

// Time: O(n), Space: O(h) for DFS, O(w) for BFS
// Works but doesn't leverage complete tree property
```

### Solution 2: Optimal Approach - Height Comparison O(log² n)

```java
public int countNodes(TreeNode root) {
    if (root == null) return 0;
    
    // Compute leftmost height
    int leftHeight = getLeftHeight(root);
    
    // Compute rightmost height
    int rightHeight = getRightHeight(root);
    
    // If equal, tree is perfect → use formula
    if (leftHeight == rightHeight) {
        return (1 << leftHeight) - 1;  // 2^h - 1
        // Or: return (int) Math.pow(2, leftHeight) - 1;
    }
    
    // If not equal, recursively count left and right subtrees
    return 1 + countNodes(root.left) + countNodes(root.right);
}

// Get height by going all the way left
private int getLeftHeight(TreeNode node) {
    int height = 0;
    while (node != null) {
        height++;
        node = node.left;
    }
    return height;
}

// Get height by going all the way right
private int getRightHeight(TreeNode node) {
    int height = 0;
    while (node != null) {
        height++;
        node = node.right;
    }
    return height;
}
```

#### How It Works - Visual Walkthrough

```
Tree:
        1              leftHeight = 3 (1→2→4)
       / \             rightHeight = 3 (1→3→7)
      2   3            Heights equal → Perfect tree!
     / \ / \           Count = 2^3 - 1 = 7
    4  5 6  7

Tree:
        1              leftHeight = 3 (1→2→4)
       / \             rightHeight = 2 (1→3)
      2   3            Heights NOT equal
     / \ /             Recurse on left and right
    4  5 6

Left subtree (2):      leftHeight = 2 (2→4)
     2                 rightHeight = 2 (2→5)
    / \                Heights equal → Perfect!
   4   5               Count = 2^2 - 1 = 3

Right subtree (3):     leftHeight = 1 (3→6)
     3                 rightHeight = 1 (3→6)
    /                  Heights equal → Perfect!
   6                   Count = 2^1 - 1 = 1

Total: 1 (root) + 3 (left) + 1 (right) = 5
BUT WAIT - right subtree (3→6) is NOT perfect!
Let me recalculate...

Actually for node 3:
  leftHeight = 1 (3→6)
  rightHeight = 0 (3→null on right)
  NOT equal, recurse:
    left of 3 = 6 → heights both 0 → count = 1
    right of 3 = null → count = 0
    Total for subtree 3 = 1 + 1 + 0 = 2

Total: 1 + 3 + 2 = 6 ✓
```

#### Key Insight: Why This is O(log² n)

```
Height of complete tree: h = log n

getLeftHeight: O(h) = O(log n)
getRightHeight: O(h) = O(log n)

In worst case, we recurse on ONE subtree at each level:
- When heights differ, we recurse on left AND right
- BUT at least one subtree will be perfect (heights equal)
- So we only recurse on ONE subtree per level

Recursion depth: O(h) = O(log n)
Work per recursion: O(log n) for height computation

Total: O(log n) × O(log n) = O(log² n)

Much better than O(n) for large trees!
Example: n = 1,000,000 nodes
  - O(n) = 1,000,000 operations
  - O(log² n) = (log₂ 1,000,000)² ≈ 20² = 400 operations
```

### Alternative: Binary Search on Last Level

```java
// More complex but interesting approach
public int countNodesBinarySearch(TreeNode root) {
    if (root == null) return 0;
    
    int height = getLeftHeight(root);
    
    if (height == 0) return 1;
    
    // Binary search on last level
    int left = 0;
    int right = (1 << (height - 1)) - 1;  // Max nodes in last level
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (nodeExists(root, height - 1, mid)) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    // Nodes in all levels except last + nodes in last level
    return ((1 << (height - 1)) - 1) + left;
}

// Check if node at given index exists in last level
private boolean nodeExists(TreeNode root, int height, int index) {
    int left = 0;
    int right = (1 << height) - 1;
    TreeNode current = root;
    
    for (int i = 0; i < height; i++) {
        int mid = left + (right - left) / 2;
        
        if (index <= mid) {
            current = current.left;
            right = mid;
        } else {
            current = current.right;
            left = mid + 1;
        }
    }
    
    return current != null;
}

// Time: O(log² n), Space: O(1) iterative
// More complex but avoids recursion
```

---

## Pattern Recognition

**When you see**:
- "Complete binary tree"
- "Better than O(n) complexity"
- "All levels filled except last"
- "Left-aligned nodes"

**Think**:
- Perfect subtree detection using height comparison
- Mathematical formula for perfect trees: 2^h - 1
- Recursive elimination of entire subtrees
- Binary search on last level (advanced)

**Key Properties of Complete Trees**:
1. At least one subtree is always perfect at each level
2. Left and right heights differ by at most 1
3. Can compute counts using formulas instead of traversal

**Similar Problems**:
1. Check Completeness of Binary Tree
2. Maximum Depth of Binary Tree (related height concept)
3. Count Good Nodes in Binary Tree

---

## Complexity Analysis Summary

| Metric | Naive Approach | Optimal Approach |
|--------|----------------|------------------|
| Time | O(n) | O(log² n) |
| Space | O(h) = O(log n) | O(log n) recursion |
| Best for | Small trees | Large complete trees |
| Trades | Simple, reliable | Complex but efficient |

### When to Use Each

**Use O(n) approach when**:
- Tree is small (< 1000 nodes)
- Code simplicity is priority
- Tree structure is not guaranteed complete

**Use O(log² n) approach when**:
- Tree is large (> 10,000 nodes)
- Complete tree property is guaranteed
- Performance is critical
- Interview optimization question

---

## Key Takeaways

1. **Complete tree property is powerful** - enables mathematical shortcuts
2. **Height comparison detects perfect subtrees** - instant counting with formula
3. **Recursive elimination** - skip entire subtrees instead of visiting nodes
4. **O(log² n) is much faster than O(n)** - 20x-1000x speedup for large trees
5. **Trade-off complexity for performance** - more code, better runtime
6. **Use bit shift for powers of 2** - `(1 << h)` instead of `Math.pow(2, h)`
