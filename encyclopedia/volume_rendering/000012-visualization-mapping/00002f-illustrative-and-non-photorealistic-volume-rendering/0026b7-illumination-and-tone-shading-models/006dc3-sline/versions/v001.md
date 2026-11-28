# Sline

Also called: Seamless Line Illustration for Interactive Biomedical Visualization

## Overview
Sline is a surface-based illustrative rendering method that provides a smooth, interactive transition between abstract silhouettes and detailed Phong shading by integrating silhouettes, contours, suggestive contours, and hatching [S1]. It distinguishes itself from sibling methods by achieving these stylistic transitions without relying on pre-existing texture coordinates or assumptions about the underlying mesh topology, making it particularly suitable for medical surface data derived from volumes [S1]. Unlike pure volume rendering methods, Sline operates on explicit surface meshes, using local geometric properties to generate line-based features that enhance shape perception [S1].

## How it works
Sline processes a scene containing one or more meshes $M$. For each mesh, a rendering style is selected from a hierarchy of abstraction levels: (1) Silhouettes and Contours, (2) Suggestive Contours, (3) Hatching, and (4) Phong Shading [S1]. The method relies on preprocessing geometric properties and then applying specific algorithms for each style level.

### Inputs and Outputs
**Inputs:**
*   Mesh $M$ with vertices and faces.
*   View vector $v$ (pointing from camera to mesh).
*   Light gradient vector field $k$.
*   Feature size $F$ (bounding sphere radius of $M$).
*   Style level parameter (1-4).

**Outputs:**
*   Final rendered image with illustrative line styles or Phong shading.

### Data Structures
*   **Mesh $M$**: Standard triangle mesh representation.
*   **Light Gradient Field $k$**: A vector field defined on the mesh surface, computed as described in [S1] (based on Lawonn et al. [LKEP14]).
*   **Radial Surface Curvature $\kappa_w(p)$**: Scalar field on $M$, where $w$ is the projection of view vector $v$ onto the tangent plane at $p$ [S1].
*   **Directional Derivative $D_w \kappa_w$**: Derivative of $\kappa_w$ in direction $w$ [S1].
*   **Preparation Texture $TEX_p$**: Screen-space texture storing fragment 3D locations and normals for hatching seed computation [S1].
*   **Stencil Buffer**: Used to isolate silhouette contours from inner contours [S1].

### Preprocessing
1.  Compute the light gradient vector field $k$ on $M$ [S1].
2.  Compute radial surface curvature $\kappa_w(p)$ for all points $p \in M$ [S1].
3.  Compute the directional derivative $D_w \kappa_w$ using the method by Judd et al. [JDA07] [S1].
4.  Define feature size $F$ as the radius of the bounding sphere of $M$ [S1].

### Step 1: Silhouettes and Contours
This step identifies object boundaries and internal feature lines.

1.  **Silhouette/Contour Detection**: Iterate over all triangle edges. For each edge, interpolate the normal $\hat{n}$ along the edge. Find zero crossings where $\langle \hat{n}, v \rangle = 0$ [S1].
2.  **Contour Patch Generation**: If zero crossings are found on two edges of a triangle, create a contour patch with four vertices. The vertices are defined as:
    $$ \hat{p}_1 = \hat{p}_0 + a \cdot \hat{n} \cdot F $$
    where $\hat{p}_0$ is the zero crossing location, $\hat{n}$ is the interpolated normal at that location, $a$ is a parameter controlling contour width, and $F$ is the feature size [S1].
3.  **Silhouette-Only Mode (Optional)**: If only silhouettes are required:
    *   Render mesh $M$ into the stencil buffer.
    *   Render the generated contour patches with the stencil test enabled. This ensures contours are drawn only where $M$ has not been drawn, effectively discarding inner contour patches [S1].

### Step 2: Suggestive Contours
Suggestive contours extend actual contours to support shape perception by identifying contours that would appear as silhouettes from nearby viewpoints [S1].

1.  **Zero Crossing Search**: For each triangle edge, search for a point $\hat{p}$ where $\kappa_w(\hat{p}) = 0$ [S1].
2.  **Condition Checks**: For each zero crossing $\hat{p}$, verify two conditions:
    *   **Condition 2**: $0 < t_d < \frac{D_w \kappa_w(\hat{p})}{\|w\|}$. Here, $t_d$ is a threshold parameter ensuring $D_w \kappa_w$ is positive and suppressing noise-induced sign changes [S1].
    *   **Condition 3**: $0 < \theta_c < \arccos\left( \frac{-\langle n(\hat{p}), v(\hat{p}) \rangle}{\|v(\hat{p})\|} \right)$. This discards contours where the angle between view vector $v$ and normal $n$ is too small. $\theta_c$ is a threshold parameter [S1].
3.  **Patch Generation**: If two or more edges of a triangle meet the conditions, generate suggestive contour patches. These patches are oriented orthogonally to the corresponding triangle surface [S1].

### Step 3: Hatching
Hatching generates thick, non-merging lines using "smart seeds" mapped to the mesh without texture coordinates [S1].

