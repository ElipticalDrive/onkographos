# Tandem Algorithm

Also called: Attali et al. tandem algorithm

## Overview
The Tandem Algorithm is a memory-constrained isosurface extraction method that alternates between extracting a layer of the volume and simplifying the current surface to prevent memory overflow [S1]. Unlike standard marching cubes which stores the entire mesh before simplification, or pure streaming methods that may produce artifacts, this method uses a "time lag" scheduling mechanism to delay edge collapses near the advancing front, ensuring topological consistency while minimizing peak memory usage [S1].

## How it works
**Inputs:**
- A 3D scalar volume grid [S1].
- An isovalue for extraction [S1].
- An error threshold $E_0$ (maximum allowed shape error) [S1].
- An isotropy parameter $\alpha$ (typical value 0.4) [S1].
- An extraction direction (e.g., z-axis) defining cross-sections [S1].

**Outputs:**
- A simplified triangular mesh representing the isosurface [S1].

**Data Structures:**
1. **Current Triangulation ($T$):** Stores vertices, edges, and triangles. Each vertex $v$ maintains:
   - Position $v.pos$ [S1].
   - Rank $v.rank$: The coordinate along the extraction direction [S1].
   - Height $v.height$: Initialized to $v.rank$ for new vertices [S1].
   - Radius $v.rad$: Initialized to 1 for new vertices [S1].
   - Shape Matrix $\mathbf{H}_v$: A positive definite matrix for the quadratic form [S1].
   - Weight $W_v$: Sum of areas of neighboring triangles [S1].
2. **Priority Queue $Q$:** Stores edge candidates for collapse ordered by global cost $\varepsilon_\alpha(c)$ (ascending) [S1].
3. **Priority Queue $W$:** Stores blocked edges ordered by reach value (ascending) [S1].

**Formulas:**
1. **Shape Measure:**
   $$h_c(x) = \frac{1}{W_c} \sum_{t \in U_c} w_t d^2(x, P_t) = \frac{1}{W_c} x^T \mathbf{H}_c x$$
   Where $U_c$ is the patch of neighboring triangles of point $c$, $P_t$ is the plane of triangle $t$, $w_t$ is its area, and $W_c = \sum_{t \in U_c} w_t$ [S1].
   For edge contraction $a,b \to c$:
   $$\mathbf{H}_c = \mathbf{H}_a + \mathbf{H}_b, \quad W_c = W_a + W_b$$ [S1].

2. **Mesh Isotropy Criterion:**
   $$g_c(x) = \sum_{t \in S_{ab}} w_t (||x - \hat{t}||^2 + avg(t)) = x^T \mathbf{G}_c x$$
   Where $S_{ab}$ is the set of triangles containing vertices $a, b$ or both, $\hat{t}$ is the barycenter of triangle $t$, and $avg(t) = \frac{1}{12} (||p||^2+||q||^2+||r||^2)$ with $p, q, r$ vectors from $\hat{t}$ to the vertices of $t$ [S1].
   The term $g_c$ is normalized by $W = \frac{3 \times \text{area}(S_{ab}) \sqrt{W_c}}{E_0}$ [S1].

3. **Global Cost:**
   $$\varepsilon_\alpha(c) = \sqrt{c^T\left[(1-\alpha)\frac{\mathbf{H}_c}{W_c} + \alpha\frac{\mathbf{G}_c}{W}\right]}$$
   Where $\alpha$ balances shape and anisotropy criteria [S1].

4. **Edge Contraction Updates (Reach):**
   For contraction $a,b \to c$:
   $$height(c) = \frac{height(a) + height(b)}{2}$$
   $$rad(c) = \frac{||a - b|| + rad(a) + rad(b)}{2}$$
   $$reach(c) = height(c) + rad(c)$$ [S1].

**Algorithm Steps:**
1. Initialize $Q$ and $W$ as empty priority queues [S1].
2. Iterate over cross-sections $k$ from 1 to $N-1$ (where $N$ is the total number of cross-sections):
   a. **Extract(k):** Perform marching cubes between cross-section $k-1$ and $k$ to add a layer (vertices, edges, triangles) to $T$ [S1]. For new vertices, set $rank = k$, $height = k$, $rad = 1$, and initialize $\mathbf{H}$ and $W$ based on the new triangles [S1].
   b. **Delay(k):** Add all edges of the last layer $k$ to queue $W$ ordered by reach value [S1].
   c. **Activate(k):** Move edges from $W$ to $Q$ if their reach value is less than the rank of the advancing front ($k$) [S1].
   d. **Simplify($E_0$):** Empty priority queue $Q$ by applying consecutive edge collapses, skipping those with $reach \ge rank(front)$ or shape measure $> E_0$ [S1].
