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
