# Tries (Prefix Trees)

## What You'll Learn
- Trie structure and why it's powerful for prefix operations
- Building intuition for when to use tries vs hash tables
- Insert, search, and prefix search operations
- Space optimization techniques (compressed tries)
- Real-world applications in autocomplete, spell-checking, and routing
- Common trie patterns and problem-solving strategies

## Why This Matters

Tries are specialized tree structures optimized for **string operations**, particularly prefix-based queries. While hash tables provide O(1) average lookups, tries excel at **prefix matching**, **autocomplete**, and **lexicographic ordering**—operations that are expensive or impossible with hash tables. Understanding tries means understanding how search engines suggest queries, how IP routing works, and how spell checkers find corrections efficiently.

---

## Building the Intuition

### The Problem Hash Tables Can't Solve Efficiently

```java
// Given dictionary: ["cat", "car", "card", "care", "careful", "dog"]
// Query: Find all words starting with "car"

// Hash Table approach:
Set<String> dict = new HashSet<>(Arrays.asList("cat", "car", "card", "care", "careful", "dog"));

public List<String> findWordsWithPrefix(String prefix) {
    List<String> result = new ArrayList<>();
    for (String word : dict) {  // O(n) - must check every word
        if (word.startsWith(prefix)) {
            result.add(word);
        }
    }
    return result;
}
// Time: O(n × m) where n = dictionary size, m = word length
// Hash tables don't group words by common prefixes!

// Trie approach:
// All words starting with "car" are in the same subtree
// Navigate to "c" → "a" → "r" → collect all children
// Time: O(p + k) where p = prefix length, k = results count
```

**Key insight**: Tries organize data by **shared prefixes**, making prefix operations extremely efficient.

---

## Trie Structure

### Visual Representation

```
Dictionary: ["cat", "car", "card", "dog", "door"]

Trie Structure:
                (root)
               /      \
              c        d
              |        |
              a        o
             / \       |
            t   r      o
           *    |      |
               (*)     r
                |      *
                d
                *

* = end of word marker

Key observations:
1. Each node represents a character
2. Path from root to * represents a complete word
3. Common prefixes share paths: "car" and "card" share "c-a-r"
4. Sibling nodes represent alternative continuations
5. Space grows with unique prefixes, not total words
```

### Node Structure

```java
class TrieNode {
    // Children nodes (26 letters for lowercase English)
    TrieNode[] children;
    
    // Marks if this node is end of a word
    boolean isEndOfWord;
    
    // Optional: store the actual word here for easier retrieval
    String word;  // Used in some problems
    
    // Optional: count of words passing through this node
    int count;  // Used for frequency tracking
    
    public TrieNode() {
        children = new TrieNode[26];  // For 'a' to 'z'
        isEndOfWord = false;
    }
}

// Why array of 26 instead of HashMap?
// Array: O(1) lookup, fixed memory per node
// HashMap: O(1) average, but overhead per map
// For dense character sets (a-z), array is faster and simpler
```

### Alternative Node Structure (Space-Optimized)

```java
class TrieNodeOptimized {
    // Use HashMap for sparse character sets
    Map<Character, TrieNodeOptimized> children;
    boolean isEndOfWord;
    
    public TrieNodeOptimized() {
        children = new HashMap<>();
        isEndOfWord = false;
    }
}

// When to use HashMap:
// - Large character set (Unicode, special characters)
// - Sparse usage (not all characters present)
// - Trade-off: Slower lookup but less memory per node
```

---

## Core Operations

### Pattern 1: Insert

```java
public class Trie {
    private TrieNode root;
    
    public Trie() {
        root = new TrieNode();
    }
    
    // Insert a word into trie
    public void insert(String word) {
        TrieNode current = root;
        
        for (char ch : word.toCharArray()) {
            int index = ch - 'a';  // Map 'a'-'z' to 0-25
            
            // Create node if doesn't exist
            if (current.children[index] == null) {
                current.children[index] = new TrieNode();
            }
            
            // Move to child node
            current = current.children[index];
        }
        
        // Mark end of word
        current.isEndOfWord = true;
    }
    
    // Time: O(m) where m = word length
    // Space: O(m) in worst case (all new nodes)
}
```

