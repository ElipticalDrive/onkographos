# Face Subsampling

Also called: Adaptive Face Subsampling, Internal Face Subsampling

## Overview
Face Subsampling is a Level-of-Detail (LOD) technique for tetrahedral volume rendering that dynamically selects a subset of mesh faces to render, balancing visual fidelity against frame rate constraints [S1, unofficial]. It solves the problem of interactive rendering on large tetrahedral meshes by reducing the number of rasterized primitives without pre-aggregating data into voxels or bricks [S1, unofficial]. It differs from sibling methods like [Hierarchical Culling & Multi-resolution Approximation](2010-hierarchical-culling-multi-resolution-approximation.md) by operating directly on the unstructured mesh topology rather than a hierarchical grid, and from [Focus Point Based Brick LOD Selection](5282-focus-point-based-brick-lod-selection.md) by being primarily face-centric rather than brick-centric [S1, unofficial].

## How it works
The method operates on a tetrahedral mesh where the domain $D$ is bounded by a set of unique faces $F$. The core mechanism separates $F$ into boundary faces $B$ and internal faces $I$, such that $B \cup I = F$ [S1, unofficial]. The boundary faces $B$ are kept constant to ensure the domain bounds are always rendered [S1, unofficial]. The internal faces $I$ are sampled adaptively to form the renderable set $B \cup I_{selected}$ [S1, unofficial].

### Inputs and Outputs
*   **Input:** A tetrahedral mesh with unique faces $F$, boundary faces $B$, internal faces $I$, and a target frame time $T_{target}$ (typically 0.01 seconds for interactive viewing) [S1, unofficial].
*   **Output:** A subset of internal faces $I_{selected}$ to be rendered alongside $B$ [S1, unofficial].

### Data Structures
*   **Face List:** Each face contains vertex indices and scalar values. Internal faces store a reference to a neighboring face with identical vertices and scalar values to avoid redundant computation of zero-length inter-cell distances [S1, unofficial].
*   **Layer List:** Used in Topology Sampling, faces are assigned an integer layer value during preprocessing [S1, unofficial].

### Adaptive Selection Formula
The number of internal faces to select, $|I_{selected}|$, is determined dynamically based on the previous frame's render time $T_{render}$ [S1, unofficial]:

$$
|I_{selected}| = \left\lfloor |I| \times \frac{T_{target}}{T_{render}} \right\rfloor
$$

If the calculated $|I_{selected}|$ exceeds the available internal face count, or if all layer faces cannot be drawn due to limits, a subset of layers is rendered by picking an even distribution of layers throughout the mesh until the target internal face count is reached [S1, unofficial].

### Sampling Strategies
Four strategies exist for selecting $I_{selected}$ from $I$. These are detailed in Subtopics.

### Edge Cases and Pitfalls
*   **Exceeding Limits:** If the required number of faces exceeds the limit for interactive rendering, the system must subsample the selected layers evenly [S1, unofficial].
*   **Connectivity Discontinuities:** Field Sampling ignores mesh connectivity, leading to obvious discontinuities in intensity [S1, unofficial].
*   **Fill Rate:** Area Sampling prioritizes larger faces, which lowers visual error but increases fill requirements, potentially lowering frame rates if the GPU is fill-rate limited [S1, unofficial].

## Subtopics

### Topology Sampling
Topology Sampling assigns layers to faces by peeling the tetrahedral mesh from the outside in [S1, unofficial]. It selects faces that are topologically connected, resulting in images that look more continuous and lack obvious holes, though it may produce correlated regions in the image [S1, unofficial].

**Preprocessing Steps:**
1.  Initialize `CurrentLayer` to 0.
2.  Mark all tetrahedra as unpeeled.
3.  Loop while there exists an unpeeled tetrahedron $t$:
    *   Identify all external faces $f$ of the unpeeled tetrahedra.
    *   Assign `CurrentLayer` to these faces.
    *   Mark tetrahedra sharing these faces as peeled.
    *   Increment `CurrentLayer`.