1.  **Seed Mapping (Polycube Projection)**:
    *   Define a 3D grid in local coordinates of $M$.
    *   For each surface point $p$ with normal $p_n$, project $p$ onto the grid.
    *   The point is projected onto the grid face $f$ with normal $f_n$ if $\langle p_n, f_n \rangle$ is the maximum dot product among all face normals of the grid cell [S1].
    *   This projection minimizes distortion and allows points to project outside their original cell [S1].
2.  **Smart Seed Definition**:
    *   Seeds consist of three concentric regions based on distance from the cell center $c$:
        *   **Core**: Radius $r_c = \frac{1}{2} s_r$. Always black (actual stroke).
        *   **Halo**: Radius $r_h = s_r$. Adjustable brightness for contrast.
        *   **Contact**: Radius $r_{cont} = \beta s_r$. Drawn lighter to prevent merging.
    *   $s_r = \frac{1}{4} c_s$, where $c_s$ is the cell size.
    *   $\beta = 1.8$ is the contact region size factor [S1].
    *   Cell size $c_s$ is chosen dependent on feature size $F$ and camera distance [S1].
3.  **Seed Trimming**:
    *   Render mesh $M$ to preparation texture $TEX_p$ to get screen-space fragment location $p$ and normal.
    *   Project $p$ to grid point $p'$ based on normal.
    *   Project light gradient $k(p)$ to grid cell face to get normalized vector $k'(p)$ [S1].
    *   Trim the seed in the direction of $k'(p)$ to ensure accessibility from outside. Compute:
        $$ q_1 = c + s_r \cdot \frac{k'(p)}{4} $$
        $$ q_2 = c - s_r \cdot \frac{k'(p)}{4} $$
        $$ d_1 = \langle k'(p), (q_1 - p') \rangle $$
        $$ d_2 = \langle k'(p), (q_2 - p') \rangle $$
    *   If $d_1$ and $d_2$ have the same sign, the fragment is invalid (discarded) [S1].
4.  **Perspective Compensation**:
    *   For faces turning away from the viewer, increase the seed radius:
        $$ s_{rm} = s_r (2 - |\langle v, n \rangle|) $$
    *   This compensates for perspective distortion and prevents rasterizer discard of small, clinched seeds [S1].
5.  **Line Search**:
    *   Perform a line search from the seed regions to generate thick lines that do not merge. The core produces the black stroke, the halo adjusts contrast, and the contact region ensures strokes remain distinct [S1].

### Step 4: Phong Shading
Standard Phong illumination model is applied if the highest level of abstraction is selected [S1].

### Parameters
*   $a$: Contour width factor [S1].
*   $t_d$: Threshold for suggestive contour condition 2 [S1].
*   $\theta_c$: Threshold for suggestive contour condition 3 [S1].
*   $\beta$: Contact region size factor, typically $1.8$ [S1].
*   $c_s$: Cell size for hatching grid, dependent on $F$ and camera distance [S1].
*   $F$: Feature size (bounding sphere radius) [S1].

### Edge Cases and Pitfalls
*   **Noise**: Frequent sign changes in $D_w \kappa_w$ due to noise can generate spurious suggestive contours. Condition 2 with $t_d$ suppresses these [S1].
*   **Merging Strokes**: Without the contact region and trimming, hatching strokes may merge into broad, indistinct bands. The smart seed design and line search address this [S1].
*   **Perspective Distortion**: Small seeds on faces turning away from the viewer may be discarded by the rasterizer. The radius modification $s_{rm}$ mitigates this [S1].
*   **Lack of Texture Coordinates**: Sline is designed to work without texture coordinates, which are often absent in medical surface data [S1].

## Operating characteristics
*   **Time Complexity**: Depends on mesh resolution and grid density for hatching. Preprocessing requires curvature and derivative computations per vertex/face [S1].
*   **Memory Complexity**: Requires storage for mesh data, curvature fields, and preparation texture $TEX_p$ [S1].
*   **Strengths**: Seamless transition between abstraction levels; no reliance on texture coordinates; interactive performance [S1].
*   **Limitations**: Surface-based only (not direct volume rendering); requires preprocessing of curvature fields; parameter tuning for $t_d$, $\theta_c$, and $c_s$ [S1].
*   **When to Use**: Interactive biomedical visualization where abstract to detailed transitions are needed, especially for meshes without texture coordinates [S1].
*   **When Not to Use**: Direct volume rendering without surface extraction; scenes requiring texture-based hatching [S1].

## Background
Sline was introduced in the paper "Sline: Seamless Line Illustration for Interactive Biomedical Visualization" [S1]. It builds upon previous work in illustrative rendering, such as suggestive contours by De Carlo et al. [DFRS03] and hatching techniques by Lawonn et al. [LKEP14]. The method was developed to address the need for interactive, stylized visualization of medical data, where texture coordinates are often unavailable [S1].

## Sources
[S1] Sline: Seamless Line Illustration for Interactive Biomedical Visualization, https://doi.org/10.2312/vcbm.20161281, fetched 2026-10-29