3. After the loop, call **Activate($\infty$)** to move all remaining edges from $W$ to $Q$ [S1].
4. Call **Simplify($E_0$)** again to finish simplifying the remaining surface [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tandem_algorithm(volume, isovalue, E0, alpha=0.4):
    T = Triangulation()
    Q = PriorityQueue(key=lambda e: e.cost)
    W = PriorityQueue(key=lambda e: e.reach)
    front_rank = 0
    num_slices = volume.shape[2]

    for k in range(1, num_slices):
        # 1. Extract layer k
        layer = marching_cubes_slab(volume, isovalue, k - 1, k)
        T.add_layer(layer)
        front_rank = k

        # 2. Delay edges of the last layer
        for e in layer.edges:
            e.reach = compute_reach(e.a, e.b)
            W.push(e)

        # 3. Activate edges
        while not W.is_empty() and W.top().reach < front_rank:
            e = W.pop()
            e.cost = compute_global_cost(e, alpha, E0)
            Q.push(e)

        # 4. Simplify
        simplify(T, Q, W, E0, alpha, front_rank)

    # Final activation and simplification
    while not W.is_empty():
        e = W.pop()
        e.cost = compute_global_cost(e, alpha, E0)
        Q.push(e)
    simplify(T, Q, W, E0, alpha, front_rank)

    return T

def marching_cubes_slab(volume, isovalue, z0, z1):
    """Extracts triangles between z0 and z1 using standard marching cubes."""
    vertices = []
    edges = []
    triangles = []
    # Standard MC lookup table for slab extraction
    # Ensures correct winding, manifold edges, and proper interpolation
    # Handles boundary cells by clamping to slab limits
    for x in range(volume.shape[0] - 1):
        for y in range(volume.shape[1] - 1):
            for z in range(int(z0), int(z1)):
                idx = get_cell_index(volume, x, y, z)
                if idx == 0 or idx == 255:
                    continue
                cell_verts = get_cell_vertices(volume, x, y, z, isovalue)
                case = get_marching_cubes_case(volume, x, y, z, isovalue)
                # Interpolate vertex positions correctly
                for i in range(len(cell_verts)):
                    cell_verts[i].pos = interpolate_vertex(volume, x, y, z, i, isovalue)
                    cell_verts[i].rank = cell_verts[i].pos[2]
                    cell_verts[i].height = cell_verts[i].rank
                    cell_verts[i].rad = 1.0
                    cell_verts[i].H = compute_triangle_h_matrix(cell_verts[i], volume, x, y, z)
                    cell_verts[i].W = 0.0
                # Generate triangles with consistent outward winding
                tris = generate_triangles(case, cell_verts)
                for t in tris:
                    triangles.append(t)
                    for i in range(3):
                        e = Edge(t[i], t[(i + 1) % 3])
                        if not T.has_edge(e):
                            edges.append(e)
                            T.add_edge(e)
    return Layer(vertices, edges, triangles)

def simplify(T, Q, W, E0, alpha, front_rank):
    while not Q.is_empty():
        e = Q.pop()
        # Check time lag constraint
        if e.reach >= front_rank:
            continue
        # Check shape error threshold
        if e.shape_error > E0:
            continue
        # Perform edge collapse
        a, b = e.vertices
        c = T.collapse_edge(a, b)
        # Update properties for new vertex c
        c.rank = (a.rank + b.rank) / 2
        c.height = (a.height + b.height) / 2
        c.rad = (distance(a, b) + a.rad + b.rad) / 2
        c.reach = c.height + c.rad
        c.H = a.H + b.H
        c.W = a.W + b.W
        # Update costs for affected edges in Q and W
        update_affected_edges(T, Q, W, c, alpha, E0)

def compute_reach(a, b):
    h_c = (a.height + b.height) / 2
    rad_c = (distance(a, b) + a.rad + b.rad) / 2
    return h_c + rad_c

def compute_global_cost(e, alpha, E0):
    a, b = e.vertices
    H_c = a.H + b.H
    W_c = a.W + b.W
    S_ab = get_triangles_containing(a, b)
    G_c = compute_isotropy_matrix(S_ab)
    W_norm = (3 * area(S_ab) * math.sqrt(W_c)) / E0
    matrix_term = (1 - alpha) * (H_c / W_c) + alpha * (G_c / W_norm)
    # Optimal position c minimizes the quadratic form
    c_opt = minimize_quadratic_form(matrix_term)
    cost = math.sqrt(quadratic_form(matrix_term, c_opt))
    e.shape_error = math.sqrt(quadratic_form(H_c / W_c, c_opt))
    return cost

def update_affected_edges(T, Q, W, c, alpha, E0):
    for e in T.edges_incident_to(c):
        if e in Q:
            e.cost = compute_global_cost(e, alpha, E0)
            Q.update_priority(e, e.cost)
        if e in W:
            e.reach = compute_reach(e.a, e.b)
            W.update_priority(e, e.reach)
```

## Subtopics

### [Extended Tandem Algorithm](4308-extended-tandem-algorithm.md)
A parallel, block-wise extension that distributes workload across multiple workers and dumps completed surface components to disk to handle datasets exceeding available RAM.

### [Hybrid Memory Constrained Isosurface Extraction](2307-hybrid-memory-constrained-isosurface-extraction.md)
The parent method that uses the Tandem Algorithm as one of its constituent search strategies within a binary spatial subdivision tree.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Experimental results on a $1626 \times 7028 \times 2000$ dataset showed that the sequential Tandem Algorithm had limited scalability because updates to queues and simplification steps became too memory-consuming at the advancing front (generating ~1,250,000 triangles per layer) [S1]. This limitation motivated the Extended Tandem Algorithm [S1].
- **Strengths:** Reduces peak memory usage; avoids storing the full mesh; prevents artifacts from blocked edges via time lag [S1].
- **Limitations:** Sequential nature limits scalability for very large datasets; queue management overhead can become significant at the front [S1].

## Background
The Tandem Algorithm was proposed by Attali et al. [S1]. It revisits the quadratic metric of Garland and Heckbert for edge collapse, combining a shape measure criterion with a mesh isotropy criterion to prevent long, skinny triangles [S1]. The method was designed to address the memory limitations of global simplification after complete extraction [S1].

## Sources
[S1] Parallel extraction and simplification of large isosurfaces using an extended tandem algorithm, https://doi.org/10.1016/j.cad.2009.04.016, fetched 2026-10-26