**Intuition**: Follow/create path for each character. It's like navigating a file system—create directories as needed, mark the final directory as "file location."

### Pattern 2: Search

```java
// Search for exact word
public boolean search(String word) {
    TrieNode node = searchNode(word);
    return node != null && node.isEndOfWord;
}

// Helper: Navigate to node representing word
private TrieNode searchNode(String word) {
    TrieNode current = root;
    
    for (char ch : word.toCharArray()) {
        int index = ch - 'a';
        
        // Path doesn't exist
        if (current.children[index] == null) {
            return null;
        }
        
        current = current.children[index];
    }
    
    return current;
}

// Time: O(m) where m = word length
// Space: O(1)

// Example:
// Trie has: "car", "card"
// search("car") → true (end marker exists)
// search("ca") → false (no end marker)
// search("card") → true
// search("care") → false (path doesn't exist)
```

### Pattern 3: Prefix Search (StartsWith)

```java
// Check if any word starts with given prefix
public boolean startsWith(String prefix) {
    TrieNode node = searchNode(prefix);
    return node != null;
}

// Note: We don't check isEndOfWord!
// Just need to verify path exists

// Example:
// Trie has: "car", "card", "careful"
// startsWith("car") → true (3 words have this prefix)
// startsWith("care") → true (1 word: "careful")
// startsWith("dog") → false (no such prefix)

// Time: O(p) where p = prefix length
// Space: O(1)
```

**Key difference**: Search requires end marker, startsWith only needs path to exist.

---

## Advanced Operations

### Pattern 4: Find All Words with Prefix (Autocomplete)

```java
// Return all words starting with given prefix
public List<String> findWordsWithPrefix(String prefix) {
    List<String> result = new ArrayList<>();
    TrieNode node = searchNode(prefix);
    
    if (node == null) {
        return result;  // Prefix doesn't exist
    }
    
    // DFS from this node to collect all words
    dfsCollect(node, prefix, result);
    return result;
}

private void dfsCollect(TrieNode node, String currentWord, List<String> result) {
    // If this node marks end of word, add it
    if (node.isEndOfWord) {
        result.add(currentWord);
    }
    
    // Explore all children
    for (int i = 0; i < 26; i++) {
        if (node.children[i] != null) {
            char ch = (char) ('a' + i);
            dfsCollect(node.children[i], currentWord + ch, result);
        }
    }
}

// Time: O(p + k × m) where:
//   p = prefix length (navigate to starting node)
//   k = number of matching words
//   m = average word length (collecting results)
// Space: O(k × m) for results

// Example:
// Trie: ["car", "card", "care", "careful", "cat"]
// findWordsWithPrefix("car") → ["car", "card", "care", "careful"]
// findWordsWithPrefix("care") → ["care", "careful"]
```

**Real-world use**: Search bar autocomplete, code editor suggestions.

### Pattern 5: Delete Word

```java
public void delete(String word) {
    delete(root, word, 0);
}

private boolean delete(TrieNode node, String word, int depth) {
    if (node == null) {
        return false;  // Word doesn't exist
    }
    
    // Reached end of word
    if (depth == word.length()) {
        if (!node.isEndOfWord) {
            return false;  // Word doesn't exist
        }
        
        // Unmark end of word
        node.isEndOfWord = false;
        
        // If node has no children, it can be deleted
        return isEmpty(node);
    }
    
    int index = word.charAt(depth) - 'a';
    
    // Recursively delete in child
    boolean shouldDeleteChild = delete(node.children[index], word, depth + 1);
    
    // If true, delete child node
    if (shouldDeleteChild) {
        node.children[index] = null;
        
        // Return true if current node can also be deleted
        // (no other children and not end of another word)
        return !node.isEndOfWord && isEmpty(node);
    }
    
    return false;
}

private boolean isEmpty(TrieNode node) {
    for (TrieNode child : node.children) {
        if (child != null) {
            return false;
        }
    }
    return true;
}

// Time: O(m) where m = word length
// Space: O(m) recursion stack

// Critical: Only delete nodes that aren't part of other words!
// Trie: ["car", "card"]
// delete("card") → keep "c-a-r" (part of "car"), only remove "d"
```

### Pattern 6: Longest Common Prefix