4.  Separate faces into the boundary face list (layer 0) and internal face list (layers > 0) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_topology_layers(tetrahedra):
    peeled = {t: False for t in tetrahedra}
    current_layer = 0
    boundary_faces = []
    internal_faces = []
    
    while any(not p for p in peeled.values()):
        current_layer_faces = []
        for t in tetrahedra:
            if peeled[t]:
                continue
            for f in t.faces:
                if f.is_external():
                    current_layer_faces.append(f)
        
        for f in current_layer_faces:
            if f in boundary_faces:
                continue
            if current_layer == 0:
                boundary_faces.append(f)
            else:
                internal_faces.append(f)
        
        for t in tetrahedra:
            if peeled[t]:
                continue
            for f in t.faces:
                if f in current_layer_faces:
                    peeled[t] = True
                    break
        
        current_layer += 1
        
    return boundary_faces, internal_faces
```

### Field Sampling
Field Sampling assigns importance to a mesh face based on the average scalar value at its vertices [S1, unofficial]. It uses stratified sampling over a histogram of normalized scalar values to ensure a representative distribution of field values in the rendered subset [S1, unofficial]. It is robust and requires no per-viewpoint recomputation, but ignores connectivity, leading to intensity discontinuities [S1, unofficial].

**Parameters:**
*   **Histogram Intervals:** Number of evenly spaced intervals for stratified sampling. Typical value: 128 [S1, unofficial].

**Steps:**
1.  Build a histogram of normalized scalar values at the mesh vertices.
2.  Divide the histogram into 128 evenly spaced intervals.
3.  Randomly pick the same number of faces from each interval to fill the internal face list [S1, unofficial].

### Area Sampling
Area Sampling selects internal faces based on their geometric size (area in $\mathbb{R}^3$) [S1, unofficial]. It prioritizes faces to minimize total error magnitude, which is proportional to the triangle area [S1, unofficial]. It is viewpoint-independent, avoiding flickering artifacts, but favors larger triangles which may increase fill rate requirements [S1, unofficial].

**Formula:**
Priority for face $i$ is calculated as:

$$
P_i = A_i \times \xi_i
$$

where $A_i$ is the area of the $i$-th face in $\mathbb{R}^3$ and $\xi_i$ is a uniform random number [S1, unofficial].

**Steps:**
1.  Calculate the area $A_i$ for each internal face.
2.  Generate a uniform random number $\xi_i$ for each face.
3.  Sort faces by priority $P_i$ and select the top $|I_{selected}|$ faces [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing (Topology/Field) is $O(N)$ where $N$ is the number of faces/tetrahedra. Adaptive selection is $O(1)$ per frame after preprocessing. View-aligned sampling (not detailed in notes but mentioned as a sibling strategy in the text) requires per-viewpoint computation [S1, unofficial].
*   **Memory Complexity:** Requires storing the mesh topology, face lists, and potentially layer assignments or histograms. Internal faces store neighbor references [S1, unofficial].
*   **Performance:** On the Fighter dataset (1.4 million tetrahedra), full quality rendered at 1.3 fps. At 15% LOD, it reached 4.5 fps. At 5% LOD, it reached 10.0 fps [S1, unofficial].
*   **Strengths:** Natural representation for traversing LODs; no need for explicit simplification hierarchies like those required for cells; viewpoint-independent methods avoid flickering [S1, unofficial].
*   **Limitations:** Field Sampling causes intensity discontinuities; Area Sampling increases fill rate; Topology Sampling may create correlated visual artifacts; View-aligned sampling causes frame-to-frame coherence issues [S1, unofficial].

## Background
Face Subsampling was proposed by Steve Callahan in his Master's thesis, "The K-Buffer and Its Applications to Volume Rendering" (2006) [S1, unofficial]. It builds on the Hardware-Assisted Visibility Sorting (HAVS) volume rendering system [S1, unofficial]. The method addresses the limitations of uniform random sampling, which suffers from clumping and holes, by introducing structured sampling strategies (Topology, Field, Area) [S1, unofficial].

## Sources
[S1] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
