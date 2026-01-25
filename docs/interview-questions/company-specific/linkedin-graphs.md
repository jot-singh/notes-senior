# LinkedIn Interview Questions: Graphs

**CRITICAL PRIORITY** - Graphs are LinkedIn's core domain. These are the most important questions for LinkedIn interviews since the entire platform is built on social graph concepts.

---

## Must-Practice Problems

### 1. Number of Islands (Medium) ⭐⭐⭐⭐⭐
**Frequency**: Very High | **Pattern**: DFS/BFS on 2D Grid

```java
/*
Problem: Count number of islands in 2D grid ('1' = land, '0' = water).

Example:
Input: grid = [
  ["1","1","0","0","0"],
  ["1","1","0","0","0"],
  ["0","0","1","0","0"],
  ["0","0","0","1","1"]
]
Output: 3

LinkedIn Context: Finding connected components in networks
*/

public int numIslands(char[][] grid) {
    if (grid == null || grid.length == 0) return 0;
    
    int count = 0;
    int rows = grid.length;
    int cols = grid[0].length;
    
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            if (grid[i][j] == '1') {
                count++;
                dfs(grid, i, j);
            }
        }
    }
    
    return count;
}

private void dfs(char[][] grid, int i, int j) {
    if (i < 0 || i >= grid.length || j < 0 || j >= grid[0].length || 
        grid[i][j] == '0') {
        return;
    }
    
    grid[i][j] = '0';  // Mark as visited
    
    // Explore all 4 directions
    dfs(grid, i + 1, j);
    dfs(grid, i - 1, j);
    dfs(grid, i, j + 1);
    dfs(grid, i, j - 1);
}

// Time: O(m × n)
// Space: O(m × n) for recursion stack
```

---

### 2. Clone Graph (Medium) ⭐⭐⭐⭐⭐
**Frequency**: Very High | **Pattern**: Graph Traversal + Hash Map

```java
/*
Problem: Deep copy undirected graph.

LinkedIn Context: Copying connection networks, duplicating graph structures
*/

class Node {
    public int val;
    public List<Node> neighbors;
    
    public Node(int _val) {
        val = _val;
        neighbors = new ArrayList<>();
    }
}

public class Solution {
    private Map<Node, Node> visited = new HashMap<>();
    
    public Node cloneGraph(Node node) {
        if (node == null) return null;
        
        if (visited.containsKey(node)) {
            return visited.get(node);
        }
        
        // Create clone
        Node clone = new Node(node.val);
        visited.put(node, clone);
        
        // Clone neighbors
        for (Node neighbor : node.neighbors) {
            clone.neighbors.add(cloneGraph(neighbor));
        }
        
        return clone;
    }
    
    // BFS approach
    public Node cloneGraphBFS(Node node) {
        if (node == null) return null;
        
        Map<Node, Node> visited = new HashMap<>();
        Queue<Node> queue = new LinkedList<>();
        
        visited.put(node, new Node(node.val));
        queue.offer(node);
        
        while (!queue.isEmpty()) {
            Node current = queue.poll();
            
            for (Node neighbor : current.neighbors) {
                if (!visited.containsKey(neighbor)) {
                    visited.put(neighbor, new Node(neighbor.val));
                    queue.offer(neighbor);
                }
                visited.get(current).neighbors.add(visited.get(neighbor));
            }
        }
        
        return visited.get(node);
    }
}

// Time: O(V + E)
// Space: O(V)
```

---

### 3. Course Schedule (Medium) ⭐⭐⭐⭐⭐
**Frequency**: Very High | **Pattern**: Topological Sort + Cycle Detection

