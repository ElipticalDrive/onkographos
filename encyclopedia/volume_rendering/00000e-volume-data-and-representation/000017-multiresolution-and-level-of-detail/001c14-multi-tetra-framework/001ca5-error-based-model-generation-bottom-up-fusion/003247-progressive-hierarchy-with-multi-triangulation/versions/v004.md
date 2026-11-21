# Progressive Hierarchy with Multi-triangulation

Also called: Multi-triangulation LOD

## Overview
Progressive Hierarchy with Multi-triangulation is a dynamic Level-of-Detail (LOD) technique for volume rendering on unstructured grids that creates a progressive hierarchy of tetrahedra stored in a multi-triangulation data structure [S1, S2, S3, unofficial]. It solves the problem of interactive rendering performance by allowing the mesh to be dynamically updated and selectively refined to a target LOD using selection heuristics [S1, S2, S3, unofficial]. It differs from static LOD methods by encoding all edge-collapse operations required to reduce the mesh to a base state, enabling on-the-fly reconstruction [S1, S2, S3, unofficial].

## How it works
**Inputs:** An unstructured tetrahedral grid; a target Level-of-Detail (LOD) metric or threshold; selection heuristics for determining which regions require refinement.
**Outputs:** A dynamically updated multi-triangulation mesh representation suitable for rendering.

**Data Structures:**
The core data structure is the **multi-triangulation model** [S1, S2, S3, unofficial]. This structure encodes the sequence of edge-collapse operations necessary to reduce the full-resolution tetrahedral mesh to a base mesh (the coarsest level) [S1, S2, S3, unofficial].
*   **Base Mesh:** The coarsest tetrahedral mesh in the hierarchy.
*   **Edge-Collapse Operations:** A list or stack of operations, each specifying an edge to collapse and the resulting topological change, effectively storing the "history" of simplification from base to full resolution [S1, S2, S3, unofficial].
*   **Multi-triangulation Structure:** A data structure that supports the dynamic application of these edge-collapses in reverse (refinement) or forward (coarsening) order to reconstruct intermediate LOD levels [S1, S2, S3, unofficial]. Specific field layouts for the multi-triangulation structure are not detailed in the sources beyond its function of encoding edge-collapses [S1, S2, S3, unofficial].

**Steps:**
1.  **Preprocessing (Model Creation):**
    1.  Start with the full-resolution tetrahedral mesh.
    2.  Perform a bottom-up simplification process (as described in [Error-Based Model Generation (Bottom-Up Fusion)](1306-error-based-model-generation-bottom-up-fusion.md)) to reduce the mesh to a base mesh.
    3.  Record every edge-collapse operation performed during this reduction.
    4.  Store these operations in the multi-triangulation data structure [S1, S2, S3, unofficial].
2.  **Dynamic LOD Selection:**
    1.  Determine the desired LOD for the current frame based on hardware capabilities, camera distance, or other selection heuristics [S1, S2, S3, unofficial].
    2.  Identify which edge-collapse operations must be "undone" (i.e., which edges must be split) to reach the target LOD from the base mesh [S1, S2, S3, unofficial].
3.  **Mesh Reconstruction:**
    1.  Apply the necessary edge-split operations (reverse of edge-collapse) to the base mesh using the multi-triangulation structure [S1, S2, S3, unofficial].
    2.  Dynamically update the mesh to reflect the selected LOD [S1, S2, S3, unofficial].
4.  **Rendering:**
    1.  Render the dynamically updated multi-triangulation mesh [S1, S2, S3, unofficial].

**Parameters:**
*   **Selection Heuristics:** The specific criteria for choosing which edges to split or collapse are not defined in the sources [S1, S2, S3, unofficial]. [G] Typical heuristics might include error metrics (e.g., distance from camera, scalar value gradient) or frame-rate targets.
*   **Base Mesh Resolution:** The resolution of the initial coarse mesh. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Conformality:** The method relies on edge-collapse operations which generally preserve mesh conformality if applied correctly. However, dynamic updates must ensure that shared edges between adjacent tetrahedra are handled consistently to avoid non-manifold geometry or cracks [S1, S2, S3, unofficial]. [G] Standard multi-triangulation implementations handle this by tracking edge status globally.
*   **Precision:** Edge-collapse operations involve geometric calculations. Precision issues may arise if the collapse distance is very small relative to the mesh scale. Not given in the sources.
*   **Degenerate Input:** If the initial mesh contains degenerate tetrahedra (zero volume), the simplification process may fail or produce invalid base meshes. Not given in the sources.

## Subtopics

### Tetrahedral Hierarchy with Video Metaphor
A variant of the progressive hierarchy approach applied to time-varying volumetric data, proposed by Gregorski et al. [S4]. It represents the spatial component of the data as a hierarchy of tetrahedra, allowing level-of-detail control over the extracted mesh in each frame [S4]. To advance to the next frame, it refines or coarsens each node of the previous frame’s mesh [S4]. Unlike standard conformal refinement, it uses a lookup table to locally subdivide tetrahedra that do not align with their neighbors, rather than refining neighboring geometry [S4]. This approach improves run-time performance due to increased parallelizability but results in limited frame-to-frame coherence [S4].

## Operating characteristics
*   **Time Complexity:** Dynamic updates allow for interactive rendering, but the complexity of applying edge-split operations depends on the number of operations required to reach the target LOD [S1, S2, S3, unofficial]. The video metaphor variant claims improved run-time performance due to parallelizability [S4].
*   **Memory Complexity:** The multi-triangulation structure stores all edge-collapse operations, which may require significant memory for high-resolution meshes [S1, S2, S3, unofficial].
*   **Measured Performance:** Specific performance metrics (FPS, memory usage) are not provided in the sources [S1, S2, S3, unofficial][S4].
*   **Strengths:** Enables interactive rendering of unstructured grids by dynamically adjusting mesh complexity [S1, S2, S3, unofficial]. The video metaphor variant offers improved parallelizability [S4].
*   **Limitations:** The video metaphor variant has limited frame-to-frame coherence [S4]. The dynamic nature of the updates may introduce overhead compared to static LOD [S1, S2, S3, unofficial].
*   **When to Use:** Suitable for interactive visualization of unstructured grid data where frame rates are critical and the scene complexity varies [S1, S2, S3, unofficial]. The video metaphor variant is suitable for time-varying data where parallel processing is available [S4].
*   **When Not to Use:** Not suitable for offline rendering where maximum quality is prioritized over interactivity, or for data with very high temporal coherence where static LOD might suffice.

## Background
The technique was proposed by Cignoni et al. [S1, S2, S3, unofficial]. It is part of the broader effort to enable GPU-based volume rendering of unstructured grids, addressing the limitations of hardware capabilities at the time of publication [S1, S2, S3, unofficial]. The multi-triangulation data structure is based on prior work by Cignoni et al. [25] (as cited in [S1, S2, S3, unofficial]). The video metaphor variant was introduced by Gregorski et al. [S4] for time-varying data.

## Sources
[S1] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
[S3] A Survey of GPU-Based Volume Rendering of ..., https://www.sci.utah.edu/~csilva/papers/rita2005.pdf, fetched 2026-10-29 (unofficial)
[S4] Modeling and Visualization Approaches for Time-Varying Volumetric Data, https://doi.org/10.1007/978-3-540-89646-3_100, fetched 2026-10-30
