# 3D Gaussian Ray Tracing

Also called: 3DGRT, BVH Ray Tracing for 3D Gaussians, Point-based Ray Tracing on 3D Gaussians, Ray-Splat Intersection via BVH, Splat BVH Traversal

## Overview
3D Gaussian Ray Tracing (3DGRT) replaces the rasterization-based splatting of standard 3D Gaussian Splatting with hardware-accelerated ray tracing to correctly handle semi-transparent particles and complex lighting [S1, unofficial][S4, unofficial]. It solves the linear scaling and incorrect alpha blending of tile-based rasterization in dense scenes by using a Bounding Volume Hierarchy (BVH) to evaluate only Gaussians intersected by each ray, enabling logarithmic complexity in visibility queries [S1, unofficial][S4, unofficial]. Unlike rasterization, which processes tiles and evaluates all overlapping Gaussians per pixel, 3DGRT uses proxy triangle meshes or tight bounding boxes to perform exact ray-splat intersection tests, enabling correct front-to-back transmittance accumulation and shadow casting [S2][S4, unofficial].

## How it works
**Inputs:**
*   Ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin and $\mathbf{d}$ is the direction [G].
*   Set of $N$ 3D Gaussians, each defined by mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, and color $\mathbf{c}$ [G].
*   BVH structure built from the Gaussians [S2].

**Outputs:**
*   Pixel color $\mathbf{C}$ and transmittance $T$ [G].

**Data Structures:**
The BVH is stored in textures for hardware acceleration (e.g., OptiX) [S3, unofficial].
1.  **AABB Texture (`tAABBTexture`):**
    *   Size: $4096 \times 4096$ [S3, unofficial].
    *   Layout: Each node is one `uvec4`.
        *   `x`: Node ID (negative for inner nodes, positive for leaf splat IDs) [S3, unofficial].
        *   `y`: Min/Max X of AABB (packed via `unpackHalf2x16`) [S3, unofficial].
        *   `z`: Min/Max Y of AABB [S3, unofficial].
        *   `w`: Min/Max Z of AABB [S3, unofficial].
2.  **Triangle Texture (`tTriangleTexture`):**
    *   Size: $2048 \times 2048$ [S3, unofficial].
    *   Layout: Each splat occupies two texels (`id*2` and `id*2+1`) [S3, unofficial].
        *   `vd0.x, y, z`: Position $\boldsymbol{\mu}$ (stored as `uintBitsToFloat`) [S3, unofficial].
        *   `vd0.w, vd1.x`: First axis vector $\mathbf{u}$ components (packed via `unpackHalf2x16`) [S3, unofficial].
        *   `vd1.y`: Second component of $\mathbf{u}$ [S3, unofficial].
        *   `vd1.z, w`: RGB color and Alpha (z=RGB, w=Alpha) [S3, unofficial].

**Algorithm Steps:**

1.  **BVH Construction:**
    *   Construct a binary radix tree from the 3D Gaussians [S2].
    *   Leaf nodes represent tight bounding boxes (or proxy triangle meshes) of individual Gaussians [S2][S4, unofficial].
    *   Internal nodes denote the bounding box of their two children [S2].
    *   *Note:* For proxy meshes, each Gaussian is bound with a triangle mesh [S4, unofficial].

2.  **Ray Traversal Mode Selection:**
    *   **Closest-Hit Mode (`USE_ANYHIT=0`):** Sort children by $t_0$ (near) to find the nearest splat first [S3, unofficial].
    *   **Any-Hit Mode (`USE_ANYHIT=1`):** Sort children by $t_1$ (far) to collect all intersected splats [S3, unofficial].

3.  **Traversal Loop:**
    *   Start from the root node [S2].
    *   Perform intersection tests between the ray and the bounding volumes of children [S2].
    *   **AABB Intersection Test:**
        *   Compute near/far $t$ values for x, y, z axes [S3, unofficial].
        *   $t_0 = \max(\max(t_{min,x}, t_{min,y}), t_{min,z})$ [S3, unofficial].
        *   $t_1 = \min(\min(t_{max,x}, t_{max,y}), t_{max,z})$ [S3, unofficial].
        *   Valid if $\max(t_0, 0.0) \le t_1$ [S3, unofficial].
    *   If the node is an inner node (ID < 0), push children to stack and continue [S3, unofficial].
    *   If the node is a leaf (ID $\ge$ 0), process the splat [S3, unofficial].

4.  **Leaf Processing (Splat Intersection):**
    *   Fetch triangle texture for ID [S3, unofficial].
    *   Decode position $\mathbf{p}$, axes $\mathbf{u}, \mathbf{v}$, and color/alpha [S3, unofficial].
    *   Solve for intersection parameters $u, v, t$ in the local space:
        $$ [\mathbf{u} \ \mathbf{v} \ \mathbf{d}] \begin{bmatrix} u \\ v \\ -t \end{bmatrix} = \mathbf{o} - \mathbf{p} $$
        *   $\mathbf{uvt} = \text{solve}(\dots)$
        *   $uv = \mathbf{uvt}.xy$
        *   $t = -\mathbf{uvt}.z$ [S3, unofficial].
    *   **Hit Condition:** $t > 0$ and $\text{dot}(uv, uv) < 1.0$ (inside unit circle in local space) [S3, unofficial].