```java
// Find longest common prefix among all words in trie
public String longestCommonPrefix() {
    StringBuilder prefix = new StringBuilder();
    TrieNode current = root;
    
    // Continue while:
    // 1. Exactly one child exists
    // 2. Current node is not end of word
    while (countChildren(current) == 1 && !current.isEndOfWord) {
        // Find the single child
        for (int i = 0; i < 26; i++) {
            if (current.children[i] != null) {
                prefix.append((char) ('a' + i));
                current = current.children[i];
                break;
            }
        }
    }
    
    return prefix.toString();
}

private int countChildren(TrieNode node) {
    int count = 0;
    for (TrieNode child : node.children) {
        if (child != null) count++;
    }
    return count;
}

// Example:
// Words: ["flower", "flow", "flight"]
// LCP: "fl" (stops at 'o' vs 'i' branch)

// Words: ["dog", "racecar", "car"]
// LCP: "" (multiple branches from root)
```

---

## Space Optimization Techniques

### Technique 1: Compressed Trie (Radix Tree)

```java
// Standard Trie for "test", "testing", "tested":
//     t → e → s → t* → i → n → g*
//                  ↓
//                  e → d*
// 7 nodes, many single-child paths

// Compressed Trie (store strings on edges):
//     "test"* → "ing"*
//            ↘ "ed"*
// 3 nodes, paths compressed into edges

class RadixNode {
    Map<String, RadixNode> children;  // Edge labels are strings
    boolean isEndOfWord;
}

// Advantage: Fewer nodes for words with long common prefixes
// Disadvantage: More complex implementation
// Used in: Patricia Trie, Radix Tree
```

### Technique 2: HashMap Children for Sparse Sets

```java
// Instead of TrieNode[26] for every node:
class TrieNodeSparse {
    Map<Character, TrieNodeSparse> children;  // Only store present children
    boolean isEndOfWord;
}

// Standard array: 26 × 8 bytes = 208 bytes per node
// HashMap: Only stores actual children

// When to use:
// - Unicode characters (array would be huge)
// - Sparse character usage (few children per node)
// - Trade-off: Slower lookup (HashMap overhead)
```

### Technique 3: Suffix Compression

```java
// For memory-critical applications:
// Store common suffixes once

class TrieNodeCompressed {
    Map<Character, TrieNodeCompressed> children;
    String suffix;  // Store remaining string if single path
    
    // "testing" → t-e-s-t, suffix="ing"
    // Saves nodes for linear chains
}
```

---

## Real-World Applications

### Application 1: Autocomplete System

```java
public class AutocompleteSystem {
    private TrieNode root;
    
    class TrieNode {
        Map<Character, TrieNode> children;
        String word;
        int frequency;  // How often this word is used
        
        TrieNode() {
            children = new HashMap<>();
        }
    }
    
    public void addWord(String word, int frequency) {
        TrieNode current = root;
        
        for (char ch : word.toCharArray()) {
            current.children.putIfAbsent(ch, new TrieNode());
            current = current.children.get(ch);
        }
        
        current.word = word;
        current.frequency = frequency;
    }
    
    public List<String> getTopSuggestions(String prefix, int k) {
        TrieNode node = searchNode(prefix);
        if (node == null) return new ArrayList<>();
        
        // Collect all words under this prefix
        PriorityQueue<TrieNode> pq = new PriorityQueue<>(
            (a, b) -> b.frequency - a.frequency  // Max heap by frequency
        );
        
        collectWords(node, pq);
        
        // Return top k
        List<String> result = new ArrayList<>();
        for (int i = 0; i < k && !pq.isEmpty(); i++) {
            result.add(pq.poll().word);
        }
        
        return result;
    }
    
    private void collectWords(TrieNode node, PriorityQueue<TrieNode> pq) {
        if (node.word != null) {
            pq.offer(node);
        }
        
        for (TrieNode child : node.children.values()) {
            collectWords(child, pq);
        }
    }
}

// Used by: Google Search, IDE code completion, mobile keyboards
```

### Application 2: Spell Checker with Edit Distance

