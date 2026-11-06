# Level-of-detail (LOD) methods

Also called: Adaptive resolution rendering, Dynamic LOD, Dynamic Level-of-Detail (LOD) Framework, Sample-Based LOD

## Overview
Level-of-detail (LOD) methods change the resolution of the rendering during viewpoint interaction to maintain responsive frame rates [S2, book]. They solve the problem of visualizing large scalar fields or unstructured grids by generating reduced-resolution approximations of the original data, thereby minimizing memory bandwidth and sample counts [S1, unofficial][S2, book]. Unlike [Dynamic Volume Resolution & Alpha Mask Update](2539-dynamic-volume-resolution-alpha-mask-update.md), which often adjusts the grid structure itself, LOD methods typically maintain the original geometry or data structure and vary the sampling density or number of rendered primitives [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Original scalar field $f$ defined on domain $D$ (e.g., tetrahedral mesh or voxel grid), current render time, target frame time.
*   **Output:** Reduced-resolution approximation $\bar{f}$ on domain $\bar{D} \subseteq D$, or a subset of rendering primitives (faces, slices, rays).

**Data Structures:**
*   **Original Domain $D$:** The full-resolution representation (e.g., tetrahedral mesh with boundary faces $B$ and internal faces $I$, or a voxel grid).
*   **Approximation Domain $\bar{D}$:** A subset of $D$ selected for rendering. For mesh-based LOD, $\bar{D}$ is represented by $B \cup I_{selected}$, where $I_{selected} \subset I$ [S1, unofficial].

**Steps:**
1.  **Metric Evaluation:** Measure the render time of the previous frame, denoted as $\text{RenderTime}$ [S1, unofficial].
2.  **LOD Selection (Time-Driven):** Calculate the number of internal elements (e.g., faces $|I|$) to include in the current frame to meet the $\text{TargetTime}$ [S1, unofficial]. The formula is:
    $$
    |I_{selected}| = \frac{|I| \times \text{TargetTime}}{\text{RenderTime}}
    $$
    where $|I|$ is the total number of internal faces in the original mesh, and $\text{TargetTime}$ is the desired frame time (typically 0.01 seconds for interactive viewing) [S1, unofficial].
3.  **Approximation Generation:**
    *   **Sample-Based Simplification:** Use the original geometry to represent $D$, but compute an approximate volume integral over the original geometry using coarser sampling [S1, unofficial]. This involves:
        *   For ray-casting: Using larger step sizes or casting fewer rays [S2, book].
        *   For texture slicing: Using fewer slices through the data (e.g., every fourth slice) [S2, book].
    *   **Domain-Based Simplification:** Approximate the domain geometry first (e.g., by collapsing edges in a tetrahedral mesh) and then compute an exact volume integral over the approximate geometry [S1, unofficial].
4.  **Rendering:** Render the selected approximation $\bar{f}$ or subset of primitives.
5.  **State Update:** If the view changes significantly or performance deviates, repeat steps 1-3. The system allows an easy return to full quality by selecting all internal faces ($I_{selected} = I$) [S1, unofficial].

**Parameters:**
*   **TargetTime:** The amount of time desired for each frame for interactive viewing. Typical value: 0.01 seconds [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Visual Error:** Sample-based simplification provides different, though not necessarily better, results than domain-based simplification [S1, unofficial].
*   **Smoothness:** To maximize visual smoothness when the viewpoint or transfer function changes, face selection should be based only on mesh geometry and scalar values, not on the transfer function or viewpoint [S1, unofficial].
*   **Precision:** The formula assumes linear relationship between complexity and time, which may not hold for all hardware or scene complexities [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lod_render_step(mesh, target_time=0.01):
    """
    Render a frame using LOD based on previous frame's render time.
    
    Args:
        mesh: Object containing 'boundary_faces' (B), 'internal_faces' (I), 
              and 'render()' method.
        target_time: Desired frame time in seconds.
        
    Returns:
        None
    """
    # Step 1: Get previous render time
    # [G] In a real system, this would be retrieved from the previous frame's execution log.
    # If this is the first frame, use a default estimate.
    render_time = get_previous_render_time()
    if render_time is None:
        render_time = target_time  # Default to target time if no history

    # Step 2: Calculate number of internal faces to select
    total_internal_faces = len(mesh.internal_faces)
    
    # Avoid division by zero and ensure at least 1 face is selected if possible
    if render_time == 0:
        render_time = 1e-9
        
    # Formula: |I_selected| = (|I| * TargetTime) / RenderTime
    # [G] Clamp the result to the valid range [0, |I|]
    selected_count = int((total_internal_faces * target_time) / render_time)
    selected_count = max(0, min(selected_count, total_internal_faces))

    # Step 3: Select faces
    # [G] In a real system, faces would be selected based on error criteria 
    # (e.g., edge collapse error) to minimize visual loss. 
    # Here we assume a simple selection for demonstration.
    selected_internal_faces = mesh.internal_faces[:selected_count]
    
    # Step 4: Construct approximation domain
    # The approximation domain D_bar is represented by B U I_selected
    current_faces = mesh.boundary_faces + selected_internal_faces

    # Step 5: Render
    # [G] The render function would compute the volume integral over the selected faces.
    # For sample-based simplification, this might involve coarser sampling.
    render(current_faces)
    
    # Step 6: Store render time for next frame
    # [G] This would be handled by the rendering engine's timing loop.
    store_render_time(get_current_frame_time())
```

## Subtopics

### [Frustum-based LOD Selection](2986-frustum-based-lod-selection.md)
Determines resolution based on distance from camera to entity relative to frustum faces. Chooses lower resolutions for distant/peripheral rays.

### [Footprint-based LOD Selection](3088-footprint-based-lod-selection.md)
Ensures projected voxel size does not exceed a pixel threshold to prevent aliasing.

## Operating characteristics
*   **Time Complexity:** Depends on the number of selected primitives. For sample-based LOD, it is proportional to the number of rays or slices used [S2, book]. For domain-based LOD, it depends on the number of simplices in the simplified mesh [S1, unofficial].
*   **Memory Complexity:** Sample-based LOD requires storing the full original data structure but only accesses a subset of samples [S1, unofficial]. Domain-based LOD may require storing simplified mesh structures [S1, unofficial].
*   **Strengths:** Maintains interactive frame rates for large datasets [S2, book]. Simple to implement for sample-based approaches [S1, unofficial].
*   **Limitations:** May introduce visual artifacts if simplification is too aggressive [S1, unofficial]. Performance may not scale linearly with complexity [S1, unofficial].
*   **When to Use:** Interactive visualization of large scalar fields or unstructured grids [S1, unofficial].
*   **When Not to Use:** When high visual fidelity is required regardless of frame rate, or when the scene complexity does not vary significantly with viewpoint [S2, book].

## Background
LOD methods originated in the context of interactive visualization of large scalar fields represented as tetrahedral meshes [S1, unofficial]. The concept of using coarser sampling for ray-casting was introduced by Levoy [S2, book]. Hierarchical representations like octrees were proposed by Danskin and Hanrahan [S2, book]. Advances in GPU hardware have further improved the efficiency of LOD techniques, particularly for ray-casting and texture-based approaches [S2, book].

## Sources
[S1] THE K-BUFFER AND ITS APPLICATIONS TO VOLUME RENDERING, https://ctsilva.github.io/thesis/steve-callahan-ms-thesis.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
