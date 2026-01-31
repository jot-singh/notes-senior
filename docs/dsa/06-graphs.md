# Graphs Fundamentals

## What You'll Learn
- Graph representations and when to use each
- BFS and DFS traversal with problem-solving patterns
- Topological sorting for dependency resolution
- Shortest path algorithms (BFS, Dijkstra's, Bellman-Ford)
- Cycle detection and bipartite checking
- Union-Find for connected components
- Minimum spanning trees (Kruskal's, Prim's)
- Real-world applications: social networks, maps, recommendations

## Why This Matters

Graphs model **relationships without hierarchy**—the most flexible structure for representing connections. They're everywhere: social networks (LinkedIn connections), route planning (Google Maps), dependency resolution (build systems), recommendation engines (Netflix), web crawlers (page links), network topology (routers). Unlike trees with rigid parent-child relationships, graphs capture arbitrary relationships. Mastering graphs means understanding how the connected world works and solving problems from scheduling to pathfinding to influence mapping.

---

## Graph Fundamentals

### What Makes Graphs Unique

Unlike trees, graphs allow:
- **Cycles**: Nodes can form loops
- **Multiple paths**: Two nodes can be connected multiple ways
- **Directionality**: Edges can be one-way or bidirectional
- **Weights**: Edges can have costs/distances

```java
// Graph terminology
public class Graph {
    /*
    Undirected Graph:           Directed Graph:
        1 --- 2                     1 ---> 2
        |     |                     |      |
        3 --- 4                     v      v
                                    3 ---> 4
    
    Undirected: edges work both ways (1-2 and 2-1)
    Directed: edges work one way (1->2 but not 2->1)
    
    Weighted: edges have costs
        1 ---(5)--- 2
        |           |
       (3)         (4)
        |           |
        3 ---(2)--- 4
    
    Properties:
    - Node/Vertex: A point in the graph
    - Edge: Connection between nodes
    - Degree: Number of edges connected to node
    - Path: Sequence of edges connecting nodes
    - Cycle: Path that starts and ends at same node
    */
}
```

---

## Graph Representations

### Representation 1: Adjacency Matrix

**When to use**: Dense graphs (many edges), quick edge lookup, dense adjacency queries.

```java
public class AdjacencyMatrix {
    private int[][] matrix;
    private int numNodes;
    
    public AdjacencyMatrix(int n) {
        numNodes = n;
        matrix = new int[n][n];
        // matrix[i][j] = weight of edge from i to j (0 if no edge)
    }
    
    public void addEdge(int u, int v, int weight) {
        matrix[u][v] = weight;      // Directed
        matrix[v][u] = weight;      // Undirected (both directions)
    }
    
    public void removeEdge(int u, int v) {
        matrix[u][v] = 0;
        matrix[v][u] = 0;
    }
    
    public boolean hasEdge(int u, int v) {
        return matrix[u][v] != 0;
    }
    
    // Get all neighbors
    public List<Integer> getNeighbors(int u) {
        List<Integer> neighbors = new ArrayList<>();
        for (int v = 0; v < numNodes; v++) {
            if (matrix[u][v] != 0) {
                neighbors.add(v);
            }
        }
        return neighbors;
    }
}

// Space: O(V²) where V is number of nodes
// Edge lookup: O(1)
// Getting neighbors: O(V)
// Used for: Dense graphs, all-pairs shortest path, transitivity closure
```

**Trade-offs**: 
- ✅ O(1) edge lookup
- ❌ O(V²) space (wasteful for sparse graphs)
- ❌ O(V) to iterate neighbors

### Representation 2: Adjacency List

**When to use**: Sparse graphs (few edges), most real-world applications.

```java
public class AdjacencyList {
    private Map<Integer, List<Integer>> graph;
    
    public AdjacencyList(int n) {
        graph = new HashMap<>();
        for (int i = 0; i < n; i++) {
            graph.put(i, new ArrayList<>());
        }
    }
    
    public void addEdge(int u, int v) {
        graph.get(u).add(v);           // Directed
        graph.get(v).add(u);           // Undirected
    }
    
    public void removeEdge(int u, int v) {
        graph.get(u).remove(Integer.valueOf(v));
        graph.get(v).remove(Integer.valueOf(u));
    }
    
    public List<Integer> getNeighbors(int u) {
        return graph.get(u);
    }
    
    public boolean hasEdge(int u, int v) {
        return graph.get(u).contains(v);
    }
}

// Space: O(V + E) where V = nodes, E = edges
// Edge lookup: O(degree) average, O(V) worst
// Getting neighbors: O(1) - direct access to list
// Used for: Sparse graphs, BFS/DFS, most practical applications
```

**Trade-offs**:
- ✅ O(V + E) space (efficient for sparse graphs)
- ✅ Fast neighbor iteration
- ❌ O(degree) edge lookup

---

## Graph Traversals

### Pattern 1: BFS (Breadth-First Search)

**When to use**: Finding shortest path in unweighted graph, level-order exploration, flood fill.

**The Intuition**: Explore all neighbors of current node before moving deeper. Uses queue to maintain "frontier" of nodes to visit.

```java
public class BFS {
    // Explore all reachable nodes, distance from start
    public Map<Integer, Integer> bfs(AdjacencyList graph, int start) {
        Map<Integer, Integer> distance = new HashMap<>();
        Queue<Integer> queue = new LinkedList<>();
        Set<Integer> visited = new HashSet<>();
        
        queue.offer(start);
        visited.add(start);
        distance.put(start, 0);
        
        while (!queue.isEmpty()) {
            int node = queue.poll();
            
            for (int neighbor : graph.getNeighbors(node)) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    distance.put(neighbor, distance.get(node) + 1);
                    queue.offer(neighbor);
                }
            }
        }
        
        return distance;
    }
    
    // Find shortest path from source to target
    public List<Integer> shortestPath(AdjacencyList graph, int source, int target) {
        if (source == target) return List.of(source);
        
        Map<Integer, Integer> parent = new HashMap<>();
        Queue<Integer> queue = new LinkedList<>();
        Set<Integer> visited = new HashSet<>();
        
        queue.offer(source);
        visited.add(source);
        parent.put(source, -1);
        
        while (!queue.isEmpty()) {
            int node = queue.poll();
            
            if (node == target) {
                // Reconstruct path
                List<Integer> path = new ArrayList<>();
                int current = target;
                while (current != -1) {
                    path.add(0, current);
                    current = parent.get(current);
                }
                return path;
            }
            
            for (int neighbor : graph.getNeighbors(node)) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    parent.put(neighbor, node);
                    queue.offer(neighbor);
                }
            }
        }
        
        return new ArrayList<>();  // No path
    }
}

// Time: O(V + E) - visit each node once, each edge once
// Space: O(V) for queue and visited set
```

### Pattern 2: DFS (Depth-First Search)

**When to use**: Cycle detection, connected components, topological sort, backtracking problems.

**The Intuition**: Explore as far as possible along each branch before backtracking. Uses recursion (implicit stack) or explicit stack.

```java
public class DFS {
    // Explore all reachable nodes
    public void dfs(AdjacencyList graph, int start) {
        Set<Integer> visited = new HashSet<>();
        dfsHelper(graph, start, visited);
    }
    
    private void dfsHelper(AdjacencyList graph, int node, Set<Integer> visited) {
        visited.add(node);
        System.out.println(node);
        
        for (int neighbor : graph.getNeighbors(node)) {
            if (!visited.contains(neighbor)) {
                dfsHelper(graph, neighbor, visited);
            }
        }
    }
    
    // Detect cycle in directed graph
    public boolean hasCycle(AdjacencyList graph, int numNodes) {
        Set<Integer> visited = new HashSet<>();
        Set<Integer> recursionStack = new HashSet<>();
        
        for (int i = 0; i < numNodes; i++) {
            if (!visited.contains(i)) {
                if (hasCycleDFS(graph, i, visited, recursionStack)) {
                    return true;
                }
            }
        }
        
        return false;
    }
    
    private boolean hasCycleDFS(AdjacencyList graph, int node, 
                                Set<Integer> visited, Set<Integer> stack) {
        visited.add(node);
        stack.add(node);
        
        for (int neighbor : graph.getNeighbors(node)) {
            if (!visited.contains(neighbor)) {
                if (hasCycleDFS(graph, neighbor, visited, stack)) {
                    return true;
                }
            } else if (stack.contains(neighbor)) {
                // Back edge = cycle
                return true;
            }
        }
        
        stack.remove(node);
        return false;
    }
    
    // Find connected components
    public int countComponents(AdjacencyList graph, int numNodes) {
        Set<Integer> visited = new HashSet<>();
        int count = 0;
        
        for (int i = 0; i < numNodes; i++) {
            if (!visited.contains(i)) {
                dfsComponentHelper(graph, i, visited);
                count++;
            }
        }
        
        return count;
    }
    
    private void dfsComponentHelper(AdjacencyList graph, int node, 
                                    Set<Integer> visited) {
        visited.add(node);
        for (int neighbor : graph.getNeighbors(node)) {
            if (!visited.contains(neighbor)) {
                dfsComponentHelper(graph, neighbor, visited);
            }
        }
    }
}

// Time: O(V + E)
// Space: O(V) for recursion stack and visited set
```

### Pattern 3: Topological Sorting

**When to use**: Dependency resolution, build systems, course scheduling, DAG processing.

```java
// Kahn's Algorithm (DFS-based would be Kosaraju/Tarjan)
public class TopologicalSort {
    // Valid for DAGs only (Directed Acyclic Graphs)
    public List<Integer> topologicalSort(AdjacencyList graph, int numNodes) {
        // In-degree: number of incoming edges
        int[] inDegree = new int[numNodes];
        for (int i = 0; i < numNodes; i++) {
            for (int neighbor : graph.getNeighbors(i)) {
                inDegree[neighbor]++;
            }
        }
        
        // Start with nodes having no dependencies
        Queue<Integer> queue = new LinkedList<>();
        for (int i = 0; i < numNodes; i++) {
            if (inDegree[i] == 0) {
                queue.offer(i);
            }
        }
        
        List<Integer> result = new ArrayList<>();
        while (!queue.isEmpty()) {
            int node = queue.poll();
            result.add(node);
            
            // Process all neighbors (dependent nodes)
            for (int neighbor : graph.getNeighbors(node)) {
                inDegree[neighbor]--;
                if (inDegree[neighbor] == 0) {
                    queue.offer(neighbor);
                }
            }
        }
        
        // If result.size() != numNodes, cycle exists
        return result.size() == numNodes ? result : new ArrayList<>();
    }
}

// Example: Course Prerequisites
// Course 0 requires Course 1
// Course 1 requires Course 2
// Topological order: [2, 1, 0] (take 2, then 1, then 0)

// Time: O(V + E)
// Space: O(V)
// Used in: Build systems (Make, Gradle), dependency management, scheduling
```

---

## Real-World Applications

### 1. **Social Networks**

```java
public class SocialNetwork {
    private AdjacencyList graph;
    
    // Find all friends of a person (1 degree)
    public List<Integer> getFriends(int person) {
        return graph.getNeighbors(person);
    }
    
    // Find all friends of friends (2 degrees)
    public Set<Integer> getFriendsOfFriends(int person) {
        Set<Integer> result = new HashSet<>();
        
        for (int friend : getFriends(person)) {
            for (int foaf : getFriends(friend)) {
                if (foaf != person) {
                    result.add(foaf);
                }
            }
        }
        
        return result;
    }
    
    // Find connection between two people (shortest path)
    public List<Integer> findConnection(int person1, int person2) {
        BFS bfs = new BFS();
        return bfs.shortestPath(graph, person1, person2);
    }
    
    // Find communities (connected components)
    public int countCommunities(int numPeople) {
        DFS dfs = new DFS();
        return dfs.countComponents(graph, numPeople);
    }
}

// Used in: LinkedIn, Facebook, Twitter friend recommendations
```

### 2. **Maps and Route Planning**

```java
public class MapNavigation {
    private Map<String, List<Route>> graph;  // City -> Routes to neighbors
    
    public class Route {
        String destination;
        double distance;
        
        Route(String dest, double dist) {
            destination = dest;
            distance = dist;
        }
    }
    
    // Find nearest neighbors (BFS)
    public List<String> nearbyLocations(String start, int maxDistance) {
        List<String> result = new ArrayList<>();
        Queue<String> queue = new LinkedList<>();
        Map<String, Double> distances = new HashMap<>();
        
        queue.offer(start);
        distances.put(start, 0.0);
        
        while (!queue.isEmpty()) {
            String location = queue.poll();
            double currentDistance = distances.get(location);
            
            if (currentDistance <= maxDistance) {
                result.add(location);
                
                for (Route route : graph.get(location)) {
                    double newDistance = currentDistance + route.distance;
                    
                    if (newDistance <= maxDistance && 
                        !distances.containsKey(route.destination)) {
                        distances.put(route.destination, newDistance);
                        queue.offer(route.destination);
                    }
                }
            }
        }
        
        return result;
    }
}

// Used in: Google Maps, Uber routing, GPS navigation
```

### 3. **Recommendation Engine**

```java
public class RecommendationEngine {
    // Graph: Products -> Similar Products (or Users -> Similar Interests)
    
    public List<String> recommendProducts(String productId, int depth) {
        Set<String> recommendations = new HashSet<>();
        DFS dfs = new DFS();
        
        // Find similar products up to depth N
        Queue<String> queue = new LinkedList<>();
        Set<String> visited = new HashSet<>();
        Map<String, Integer> currentDepth = new HashMap<>();
        
        queue.offer(productId);
        visited.add(productId);
        currentDepth.put(productId, 0);
        
        while (!queue.isEmpty()) {
            String product = queue.poll();
            int d = currentDepth.get(product);
            
            if (d < depth) {
                for (String similar : getSimilarProducts(product)) {
                    if (!visited.contains(similar)) {
                        visited.add(similar);
                        currentDepth.put(similar, d + 1);
                        recommendations.add(similar);
                        queue.offer(similar);
                    }
                }
            }
        }
        
        return new ArrayList<>(recommendations);
    }
    
    private List<String> getSimilarProducts(String product) {
        // Would fetch from graph
        return new ArrayList<>();
    }
}

// Used in: Amazon, Netflix, Spotify recommendations
```

### 4. **Build Dependency Resolution**

```java
public class BuildSystem {
    // Tasks with dependencies (DAG)
    
    public List<String> buildOrder(Map<String, List<String>> dependencies) {
        // Build order respects dependencies
        // dependencies: task -> list of tasks it depends on
        
        int numTasks = dependencies.size();
        Map<String, Integer> inDegree = new HashMap<>();
        
        // Initialize
        for (String task : dependencies.keySet()) {
            inDegree.putIfAbsent(task, 0);
            for (String dep : dependencies.get(task)) {
                inDegree.put(dep, inDegree.getOrDefault(dep, 0) + 1);
            }
        }
        
        // Find tasks with no dependencies
        Queue<String> queue = new LinkedList<>();
        for (String task : inDegree.keySet()) {
            if (inDegree.get(task) == 0) {
                queue.offer(task);
            }
        }
        
        List<String> buildOrder = new ArrayList<>();
        while (!queue.isEmpty()) {
            String task = queue.poll();
            buildOrder.add(task);
            
            for (String dependent : dependencies.getOrDefault(task, new ArrayList<>())) {
                inDegree.put(dependent, inDegree.get(dependent) - 1);
                if (inDegree.get(dependent) == 0) {
                    queue.offer(dependent);
                }
            }
        }
        
        return buildOrder.size() == numTasks ? buildOrder : new ArrayList<>();
    }
}

// Used in: Maven, Gradle, Make, build systems
```

---

## Graph Problem Decision Tree

```
What am I solving?

├─ Shortest path (unweighted)?
│  └─ BFS
│
├─ Shortest path (weighted)?
│  └─ Dijkstra's algorithm
│
├─ All pairs shortest path?
│  └─ Floyd-Warshall
│
├─ Detect cycle?
│  └─ DFS with recursion stack
│
├─ Connected components?
│  └─ DFS from each unvisited node
│
├─ Topological order (DAG)?
│  └─ Kahn's algorithm or DFS post-order
│
├─ Strongly connected components?
│  └─ Kosaraju or Tarjan
│
└─ Minimum spanning tree?
   └─ Kruskal's or Prim's algorithm
```

---

## Common Mistakes (Anti-Patterns)

### ❌ Forgetting to Track Visited Nodes
```java
// ❌ Infinite loop if graph has cycles
public void dfs(int node) {
    System.out.println(node);
    for (int neighbor : graph.get(node)) {
        dfs(neighbor);  // No visited check = infinite recursion
    }
}

// ✅ Always track visited
public void dfs(int node, Set<Integer> visited) {
    visited.add(node);
    System.out.println(node);
    for (int neighbor : graph.get(node)) {
        if (!visited.contains(neighbor)) {
            dfs(neighbor, visited);
        }
    }
}
```

### ❌ Confusing Directed and Undirected
```java
// ❌ Treating directed as undirected
public void addEdge(int u, int v) {
    graph.get(u).add(v);
    graph.get(v).add(u);  // Wrong for directed graphs!
}

// ✅ Be explicit about directionality
public void addDirectedEdge(int u, int v) {
    graph.get(u).add(v);
}

public void addUndirectedEdge(int u, int v) {
    graph.get(u).add(v);
    graph.get(v).add(u);
}
```

### ❌ BFS for Shortest Path in Weighted Graphs
```java
// ❌ BFS doesn't work with weights
Queue<Integer> queue = new LinkedList<>();
queue.offer(start);

while (!queue.isEmpty()) {
    int node = queue.poll();
    // Won't find shortest path with weights!
}

// ✅ Use Dijkstra's for weighted graphs
// or weight-aware BFS with priority queue
```

---

## Complexity Analysis

| Algorithm | Time | Space | Notes |
|-----------|------|-------|-------|
| BFS | O(V + E) | O(V) | Shortest path (unweighted) |
| DFS | O(V + E) | O(V) | Recursion depth |
| Topological Sort | O(V + E) | O(V) | DAGs only |
| Cycle Detection | O(V + E) | O(V) | Directed graphs |
| Connected Components | O(V + E) | O(V) | Count separate graphs |

---

## Advanced Graph Algorithms

### Pattern 4: Dijkstra's Algorithm (Shortest Path in Weighted Graph)

**When to use**: Finding shortest path in weighted graph with non-negative weights.

```java
public class Dijkstra {
    // Find shortest distances from source to all nodes
    public Map<Integer, Integer> shortestPath(Map<Integer, List<Edge>> graph, int source) {
        Map<Integer, Integer> dist = new HashMap<>();
        
        // Min-heap: (distance, node)
        PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
        pq.offer(new int[]{0, source});
        dist.put(source, 0);
        
        while (!pq.isEmpty()) {
            int[] current = pq.poll();
            int d = current[0];
            int u = current[1];
            
            // Skip if we've found shorter path already
            if (d > dist.getOrDefault(u, Integer.MAX_VALUE)) {
                continue;
            }
            
            // Relax edges
            for (Edge edge : graph.getOrDefault(u, new ArrayList<>())) {
                int v = edge.to;
                int weight = edge.weight;
                int newDist = d + weight;
                
                if (newDist < dist.getOrDefault(v, Integer.MAX_VALUE)) {
                    dist.put(v, newDist);
                    pq.offer(new int[]{newDist, v});
                }
            }
        }
        
        return dist;
    }
    
    // With path reconstruction
    public List<Integer> shortestPathWithRoute(Map<Integer, List<Edge>> graph, 
                                               int source, int target) {
        Map<Integer, Integer> dist = new HashMap<>();
        Map<Integer, Integer> parent = new HashMap<>();
        
        PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);
        pq.offer(new int[]{0, source});
        dist.put(source, 0);
        parent.put(source, -1);
        
        while (!pq.isEmpty()) {
            int[] current = pq.poll();
            int d = current[0];
            int u = current[1];
            
            if (u == target) break;  // Found shortest path to target
            
            if (d > dist.getOrDefault(u, Integer.MAX_VALUE)) continue;
            
            for (Edge edge : graph.getOrDefault(u, new ArrayList<>())) {
                int v = edge.to;
                int newDist = d + edge.weight;
                
                if (newDist < dist.getOrDefault(v, Integer.MAX_VALUE)) {
                    dist.put(v, newDist);
                    parent.put(v, u);
                    pq.offer(new int[]{newDist, v});
                }
            }
        }
        
        // Reconstruct path
        List<Integer> path = new ArrayList<>();
        if (!parent.containsKey(target)) return path;  // No path
        
        int current = target;
        while (current != -1) {
            path.add(0, current);
            current = parent.get(current);
        }
        
        return path;
    }
    
    static class Edge {
        int to, weight;
        Edge(int to, int weight) {
            this.to = to;
            this.weight = weight;
        }
    }
}

// Time: O((V + E) log V) with binary heap
// Space: O(V) for distances and heap

// Why priority queue?
// Always process closest unvisited node
// Ensures we find shortest path first

// Example:
//     1 --(4)-- 2
//     |         |
//    (1)       (2)
//     |         |
//     3 --(5)-- 4
//
// From 1: dist[1]=0, dist[3]=1, dist[2]=4, dist[4]=3 (via 3→4)
```

### Pattern 5: Bellman-Ford (Shortest Path with Negative Weights)

**When to use**: Weighted graph with negative edges, detect negative cycles.

```java
public class BellmanFord {
    public Map<Integer, Integer> shortestPath(int n, List<Edge> edges, int source) {
        Map<Integer, Integer> dist = new HashMap<>();
        
        // Initialize distances
        for (int i = 0; i < n; i++) {
            dist.put(i, Integer.MAX_VALUE);
        }
        dist.put(source, 0);
        
        // Relax all edges V-1 times
        for (int i = 0; i < n - 1; i++) {
            for (Edge edge : edges) {
                if (dist.get(edge.from) != Integer.MAX_VALUE) {
                    int newDist = dist.get(edge.from) + edge.weight;
                    if (newDist < dist.get(edge.to)) {
                        dist.put(edge.to, newDist);
                    }
                }
            }
        }
        
        // Check for negative cycles
        for (Edge edge : edges) {
            if (dist.get(edge.from) != Integer.MAX_VALUE) {
                int newDist = dist.get(edge.from) + edge.weight;
                if (newDist < dist.get(edge.to)) {
                    throw new IllegalStateException("Negative cycle detected");
                }
            }
        }
        
        return dist;
    }
    
    static class Edge {
        int from, to, weight;
        Edge(int from, int to, int weight) {
            this.from = from;
            this.to = to;
            this.weight = weight;
        }
    }
}

// Time: O(V × E)
// Space: O(V)

// Why V-1 iterations?
// Longest simple path has at most V-1 edges
// After V-1 iterations, all shortest paths found

// Use when:
// - Graph has negative weights
// - Need to detect negative cycles
// - Simpler to implement than Dijkstra for small graphs
```

### Pattern 6: Union-Find (Disjoint Set Union)

**When to use**: Connected components, cycle detection in undirected graphs, Kruskal's MST.

```java
public class UnionFind {
    private int[] parent;
    private int[] rank;
    private int components;
    
    public UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        components = n;
        
        for (int i = 0; i < n; i++) {
            parent[i] = i;  // Each node is its own parent initially
            rank[i] = 0;
        }
    }
    
    // Find with path compression
    public int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]);  // Path compression
        }
        return parent[x];
    }
    
    // Union by rank
    public boolean union(int x, int y) {
        int rootX = find(x);
        int rootY = find(y);
        
        if (rootX == rootY) {
            return false;  // Already in same set
        }
        
        // Union by rank: attach smaller tree under larger
        if (rank[rootX] < rank[rootY]) {
            parent[rootX] = rootY;
        } else if (rank[rootX] > rank[rootY]) {
            parent[rootY] = rootX;
        } else {
            parent[rootY] = rootX;
            rank[rootX]++;
        }
        
        components--;
        return true;
    }
    
    public boolean connected(int x, int y) {
        return find(x) == find(y);
    }
    
    public int getComponents() {
        return components;
    }
}

// Time: O(α(n)) per operation where α is inverse Ackermann (nearly O(1))
// Space: O(n)

// Applications:
// 1. Cycle detection in undirected graph
public boolean hasCycle(int n, int[][] edges) {
    UnionFind uf = new UnionFind(n);
    
    for (int[] edge : edges) {
        if (!uf.union(edge[0], edge[1])) {
            return true;  // Edge creates cycle
        }
    }
    
    return false;
}

// 2. Count connected components
public int countComponents(int n, int[][] edges) {
    UnionFind uf = new UnionFind(n);
    
    for (int[] edge : edges) {
        uf.union(edge[0], edge[1]);
    }
    
    return uf.getComponents();
}

// 3. Check if graph is connected
public boolean isConnected(int n, int[][] edges) {
    UnionFind uf = new UnionFind(n);
    
    for (int[] edge : edges) {
        uf.union(edge[0], edge[1]);
    }
    
    return uf.getComponents() == 1;
}
```

### Pattern 7: Minimum Spanning Tree (Kruskal's Algorithm)

**When to use**: Connect all nodes with minimum total edge weight, network design.

```java
public class KruskalMST {
    public List<Edge> findMST(int n, List<Edge> edges) {
        // Sort edges by weight
        edges.sort((a, b) -> a.weight - b.weight);
        
        UnionFind uf = new UnionFind(n);
        List<Edge> mst = new ArrayList<>();
        int totalWeight = 0;
        
        for (Edge edge : edges) {
            // Add edge if it doesn't create cycle
            if (uf.union(edge.from, edge.to)) {
                mst.add(edge);
                totalWeight += edge.weight;
                
                // MST has exactly n-1 edges
                if (mst.size() == n - 1) {
                    break;
                }
            }
        }
        
        return mst;
    }
    
    static class Edge {
        int from, to, weight;
        Edge(int from, int to, int weight) {
            this.from = from;
            this.to = to;
            this.weight = weight;
        }
    }
}

// Time: O(E log E) for sorting + O(E × α(V)) for union-find ≈ O(E log E)
// Space: O(V) for union-find

// Example: Network cable installation
// Nodes = buildings, weights = cable costs
// MST = minimum cost to connect all buildings

// 0 --(4)-- 1
// |         |
// (5)      (3)
// |         |
// 2 --(2)-- 3
//
// MST: edges (2,3), (1,3), (0,1) with total weight 9
```

### Pattern 8: Prim's Algorithm (Alternative MST)

**When to use**: MST with dense graphs, incrementally growing tree.

```java
public class PrimMST {
    public List<Edge> findMST(Map<Integer, List<Edge>> graph, int start) {
        List<Edge> mst = new ArrayList<>();
        Set<Integer> visited = new HashSet<>();
        
        // Min-heap: (weight, from, to)
        PriorityQueue<Edge> pq = new PriorityQueue<>((a, b) -> a.weight - b.weight);
        
        // Start with arbitrary node
        visited.add(start);
        for (Edge edge : graph.get(start)) {
            pq.offer(edge);
        }
        
        while (!pq.isEmpty() && visited.size() < graph.size()) {
            Edge edge = pq.poll();
            
            // Skip if already visited
            if (visited.contains(edge.to)) {
                continue;
            }
            
            // Add edge to MST
            mst.add(edge);
            visited.add(edge.to);
            
            // Add new edges
            for (Edge nextEdge : graph.get(edge.to)) {
                if (!visited.contains(nextEdge.to)) {
                    pq.offer(nextEdge);
                }
            }
        }
        
        return mst;
    }
    
    static class Edge {
        int from, to, weight;
        Edge(int from, int to, int weight) {
            this.from = from;
            this.to = to;
            this.weight = weight;
        }
    }
}

// Time: O(E log V) with binary heap
// Space: O(V + E)

// Kruskal vs Prim:
// Kruskal: Global view, sort all edges, good for sparse graphs
// Prim: Local view, grow from one node, good for dense graphs
```

### Pattern 9: Bipartite Graph Check (Two-Coloring)

**When to use**: Check if graph can be divided into two groups with no internal connections.

```java
public class BipartiteCheck {
    // Color nodes with 2 colors such that no adjacent nodes have same color
    public boolean isBipartite(Map<Integer, List<Integer>> graph, int n) {
        int[] color = new int[n];  // 0 = uncolored, 1 = red, 2 = blue
        
        for (int i = 0; i < n; i++) {
            if (color[i] == 0) {
                if (!bfsColor(graph, i, color)) {
                    return false;
                }
            }
        }
        
        return true;
    }
    
    private boolean bfsColor(Map<Integer, List<Integer>> graph, int start, int[] color) {
        Queue<Integer> queue = new LinkedList<>();
        queue.offer(start);
        color[start] = 1;  // Start with red
        
        while (!queue.isEmpty()) {
            int node = queue.poll();
            int currentColor = color[node];
            int neighborColor = currentColor == 1 ? 2 : 1;
            
            for (int neighbor : graph.get(node)) {
                if (color[neighbor] == 0) {
                    color[neighbor] = neighborColor;
                    queue.offer(neighbor);
                } else if (color[neighbor] != neighborColor) {
                    return false;  // Conflict: same color as parent
                }
            }
        }
        
        return true;
    }
    
    // DFS version
    public boolean isBipartiteDFS(Map<Integer, List<Integer>> graph, int n) {
        int[] color = new int[n];
        
        for (int i = 0; i < n; i++) {
            if (color[i] == 0) {
                if (!dfsColor(graph, i, 1, color)) {
                    return false;
                }
            }
        }
        
        return true;
    }
    
    private boolean dfsColor(Map<Integer, List<Integer>> graph, int node, 
                            int c, int[] color) {
        color[node] = c;
        
        for (int neighbor : graph.get(node)) {
            if (color[neighbor] == c) {
                return false;  // Same color
            }
            if (color[neighbor] == 0) {
                if (!dfsColor(graph, neighbor, -c, color)) {
                    return false;
                }
            }
        }
        
        return true;
    }
}

// Time: O(V + E)
// Space: O(V)

// Applications:
// - Job scheduling (day/night shifts)
// - Coloring maps
// - Matching problems (students to courses)
// - Detecting odd-length cycles (bipartite ↔ no odd cycles)
```

---

## Common Graph Patterns

### Pattern: Clone Graph

```java
// Deep copy of graph
public Node cloneGraph(Node node) {
    if (node == null) return null;
    
    Map<Node, Node> cloned = new HashMap<>();
    return cloneHelper(node, cloned);
}

private Node cloneHelper(Node node, Map<Node, Node> cloned) {
    if (cloned.containsKey(node)) {
        return cloned.get(node);
    }
    
    Node copy = new Node(node.val);
    cloned.put(node, copy);
    
    for (Node neighbor : node.neighbors) {
        copy.neighbors.add(cloneHelper(neighbor, cloned));
    }
    
    return copy;
}

// Used in: Deep copying data structures, serialization
```

### Pattern: Course Schedule (Cycle Detection + Topological Sort)

```java
public boolean canFinish(int numCourses, int[][] prerequisites) {
    // Build graph
    Map<Integer, List<Integer>> graph = new HashMap<>();
    int[] inDegree = new int[numCourses];
    
    for (int i = 0; i < numCourses; i++) {
        graph.put(i, new ArrayList<>());
    }
    
    for (int[] prereq : prerequisites) {
        int course = prereq[0];
        int pre = prereq[1];
        graph.get(pre).add(course);
        inDegree[course]++;
    }
    
    // Topological sort
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

// If completed < numCourses, cycle exists
```

### Pattern: Number of Islands (Connected Components in Grid)

```java
public int numIslands(char[][] grid) {
    if (grid == null || grid.length == 0) return 0;
    
    int count = 0;
    
    for (int i = 0; i < grid.length; i++) {
        for (int j = 0; j < grid[0].length; j++) {
            if (grid[i][j] == '1') {
                dfs(grid, i, j);
                count++;
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
    
    // Explore 4 directions
    dfs(grid, i + 1, j);
    dfs(grid, i - 1, j);
    dfs(grid, i, j + 1);
    dfs(grid, i, j - 1);
}

// BFS version
public int numIslandsBFS(char[][] grid) {
    if (grid == null || grid.length == 0) return 0;
    
    int count = 0;
    
    for (int i = 0; i < grid.length; i++) {
        for (int j = 0; j < grid[0].length; j++) {
            if (grid[i][j] == '1') {
                bfs(grid, i, j);
                count++;
            }
        }
    }
    
    return count;
}

private void bfs(char[][] grid, int i, int j) {
    Queue<int[]> queue = new LinkedList<>();
    queue.offer(new int[]{i, j});
    grid[i][j] = '0';
    
    int[][] dirs = {{1,0}, {-1,0}, {0,1}, {0,-1}};
    
    while (!queue.isEmpty()) {
        int[] cell = queue.poll();
        
        for (int[] dir : dirs) {
            int ni = cell[0] + dir[0];
            int nj = cell[1] + dir[1];
            
            if (ni >= 0 && ni < grid.length && nj >= 0 && nj < grid[0].length &&
                grid[ni][nj] == '1') {
                grid[ni][nj] = '0';
                queue.offer(new int[]{ni, nj});
            }
        }
    }
}
```

### Pattern: Word Ladder (Shortest Transformation)

```java
public int ladderLength(String beginWord, String endWord, List<String> wordList) {
    Set<String> dict = new HashSet<>(wordList);
    if (!dict.contains(endWord)) return 0;
    
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
            
            // Try all possible transformations
            char[] chars = word.toCharArray();
            for (int j = 0; j < chars.length; j++) {
                char original = chars[j];
                
                for (char c = 'a'; c <= 'z'; c++) {
                    if (c == original) continue;
                    
                    chars[j] = c;
                    String transformed = new String(chars);
                    
                    if (dict.contains(transformed)) {
                        queue.offer(transformed);
                        dict.remove(transformed);  // Mark as visited
                    }
                }
                
                chars[j] = original;  // Restore
            }
        }
        
        level++;
    }
    
    return 0;
}

// Example: "hit" → "hot" → "dot" → "dog" → "cog"
// Shortest transformation sequence length = 5
```

---

## Graph Comparison Table

| Algorithm | Use Case | Time | Space | Notes |
|-----------|----------|------|-------|-------|
| BFS | Shortest path (unweighted) | O(V + E) | O(V) | Uses queue |
| DFS | Explore all paths, cycle detection | O(V + E) | O(V) | Uses stack/recursion |
| Dijkstra | Shortest path (weighted, non-negative) | O((V + E) log V) | O(V) | Uses priority queue |
| Bellman-Ford | Shortest path (with negative weights) | O(V × E) | O(V) | Detects negative cycles |
| Topological Sort | Dependency resolution (DAG) | O(V + E) | O(V) | Kahn's algorithm |
| Union-Find | Connected components, cycle detection | O(α(V)) per op | O(V) | Nearly constant time |
| Kruskal's MST | Minimum spanning tree | O(E log E) | O(V) | Sort edges + union-find |
| Prim's MST | Minimum spanning tree | O(E log V) | O(V + E) | Priority queue |
| Floyd-Warshall | All-pairs shortest paths | O(V³) | O(V²) | Dynamic programming |

---

## Next Steps

- Master BFS and DFS until they're second nature
- Understand when to use each graph representation
- Learn Dijkstra's for weighted graphs
- Move to advanced graph algorithms: flow networks, strongly connected components
- Study [Sorting Algorithms](07-sorting-algorithms.md) for comparison-based techniques
