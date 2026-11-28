# Contour Tree

Also called: Join Tree, Split Tree

## Overview
The Contour Tree is a topological data structure that represents the connectivity of level sets (contours) of a scalar field as the isovalue varies [S1, unofficial][S2]. It solves the problem of tracking how contours are created, split, merged, and destroyed by mapping these events to a tree structure where edges correspond to individual contour classes and vertices correspond to critical points [S1, unofficial][S2]. It sets itself apart from the [Reeb Graph](9315-reeb-graph.md) by requiring a simply connected domain without loops [S1, unofficial], and from the [Topological Landscape](8481-topological-landscape.md) by providing the explicit graph structure rather than just a visual metaphor of peaks and valleys [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field $f$ defined on a simply connected domain (e.g., a grid or mesh) [S1, unofficial][S2].
*   **Output:** A tree structure $T = (V, E)$ where $V$ is the set of critical points and $E$ is the set of edges connecting them [S2].

**Data Structures:**
The contour tree consists of two primary elements [S1, unofficial]:
1.  **Vertices (Supernodes):**
    *   **Leaf Nodes:** Represent local extrema [S1, unofficial].
        *   Local maxima: Where a contour is created (Join Tree context) or destroyed (Split Tree context) [S1, unofficial].
        *   Local minima: Where a contour is destroyed (Join Tree context) or created (Split Tree context) [S1, unofficial].
    *   **Interior Nodes:** Represent saddle points [S1, unofficial].
        *   Split saddles: Where one contour splits into two or more disjoint contours [S1, unofficial].
        *   Join saddles: Where two or more disjoint contours merge into one [S1, unofficial].
2.  **Edges (Superarcs):**
    *   Represent a single continuous contour class between the supernode where it is created and the supernode where it is destroyed [S1, unofficial].
    *   A contour class is defined as a maximal set of continuous contours that do not contain critical points [S2].

**Construction Algorithm:**
The contour tree is typically constructed by first building two auxiliary structures: the Join Tree (JT) and the Split Tree (ST), and then merging them [S2]. The Join Tree captures connected sub-level sets (regions below an isovalue), while the Split Tree captures connected super-level sets (regions above an isovalue) [S1, unofficial].

**Join Tree (JT) Construction:**
1.  Start from the global maximum function value [S2].
2.  Continuously decrease the isovalue $w$.
3.  Define the marked region $X(w) = \{x | f(x) \ge w\}$ [S2].
4.  Identify connected components of $X(w)$ as "upper objects" [S2].
5.  **Node Creation (Upper Leaf):** When $w$ passes a local maximum, a new connected component is created. Create a JT node for this upper leaf [S2].
6.  **Merge Event:** When $w$ passes a join saddle point, two or more components merge into one. Create a new JT node for the saddle point and connect it to the node of the latest critical point belonging to the same object [S2].
7.  **Termination:** When $w$ reaches the global minimum, create a final JT node and connect it to the node of the latest critical point [S2].
8.  **Implementation Detail:** Vertices in the domain mesh are marked in decreasing order based on function value [S2].

**Split Tree (ST) Construction:**
Symmetric to JT construction, but starts from the global minimum and increases the isovalue to track connected super-level sets [S2].

**Contour Tree (CT) Formation:**
1.  **Augmented Contour Tree (ACT):** Successively delete upper leaves of the JT and lower leaves of the ST, inserting adjacent edges to form the ACT [S2].
2.  **Final CT:** Successively delete regular (non-critical) vertices in the ACT to obtain the final Contour Tree [S2].

**Optimizations and Variants:**
*   **Sorting:** Chiang et al. improve processing time by only sorting critical vertices during the merge tree construction stage, rather than sorting all elements [S1, unofficial].
*   **Multithreading:** Gueunet et al. present a "contour forest" algorithm using a shared memory approach for multithreaded computation of the augmented contour tree [S1, unofficial].
*   **Seeds:** To facilitate contour tracing, "seeds" are identified as vertices where contours can be built from. An entire contour for a specified isovalue is computed from the seed bounded by the two supernodes representing the critical points of that contour [S1, unofficial]. Using seeds reduces the number of seeds required for representation but may increase overall storage size [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_contour_tree(scalar_field, domain_mesh):
    """
    Computes the Contour Tree by constructing Join and Split Trees.
    scalar_field: dict mapping vertex -> float value.
    domain_mesh: dict mapping vertex -> list of neighbor vertices.
    Returns: Contour Tree T = (V, E) as lists of nodes and edges.
    """
    # 1. Sort vertices by function value in descending order
    sorted_vertices = sorted(domain_mesh.keys(), key=lambda v: scalar_field[v], reverse=True)

    # Union-Find structures to track connected components
    parent = {v: v for v in domain_mesh}
    node_map = {}  # Maps component root to its JT node ID

    def find(v):
        while parent[v] != v:
            parent[v] = parent[parent[v]]
            v = parent[v]
        return v

    def union(u, v):
        root_u, root_v = find(u), find(v)
        if root_u != root_v:
            # Connect to the lowest vertex of neighboring objects [S2]
            if scalar_field[root_u] > scalar_field[root_v]:
                parent[root_u] = root_v
            else:
                parent[root_v] = root_u
            return True
        return False

    jt_nodes = []
    jt_edges = []
    node_counter = 0

    def create_node(vertex, node_type):
        nonlocal node_counter
        nid = f"node_{node_counter}"
        node_counter += 1
        jt_nodes.append((nid, vertex, node_type))
        return nid

    # 2. Sweep and build JT
    for vertex in sorted_vertices:
        neighbors = domain_mesh[vertex]
        neighbor_roots = set()
        for n in neighbors:
            if n in parent:
                neighbor_roots.add(find(n))

        if not neighbor_roots:
            # Node Creation (Upper Leaf): When w passes a local maximum, a new component is created [S2]
            nid = create_node(vertex, 'upper_leaf')
            node_map[vertex] = nid
            parent[vertex] = vertex
        else:
            roots_list = list(neighbor_roots)
            if len(roots_list) > 1:
                # Merge Event: When w passes a join saddle point, two or more components merge into one [S2]
                saddle_nid = create_node(vertex, 'join_saddle')
                jt_nodes.append(saddle_nid)
                for r in roots_list:
                    jt_edges.append((saddle_nid, node_map[r]))
                    parent[r] = vertex  # Union all components to this vertex
                node_map[vertex] = saddle_nid
            else:
                # Single component continues
                r = roots_list[0]
                parent[vertex] = r
                jt_edges.append((vertex, node_map[r]))
                node_map[vertex] = node_map[r]

    # Termination: When w reaches the global minimum, a JT node is created and connected to the node for the latest critical point [S2]
    global_min_vertex = min(domain_mesh.keys(), key=lambda v: scalar_field[v])
    min_nid = create_node(global_min_vertex, 'global_min')
    jt_nodes.append(min_nid)
    jt_edges.append((min_nid, node_map[find(global_min_vertex)]))

    # 3. Construct Split Tree (ST) symmetrically [S2]
    # ST construction follows identical logic but processes vertices in ascending order to track super-level sets.

    # 4. Form Augmented Contour Tree (ACT): Successively delete upper leaves of JT and lower leaves of ST, inserting adjacent edges [S2].
    # 5. Form Final CT: Successively delete regular vertices in ACT to obtain the final Contour Tree [S2].
    # Implementation note: These steps merge edges between critical points to form the final tree structure.

    return jt_nodes, jt_edges
```

**Parameters:**
*   **Neighborhood Definition:** Changing the method of defining neighbors during split/merge tree computations can slightly alter the location of critical points, though the underlying structure (number of supernodes and superarcs) remains fixed [S1, unofficial].
*   **Genus Information:** The contour tree typically does not contain information regarding changes in topological genus (loops); this must be added to the output separately if required [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Simply Connected Domains:** The input domain must be simply connected and free of loops [S1, unofficial]. The merge tree computation phase cannot correctly handle data without a boundary or with loops [S1, unofficial]. For such cases, the [Reeb Graph](9315-reeb-graph.md) should be used [S1, unofficial].
*   **High Dimensionality:** Critical vertices are difficult to identify in data with dimensionality of four or more [S1, unofficial].
*   **Load Imbalance:** In multithreaded implementations (e.g., contour forest), speedup is limited by load imbalance and redundant computations [S1, unofficial].

## Subtopics

### [Reeb Graph](9315-reeb-graph.md)
The Reeb graph is a topological data structure that generalizes the contour tree to compute the topology of scalar data on domains that are not simply connected (contain loops) or lack a boundary. It solves the problem of capturing global topology in complex geometries where standard contour trees fail. It sets itself apart from the [Contour Tree](9312-contour-tree.md) by explicitly handling domains with loops or missing boundaries, and from the [Joint Contour Net](9316-joint-contour-net.md) by operating on single-variable scalar fields rather than multivariate data.

### [Topological Zone Transfer Function Application](11654-topological-zone-transfer-function-application.md)
This method applies distinct transfer functions to different topological zones of a scalar field, determined by mapping sample points to arcs in a contour tree via monotone paths. It sets itself apart from global transfer functions by allowing topology-aware color and opacity mapping, and from simple gradient reversal or flattening by using dynamic tube sweeping to broaden connections between components, providing a smoother approximation of the scalar field.

### [Contour-Tree-Defined Segmentation](11672-contour-tree-defined-segmentation.md)
Contour-Tree-Defined Segmentation partitions a scalar volume into topological zones, where each zone corresponds to an arc in the contour tree of the underlying trilinear interpolant. It solves the problem of assigning independent transfer functions to distinct topological structures, allowing for the isolation of specific features based on their creation and destruction isovalue ranges. It builds on the concept of topological zones defined by Cox et al. and extends prior work by allowing independent transfer functions per zone and using trilinear interpolation over hexahedral cells.

### [Stratified Morse Theory for Euler Characteristic](11757-stratified-morse-theory-for-euler-characteristic.md)
Stratified Morse Theory (SMT) for Euler Characteristic is a mathematical framework and verification pipeline used to compute the Euler characteristic ($\chi$) of isosurfaces generated from piecewise trilinear scalar fields, specifically handling non-smoothness at cell boundaries and surfaces with boundary components. It solves the problem of verifying the topological correctness of isosurface extraction algorithms (such as Marching Cubes) by comparing the Euler characteristic derived from critical point analysis against the Euler characteristic of the extracted mesh. Unlike its siblings, which often rely on smooth manifold assumptions or digital topology constraints, SMT explicitly stratifies the grid into vertices, edges, faces, and cells to apply Morse theory locally, making it robust for verifying algorithms that guarantee topological correctness in the presence of ambiguous configurations.

### [Time-varying Contour Topology Tracking](17245-time-varying-contour-topology-tracking.md)
Time-varying Contour Topology Tracking is a method for establishing temporal correspondence between isosurfaces in time-varying scalar fields by maintaining and intersecting labeled Join Trees (JT) and Separate Trees (ST) across consecutive timesteps. It solves the problem of tracking topological changes (merges, splits, collisions) in contour evolution by labeling the JT from time $t$ with edge sets derived from time $t+1$, then intersecting these labels with those from the ST to determine precise contour mapping. Unlike static contour tree analysis, this method explicitly handles the dynamic intersection of upper and lower objects between timesteps to resolve topology changes.

### Contour Tree Branch Navigation
After determining high and low vertices, look up their corresponding high and low arcs in the contour tree [S3, unofficial]. Walk from the high arc towards the low arc to find the arc containing the sampled value $f$ [S3, unofficial]. Use the branch decomposition (a rooted tree) instead of the contour tree (a free tree) to find the path between branches $b$ and $c$ [S3, unofficial]. Walk up the parent link of the deeper of the two branches $b$ and $c$ simultaneously [S3, unofficial]. If neither stopping condition is met, the walk terminates when both paths reach the common ancestor $a$, which is the desired branch [S3, unofficial]. When walking towards increasing function value, stop if the next step goes past a saddle with a higher value than the sampled point [S3, unofficial]. When walking towards decreasing function value, stop if the next step goes past a saddle with a lower value than the sample [S3, unofficial].

## Operating characteristics
*   **Complexity:** The construction of the contour tree involves sorting critical points and processing connectivity changes [S1, unofficial]. Chiang et al. note that sorting only critical vertices improves processing time in very large datasets [S1, unofficial].
*   **Memory:** Using seeds reduces the number of seeds required for representation significantly but often increases the overall storage size of the contour tree [S1, unofficial].
*   **Performance:** Multithreaded approaches (e.g., contour forest) offer quicker computations, but speedup is limited by load imbalance and redundant computations [S1, unofficial].
*   **Limitations:** The method is restricted to simply connected domains without loops [S1, unofficial]. It does not inherently capture genus changes [S1, unofficial]. Critical point identification becomes difficult in dimensions four or higher [S1, unofficial].
*   **Usage:** Merge trees are often used as a computation step in the construction of more complex topological data structures such as the contour tree [S1, unofficial]. Alternatively, they can be used directly for analysis and feature extraction in complex data sets [S1, unofficial].

## Background
The contour tree was introduced by Boyell and Ruston [S1, unofficial]. The contour tree uses concepts from Morse theory to capture changes in the topology of a scalar field at critical points [S1, unofficial]. Carr et al. investigated the use of contour trees in higher dimensional data sets and introduced the concept of augmented contour trees [S1, unofficial]. The structure is fundamental in computational topology for tracking connected level sets [S1, unofficial].

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
[S2] Time-varying contour topology, https://doi.org/10.1109/tvcg.2006.16, fetched 2026-11-08
[S3] Topology-Controlled Volume Rendering, https://www.computer.org/csdl/journal/tg/2007/02/04069241/13rRUytF41s, fetched 2026-10-31 (unofficial)
