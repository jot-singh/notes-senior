# Interview Questions Directory

Comprehensive collection of coding interview questions organized by company and topic, with emphasis on pattern recognition and problem-solving approaches used in FAANG-level technical interviews.

---

## Quick Navigation

### Company-Specific Collections

| Company | Topics Covered | Link |
|---------|----------------|------|
| LinkedIn | Arrays/Strings, Graphs, Dynamic Programming | [company-specific/](company-specific/) |

### Topic-Based Collections

| Topic | Problems | Companies | Difficulty Range | Link |
|-------|----------|-----------|------------------|------|
| Trees | Connect Nodes at Same Level | Google, Amazon | Hard | [topics/trees/](topics/trees/) |

---

## Interview Structure at LinkedIn

### Coding Rounds (45-60 minutes each)
1. **Round 1**: Array/String manipulation + follow-ups
2. **Round 2**: Graph/Tree problems + optimization
3. **Round 3**: Design + coding (system-level thinking)
4. **Round 4**: Dynamic Programming or complex algorithms

### What LinkedIn Looks For
- **Pattern recognition**: Identifying the right approach quickly
- **Code quality**: Clean, readable, maintainable code
- **Communication**: Explaining thought process clearly
- **Optimization**: Time/space complexity awareness
- **Edge cases**: Thorough testing mindset
- **Trade-offs**: Understanding when to optimize vs when to ship

---

## Question Categories

### 1. [Arrays and Strings](linkedin-arrays-strings.md)
**Priority: HIGH** - Most common first-round questions
- Two Sum variations
- Sliding Window problems
- String manipulation
- Subarray problems

**LinkedIn Focus**: Nested connections, recommendation feeds, message parsing

### 2. [Linked Lists and Two-Pointers](linkedin-linked-lists.md)
**Priority: MEDIUM** - Core data structure understanding
- Fast/slow pointer techniques
- List reversal and reordering
- Cycle detection
- Merge operations

**LinkedIn Focus**: Connection chains, timeline feeds, LRU caching

### 3. [Trees and Binary Search](linkedin-trees.md)
**Priority: HIGH** - Fundamental recursion and tree traversal
- Connect nodes at same level
- BST operations
- Tree traversals
- Path problems
- Serialization/deserialization

**LinkedIn Focus**: Organization hierarchies, skill trees, recommendation trees

### 4. [Graphs and BFS/DFS](linkedin-graphs.md)
**Priority: CRITICAL** - LinkedIn is built on graphs!
- Network traversal
- Shortest path
- Connected components
- Topological sorting

**LinkedIn Focus**: Degree of separation, connection recommendations, influence mapping

### 5. [Dynamic Programming](linkedin-dp.md)
**Priority: HIGH** - Advanced optimization problems
- 1D/2D DP patterns
- Knapsack variations
- String problems
- Optimization problems

**LinkedIn Focus**: Feed optimization, matching algorithms, resource allocation

### 6. [Hash Tables and Design](linkedin-hashmap-design.md)
**Priority: MEDIUM** - Data structure design questions
- LRU Cache
- LFU Cache
- Rate Limiter
- Custom data structures

**LinkedIn Focus**: Caching layers, API rate limiting, session management

### 7. [Heaps and Priority Queues](linkedin-heaps.md)
**Priority: MEDIUM** - Ranking and prioritization
- Top K problems
- Median finding
- Merge K sorted
- Task scheduling

**LinkedIn Focus**: News feed ranking, job recommendations, trending posts

### 8. [Backtracking and Recursion](linkedin-backtracking.md)
**Priority: LOW-MEDIUM** - Problem-solving depth
- Combinations/Permutations
- Subset problems
- Constraint satisfaction
- Graph coloring

**LinkedIn Focus**: Search suggestions, matching algorithms

---

## Study Plan

### Week 1-2: Foundations (Must-Know)
- [ ] Two Sum and variants (3 sum, 4 sum)
- [ ] Sliding window maximum
- [ ] Merge intervals
- [ ] Valid parentheses
- [ ] Binary tree traversals
- [ ] Graph BFS/DFS basics

### Week 3-4: Core Patterns
- [ ] LRU Cache implementation
- [ ] Lowest Common Ancestor
- [ ] Word Search (2D grid DFS)
- [ ] Clone Graph
- [ ] Longest increasing subsequence
- [ ] Coin change

### Week 5-6: LinkedIn Specific
- [ ] Degree of separation (BFS in social graph)
- [ ] Friend recommendations
- [ ] News feed ranking
- [ ] Rate limiter design
- [ ] Autocomplete system
- [ ] Mutual friends

### Week 7-8: Advanced Optimization
- [ ] Edit distance
- [ ] Maximum profit with K transactions
- [ ] Word break II
- [ ] Alien dictionary
- [ ] Critical connections
- [ ] Minimum window substring

---

## Interview Tips

### Before the Interview
1. **Practice on a whiteboard** - LinkedIn uses virtual whiteboards
2. **Think out loud** - Explain your thought process
3. **Ask clarifying questions** - Don't assume requirements
4. **Consider edge cases** - Empty inputs, single elements, duplicates

### During Problem Solving
1. **Understand the problem** (2-3 min)
   - Restate in your own words
   - Ask about constraints (size, range, special cases)
   - Confirm expected output format

2. **Design approach** (5-7 min)
   - Discuss brute force first
   - Identify optimization opportunities
   - Explain time/space complexity

3. **Code implementation** (20-30 min)
   - Start with clean structure
   - Use meaningful variable names
   - Add comments for complex logic
   - Handle edge cases

4. **Testing** (5-10 min)
   - Walk through with example
   - Test edge cases
   - Discuss potential bugs

### Red Flags to Avoid
❌ Jumping to code without understanding  
❌ Not discussing trade-offs  
❌ Ignoring edge cases  
❌ Poor variable naming  
❌ Not testing the solution  
❌ Getting stuck without asking for hints  

---

## Difficulty Distribution

### Easy (Warmup - 15%)
Foundation problems, mostly arrays/strings, single pattern

### Medium (Core - 70%)
Combination of 2-3 patterns, optimization required

### Hard (Senior - 15%)
Complex algorithms, multiple optimizations, design considerations

---

## Real Interview Questions Breakdown

### Most Frequently Asked (Last 2 Years)
1. **Degree of Separation** - Graph BFS (Asked ~30% of candidates)
2. **LRU Cache** - Design + Hash Map (Asked ~25%)
3. **Merge Intervals** - Array sorting (Asked ~20%)
4. **Clone Graph** - Graph traversal (Asked ~20%)
5. **Maximum Subarray** - Kadane's algorithm (Asked ~15%)

### LinkedIn-Specific Patterns
- **Social graph problems**: Connections, recommendations
- **Feed optimization**: Ranking, filtering, pagination
- **Rate limiting**: Design problems
- **Search/autocomplete**: Trie, prefix matching
- **Caching**: LRU, distributed caching

---

## Next Steps

1. Start with [Arrays and Strings](linkedin-arrays-strings.md) - highest frequency
2. Master [Graphs](linkedin-graphs.md) - LinkedIn's core domain
3. Practice [Dynamic Programming](linkedin-dp.md) - separates senior candidates
4. Build [System Design](linkedin-hashmap-design.md) understanding

---

## Resources

- **Practice Platforms**: LeetCode (LinkedIn tag), HackerRank
- **Mock Interviews**: Pramp, Interviewing.io
- **Study Groups**: AlgoExpert, Blind 75

**Remember**: LinkedIn values clear communication and practical problem-solving over just finding the optimal solution. Explain your thinking!
