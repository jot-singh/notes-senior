# Graphs Fundamentals

## What You'll Learn
- Graph representations and when to use each
- Breadth-First Search (BFS) and Depth-First Search (DFS) traversal
- Topological sorting for dependency resolution
- Cycle detection and connectivity analysis
- Real-world graph applications: social networks, maps, recommendations

## Why This Matters

Graphs model **relationships without hierarchy** - the most flexible data structure for representing connections. Social networks are graphs, route planning is graphs, recommendation systems are graphs. Mastering graph algorithms unlocks solutions to countless real-world problems.

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

## Next Steps

- Master BFS and DFS until they're second nature
- Understand when to use each graph representation
- Learn Dijkstra's for weighted graphs
- Move to advanced graph algorithms: flow networks, strongly connected components
- Study [Sorting Algorithms](07-sorting-algorithms.md) for comparison-based techniques