```java
/*
Problem: Determine if you can finish all courses given prerequisites.

Example:
Input: numCourses = 2, prerequisites = [[1,0]]
Output: true (take course 0, then course 1)

LinkedIn Context: Dependency resolution, skill prerequisites
*/

public boolean canFinish(int numCourses, int[][] prerequisites) {
    // Build adjacency list
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) {
        graph.add(new ArrayList<>());
    }
    
    int[] inDegree = new int[numCourses];
    
    for (int[] prereq : prerequisites) {
        int course = prereq[0];
        int prerequisite = prereq[1];
        graph.get(prerequisite).add(course);
        inDegree[course]++;
    }
    
    // BFS with courses having no prerequisites
    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) {
        if (inDegree[i] == 0) {
            queue.offer(i);
        }
    }
    
    int completed = 0;
    while (!queue.isEmpty()) {
        int course = queue.poll();
        completed++;
        
        for (int next : graph.get(course)) {
            inDegree[next]--;
            if (inDegree[next] == 0) {
                queue.offer(next);
            }
        }
    }
    
    return completed == numCourses;
}

// Follow-up: Return the course order
public int[] findOrder(int numCourses, int[][] prerequisites) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) {
        graph.add(new ArrayList<>());
    }
    
    int[] inDegree = new int[numCourses];
    
    for (int[] prereq : prerequisites) {
        graph.get(prereq[1]).add(prereq[0]);
        inDegree[prereq[0]]++;
    }
    
    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) {
        if (inDegree[i] == 0) {
            queue.offer(i);
        }
    }
    
    int[] order = new int[numCourses];
    int index = 0;
    
    while (!queue.isEmpty()) {
        int course = queue.poll();
        order[index++] = course;
        
        for (int next : graph.get(course)) {
            inDegree[next]--;
            if (inDegree[next] == 0) {
                queue.offer(next);
            }
        }
    }
    
    return index == numCourses ? order : new int[0];
}

// Time: O(V + E)
// Space: O(V + E)
```

---

### 4. Word Ladder (Hard) ⭐⭐⭐⭐⭐
**Frequency**: High | **Pattern**: BFS + Graph

```java
/*
Problem: Find shortest transformation sequence from beginWord to endWord.
Each transformation changes one letter and must be in wordList.

Example:
Input: beginWord = "hit", endWord = "cog", 
       wordList = ["hot","dot","dog","lot","log","cog"]
Output: 5 ("hit" -> "hot" -> "dot" -> "dog" -> "cog")

LinkedIn Context: Skill transformation paths, connection chains
*/

public int ladderLength(String beginWord, String endWord, List<String> wordList) {
    Set<String> wordSet = new HashSet<>(wordList);
    if (!wordSet.contains(endWord)) return 0;
    
    Queue<String> queue = new LinkedList<>();
    queue.offer(beginWord);
    
    int level = 1;
    
    while (!queue.isEmpty()) {
        int size = queue.size();
        
        for (int i = 0; i < size; i++) {
            String word = queue.poll();
            
            if (word.equals(endWord)) {
                return level;
            }
            
            // Try changing each character
            char[] chars = word.toCharArray();
            for (int j = 0; j < chars.length; j++) {
                char original = chars[j];
                
                // Try all 26 letters
                for (char c = 'a'; c <= 'z'; c++) {
                    if (c == original) continue;
                    
                    chars[j] = c;
                    String newWord = new String(chars);
                    
                    if (wordSet.contains(newWord)) {
                        queue.offer(newWord);
                        wordSet.remove(newWord);  // Mark as visited
                    }
                }
                
                chars[j] = original;  // Restore
            }
        }
        
        level++;
    }
    
    return 0;
}

// Time: O(M² × N) where M = word length, N = wordList size
// Space: O(M × N)

// Bidirectional BFS optimization (much faster!)
public int ladderLengthBidirectional(String beginWord, String endWord, 
                                     List<String> wordList) {
    Set<String> wordSet = new HashSet<>(wordList);
    if (!wordSet.contains(endWord)) return 0;
    
    Set<String> beginSet = new HashSet<>();
    Set<String> endSet = new HashSet<>();
    beginSet.add(beginWord);
    endSet.add(endWord);
    
    int level = 1;
    
    while (!beginSet.isEmpty() && !endSet.isEmpty()) {
        // Always expand smaller set
        if (beginSet.size() > endSet.size()) {
            Set<String> temp = beginSet;
            beginSet = endSet;
            endSet = temp;
        }
        
        Set<String> nextLevel = new HashSet<>();
        
        for (String word : beginSet) {
            char[] chars = word.toCharArray();
            
            for (int i = 0; i < chars.length; i++) {
                char original = chars[i];
                
                for (char c = 'a'; c <= 'z'; c++) {
                    chars[i] = c;
                    String newWord = new String(chars);
                    
                    if (endSet.contains(newWord)) {
                        return level + 1;
                    }
                    
                    if (wordSet.contains(newWord)) {
                        nextLevel.add(newWord);
                        wordSet.remove(newWord);
                    }
                }
                
                chars[i] = original;
            }
        }
        
        beginSet = nextLevel;
        level++;
    }
    
    return 0;
}
```