```java
// Find words within edit distance k of query
public List<String> findSimilarWords(String query, int maxDistance) {
    List<String> result = new ArrayList<>();
    findSimilar(root, "", query, maxDistance, result);
    return result;
}

private void findSimilar(TrieNode node, String current, String query, 
                         int maxDist, List<String> result) {
    if (node.isEndOfWord) {
        if (editDistance(current, query) <= maxDist) {
            result.add(current);
        }
    }
    
    for (int i = 0; i < 26; i++) {
        if (node.children[i] != null) {
            char ch = (char) ('a' + i);
            findSimilar(node.children[i], current + ch, query, maxDist, result);
        }
    }
}

// Used by: Word processors, search engines (did you mean...)
```

### Application 3: IP Routing Table

```java
// Trie structure for IP addresses
// Each node represents a bit (0 or 1)
class IPRouter {
    class RouterNode {
        RouterNode[] children = new RouterNode[2];  // 0 and 1
        String nextHop;  // Where to forward packet
    }
    
    private RouterNode root = new RouterNode();
    
    // Insert route: IP → next hop
    public void addRoute(String ipBinary, String nextHop) {
        RouterNode current = root;
        
        for (char bit : ipBinary.toCharArray()) {
            int idx = bit - '0';
            if (current.children[idx] == null) {
                current.children[idx] = new RouterNode();
            }
            current = current.children[idx];
        }
        
        current.nextHop = nextHop;
    }
    
    // Longest prefix match for routing decision
    public String route(String ipBinary) {
        RouterNode current = root;
        String lastMatch = null;
        
        for (char bit : ipBinary.toCharArray()) {
            int idx = bit - '0';
            
            if (current.children[idx] == null) break;
            
            current = current.children[idx];
            
            if (current.nextHop != null) {
                lastMatch = current.nextHop;  // Longest prefix so far
            }
        }
        
        return lastMatch;
    }
}

// Real usage: Internet routers use tries for IP prefix matching
```

---

## Common Trie Patterns

### Pattern: Word Search II (2D Grid DFS + Trie)

```java
// Find all words from dictionary that exist in 2D grid
public List<String> findWords(char[][] board, String[] words) {
    // Build trie from dictionary
    Trie trie = new Trie();
    for (String word : words) {
        trie.insert(word);
    }
    
    List<String> result = new ArrayList<>();
    
    // Try starting DFS from each cell
    for (int i = 0; i < board.length; i++) {
        for (int j = 0; j < board[0].length; j++) {
            dfs(board, i, j, trie.root, result);
        }
    }
    
    return result;
}

private void dfs(char[][] board, int i, int j, TrieNode node, List<String> result) {
    char ch = board[i][j];
    
    // Check bounds and visited
    if (i < 0 || i >= board.length || j < 0 || j >= board[0].length || 
        ch == '#' || node.children[ch - 'a'] == null) {
        return;
    }
    
    node = node.children[ch - 'a'];
    
    // Found word
    if (node.word != null) {
        result.add(node.word);
        node.word = null;  // Avoid duplicates
    }
    
    // Mark visited
    board[i][j] = '#';
    
    // Explore neighbors
    dfs(board, i + 1, j, node, result);
    dfs(board, i - 1, j, node, result);
    dfs(board, i, j + 1, node, result);
    dfs(board, i, j - 1, node, result);
    
    // Restore
    board[i][j] = ch;
}

// Why Trie? Without trie, we'd need to check every word at every cell
// With trie: Prune search early if prefix doesn't exist in dictionary
// Time: O(m × n × 4^L) where L = max word length
// Much faster than O(m × n × W × L) where W = dictionary size
```

### Pattern: Replace Words (Shortest Prefix)

```java
// Replace words with their shortest root
// Dictionary: ["cat", "bat", "rat"]
// Sentence: "the cattle was rattled by the battery"
// Output: "the cat was rat by the bat"

public String replaceWords(List<String> roots, String sentence) {
    Trie trie = new Trie();
    for (String root : roots) {
        trie.insert(root);
    }
    
    String[] words = sentence.split(" ");
    
    for (int i = 0; i < words.length; i++) {
        words[i] = trie.findShortestRoot(words[i]);
    }
    
    return String.join(" ", words);
}

// In Trie class:
public String findShortestRoot(String word) {
    TrieNode current = root;
    StringBuilder prefix = new StringBuilder();
    
    for (char ch : word.toCharArray()) {
        int idx = ch - 'a';
        
        if (current.children[idx] == null) {
            return word;  // No root found
        }
        
        prefix.append(ch);
        current = current.children[idx];
        
        // Found shortest root
        if (current.isEndOfWord) {
            return prefix.toString();
        }
    }
    
    return word;  // No root found
}
```

