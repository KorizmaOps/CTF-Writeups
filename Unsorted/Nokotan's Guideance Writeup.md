# "Nokotan's Guidance" CTF Write-up

## Challenge Overview

The challenge asks us to solve a series of shortest path queries on a special type of graph called a "triangle graph." We are given the graph's construction process:

1.  Start with a complete graph of 3 nodes (a triangle).
2.  Iteratively add new nodes (from 4 to *n*), where each new node `i` connects to two existing, *adjacent* nodes `u` and `v`.
3.  Answer *q* queries, each asking for the minimum distance between two nodes, `s` and `t`.

The constraints are large (*n*, *q* ≤ 1.5 × 10⁵), which immediately tells us that any solution involving a simple graph traversal (like BFS) for each query will be too slow. A solution of O(*q* * *n*) would be roughly 10¹⁰ operations, so we need a much more efficient, sub-linear time approach per query.

---

## Key Observation: The Hidden Tree Structure

The critical insight lies in the graph's construction rule. When a new node `i` is added by connecting to an adjacent pair `(u, v)`, it effectively "splits" the edge `(u, v)` and replaces it with a new triangle `(u, v, i)`. This process is hierarchical and recursive.

This construction process can be mapped directly to a **ternary tree**:

*   **Tree Nodes vs. Graph Nodes:** A *graph node* is a vertex in the final triangle graph. A **tree node** represents the *operation* of adding a new graph node.
*   **The Root:** The initial triangle (graph nodes 1, 2, 3) can be considered the operation that creates the base graph. This forms the **root** of our ternary tree, which we can label `T_3`.
*   **Parent-Child Relationship:** When a new graph node `i` is created by splitting the edge `(u, v)`, the corresponding tree node `T_i` becomes a child of the tree node `T_p` that was responsible for creating the edge `(u, v)`.

Therefore, the entire graph generation process is equivalent to building a large tree where each node `T_k` represents the 3-vertex clique `{k, u, v}` it introduced. Finding the shortest path between two graph nodes `s` and `t` can now be reframed as a problem on this underlying tree.

---

## Solution Strategy: LCA with Distance Matrices

A simple Lowest Common Ancestor (LCA) on this tree isn't enough, as it doesn't directly tell us the graph distance. The path distance depends on how we traverse the cliques at each level of the tree. The optimal strategy combines two powerful techniques: **Binary Lifting for LCA** and **Dynamic Programming with Distance Matrices**.

### 1. Building the Ternary Tree

First, we parse the input and build our tree structure. For each new node `i` (from 4 to *n*) connected to `u` and `v`, we need to find the parent tree node. We can do this efficiently by maintaining a map from an edge `(u, v)` to the tree node `T_i` that created it.

```cpp
// A map from a graph edge {u, v} to the tree node ID that created it.
map<pair<int, int>, int> edge_to_treenode_map;

// The tree itself, where each node stores its parent and defining vertices.
vector<TreeNode> tree;
```

### 2. The Distance Query Problem

The shortest path between graph nodes `s` and `t` will always go "up" the tree from their respective tree nodes (`T_s`, `T_t`) to their LCA (`T_lca`), and then "down" to the target. The path must pass through one of the three vertices of the clique defined by `T_lca`.

Our goal is to efficiently calculate `dist(s, lca_vertex) + dist(t, lca_vertex)` for all three vertices in the LCA's clique and take the minimum.

### 3. Augmenting Binary Lifting with Distance Matrices

To find these distances quickly, we augment the standard binary lifting algorithm. Alongside the ancestor table `up[i][j]` (the 2<sup>j</sup>-th ancestor of `i`), we precompute a table of 3x3 distance matrices `dist_matrix_lift[i][j]`.

*   `dist_matrix_lift[i][j].data[p][q]` stores the shortest distance from the `p`-th vertex of `T_i`'s clique to the `q`-th vertex of `T_{up[i][j]}`'s clique.

These matrices can be combined using a **min-plus matrix multiplication**. The distance matrix for a jump of 2<sup>j</sup> is the product of two matrices for jumps of 2<sup>j-1</sup>:
`dist_matrix_lift[i][j] = dist_matrix_lift[i][j-1] * dist_matrix_lift[up[i][j-1]][j-1]`

This precomputation takes O(*n* log *n*) time.

### 4. Answering a Query

With the precomputed tables, a query for `(s, t)` is answered in O(log *n*) time:
1.  Identify the tree nodes `T_s` and `T_t` corresponding to graph nodes `s` and `t`.
2.  Find their LCA, `T_lca`, using the `up` table.
3.  Use the `dist_matrix_lift` table to compute the total distance matrix from `T_s` up to `T_lca`. This gives the distances from `s` to each of the 3 vertices in the LCA's clique.
4.  Repeat the process for `t`.
5.  Iterate through the three vertices of the LCA's clique, find the minimum combined path length, and return the result.

---

## Complexity Analysis & Conclusion

*   **Preprocessing:** O(*n* log *n*) to build the tree and the binary lifting tables.
*   **Query Time:** O(log *n*) for each of the *q* queries.
*   **Total Time:** O(*n* log *n* + *q* log *n*), which is well within the time limits.

### Why This Challenge Is Neat

This problem is an excellent example of finding a hidden, simpler structure within a complex problem description. It demonstrates that:
*   A generative process for a graph often implies an underlying tree structure.
*   Standard algorithms like Binary Lifting can be powerful but sometimes need to be augmented with additional data (like distance matrices) to solve a more complex variant of the problem.
*   It forces a deep understanding of how to translate a pathfinding problem on a dense graph into an efficient query on a sparse tree, which is a common and powerful pattern in competitive programming.

---

### Appendix: Core Data Structures

The solution relies on two key C++ structs to organize the data for the tree and the distance matrices.

```cpp
// Represents a node in the ternary tree.
// Each node corresponds to a 3-vertex clique added to the graph.
struct TreeNode {
    int parent;
    int depth;
    // nodes[0] = new vertex, nodes[1] & nodes[2] = existing vertices
    int nodes[3]; 
};

// A 3x3 matrix storing shortest distances between the defining vertices
// of two different tree nodes.
struct DistanceMatrix {
    int data[3][3];
};
```