---

### 5. Alien Dictionary (Hard) ⭐⭐⭐⭐⭐
**Frequency**: High | **Pattern**: Topological Sort

```java
/*
Problem: Derive order of characters from sorted alien dictionary words.

Example:
Input: words = ["wrt","wrf","er","ett","rftt"]
Output: "wertf"

LinkedIn Context: Custom ordering systems, ranking algorithms
*/

public String alienOrder(String[] words) {
    // Build graph
    Map<Character, Set<Character>> graph = new HashMap<>();
    Map<Character, Integer> inDegree = new HashMap<>();
    
    // Initialize
    for (String word : words) {
        for (char c : word.toCharArray()) {
            graph.putIfAbsent(c, new HashSet<>());
            inDegree.putIfAbsent(c, 0);
        }
    }
    
    // Add edges
    for (int i = 0; i < words.length - 1; i++) {
        String word1 = words[i];
        String word2 = words[i + 1];
        int minLen = Math.min(word1.length(), word2.length());
        
        // Check invalid case: prefix longer than word
        if (word1.length() > word2.length() && 
            word1.startsWith(word2)) {
            return "";
        }
        
        // Find first different character
        for (int j = 0; j < minLen; j++) {
            char c1 = word1.charAt(j);
            char c2 = word2.charAt(j);
            
            if (c1 != c2) {
                if (!graph.get(c1).contains(c2)) {
                    graph.get(c1).add(c2);
                    inDegree.put(c2, inDegree.get(c2) + 1);
                }
                break;
            }
        }
    }
    
    // Topological sort
    Queue<Character> queue = new LinkedList<>();
    for (char c : inDegree.keySet()) {
        if (inDegree.get(c) == 0) {
            queue.offer(c);
        }
    }
    
    StringBuilder result = new StringBuilder();
    
    while (!queue.isEmpty()) {
        char c = queue.poll();
        result.append(c);
        
        for (char neighbor : graph.get(c)) {
            inDegree.put(neighbor, inDegree.get(neighbor) - 1);
            if (inDegree.get(neighbor) == 0) {
                queue.offer(neighbor);
            }
        }
    }
    
    return result.length() == inDegree.size() ? result.toString() : "";
}

// Time: O(C) where C = total characters in all words
// Space: O(1) - at most 26 characters
```

---

### 6. Network Delay Time (Medium) ⭐⭐⭐⭐
**Frequency**: High | **Pattern**: Dijkstra's Algorithm

```java
/*
Problem: Time for signal to reach all nodes from source.
Return minimum time or -1 if impossible.

Example:
Input: times = [[2,1,1],[2,3,1],[3,4,1]], n = 4, k = 2
Output: 2

LinkedIn Context: Message propagation, notification delivery
*/

public int networkDelayTime(int[][] times, int n, int k) {
    // Build adjacency list
    Map<Integer, List<int[]>> graph = new HashMap<>();
    for (int[] time : times) {
        graph.computeIfAbsent(time[0], x -> new ArrayList<>())
             .add(new int[]{time[1], time[2]});
    }
    
    // Dijkstra's algorithm
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    pq.offer(new int[]{k, 0});  // [node, time]
    
    Map<Integer, Integer> dist = new HashMap<>();
    
    while (!pq.isEmpty()) {
        int[] current = pq.poll();
        int node = current[0];
        int time = current[1];
        
        if (dist.containsKey(node)) continue;
        dist.put(node, time);
        
        if (graph.containsKey(node)) {
            for (int[] neighbor : graph.get(node)) {
                int nextNode = neighbor[0];
                int nextTime = time + neighbor[1];
                
                if (!dist.containsKey(nextNode)) {
                    pq.offer(new int[]{nextNode, nextTime});
                }
            }
        }
    }
    
    if (dist.size() != n) return -1;
    
    int maxTime = 0;
    for (int time : dist.values()) {
        maxTime = Math.max(maxTime, time);
    }
    
    return maxTime;
}

// Time: O(E log V)
// Space: O(V + E)
```