5.  **Accumulation:**
    *   **Closest-Hit:**
        *   If hit distance $d < t_{current}$, update $t$, `splatID`, and `splatRGBA` [S3, unofficial].
        *   Loop up to 8 bounces [S3, unofficial].
        *   Accumulate color: $\mathbf{C}_{cum} += T \cdot \alpha_{hit} \cdot \mathbf{c}_{hit}$ [S3, unofficial].
        *   Update transmittance: $T *= 1.0 - \alpha_{hit}$ [S3, unofficial].
        *   Shift ray origin: $\mathbf{o} += t + 10^{-6}$ [S3, unofficial].
        *   Break if $T < 0.05$ [S3, unofficial].
    *   **Any-Hit:**
        *   Do not update $t$. Insert hit into sorted array `intersects[N_INT]` using insertion sort by depth $d$ [S3, unofficial].
        *   Store RGBA [S3, unofficial].
        *   If array full (`intersectPtr >= N_INT`), return early [S3, unofficial].
        *   After traversal, sort collected intersections by depth [S3, unofficial].
        *   Loop through sorted hits:
            *   $\mathbf{C}_{cum} += T \cdot \alpha \cdot \mathbf{c}$ [S3, unofficial].
            *   $T *= 1.0 - \alpha$ [S3, unofficial].

6.  **Equivalent Intersection Point (Alternative Formulation):**
    *   Some implementations approximate the intersection as the point where the Gaussian's contribution peaks [S2].
    *   $t_j = \frac{(\boldsymbol{\mu} - \mathbf{r_o})^T \boldsymbol{\Sigma} \mathbf{r_d}}{\mathbf{r_d}^T \boldsymbol{\Sigma} \mathbf{r_d}}$ [S2].
    *   $\mathbf{r}_x = \mathbf{r_o} + t_j \mathbf{r_d}$ [S2].
    *   Contribution is approximated as $\alpha_j$ at $\mathbf{r}_x$ [S2].
    *   Transmittance is attenuated: $T_i = (1 - \alpha_{i-1})T_{i-1}$ for $i=1, \dots, j-1$ with $T_1=1$ [S2].
    *   The order of encounter does not affect overall transmittance [S2].

7.  **Early Termination:**
    *   If transmittance $T$ drops below threshold $T_{min}$, terminate ray tracing [S2].

**Parameters:**
*   $N_{INT}$: Maximum number of splat intersections stored in any-hit mode. Value: 32 [S3, unofficial].
*   $T_{min}$: Threshold for early termination. Value: Not given in the sources.
*   $INFINITY$: Large float value for no-hit. Value: 1048576.0 [S3, unofficial].
*   Batch Size: Fixed-size groups of $N$ Gaussians to avoid register spilling. Value: Not given in the sources [S5].

**Edge Cases and Pitfalls:**
*   **Precision:** BVH traversal uses float for stack pointers and IDs. Inner nodes have negative IDs, leaves have positive. Check `id < 0.0` distinguishes them [S3, unofficial].
*   **Pruning:** Floating point comparison `currentStackData.t0 >= t` prunes branches [S3, unofficial].
*   **Complexity:** Online visibility querying is computationally expensive. Handled by pre-computing visibility for static scenes in a separate stage [S2].

## Subtopics

### [GRay](10042-gray.md)
A fast ray-tracing method for 3D Gaussian Splatting (3DGS) designed to close the performance gap between ray tracing and rasterization by leveraging the algorithmic advantage of ray tracing: it evaluates only intersected Gaussians, allowing logarithmic scaling in dense scenes with tiny Gaussians. Unlike 3DGS rasterization which operates on tiles and evaluates all Gaussians overlapping a tile for every pixel (linear scaling), GRay uses a Bounding Volume Hierarchy (BVH) to select and evaluate exactly the set of Gaussians intersecting each pixel. It builds on [3D Gaussian Ray Tracing](8101-3d-gaussian-ray-tracing.md), [EPBRR](8102-epbrr.md), and [EDGS](8103-edgs.md).

## Operating characteristics
*   **Time Complexity:** Logarithmic scaling in dense scenes with tiny Gaussians due to BVH traversal, compared to linear scaling in rasterization [S1, unofficial]. However, on tested hardware, 3DGRT optimizes about 9× slower than a recent version of 3DGS [S5].
*   **Memory Complexity:** Depends on BVH size. Textures are $4096 \times 4096$ for AABBs and $2048 \times 2048$ for triangles [S3, unofficial].
*   **Strengths:** Correct handling of semi-transparent particles, shadows, and complex lighting; efficient culling of non-intersected Gaussians [S1, unofficial][S2][S4, unofficial].
*   **Limitations:** Slower than rasterization on current hardware; computational complexity of online visibility querying [S2][S4, unofficial].
*   **When to use:** Scenes requiring physically accurate transparency, shadows, or relighting [S2].
*   **When not to use:** Real-time applications where rasterization speed is critical and visual fidelity of transparency is less important [S4, unofficial].

## Background
3D Gaussian Ray Tracing was unveiled by NVIDIA Research in July 2024 [S4, unofficial]. It was first studied by Moenne-Loccoz et al. [2024] as an alternative for rendering 3D Gaussians using hardware-accelerated ray tracing with OptiX [S5]. The method addresses the limitations of rasterization-based splatting, such as incorrect alpha blending and lack of shadow casting, by treating Gaussians as semi-transparent particles [S1, unofficial][S4, unofficial]. The method builds on earlier work in point-based ray tracing and has been further improved by subsequent methods like EPBRR [S5].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing, https://doi.org/10.1007/978-3-031-72995-9_5, fetched 2026-10-26
[S3] notlasandu/spirulae-splat, https://github.com/notlasandu/spirulae-splat, fetched 2026-10-27 (unofficial)
[S4] What Is a Radiance Field? A Complete Guide, https://radiancefields.com/what-is-a-radiance-field, fetched 2026-10-29 (unofficial)
[S5] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
