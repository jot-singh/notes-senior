# Data Structures and Algorithms (DSA)

Comprehensive senior-level notes on Data Structures and Algorithms, focused on building problem-solving intuition and recognizing patterns in real-world scenarios.

## Learning Philosophy

These notes emphasize:
- **Pattern Recognition**: Understanding when and why to use specific data structures
- **Problem-Solving Intuition**: Developing the ability to identify the right approach
- **Real-World Context**: Practical applications in production systems
- **Trade-offs Analysis**: Time/space complexity and implementation considerations

---

## Contents

### 1. [Arrays and Strings](01-arrays-strings.md)
- Array fundamentals and operations
- String manipulation patterns
- Sliding window technique
- Two-pointer patterns
- Real-world use cases: caching, buffers, indexing

### 2. [Linked Lists](02-linked-lists.md)
- Singly and doubly linked lists
- List operations and traversal patterns
- Fast and slow pointer techniques
- Real-world use cases: LRU caches, undo/redo, blockchain

### 3. [Stacks and Queues](03-stacks-queues.md)
- Stack fundamentals and use patterns
- Queue variations (FIFO, circular, priority)
- Monotonic stacks and deques
- Real-world use cases: browser history, task scheduling, parsing

### 4. [Binary Search Trees](04-binary-search-trees.md)
- BST properties and invariants
- Insert, search, delete operations
- BST validation techniques
- Self-balancing trees (AVL, Red-Black)
- Common BST patterns (kth smallest, LCA, range queries)
- Real-world use cases: database indexes, ordered caches

### 5. [Trees Fundamentals](05-trees.md)
- Binary trees and properties
- Tree traversal patterns (in-order, pre-order, post-order, level-order)
- Tree construction and manipulation
- Balanced trees overview
- Real-world use cases: file systems, DOM, expression trees

### 5.1 [Tries (Prefix Trees)](05.01-tries.md)
- Trie structure and intuition
- Insert, search, and prefix operations
- Space optimization techniques (compressed tries)
- Autocomplete and spell-checking patterns
- Real-world use cases: search engines, IP routing, dictionaries

### 6. [Graphs Fundamentals](06-graphs.md)
- Graph representations and types
- Graph traversal (BFS, DFS)
- Topological sorting
- Real-world use cases: social networks, route planning, recommendation engines

### 7. [Sorting Algorithms](07-sorting-algorithms.md)
- Comparison-based sorting (Merge Sort, Quick Sort, Heap Sort)
- Linear-time sorting (Counting Sort, Radix Sort)
- Sorting pattern recognition
- Real-world use cases: database queries, data pipelines

### 8. [Searching Algorithms](08-searching-algorithms.md)
- Linear search and optimization
- Binary search and variations
- Real-world use cases: databases, log search, version control

### 9. [Dynamic Programming Fundamentals](09-dynamic-programming.md)
- Memoization and tabulation
- DP pattern recognition
- State definition strategies
- Real-world use cases: optimization problems, route planning

### 10. [Heaps and Priority Queues](10-heaps-priority-queues.md)
- Binary heaps implementation
- Min/max heap operations
- Priority queue patterns
- Real-world use cases: task scheduling, event processing, algorithms

### 11. [Advanced Tree Structures](11-advanced-trees.md)
- AVL trees and balancing
- Red-Black trees
- B-trees
- Real-world use cases: databases, file systems

### 12. [Algorithm Patterns and Techniques](12-algorithm-patterns.md)
- Greedy algorithms
- Divide and conquer
- Bit manipulation
- Backtracking
- Real-world use cases: optimization, constraint solving

---

## How to Use These Notes

1. **Sequential Learning**: Follow the numbered order for foundational understanding
2. **Pattern Recognition**: Each topic emphasizes when to use that data structure
3. **Problem Practice**: Use the patterns as a mental framework when approaching problems
4. **Code Examples**: All examples are production-quality and language-specific
5. **Trade-offs**: Understand the cost/benefit of each approach

---

## Key Concepts by Use Case

### Caching and Storage
- Hash Tables, Arrays, Linked Lists (LRU), Bloom Filters

### Searching and Retrieval
- Binary Search, Hash Tables, Trees (BST), Tries, B-trees

### Graph Problems (Networks, Routes)
- BFS, DFS, Dijkstra's, Topological Sort

### Optimization Problems
- Dynamic Programming, Greedy Algorithms, Binary Search

### Real-time Processing
- Heaps, Priority Queues, Stacks, Queues

### Pattern Matching
- KMP, Trie, Hash Tables, Dynamic Programming

---

## Complexity Reference

### Time Complexity (Common Cases)
| Operation | Array | Linked List | Hash Table | BST | Balanced Tree | Heap |
|-----------|-------|-------------|-----------|-----|---------------|------|
| Search | O(n) | O(n) | O(1) avg | O(log n) | O(log n) | O(n) |
| Insert | O(n) | O(1)* | O(1) avg | O(log n) | O(log n) | O(log n) |
| Delete | O(n) | O(1)* | O(1) avg | O(log n) | O(log n) | O(log n) |
| Access | O(1) | O(n) | - | - | - | - |

*With pointer reference

---

## Study Strategy

### Phase 1: Foundations (1-2 weeks)
- Master Arrays, Strings, Linked Lists
- Understand time/space complexity deeply
- Practice basic operations

### Phase 2: Complex Structures (2-3 weeks)
- Trees and Graphs
- Understand traversal patterns
- Build spatial reasoning

### Phase 3: Algorithm Techniques (2-3 weeks)
- Sorting and Searching
- Dynamic Programming
- Pattern recognition

### Phase 4: Integration (Ongoing)
- Practice problems combining multiple concepts
- Study real-world system uses
- Develop intuition through varied problems

---

## Real-World Connections

Each note section includes:
- **Production Use**: How it's used in real systems
- **Problem Patterns**: Common problem types that use this structure
- **Trade-off Analysis**: Why choose this vs alternatives
- **Optimization Opportunities**: How to squeeze better performance

---

## Resources for Practice

The concepts here provide the foundation for:
- LeetCode, HackerRank, CodeSignal problem-solving
- Technical interviews at FAANG companies
- Building efficient systems in production
- Understanding database and OS internals
