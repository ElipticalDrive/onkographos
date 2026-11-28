# Slow Growing Subdivision Connectivity

Also called: SGS Connectivity, Slow subdivision, Slow-Growing Subdivision, Slow-growing volumetric subdivision, Tiered Subdivision, d-dimensional edge bisection scheme

## Overview
Slow Growing Subdivision (SGS) is a volumetric subdivision scheme that refines meshes of arbitrary topology and dimension by splitting cells in independent stages, thereby avoiding the exponential vertex proliferation characteristic of tensor-product schemes [S2, unofficial]. It generalizes Catmull-Clark refinement rules into a multi-tier connectivity structure, allowing gradual resolution increase with small stencils while naturally handling sharp features and adaptive refinement without temporary partitions [S1][S2, unofficial]. Unlike [Longest Edge Bisection (LEB) Tetrahedral Hierarchy](2486-longest-edge-bisection-leb-tetrahedral-hierarchy.md), which is restricted to simplicial complexes, SGS operates on general polyhedral cells [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A base mesh $M_0$ consisting of polyhedral cells (topological balls) in dimension $d$ [S2, unofficial].
*   **Output:** A refined mesh at level $l+1$, pass $0$, derived from level $l$, pass $3$ [S2, unofficial].

**Data Structures:**
The mesh state is tracked by a pair $(l, p)$, where $l$ is the subdivision level and $p$ is the pass index [S2, unofficial].
*   $l \in \mathbb{Z}_{\ge 0}$
*   $p \in \{0, 1, 2, 3\}$ for $d=3$. For general $d$, $p \in \{0, \dots, d\}$ [S2, unofficial].
*   Base mesh is $(0, 0)$ [S2, unofficial].
*   Transition: $(l, 3) \equiv (l+1, 0)$ [S1][S2, unofficial].
*   Elements are tagged by the tier/pass at which they exist (e.g., "tier 0 faces") [S1].

**Parameters:**
*   **Feature Tags:** Faces and edges can be tagged as "features." Feature faces constrain subdivision to 2D surface rules; feature edges constrain to 1D [S1].
*   **Smoothing Coefficient ($\alpha$):** Used in vertex repositioning. Example value $\alpha = 0.5$ [S2, unofficial].

**Algorithm Steps:**
The refinement from pass $p$ to $p+1$ involves inserting vertices and merging pyramids [S2, unofficial]. The process is recursive over passes $0$ to $d$ (or $0$ to $3$ for 3D) [S2, unofficial].

1.  **Pass 0 to Pass 1 (Cell Centers):**
    *   For each cell $C_i$ in the current mesh, compute its center $p_i$ [S2, unofficial].
    *   Decompose $C_i$ (with $n$ facets) into $n$ pyramidal cells by connecting $p_i$ to each facet $f_k$ [S2, unofficial]. Denote the pyramid connecting $p_i$ to facet $f_k$ as $p_i \diamond f_k$ [S2, unofficial].
    *   For each pair of adjacent cells $C_i, C_j$ sharing a facet $f_k$, create a new cell $F_k$ by merging the pyramids:
        $$ F_k = (p_i \diamond f_k) \cup (p_j \diamond f_k) $$
        where $f_k = C_i \cap C_j$ [S2, unofficial].
    *   **Boundary/Sharp Handling:** If $f_k$ is a boundary or a sharp feature, build only one half of $F_k$ (or two halves without merging for 2D sharp features) [S2, unofficial].

2.  **Pass 1 to Pass 2 (Edge Centers):**
    *   For each cell $F_k$ in the pass 1 mesh, determine its center $q_k$ [S2, unofficial].
    *   Decompose $F_k$ into pyramids by connecting $q_k$ to new facets $g_i$ generated in pass 1. For sharp cells, $q_k$ is joined only to new facets, not the old facet $f_k$ [S2, unofficial].
    *   Each resulting pyramid contains exactly one edge $e_j$ from the pass 0 mesh [S2, unofficial].
    *   Merge all pyramids incident to the same edge $e_j$ into a cell $E_j$. The collection of $E_j$ forms the pass 2 mesh [S2, unofficial].

3.  **Pass 2 to Pass 3 (Vertex Centers):**
    *   Determine the center $r_i$ of each cell $E_i$ [S2, unofficial].
    *   Refine $E_i$ by joining $r_i$ with each facet. For sharp cells, $r_i$ is the center of the edge $e_i$ and is shared among all cells around $e_i$ [S2, unofficial].
    *   Merge all cells that contain the same pair of vertices (specifically, a vertex of the base mesh and the center of a base cell) [S2, unofficial].
    *   **Spurious Edge Removal:** Remove any edge connecting two vertices introduced in passes differing by more than one (e.g., a pass 0 vertex connected to a pass 2 vertex) [S2, unofficial].

**General Pass $p$ to $p+1$ (for $p < d-1$):**
*   Insert a vertex $q$ at the center of a $(d-p)$-dimensional face of the base mesh $M_l$ [S2, unofficial].
*   Partition cell $c$ into pyramids by connecting $q$ to all facets [S2, unofficial].
*   Merge all pyramids containing a common $(d-p-1)$-dimensional face of $M_l$ [S2, unofficial].
*   If a $(d-p)$-dimensional face $f$ is on the boundary of $c$, place $q$ at the center of $f$ and connect to facets not containing $f$ [S2, unofficial].

**General Pass $d-1$ to Pass $d$:**
*   Insert vertex $q$ at the edge of $M_l$. For non-boundary/non-sharp features, $q$ is the center of $c$ [S2, unofficial].
*   Partition $c$ into pyramids by connecting $q$ to facets not containing $q$ [S2, unofficial].
*   Merge all pyramids containing the same vertex of $M_l$ and center of a cell of $M_l$ [S2, unofficial].

**Vertex Repositioning (Smoothing):**
A stationary averaging rule is applied for smoothing [S2, unofficial]:
$$ V_{new} = V_{old} \cdot \alpha + (1 - \alpha) \cdot w $$
where $V_{old}$ is the old position, $w$ is the average of edge-connected vertices, and $\alpha$ is the smoothing coefficient [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def slow_growing_subdivision(mesh, level, pass_idx, dimension):
    """
    Performs one subdivision step (pass) for a mesh.
    mesh: List of cells. Each cell is a dict with 'vertices', 'facets', 'center'.
    level: Current subdivision level.
    pass_idx: Current pass index (0 to dimension).
    dimension: Intrinsic dimension d.
    """
    if pass_idx == dimension:
        return slow_growing_subdivision(mesh, level + 1, 0, dimension)

    new_cells = []
    pyramids = []

    # Step 1: Insert centers and create pyramids for current pass
    for cell in mesh:
        center = compute_center(cell)
        cell['center'] = center
        for facet in cell['facets']:
            sharp = is_feature(facet) or is_boundary(facet)
            pyramids.append({'center': center, 'base': facet, 'sharp': sharp})

    # Step 2: Merge pyramids based on pass index
    if pass_idx == 0:
        new_cells = merge_by_facet(pyramids)
    elif pass_idx == 1:
        new_cells = merge_by_edge(pyramids, mesh)
    elif pass_idx == 2:
        new_cells = merge_by_vertex(pyramids, mesh)
    else:
        new_cells = merge_by_dimension(pyramids, pass_idx, dimension)

    # Step 3: Remove spurious edges
    return remove_spurious_edges(new_cells)

def compute_center(cell):
    # [G] Geometric centroid or combinatorial center calculation
    return sum(cell['vertices']) / len(cell['vertices'])

def merge_by_facet(pyramids):
    # [G] Groups pyramids by shared facet and merges them into tier 1 cells
    merged = {}
    for p in pyramids:
        key = tuple(sorted(p['base']))
        merged.setdefault(key, []).append(p)
    result = []
    for group in merged.values():
        result.append({'vertices': list(set(v for p in group for v in p['vertices'])), 'facets': group[0]['base']})
    return result

def merge_by_edge(pyramids, prev_mesh):
    # [G] Groups pyramids by shared edge from prev_mesh
    merged = {}
    for p in pyramids:
        edge = tuple(sorted(p['base'][:2]))
        merged.setdefault(edge, []).append(p)
    result = []
    for group in merged.values():
        result.append({'vertices': list(set(v for p in group for v in p['vertices'])), 'facets': group[0]['base']})
    return result

def merge_by_vertex(pyramids, prev_mesh):
    # [G] Groups pyramids by shared vertex from prev_mesh
    merged = {}
    for p in pyramids:
        vertex = p['base'][0]
        merged.setdefault(vertex, []).append(p)
    result = []
    for group in merged.values():
        result.append({'vertices': list(set(v for p in group for v in p['vertices'])), 'facets': group[0]['base']})
    return result

def merge_by_dimension(pyramids, pass_idx, dimension):
    # [G] General merging logic for d dimensions
    return merge_by_facet(pyramids)

def remove_spurious_edges(cells):
    # [G] Removes edges connecting vertices from passes differing by > 1
    return cells

def is_feature(facet):
    # [G] Checks if facet is tagged as a feature
    return False

def is_boundary(facet):
    # [G] Checks if facet is on the mesh boundary
    return False
```

## Subtopics

### [Generalized Linear B-Spline Wavelets](7672-generalized-linear-b-spline-wavelets.md)
Generalized Linear B-Spline Wavelets is a wavelet transform designed for progressive compression of volumetric subdivision meshes, specifically those generated by Slow Growing Subdivision (SGS) connectivity. It solves the problem of representing hierarchical volumetric data with controllable error tolerances by computing wavelet coefficients that capture the difference between predicted cell centers (centroids) and actual inserted vertices. Unlike the alternative SGS Wavelet Transform, which smooths original vertices using neighbor positions, Generalized Linear B-Spline Wavelets smooths original vertices exclusively through the influence of wavelet coefficients, resulting in superior compression efficiency at low bit rates.

### [Direct and Indirect Smoothing](17980-direct-and-indirect-smoothing.md)
Direct and Indirect Smoothing is a stationary averaging rule coupled with combinatorial subdivision to determine the actual positions of mesh vertices in slow-growing subdivision schemes. It solves the problem of defining geometric continuity and function values on refined non-regular volumetric meshes by distinguishing between the placement of new vertices (direct) and the repositioning of existing vertices (indirect). It sets itself apart from sibling methods like Generalized Linear B-Spline Wavelets by explicitly decoupling the combinatorial topology changes from the geometric smoothing weights, allowing for $C^1$ continuity at regular points when specific parameters are chosen.

## Operating characteristics
**Time and Memory Complexity:**
*   **Vertex Growth:** In the uniform case, the scheme doubles the number of vertices at each refinement, independent of dimension $d$ [S2, unofficial]. It is classified as a $2^{1/d}$ subdivision scheme [S2, unofficial].
*   **Cell Count:** For $d$-dimensional rectilinear grids, the total number of cells is linear in the number of vertices [S2, unofficial]. In the worst-case scenario for a general simplicial complex with $n$ vertices, the total number of cells $N$ can be as large as $O(n^{\lfloor d/2 \rfloor})$ [S2, unofficial].
*   **Refinement Rate:** The rate of vertex increase $\beta$ is $O(1)$ for $d=2$ and $d=3$, but $O(W)$ for $d=4$ [S2, unofficial].

**Measured Performance:**
*   In 3D rectilinear grid refinement: Pass 1 adds 1 vertex per original vertex (doubling total). Pass 2 adds 3 vertices per original vertex (factor of 2.5 increase). Pass 3 adds 3 vertices per original vertex (factor of 1.6 increase) [S2, unofficial].
*   Compared to tensor-product refinements, SGS doubles vertices in 2D (vs. quadrupling) and 3D (vs. factor of eight increase) [S2, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Handles arbitrary polyhedral topology; naturally embeds sharp features and non-manifold edges; adaptive refinement requires no temporary partitions; small stencil size [S2, unofficial].
*   **Limitations:** Geometric interpretation of 'center' is combinatorial; actual vertex position choices affect smoothness; spurious edges may be introduced and must be explicitly removed [S2, unofficial]. For $d > 3$, the growth rate may not be constant in the worst case [S2, unofficial].

**When to Use:**
*   Use for solid modeling, meshing, and scientific visualization where adaptive refinement of general polyhedral volumes is needed [S2, unofficial].

## Background
**Origin:**
The scheme was introduced to address the "curse of dimensionality" in volumetric subdivision, where vertex proliferation grows exponentially with dimension in standard tensor-product schemes [S2, unofficial]. It generalizes the W subdivision scheme to 3D and higher dimensions [S2, unofficial].

**History:**
The connectivity rules are a factorization of Catmull-Clark refinement rules into three distinct steps (tiers) for 3D [S1]. The scheme is combinatorially equivalent to Rivara’s edge bisection for particular tetrahedralizations of rectilinear grids, and equivalent to the d-dimensional edge bisection scheme of Maubach [S2, unofficial]. It has been used in conjunction with wavelet transforms for progressive compression of scientific data sets [S1].

## Sources
[S1] Progressive Compression of Volumetric Subdivision Meshes, https://digital.library.unt.edu/ark:/67531/metadc1405535/?q=volume data compression rendering, fetched 2026-10-28
[S2] Slow-Growing Subdivisions in any Dimension: Towards Removing the Curse of Dimensionality, https://digital.library.unt.edu/ark:/67531/metadc1413064/?q=hierarchical volume rendering wavelets multiresolution, fetched 2026-10-29 (unofficial)