---

### 7. Critical Connections (Hard) ⭐⭐⭐⭐
**Frequency**: Medium-High | **Pattern**: Tarjan's Algorithm

```java
/*
Problem: Find all critical connections (bridges) in network.
A bridge is an edge whose removal disconnects the graph.

Example:
Input: n = 4, connections = [[0,1],[1,2],[2,0],[1,3]]
Output: [[1,3]]

LinkedIn Context: Network reliability, single points of failure
*/

public List<List<Integer>> criticalConnections(int n, List<List<Integer>> connections) {
    List<List<Integer>> graph = new ArrayList<>();
    for (int i = 0; i < n; i++) {
        graph.add(new ArrayList<>());
    }
    
    for (List<Integer> conn : connections) {
        graph.get(conn.get(0)).add(conn.get(1));
        graph.get(conn.get(1)).add(conn.get(0));
    }
    
    int[] discovery = new int[n];
    int[] low = new int[n];
    boolean[] visited = new boolean[n];
    List<List<Integer>> result = new ArrayList<>();
    
    int[] time = new int[]{0};
    
    dfs(0, -1, graph, discovery, low, visited, time, result);
    
    return result;
}

private void dfs(int node, int parent, List<List<Integer>> graph,
                int[] discovery, int[] low, boolean[] visited,
                int[] time, List<List<Integer>> result) {
    visited[node] = true;
    discovery[node] = low[node] = time[0]++;
    
    for (int neighbor : graph.get(node)) {
        if (neighbor == parent) continue;
        
        if (!visited[neighbor]) {
            dfs(neighbor, node, graph, discovery, low, visited, time, result);
            low[node] = Math.min(low[node], low[neighbor]);
            
            // Bridge condition
            if (low[neighbor] > discovery[node]) {
                result.add(Arrays.asList(node, neighbor));
            }
        } else {
            low[node] = Math.min(low[node], discovery[neighbor]);
        }
    }
}

// Time: O(V + E)
// Space: O(V + E)
```

---

### 8. Evaluate Division (Medium) ⭐⭐⭐⭐
**Frequency**: Medium | **Pattern**: Graph with Weighted Edges

```java
/*
Problem: Given equations like a/b=2.0, b/c=3.0, answer queries like a/c=?

LinkedIn Context: Relationship inference, transitive connections
*/

public double[] calcEquation(List<List<String>> equations, double[] values,
                             List<List<String>> queries) {
    // Build graph
    Map<String, Map<String, Double>> graph = new HashMap<>();
    
    for (int i = 0; i < equations.size(); i++) {
        String a = equations.get(i).get(0);
        String b = equations.get(i).get(1);
        double value = values[i];
        
        graph.computeIfAbsent(a, k -> new HashMap<>()).put(b, value);
        graph.computeIfAbsent(b, k -> new HashMap<>()).put(a, 1.0 / value);
    }
    
    double[] results = new double[queries.size()];
    
    for (int i = 0; i < queries.size(); i++) {
        String start = queries.get(i).get(0);
        String end = queries.get(i).get(1);
        
        if (!graph.containsKey(start) || !graph.containsKey(end)) {
            results[i] = -1.0;
        } else if (start.equals(end)) {
            results[i] = 1.0;
        } else {
            Set<String> visited = new HashSet<>();
            results[i] = dfs(graph, start, end, 1.0, visited);
        }
    }
    
    return results;
}

private double dfs(Map<String, Map<String, Double>> graph,
                  String current, String target, double product,
                  Set<String> visited) {
    if (current.equals(target)) {
        return product;
    }
    
    visited.add(current);
    
    for (Map.Entry<String, Double> entry : graph.get(current).entrySet()) {
        String neighbor = entry.getKey();
        if (!visited.contains(neighbor)) {
            double result = dfs(graph, neighbor, target, 
                              product * entry.getValue(), visited);
            if (result != -1.0) {
                return result;
            }
        }
    }
    
    return -1.0;
}

// Time: O(equations + queries × V)
// Space: O(equations)
```