---

## Trie vs Other Data Structures

| Operation | Trie | Hash Table | BST (Balanced) |
|-----------|------|------------|----------------|
| Insert | O(m) | O(m) avg | O(m log n) |
| Search | O(m) | O(m) avg | O(m log n) |
| Prefix Search | O(p) | O(n × m) | O(log n + k) |
| Autocomplete | O(p + k) | O(n × m) | O(log n + k) |
| Lexicographic Order | O(n) | O(n log n) | O(n) |
| Space | O(ALPHABET × N × M) | O(N × M) | O(N × M) |

Where:
- m = average word length
- n = number of words
- p = prefix length
- k = number of results
- ALPHABET = character set size (26 for lowercase)

**Use Trie when**:
- Frequent prefix queries
- Autocomplete features
- Lexicographic operations
- Dictionary with common prefixes

**Use Hash Table when**:
- Only exact lookups needed
- No prefix operations
- Memory is constrained
- Character set is large

**Use BST when**:
- Need range queries
- Sorted iteration
- No specific prefix focus

---

## Space Complexity Analysis

### Memory per Node

```java
// Standard Trie Node:
class TrieNode {
    TrieNode[] children;  // 26 × 8 bytes = 208 bytes
    boolean isEndOfWord;   // 1 byte
    // Total: ~209 bytes per node
}

// For dictionary of 10,000 words, average length 8:
// Worst case: 10,000 × 8 = 80,000 nodes
// Memory: 80,000 × 209 bytes ≈ 16 MB

// Optimized with HashMap:
class TrieNode {
    Map<Character, TrieNode> children;  // Only actual children
    boolean isEndOfWord;
    // Memory: ~50-100 bytes per node with 2-3 children
}
```

### Space Optimization: When It Matters

```
Small dictionary (<1000 words): Array-based is fine
Medium dictionary (1000-100k words): HashMap-based better
Large dictionary (>100k words): Consider compressed trie

English language:
- Common prefixes: "pre", "un", "re", "in"
- Sharing saves memory
- Trie naturally deduplicates prefixes
```

---

## Common Pitfalls

### ❌ Anti-Pattern 1: Not Handling Empty String

```java
// Missing edge case
public boolean search(String word) {
    if (word.isEmpty()) return false;  // Add this check!
    // ... rest of search logic
}
```

### ❌ Anti-Pattern 2: Confusing Search vs StartsWith

```java
// WRONG: Using same logic for both
public boolean search(String word) {
    return searchNode(word) != null;  // Missing isEndOfWord check!
}

// CORRECT:
public boolean search(String word) {
    TrieNode node = searchNode(word);
    return node != null && node.isEndOfWord;  // Must check end marker
}
```

### ❌ Anti-Pattern 3: Memory Leak in Delete

```java
// WRONG: Just marking as not end of word
public void delete(String word) {
    TrieNode node = searchNode(word);
    if (node != null) {
        node.isEndOfWord = false;  // Orphaned nodes remain!
    }
}

// CORRECT: Remove unnecessary nodes (see delete implementation above)
```

---

## Interview Tips

1. **Ask about character set**: Lowercase only? Unicode? Affects choice of array vs HashMap
2. **Clarify operations**: Read-heavy? Insert-heavy? Affects optimization strategy
3. **Space constraints**: If memory critical, mention compressed tries
4. **Edge cases**: Empty strings, single character, duplicates
5. **Explain trade-offs**: Why trie over hash table for this specific problem

## Quick Reference

```java
// Build trie
Trie trie = new Trie();
trie.insert("word");

// Search operations
boolean exists = trie.search("word");        // Exact match
boolean hasPrefix = trie.startsWith("wor");  // Prefix exists

// Autocomplete
List<String> suggestions = trie.findWordsWithPrefix("wo");

// Time complexities:
// Insert/Search/StartsWith: O(m) where m = word length
// Autocomplete: O(p + k × m) where p = prefix, k = results
// Space: O(ALPHABET × N × M) worst case
```
