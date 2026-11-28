# Tandem Algorithm

Also called: Attali et al. tandem algorithm, Extraction and Simplification of Iso-surfaces in Tandem, Incremental Isosurface Extraction and Simplification

## Overview
The Tandem Algorithm is a memory-constrained isosurface extraction method that alternates between extracting a new layer of geometry and simplifying the accumulated surface to maintain a bounded memory footprint [S1]. It solves the problem of storing the entire intermediate triangulation by performing incremental simplification using edge collapses, ensuring that the number of vertices and triangles remains manageable during the extraction process [S1]. Unlike methods that extract the full volume before simplifying, or pure streaming approaches that discard history, the Tandem Algorithm maintains a global view of the surface for simplification while restricting the "active" geometry to a sliding window near the extraction front [S1].

## How it works
The algorithm processes a volumetric dataset by iterating through cross-sections (layers) along a specific extraction direction (e.g., the $z$-axis) [S1]. It maintains two priority queues: $Q$ for edge collapse candidates and $W$ for blocked edges [S1].

### Inputs and Outputs
- **Input**: Volumetric dataset, isovalue, maximum shape error threshold $E_0$, isotropy parameter $\alpha$, extraction direction defining cross-sections $k=1 \dots N$ [S1].
- **Output**: Simplified isosurface mesh [S1].

### Data Structures
1. **Priority Queue $Q$**: Stores edge candidates for collapse, ordered by their global cost $\varepsilon_\alpha(c)$ (ascending) [S1].
2. **Priority Queue $W$**: Stores blocked edges, ordered by their reach value (ascending) [S1].
3. **Mesh State**: Current triangulation $M$, including vertices with attributes:
   - $height(u)$: Rank of vertex $u$ along extraction direction [S1].
   - $rad(u)$: Radius of vertex $u$ [S1].
   - $reach(u)$: Reach value of vertex $u$ [S1].
   - $\mathbf{H}_u$: Positive definite matrix for shape measure [S1].
   - $W_u$: Weight (sum of areas) for shape measure [S1].
   - $\mathbf{G}_u$: Matrix for isotropy measure [S1].
   - $W$: Global weight for isotropy normalization [S1].

### Formulas
**Shape Measure** [S1]:
The shape measure of a point $x$ relative to patch $U_c$ (neighboring triangles of $c$) is:
$$ h_c(x) = \frac{1}{W_c} \sum_{t \in U_c} w_t d^2(x, P_t) = \frac{1}{W_c} x^T \mathbf{H}_c x $$
where $P_t$ is the plane of triangle $t$, $w_t$ is its area, $W_c = \sum_{t \in U_c} w_t$, and $\mathbf{H}_c$ is a positive definite matrix [S1]. For edge contraction $a,b \to c$:
$$ \mathbf{H}_c = \mathbf{H}_a + \mathbf{H}_b, \quad W_c = W_a + W_b $$

**Mesh Isotropy Criterion** [S1]:
To prevent long, skinny triangles, the isotropy criterion for point $c$ (result of collapsing $a,b$) is:
$$ g_c(x) = \sum_{t \in S_{ab}} w_t (||x - \hat{t}||^2 + avg(t)) = x^T \mathbf{G}_c x $$
where $S_{ab}$ is the set of triangles containing $a$, $b$, or both, $\hat{t}$ is the barycenter of $t$, and $avg(t) = \frac{1}{12} (||p||^2+||q||^2+||r||^2)$ with $p,q,r$ vectors from $\hat{t}$ to vertices of $t$ [S1]. The term $g_c$ is normalized by $W = 3 \times \text{area}(S_{ab}) W_c^{1/2} / E_0$ to balance its influence with $h_c$ [S1].

**Global Cost** [S1]:
$$ \varepsilon_\alpha(c) = \sqrt{c^T \left[ (1-\alpha)\frac{\mathbf{H}_c}{W_c} + \alpha\frac{\mathbf{G}_c}{W} \right]} $$
where $\alpha$ is the isotropy parameter (typical value 0.4) [S1].

**Edge Contraction Updates** [S1]:
When collapsing edge $a,b \to c$:
$$ height(c) = \frac{height(a) + height(b)}{2} $$
$$ rad(c) = \frac{||a - b|| + rad(a) + rad(b)}{2} $$
$$ reach(c) = height(c) + rad(c) $$
For new vertices introduced by extraction, $rad(u) = 1$ [S1].

