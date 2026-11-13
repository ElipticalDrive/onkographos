# Face Subsampling

Also called: Adaptive Face Subsampling, Internal Face Subsampling

## Overview
Face Subsampling is a Level of Detail (LOD) technique for tetrahedral volume rendering that reduces computational load by dynamically selecting a subset of internal mesh faces for rasterization, while always retaining the boundary faces to maintain domain integrity [S1, unofficial]. It solves the problem of interactive frame rates on large tetrahedral meshes by adapting the number of rendered primitives to the available rendering time [S1, unofficial]. Unlike [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md) which operates in the frequency domain, or [Multiresolution Volume Sampling: Intrablock](11149-multiresolution-volume-sampling-intrablock.md) which operates on voxel grids, Face Subsampling operates directly on the geometric topology of the tetrahedral mesh [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh defined by a set of unique faces $F$, a target frame time $T_{target}$ (typically 0.01 seconds for interactive viewing), and the measured render time of the previous frame $T_{render}$ [S1, unofficial].
*   **Output:** A subset of faces $B \cup I_{selected}$, where $B$ is the set of boundary faces and $I_{selected}$ is the selected subset of internal faces [S1, unofficial].

**Data Structures:**
*   **Faces:** Each face is defined by its vertices and associated scalar values. Internal faces have a neighboring face with the same vertices and scalar values to avoid redundancy [S1, unofficial].
*   **Boundary Faces ($B$):** Faces on the exterior of the mesh domain $D$. These are constant and always included [S1, unofficial].
*   **Internal Faces ($I$):** Faces shared between tetrahedra inside the domain. These are subject to subsampling [S1, unofficial].
*   **Layer Lists:** For Topology Sampling, faces are partitioned into a boundary face list (layer 0) and internal face lists (layers 1, 2, ...) [S1, unofficial].

**Algorithm Steps:**

1.  **Extract Boundary and Internal Faces:**
    Identify the set of unique faces $F$. Partition $F$ into boundary faces $B$ and internal faces $I$ such that $B \cup I = F$ and $B \cap I = \emptyset$ [S1, unofficial].

2.  **Determine Target Internal Face Count:**
    Calculate the number of internal faces $|I_{target}|$ to render in the current frame based on the previous frame's performance:
    $$|I_{target}| = |I| \times \frac{T_{target}}{T_{render}}$$
    where $|I|$ is the total number of internal faces available [S1, unofficial].
    *   If $|I_{target}| > |I|$, set $|I_{target}| = |I|$.
    *   If $|I_{target}| < 0$, set $|I_{target}| = 0$ [S1, unofficial].

3.  **Select Internal Faces:**
    Select $|I_{target}|$ faces from $I$ using one of the sampling strategies (see Subtopics). The selected faces form $I_{selected}$ [S1, unofficial].

4.  **Render:**
    Render the union $B \cup I_{selected}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Exceeding Limits:** If the total number of layer faces (in Topology Sampling) exceeds the allowed internal face limit, a subset of layers is rendered by picking an even distribution of layers throughout the mesh until the target internal face count is reached [S1, unofficial].
*   **Clumping:** Uniform random sampling of faces can lead to clumping and holes, resulting in poor visual quality [S1, unofficial]. Stratified or priority-based sampling is preferred [S1, unofficial].
*   **Discontinuities:** Sampling methods that ignore connectivity (e.g., Field Sampling) may produce obvious discontinuities in intensity [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def face_subsampling(mesh, target_time, prev_render_time):
    """
    Selects a subset of internal faces for rendering based on adaptive time budget.
    
    Args:
        mesh: Tetrahedral mesh object with methods get_boundary_faces(), 
              get_internal_faces(), and get_total_internal_faces().
        target_time: Desired frame time in seconds (e.g., 0.01).
        prev_render_time: Time taken to render the previous frame in seconds.
        
    Returns:
        list: A list of faces to render (Boundary + Selected Internal).
    """
    # 1. Extract Boundary and Internal Faces
    boundary_faces = mesh.get_boundary_faces()
    internal_faces = mesh.get_internal_faces()
    total_internal = len(internal_faces)
    
    # 2. Determine Target Internal Face Count
    # Avoid division by zero; assume 1.0 if prev_render_time is 0
    if prev_render_time <= 0:
        prev_render_time = 1.0
        
    target_internal_count = total_internal * (target_time / prev_render_time)
    
    # Clamp to valid range
    target_internal_count = max(0, min(target_internal_count, total_internal))
    
    # 3. Select Internal Faces
    # Default to Area Sampling if no specific strategy is provided
    selected_internal = area_sampling(internal_faces, int(target_internal_count))
    
    # 4. Return Renderable Set
    return boundary_faces + selected_internal

def area_sampling(internal_faces, count):
    """
    Selects faces based on area, prioritizing larger faces with randomization.
    
    Args:
        internal_faces: List of internal face objects.
        count: Number of faces to select.
        
    Returns:
        list: Selected subset of faces.
    """
    if count == 0:
        return []
        
    # Calculate priority for each face: Area * Random Number
    # This favors larger triangles but preserves possibility of choosing small ones
    priorities = []
    for face in internal_faces:
        area = face.get_area()
        random_val = random.uniform(0, 1)
        priority = area * random_val
        priorities.append((priority, face))
        
    # Sort by priority descending
    priorities.sort(key=lambda x: x[0], reverse=True)
    
    # Select top 'count' faces
    selected = [face for _, face in priorities[:count]]
    
    return selected
```

## Subtopics

### Topology Sampling
Topology Sampling assigns layers to faces by peeling the mesh from the outside in. It extracts the boundary, removes boundary tetrahedra, and repeats until all tetrahedra are peeled. Faces on the boundary at each step are assigned a new layer value. Boundary faces form layer 0; internal faces form subsequent layers [S1, unofficial].
*   **Step:** Initialize `CurrentLayer` to 0. Loop while there exists an unpeeled tetrahedron $t$. For each face $f$ in unpeeled tetrahedra, if $f$ is external, assign it `CurrentLayer`. Mark shared tetrahedra as peeled. Increment `CurrentLayer` [S1, unofficial].
*   **Pitfall:** If all layer faces cannot be drawn because they exceed the internal face limit, a subset of layers is rendered by picking an even distribution of layers throughout the mesh until the target internal face count is reached [S1, unofficial].
*   **Characteristics:** Produces images without obvious holes but may have correlated regions in the image [S1, unofficial].

### Field Sampling
Field Sampling assigns importance to a mesh face based on the average scalar value at its vertices. It uses stratified sampling on a histogram of normalized scalar values [S1, unofficial].
*   **Definition:** A preprocessing strategy that assigns importance to a mesh face based on the average scalar value at its vertices to select a subset of faces for rendering [S1, unofficial].
*   **Step:** Build a histogram of normalized scalar values at the mesh vertices. Divide the histogram into evenly spaced intervals (typically 128). Randomly pick the same number of faces from each interval to fill the internal face list [S1, unofficial].
*   **Parameter:** Histogram Intervals: 128 [S1, unofficial].
*   **Comparison:** Ignores connectivity, leading to obvious discontinuities in intensity, but is a robust strategy [S1, unofficial]. Unlike Monte Carlo integration, it randomly samples over the entire space of functions without extra computation per viewpoint [S1, unofficial].

### Area Sampling
Area Sampling selects internal faces based on their geometric size (area) to minimize total error magnitude, which is proportional to the area of the triangle [S1, unofficial].
*   **Definition:** A sampling strategy that selects internal faces based on their geometric size (area) [S1, unofficial].
*   **Formula:** Priority is calculated as $A_i \times \xi_i$, where $A_i$ is the area in $\mathbb{R}^3$ of the $i$-th face and $\xi_i$ is a uniform random number [S1, unofficial].
*   **Step:** Randomize the selection using the uniform random number to favor larger triangles while preserving the possibility of choosing small faces [S1, unofficial].
*   **Comparison:** Unlike view-aligned sampling, it is independent of viewpoint and does not have flickering artifacts [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The selection phase is $O(N)$ or $O(N \log N)$ depending on the sampling strategy (sorting for Area Sampling), where $N$ is the number of internal faces. The rendering time is proportional to the number of selected faces [S1, unofficial].
*   **Memory Complexity:** Requires storing the mesh topology and precomputed sampling data (e.g., histograms for Field Sampling) [S1, unofficial].
*   **Performance:** On the Fighter dataset (1.4 million tetrahedra), full quality rendered at 1.3 fps, 15% LOD at 4.5 fps, and 5% LOD at 10.0 fps using Area Sampling [S1, unofficial].
*   **Strengths:** Adaptive to rendering time; view-independent methods (Topology, Field, Area) avoid flickering artifacts [S1, unofficial].
*   **Limitations:** Field Sampling ignores connectivity, causing intensity discontinuities. Topology Sampling may produce correlated regions. Area Sampling reduces the number of rasterized triangles, which may lower visual fidelity despite error minimization [S1, unofficial].
*   **When to Use:** Interactive volume rendering of large tetrahedral meshes where frame rate stability is critical [S1, unofficial].

## Background
Face Subsampling was introduced by Steve Callahan in his 2006 Master's thesis, "The K-Buffer and Its Applications to Volume Rendering" [S1, unofficial]. It builds on the Hardware-Assisted Visibility Sorting (HAVS) volume rendering system [S1, unofficial]. The method addresses the limitations of uniform sampling and the need for adaptive LOD in tetrahedral volume rendering [S1, unofficial].

## Sources
[S1] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