---

## LinkedIn-Specific Graph Problems

### 9. Degree of Separation (Real LinkedIn Problem) ⭐⭐⭐⭐⭐
**Frequency**: VERY HIGH

```java
/*
Problem: Find shortest connection path between two users.
This is LinkedIn's famous "How you're connected" feature.

Example: You -> Friend -> Friend's Friend -> Target User = 3rd degree
*/

public class DegreeOfSeparation {
    public int findDegree(Map<Integer, List<Integer>> connections, 
                         int user1, int user2) {
        if (user1 == user2) return 0;
        
        Queue<Integer> queue = new LinkedList<>();
        Set<Integer> visited = new HashSet<>();
        queue.offer(user1);
        visited.add(user1);
        
        int degree = 0;
        
        while (!queue.isEmpty()) {
            int size = queue.size();
            degree++;
            
            for (int i = 0; i < size; i++) {
                int user = queue.poll();
                
                for (int connection : connections.getOrDefault(user, new ArrayList<>())) {
                    if (connection == user2) {
                        return degree;
                    }
                    
                    if (!visited.contains(connection)) {
                        visited.add(connection);
                        queue.offer(connection);
                    }
                }
            }
        }
        
        return -1;  // Not connected
    }
    
    // Follow-up: Return the path
    public List<Integer> findConnectionPath(Map<Integer, List<Integer>> connections,
                                           int user1, int user2) {
        if (user1 == user2) return Arrays.asList(user1);
        
        Queue<Integer> queue = new LinkedList<>();
        Map<Integer, Integer> parent = new HashMap<>();
        queue.offer(user1);
        parent.put(user1, -1);
        
        while (!queue.isEmpty()) {
            int user = queue.poll();
            
            for (int connection : connections.getOrDefault(user, new ArrayList<>())) {
                if (!parent.containsKey(connection)) {
                    parent.put(connection, user);
                    queue.offer(connection);
                    
                    if (connection == user2) {
                        return reconstructPath(parent, user1, user2);
                    }
                }
            }
        }
        
        return new ArrayList<>();
    }
    
    private List<Integer> reconstructPath(Map<Integer, Integer> parent,
                                         int start, int end) {
        List<Integer> path = new ArrayList<>();
        int current = end;
        
        while (current != -1) {
            path.add(0, current);
            current = parent.get(current);
        }
        
        return path;
    }
}
```

---

## Interview Tips for Graph Problems

### Pattern Recognition
1. **BFS**: Shortest path, level-order, degree of separation
2. **DFS**: Connected components, cycle detection, paths
3. **Topological Sort**: Dependencies, prerequisites
4. **Dijkstra**: Weighted shortest path
5. **Union-Find**: Dynamic connectivity

### Common Mistakes
❌ Forgetting to mark nodes as visited (infinite loops)  
❌ Not handling disconnected graphs  
❌ Confusing directed vs undirected edges  
❌ Not checking for cycles in topological sort  

### LinkedIn-Specific Insights
- Always think about "connections" and "relationships"
- Most problems involve finding paths or analyzing network structure
- Bidirectional BFS is often the optimization they're looking for
- Understand trade-offs between BFS (shortest path) and DFS (memory efficient)

---

## Next: [Dynamic Programming](linkedin-dp.md)
