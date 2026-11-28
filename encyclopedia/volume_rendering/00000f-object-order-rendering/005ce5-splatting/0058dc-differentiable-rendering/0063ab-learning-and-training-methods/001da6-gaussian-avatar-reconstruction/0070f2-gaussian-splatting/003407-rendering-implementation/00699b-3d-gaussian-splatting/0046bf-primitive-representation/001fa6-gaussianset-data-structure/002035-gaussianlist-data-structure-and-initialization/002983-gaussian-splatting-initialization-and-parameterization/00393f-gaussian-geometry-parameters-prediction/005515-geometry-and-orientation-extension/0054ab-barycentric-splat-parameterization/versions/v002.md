# Barycentric Splat Parameterization

Also called: GeoSplatting Face-Sampling, Heuristic Gaussian sampling from triangles, Mesh-Gaussian binding via midpoints, Splat center derivation from mesh faces

## Overview
Barycentric Splat Parameterization constrains 2D Gaussian Splat centers to lie on triangular mesh faces using barycentric coordinates, allowing splats to move freely on the face while preserving per-splat attributes [S1]. It differs from standard 3D Gaussian Splatting by deriving geometry directly from mesh topology rather than learning unconstrained centers, and from [Oriented Gaussian Splatting](9621-oriented-gaussian-splatting.md) by explicitly binding splat scales and orientations to local mesh edge vectors and face area [S2].

## How it works
**Inputs:**
- Triangular mesh face $F_i$ with vertices $v_1, v_2, v_3$.
- Vertex attributes: Position $\mathbf{P}_i = (\mathbf{p}_{i1}, \mathbf{p}_{i2}, \mathbf{p}_{i3})$, Normal $\mathbf{N}_i = (\mathbf{n}_{i1}, \mathbf{n}_{i2}, \mathbf{n}_{i3})$.
- Barycentric coordinates $u, w$ for the center, and $u, v$ for midpoints.
- Hyperparameters: $u=0.07, v=0.22, \alpha_{jk}, \beta_{jk}, \delta_{jk}$ [S2].

**Outputs:**
- Gaussian parameters: center position $\mathbf{c}$, normal $\mathbf{n}$, scaling vector $\mathbf{s}$, rotation basis vectors $\mathbf{r}_x, \mathbf{r}_y, \mathbf{r}_z$.

**Data Structures:**
- Barycentric coordinates are scalars $u, w, v \in [0, 1]$ satisfying $u+w \le 1$ for centers, and specific values for midpoints.

**Steps:**

1. **Define Barycentric Interpolation:**
   For any attribute vector $\mathbf{A}_i = (\mathbf{a}_{i1}, \mathbf{a}_{i2}, \mathbf{a}_{i3})$ defined at triangle vertices and barycentric weights $(q_1, q_2, q_3)$, compute the interpolated value:
   $$ \mathbf{A}_i \odot (q_1, q_2, q_3) = q_1 \mathbf{a}_{i1} + q_2 \mathbf{a}_{i2} + q_3 \mathbf{a}_{i3} $$
   [S2]

2. **Compute Splat Center (Optional/General):**
   If deriving a single center per face using general barycentric coordinates $u, w$:
   $$ \mathbf{m} = u \cdot \mathbf{v}_1 + w \cdot \mathbf{v}_2 + (1 - u - w) \cdot \mathbf{v}_3 $$
   where $\mathbf{v}_k$ are the vertex positions. [S1]

3. **Compute Midpoints for Heuristic Sampling:**
   Define six midpoints $\mathbf{m}_{jk}$ on the edges of the triangle using specific barycentric weights derived from hyperparameters $u$ and $v$ [S2]:
   - Edge 1-2: $\mathbf{m}_{12} = \frac{1-u}{2}\mathbf{v}_1 + \frac{1-u}{2}\mathbf{v}_2 + u\mathbf{v}_3$
   - Edge 2-3: $\mathbf{m}_{23} = \frac{1-u}{2}\mathbf{v}_1 + \frac{1-u}{2}\mathbf{v}_2 + u\mathbf{v}_3$
   - Edge 3-1: $\mathbf{m}_{31} = \frac{1-u}{2}\mathbf{v}_1 + u\mathbf{v}_2 + \frac{1-u}{2}\mathbf{v}_3$
   - Edge 4-5 (Edge 1-2 variant): $\mathbf{m}_{45} = \frac{1-v}{2}\mathbf{v}_1 + \frac{1-v}{2}\mathbf{v}_2 + v\mathbf{v}_3$
   - Edge 5-6 (Edge 2-3 variant): $\mathbf{m}_{56} = \frac{1-v}{2}\mathbf{v}_1 + \frac{1-v}{2}\mathbf{v}_2 + v\mathbf{v}_3$
   - Edge 6-4 (Edge 3-1 variant): $\mathbf{m}_{64} = \frac{1-v}{2}\mathbf{v}_1 + v\mathbf{v}_2 + \frac{1-v}{2}\mathbf{v}_3$
   
   *Note: The source [S2] lists identical formulas for (1,2) and (2,3) and for (4,5) and (5,6). This implies the midpoints are defined relative to specific edge orientations or vertex orderings not fully detailed in the excerpt, but the barycentric weights are explicitly given.* [S2]

