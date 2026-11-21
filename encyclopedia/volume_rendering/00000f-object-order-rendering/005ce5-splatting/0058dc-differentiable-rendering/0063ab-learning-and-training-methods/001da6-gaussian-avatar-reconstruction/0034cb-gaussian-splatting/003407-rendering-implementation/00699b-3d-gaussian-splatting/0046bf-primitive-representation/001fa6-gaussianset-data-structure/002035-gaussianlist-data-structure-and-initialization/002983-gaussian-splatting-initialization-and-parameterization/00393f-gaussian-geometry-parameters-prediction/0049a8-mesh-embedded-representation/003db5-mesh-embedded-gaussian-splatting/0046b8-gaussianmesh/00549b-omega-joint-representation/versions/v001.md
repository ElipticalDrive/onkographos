# OMeGa Joint Representation

Also called: Joint Optimization of Explicit Meshes and Gaussian Splats, OMeGa

## Overview
OMeGa is a joint representation framework that optimizes an explicit triangle mesh and 2D Gaussian splats simultaneously, decoupling geometric support from textural detail [S1]. Unlike methods that allocate a fixed number of splats uniformly or keep rotation/scale fixed, OMeGa binds splats to mesh faces via barycentric coordinates, allowing adaptive density control and free movement of splats within the face plane [S1]. This approach addresses inaccurate geometry in texture-less regions and leverages mesh geometry to guide splat optimization [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** Multi-view images, initial mesh topology (vertices and faces).
- **Outputs:** Optimized learnable vertex positions $V$, mesh faces $F$, Gaussian splat set $G$, and binding map $B$.

**Data Structures:**
1.  **Mesh Parameters:**
    -   $V \in \mathbb{R}^{N_v \times 3}$: Learnable vertex positions, where $N_v$ is the number of vertices.
    -   $F \in \mathbb{R}^{N_f \times 3}$: Mesh faces, where $N_f$ is the number of faces. Each face $f_j$ is defined by three vertices.
2.  **Gaussian Splat Set $G$:**
    -   Each splat $g_i$ contains texture attributes (color, opacity) and spatial parameters derived from the mesh.
    -   Spatial parameters include barycentric coordinates $(u, w)$, relative scale $s_{rel}$, and in-plane rotation $R_{2d}$.
3.  **Binding Map $B$:**
    -   $B: G \rightarrow F$ maps each splat $g_i$ to its supporting face $f_j$.
    -   Stored as a list or hash map of pairs $(g_i, f_j)$.

**Formulas:**
1.  **Splat Mean (Position):**
    The mean $\mathbf{m}$ of a splat bound to face $f_j$ with vertices $\mathbf{v}_1, \mathbf{v}_2, \mathbf{v}_3$ is:
    $$ \mathbf{m} = u \cdot \mathbf{v}_1 + w \cdot \mathbf{v}_2 + (1 - u - w) \cdot \mathbf{v}_3 $$
    where $u, w \in [0, 1]$ and $u + w \le 1$ [S1].

2.  **Splat Scale:**
    The scale $\mathbf{s}$ is the element-wise product of a learnable relative scale $\mathbf{s}_{rel}$ and a predefined base scale $\mathbf{s}_0$ derived from the mesh face:
    $$ \mathbf{s} = \mathbf{s}_{rel} \odot \mathbf{s}_0 $$
    where $\mathbf{s}_{rel} = (s_{rel,1}, s_{rel,2}) \in [s_{min}, s_{max}] \times [s_{min}, s_{max}]$ [S1].

3.  **Splat Rotation:**
    The final rotation matrix $\mathbf{R}$ is computed from the face rotation $\mathbf{R}_s$ and an in-plane rotation parameter $\mathbf{R}_{2d}$:
    $$ \mathbf{R} = \mathbf{R}_s \cdot \mathbf{R}_{2d} $$
    where $\mathbf{R}_s$ is the rotation matrix of the triangle face [S1].

**Numbered Steps:**
1.  **Initialization:** Initialize mesh vertices $V$ and faces $F$. Initialize Gaussian splats $G$ with initial barycentric coordinates $(u, w)$, relative scales $\mathbf{s}_{rel}$, and in-plane rotations $\mathbf{R}_{2d}$. Initialize binding map $B$ mapping each splat to a face.
2.  **Spatial Attribute Derivation:** For each splat $g_i$ bound to face $f_j$:
    -   Compute mean $\mathbf{m}$ using the barycentric formula [S1].
    -   Compute scale $\mathbf{s}$ using the relative scale formula [S1].
    -   Compute rotation $\mathbf{R}$ using the face rotation and in-plane rotation [S1].
3.  **Rasterization:** Render the scene using the derived Gaussian splats (means, scales, rotations, colors, opacities) [S1].
4.  **Loss Computation:** Compute photometric loss from rasterized images and mesh regularization losses (mesh constraint, monocular normal supervision) [S1].
5.  **Optimization:** Update learnable parameters ($V$, splat attributes) via gradient descent [S1].
6.  **Adaptive Density Control (Splat Management):**
    -   **Cloning/Splitting:** When a Gaussian $g_i$ bound to $f_j$ is cloned or split, create $g_{new}$. Bind $g_{new}$ to $f_j$. Inherit $g_i$'s barycentric position with a small in-plane offset onto the face plane $\Pi_{f_j}$. Insert $(g_{new}, f_j)$ into $B$ [S1].
    -   **Removal:** If a splat is pruned by the adaptive density controller, remove its entry from $B$ [S1].
7.  **Mesh Refinement (Topology Management):**
    -   **Subdivision:** When face $f_j$ is subdivided into children $\{f_{jk}\}$, reassign each splat previously bound to $f_j$ to the child whose domain contains its barycentric coordinates. Preserve per-splat attributes [S1].
    -   **Removal:** When face $f_j$ is removed, identify all splats mapped to $f_j$ and flag them for removal in subsequent steps [S1].

**Parameters:**
-   $s_{min}, s_{max}$: Lower and upper bounds for Gaussian Splat scale components to prevent discrepancies with triangular face scales [S1].
-   Typical values for $s_{min}, s_{max}$: Not given in the sources.

**Edge Cases and Pitfalls:**
-   **Scale Discrepancy:** Significant differences between Gaussian Splat scales and triangular face scales can occur. This is prevented by imposing upper and lower bounds on $\mathbf{s}_{rel}$ [S1].
-   **Barycentric Validity:** Ensure $u, w \in [0, 1]$ and $u + w \le 1$ to keep splats within the face boundaries [S1].
-   **Topology Changes:** When mesh topology changes (subdivision/removal), the binding map $B$ must be updated consistently to maintain valid splat-face associations [S1].

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** Not given in the sources.
-   **Measured Performance:** OMeGa achieves state-of-the-art performance on challenging indoor scenes [S1].
-   **Strengths:** Decouples geometric support from textural detail; allows adaptive density control; enables free movement of splats within the face plane; regularizes geometry learning via mesh constraints and monocular normal supervision [S1].
-   **Limitations:** Not given in the sources.
-   **When to use:** When accurate geometry in texture-less regions is required, and when joint optimization of mesh and splats is beneficial for scene-level surface reconstruction [S1].
-   **When not to use:** Not given in the sources.

## Background
OMeGa was introduced in the paper "OMeGa: Joint Optimization of Explicit Meshes and Gaussian Splats for Robust Scene-Level Surface Reconstruction" [S1]. It addresses limitations of previous works, such as inaccurate geometry in texture-less indoor regions and the decoupling of mesh extraction from optimization [S1]. The method integrates mesh constraints and monocular normal supervision to regularize geometry learning and proposes a heuristic, iterative mesh-refinement strategy [S1].

## Sources
[S1] OMeGa: Joint Optimization of Explicit Meshes and Gaussian Splats for Robust Scene-Level Surface Reconstruction, https://doi.org/10.1109/WACV61042.2026.00427, fetched 2026-10-30