### Parameters
- **$E_0$**: Maximum shape error allowed. A candidate edge collapse is rejected if its shape measure $\varepsilon_0(c)$ exceeds $E_0$ [S1].
- **$\alpha$**: Isotropy parameter balancing shape and anisotropy criteria. Typical value: 0.4 [S1].

### Algorithm Steps
1. **Initialize**: Create empty mesh $M$, empty queues $Q$ and $W$ [S1].
2. **Loop** for $k$ from 1 to $N-1$ (where $N$ is the number of cross-sections):
   1. **Extract($k$)**: Add vertices, edges, and triangles from marching cubes between cross-sections $k-1$ and $k$ to $M$ [S1].
   2. **Delay($k$)**: Add all edges of the newly extracted layer $k$ to priority queue $W$ ordered by reach value [S1].
   3. **Activate($k$)**: Move edges from $W$ to $Q$ if their reach value is less than the rank of the advancing front $k$ [S1].
   4. **Simplify($E_0$)**: Empty priority queue $Q$ by applying consecutive edge collapses, rejecting candidates where $\varepsilon_0(c) > E_0$ [S1].
3. **Finalize**: Call **Activate($\infty$)** to move all remaining edges from $W$ to $Q$, then call **Simplify($E_0$)** to empty $Q$ completely [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tandem_algorithm(volume, isovalue, E0, alpha, num_cross_sections):
    mesh = Mesh()
    Q = PriorityQueue()
    W = PriorityQueue()
    current_front_rank = 0

    for k in range(1, num_cross_sections):
        current_front_rank = k
        layer_vertices, layer_edges, layer_triangles = marching_cubes(volume, k-1, k, isovalue)
        mesh.add(layer_vertices, layer_edges, layer_triangles)

        for v in layer_vertices:
            v.height = k
            v.rad = 1.0
            v.reach = k + 1.0
            v.H = compute_quadratic_metric(v)
            v.W = sum(t.area for t in v.neighbors)
            v.G = compute_isotropy_metric(v)

        for e in layer_edges:
            a, b = e.vertices
            c = midpoint(a, b)
            reach_val = c.height + c.rad
            W.push(e, priority=reach_val)

        while W and W.top().priority < current_front_rank:
            blocked_edge = W.pop()
            a, b = blocked_edge.vertices
            c = midpoint(a, b)
            cost = compute_global_cost(c, alpha)
            Q.push(blocked_edge, priority=cost)

        simplify(mesh, Q, W, E0, alpha, current_front_rank)

    while W:
        blocked_edge = W.pop()
        a, b = blocked_edge.vertices
        c = midpoint(a, b)
        cost = compute_global_cost(c, alpha)
        Q.push(blocked_edge, priority=cost)

    simplify(mesh, Q, W, E0, alpha, current_front_rank)
    return mesh

def simplify(mesh, Q, W, E0, alpha, current_front_rank):
    while Q:
        edge = Q.pop()
        a, b = edge.vertices
        if not mesh.vertex_exists(a) or not mesh.vertex_exists(b):
            continue
        c = midpoint(a, b)
        shape_error = compute_shape_error(c)
        if shape_error <= E0:
            mesh.collapse_edge(edge, c)
            c.height = (a.height + b.height) / 2.0
            c.rad = (dist(a, b) + a.rad + b.rad) / 2.0
            c.reach = c.height + c.rad
            c.H = a.H + b.H
            c.W = a.W + b.W
            c.G = a.G + b.G
            update_connectivity(mesh, c)
            for neighbor_edge in c.incident_edges:
                if neighbor_edge not in Q and neighbor_edge not in W:
                    reach_val = compute_reach(neighbor_edge)
                    if reach_val >= current_front_rank:
                        W.push(neighbor_edge, priority=reach_val)
                    else:
                        cost = compute_global_cost(neighbor_edge, alpha)
                        Q.push(neighbor_edge, priority=cost)

# Standard operations [G]
def compute_quadratic_metric(v): return standard_quadratic_metric(v)
def compute_isotropy_metric(v): return standard_isotropy_metric(v)
def compute_global_cost(c, alpha): return sqrt(c.T @ ((1-alpha)*c.H/c.W + alpha*c.G/W) @ c)
def compute_shape_error(c): return sqrt(c.T @ (c.H/c.W) @ c)
def compute_reach(e): a, b = e.vertices; c = midpoint(a,b); return c.height + c.rad
```

### Edge Cases and Pitfalls
- **Blocked Edges**: Contraction of an edge is prevented if its reach value is greater than or equal to the rank of the advancing front. This prevents artifacts from blocked edges on the last extracted layer [S1].
- **Precision**: The shape measure $\varepsilon_0(c)$ must be compared against $E_0$. A candidate edge collapse is rejected if its shape measure exceeds $E_0$ [S1].
- **Degenerate Input**: Noisy datasets may produce many small disconnected components. The standard Tandem Algorithm processes them sequentially, but the Extended Tandem Algorithm is recommended for such cases to dump components to disk [S3].

## Subtopics

### [Extended Tandem Algorithm](4308-extended-tandem-algorithm.md)
The Extended Tandem Algorithm is a parallel, bloc-wise extension of the Tandem Algorithm designed to extract and simplify isosurfaces from very large volumetric datasets while minimizing memory consumption. It solves the scalability bottleneck of the sequential Tandem Algorithm, where queue updates and simplification steps become too memory-intensive for high-resolution data, such as generating 1,250,000 triangles per layer on a $1626 \times 7028 \times 2000$ dataset. Unlike the parent method, which processes the entire volume in a single pass, this method splits the dataset into blocks, distributes them across computing nodes, and employs a component dumping mechanism to persist finalized surface components to disk. This allows the extraction process to continue on sub-parts of the dataset without losing previously computed geometry, making it suitable for geophysical datasets with numerous small, disconnected components.

### Components Dumping
A mechanism that frees memory by migrating independent surface components to disk along with meta-data (oriented bounding box, volume) as soon as they are extracted, simplified, and detected as finished [S3]. It detects when a surface component is finished (simplification complete), dumps the completed surface component to disk, and stores meta-data with the dumped component, including oriented bounding box and volume, for later filtering and exploration [S3]. This is useful for noisy datasets or when the isovalue is not well determined, leading to many small disconnected components [S3]. Meta-data allows filtering, such as discarding components with too small volumes or selecting aligned ones [S3].

## Operating characteristics
- **Time Complexity**: The algorithm performs one pass over the cross-sections. Each extraction step is $O(N_{layer})$ where $N_{layer}$ is the number of cells in the layer. Simplification involves edge collapses, each taking time proportional to the degree of the vertices involved. The total time is dominated by the number of edge collapses, which is bounded by the number of triangles in the final mesh.
- **Memory Complexity**: The memory usage is bounded by the size of the current layer plus the simplified history. The Tandem Algorithm ensures that the number of vertices and triangles in memory is proportional to the surface area of the extracted portion, not the volume. The memory constraint is controlled by $E_0$ and the simplification rate [S1].
- **Measured Performance**: On a noisy dataset of size $1626 \times 7028 \times 2000$, the sequential Tandem Algorithm generated 1,250,000 triangles per layer, leading to memory bottlenecks in queue updates and simplification [S1][S2, unofficial]. The Extended Tandem Algorithm addresses this by distributing work and dumping components [S2, unofficial].
- **Strengths**: Reduces memory consumption significantly compared to global extraction. Maintains global simplification quality.
- **Limitations**: Sequential version does not scale well to extremely large datasets due to queue management overhead. Requires careful tuning of $E_0$ and $\alpha$ [S1].

## Background
The Tandem Algorithm was proposed by Attali, Cohen-Steiner, and Edelsbrunner in 2005 in 'Extraction and Simplification of Iso-surfaces in Tandem' at the Eurographics Symposium on Geometry Processing [S2, unofficial]. It revisits the quadratic metric of Garland and Heckbert for edge collapse simplification, introducing a shape measure and a mesh isotropy criterion [S1]. The algorithm alternates extraction and simplification to reduce memory occupancy [S1]. The Extended Tandem Algorithm was later proposed to address scalability issues on very large datasets, particularly in geophysics, by introducing parallel processing and component dumping [S3].

## Sources
[S1] Parallel extraction and simplification of large isosurfaces using an extended tandem algorithm, https://doi.org/10.1016/j.cad.2009.04.016, fetched 2026-10-26
[S2] July 2012 – 0 FPS, https://0fps.net/2012/07/, fetched 2026-10-30 (unofficial)
[S3] Isosurface extraction and interpretation on very large datasets in geophysics, https://inria.hal.science/inria-00277070v1, fetched 2026-10-26