4. **Sample Gaussian Parameters for Each Midpoint $\mathbf{m}_{jk}$:**
   For each midpoint $\mathbf{m}_{jk}$, calculate the Gaussian parameters [S2]:
   
   a. **Center Position:**
      $$ \mathbf{c} = \mathbf{m}_{jk} \odot \mathbf{P}_i $$
   
   b. **Normal:**
      $$ \mathbf{n} = \mathbf{m}_{jk} \odot \mathbf{N}_i $$
   
   c. **Edge Vector:**
      Let $\mathbf{b}_k$ be the vertex index of the "end" of the edge corresponding to $k$ (e.g., for edge 1-2, $k=2$). The vector from the midpoint to the vertex is:
      $$ \mathbf{e} = (\mathbf{b}_k \odot \mathbf{P}_i) - \mathbf{m}_{jk} \odot \mathbf{P}_i $$
   
   d. **Scaling X:**
      $$ s_x = \alpha_{jk} \|\mathbf{e}\|_2 $$
   
   e. **Rotation X:**
      $$ \mathbf{r}_x = \frac{\mathbf{e}}{\|\mathbf{e}\|_2} $$
   
   f. **Scaling Y:**
      $$ s_y = \frac{\text{Area}(F_i)}{\beta_{jk} \|\mathbf{e}\|_2} $$
      where $\text{Area}(F_i)$ is the area of the triangular face.
   
   g. **Rotation Y:**
      $$ \mathbf{r}_y = \mathbf{n} \times \mathbf{r}_x $$
   
   h. **Scaling Z:**
      $$ s_z = \delta_{jk} $$
   
   i. **Rotation Z:**
      $$ \mathbf{r}_z = \mathbf{n} $$

**Parameters:**
- Hyperparameters for generalized geometric alignment [S2]:
  - $u = 0.07$
  - $v = 0.22$
  - $\alpha_{12}=\alpha_{23}=\alpha_{31}=0.80$
  - $\alpha_{45}=\alpha_{56}=\alpha_{64}=2.08$
  - $\beta_{12}=\beta_{23}=\beta_{31}=15.0$
  - $\beta_{45}=\beta_{56}=\beta_{64}=13.0$
  - $\delta_{12}=\delta_{23}=\delta_{31}=\delta_{45}=\delta_{56}=\delta_{64}=4.5 \times 10^{-5}$

**Edge Cases and Pitfalls:**
- **Degenerate Triangles:** If $\|\mathbf{e}\|_2 \approx 0$ (midpoint coincides with vertex), normalization of $\mathbf{r}_x$ fails. [G] Guard against division by zero by clamping $\|\mathbf{e}\|_2$ to a small epsilon or skipping such splats.
- **Face Removal:** When a mesh face is removed, all splats mapped to it must be identified and flagged for removal [S1].

## Operating characteristics
Not given in the sources.

## Background
Originating from GeoSplatting [S2] and OMeGa [S1], this method addresses the lack of inherent geometry boundaries in standard 3D Gaussian Splatting by binding splats to explicit mesh surfaces. It enables physically-based inverse rendering by ensuring shape consistency between the Gaussian representation and the underlying mesh geometry [S2].

## Sources
[S1] OMeGa: Joint Optimization of Explicit Meshes and Gaussian Splats for Robust Scene-Level Surface Reconstruction, https://doi.org/10.1109/WACV61042.2026.00427, fetched 2026-10-30
[S2] GeoSplatting: Towards Geometry Guided Gaussian Splatting for Physically-Based Inverse Rendering, https://doi.org/10.1109/ICCV51701.2025.02692, fetched 2026-10-30
