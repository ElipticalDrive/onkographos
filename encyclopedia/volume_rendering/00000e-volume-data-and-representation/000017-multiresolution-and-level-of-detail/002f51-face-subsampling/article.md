# Face Subsampling

Also called: Adaptive Face Subsampling, Internal Face Subsampling

## Overview
Face Subsampling is a Level of Detail (LOD) technique for tetrahedral volume rendering that reduces computational load by dynamically selecting a subset of internal mesh faces for rasterization, while keeping the boundary faces constant [S1, unofficial]. It solves the problem of maintaining interactive frame rates on complex volumetric datasets by adapting the number of rendered primitives to the available rendering time [S1, unofficial]. Unlike [Wavelet-based Volume Rendering](1052-wavelet-based-volume-rendering.md) which operates in the frequency domain, or [Mesh Simplification](1032-mesh-simplification.md) which modifies geometry topology, Face Subsampling preserves the original mesh structure and instead culls faces based on adaptive criteria such as topology, field values, or geometric area [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A tetrahedral mesh defined by a set of unique faces $F$, a target frame time $T_{target}$ (typically 0.01 seconds for interactive viewing), and the measured render time of the previous frame $T_{render}$ [S1, unofficial].
*   **Output:** A subset of faces $F_{render} = B \cup I_{selected}$, where $B$ is the set of boundary faces and $I_{selected}$ is the selected subset of internal faces [S1, unofficial].

**Data Structures:**
The mesh faces are partitioned into two lists:
1.  **Boundary Face List ($B$):** Faces on the exterior of the domain. These are never culled and remain constant across LOD levels [S1, unofficial].
2.  **Internal Face List ($I$):** Faces shared between tetrahedra. This list is subdivided based on the chosen sampling strategy (e.g., layers for Topology Sampling, histogram bins for Field Sampling) [S1, unofficial].

**Algorithm Steps:**

1.  **Extract Boundary and Internal Faces:**
    Identify all unique faces $F$ in the tetrahedral mesh. Partition $F$ into boundary faces $B$ and internal faces $I$ such that $B \cup I = F$ and $B \cap I = \emptyset$ [S1, unofficial].

2.  **Determine Target Internal Face Count:**
    Calculate the number of internal faces $|I_{target}|$ to render in the current frame using the adaptive formula:
    $$ |I_{target}| = |I| \times \frac{T_{target}}{T_{render}} $$
    where $|I|$ is the total number of internal faces available [S1, unofficial].
    *   If $|I_{target}| > |I|$, set $|I_{target}| = |I|$.
    *   If $|I_{target}| < 0$, set $|I_{target}| = 0$ [S1, unofficial].

3.  **Select Internal Faces:**
    Select $|I_{target}|$ faces from $I$ based on one of the following strategies (see Subtopics):
    *   **Topology Sampling:** Select faces from evenly distributed layers.
    *   **Field Sampling:** Select faces via stratified random sampling based on scalar value histograms.
    *   **Area Sampling:** Select faces based on geometric area weighted by a random number.
    *   **View-Aligned Sampling:** Select faces based on alignment with the view vector (not detailed in notes, but mentioned as a sibling strategy) [S1, unofficial].

4.  **Render:**
    Rasterize the union of the constant boundary faces $B$ and the selected internal faces $I_{selected}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def face_subsampling(mesh, target_time=0.01):
    # 1. Preprocessing: Extract Boundary and Internal Faces
    # This step is typically done once or when the mesh changes.
    boundary_faces, internal_faces = extract_boundary_and_internal(mesh)
    
    # 2. Adaptive Selection
    # Get render time from previous frame. If first frame, use a default or full count.
    render_time = get_previous_frame_time()
    if render_time is None:
        render_time = target_time # Fallback to render all if no history
        
    total_internal = len(internal_faces)
    
    # Calculate target count using the adaptive formula
    # |I| = |I| * (TargetTime / RenderTime)
    target_count = int(total_internal * (target_time / render_time))
    
    # Clamp to valid range
    target_count = max(0, min(target_count, total_internal))
    
    # 3. Select Faces based on Strategy
    # Note: The specific selection logic depends on the variant (Topology, Field, Area)
    # For a generic implementation, we assume a 'select_faces' function exists
    # which implements the specific variant's logic on the internal_faces list.
    selected_internal = select_faces(internal_faces, target_count)
    
    # 4. Form Render Set
    render_set = boundary_faces + selected_internal
    
    return render_set
```

**Edge Cases and Pitfalls:**
*   **Exceeding Limit:** If the number of layer faces (in Topology Sampling) exceeds the target internal face count, a subset of layers is rendered by picking an even distribution of layers throughout the mesh until the target count is reached [S1, unofficial].
*   **Discontinuities:** Field Sampling ignores connectivity, which can lead to obvious discontinuities in intensity [S1, unofficial].
*   **Flickering:** View-Aligned Sampling can cause flickering artifacts between frames due to view-dependence, whereas Topology, Field, and Area sampling are view-independent and do not flicker [S1, unofficial].

## Subtopics

### Topology Sampling
A preprocessing strategy that assigns layers to faces by peeling the tetrahedral mesh from the outside in. It extracts the boundary, removes boundary tetrahedra, and repeats until all tetrahedra are peeled, assigning a layer value to faces at each step [S1, unofficial].
*   **Step:** Initialize `CurrentLayer` to 0. Loop while unpeeled tetrahedra exist. For each face of unpeeled tetrahedra, if the face is external, assign it `CurrentLayer`. Mark shared tetrahedra as peeled. Increment `CurrentLayer` [S1, unofficial].
*   **Characteristics:** Produces images without obvious holes and maintains connected regions, but may result in correlated regions in the image [S1, unofficial].
*   **When to choose:** When visual continuity and lack of holes are prioritized over uniform distribution [S1, unofficial].

### Field Sampling
A strategy that assigns importance to faces based on the average scalar value at their vertices. It uses stratified sampling on a histogram of normalized scalar values [S1, unofficial].
*   **Step:** Build a histogram of normalized scalar values at mesh vertices. Divide the histogram into evenly spaced intervals (typically 128). Randomly pick the same number of faces from each interval to fill the internal face list [S1, unofficial].
*   **Characteristics:** Robust and view-independent. Ignores connectivity, leading to intensity discontinuities. Unlike Monte Carlo integration, it samples over the entire space of functions without extra computation per viewpoint [S1, unofficial].
*   **When to choose:** When the scalar field distribution is important and uniform coverage of value ranges is desired [S1, unofficial].

### Area Sampling
A strategy that selects internal faces based on their geometric area in $\mathbb{R}^3$. It prioritizes larger triangles to minimize total error magnitude, which is proportional to triangle area [S1, unofficial].
*   **Formula:** Priority is calculated as $A_i \times \xi_i$, where $A_i$ is the area of the $i$-th face and $\xi_i$ is a uniform random number [S1, unofficial].
*   **Characteristics:** View-independent and avoids flickering. Randomizes selection to preserve the possibility of choosing small faces. Lowers visual error but may reduce the number of rasterized triangles due to larger fill requirements for large triangles [S1, unofficial].
*   **When to choose:** When minimizing geometric error is the primary goal and fill rate is not the bottleneck [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The adaptive selection step is $O(1)$ after preprocessing. Preprocessing (e.g., layer extraction for Topology Sampling) is $O(N)$ where $N$ is the number of tetrahedra [S1, unofficial].
*   **Memory Complexity:** Requires storage for the mesh, boundary faces, and internal faces partitioned by the chosen strategy (e.g., layers or histogram bins) [S1, unofficial].
*   **Performance:** In tests with the Fighter dataset (1.4 million tetrahedra), full quality rendered at 1.3 fps. At 15% LOD, it achieved 4.5 fps, and at 5% LOD, 10.0 fps using Area Sampling [S1, unofficial].
*   **Strengths:** Provides natural traversal of LODs; view-independent variants (Topology, Field, Area) avoid flickering; adaptive mechanism maintains target frame rates [S1, unofficial].
*   **Limitations:** Field Sampling has intensity discontinuities; Topology Sampling has correlated regions; Area Sampling may suffer from fill-rate limitations due to large triangles; View-Aligned Sampling suffers from flickering [S1, unofficial].
*   **When to use:** Interactive volume rendering of tetrahedral meshes where frame rate stability is critical [S1, unofficial].
*   **When not to use:** When visual continuity is paramount and Topology Sampling's correlated regions are unacceptable, or when fill rate is the primary bottleneck and large triangles are prevalent [S1, unofficial].

## Background
Face Subsampling was introduced by Steve Callahan in his 2006 Master's thesis, "The K-Buffer and Its Applications to Volume Rendering" [S1, unofficial]. The method builds on the Hardware-Assisted Visibility Sorting (HAVS) volume rendering system [S1, unofficial]. It is part of a family of multiresolution techniques that operate on tetrahedral meshes, offering an alternative to grid-based or wavelet-based LOD methods [S1, unofficial].

## Sources
[S1] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
