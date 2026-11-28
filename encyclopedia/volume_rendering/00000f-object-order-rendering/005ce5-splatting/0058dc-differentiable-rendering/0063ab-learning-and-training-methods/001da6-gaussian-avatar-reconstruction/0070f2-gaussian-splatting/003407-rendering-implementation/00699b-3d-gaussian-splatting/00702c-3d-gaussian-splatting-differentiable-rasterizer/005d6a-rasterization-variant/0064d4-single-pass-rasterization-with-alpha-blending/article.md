# Single-Pass Rasterization with Alpha-Blending

Also called: Single-Pass Anti-Aliased Rasterization, Unified Rasterization

## Overview
This method provides a unified rendering pipeline that rasterizes both 3D Gaussian Splatting (3DGS) primitives and triangle meshes in a single pass using alpha-blending, ensuring correct spatial occlusion and color blending without dropping rendering rates [S1]. It solves the problem of integrating mesh-based deformation or proxy geometry with 3DGS by treating depth-adjacent triangles as a single entity for transmittance calculation, thereby avoiding color overflow artifacts that occur when Gaussians are present between triangles [S1]. Unlike standard 3DGS rasterization which only handles splats, or standard rasterization which only handles triangles, this variant unifies them by applying Multi-Sample Anti-Aliasing (MSAA) logic to triangle fragments while maintaining the standard front-to-back alpha compositing for Gaussians [S1].

## How it works
**Inputs:**
- A set of 3D Gaussian fragments, each defined by color $\mathbf{c}_i$, opacity $\alpha_i$, and depth.
- A set of triangle fragments, each defined by color $\mathbf{c}_i$, opacity $\alpha_i$ (typically 1.0 for opaque triangles, but treated as alpha for blending), and depth.
- A background color $\mathbf{C}_{bg}$ and background opacity $\alpha_{bg}$.
- Initial transmittance $T_{in}$ (typically 1.0).
- Number of sub-pixels $M$ for MSAA (typically 4) [S1].

**Outputs:**
- Final pixel color $\mathbf{C}_{pixel}$ [S1].

**Data Structures:**
- Fragments are sorted by depth (front-to-back) within each tile/pixel.
- For triangles, sub-pixel coverage is tracked. Let $o_{ji} \in \{0, 1\}$ be a boolean indicating whether the $i$-th triangle fragment covers the $j$-th sub-pixel [S1].

**Steps:**

1.  **Initialize:** Set initial transmittance $T_{in} = 1.0$ [S1].

2.  **Process Gaussian Fragments:**
    For each Gaussian fragment $i$ (in depth order):
    - Compute transmittance $T_i$ representing the visibility of the fragment:
      $$ T_i = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
      where $T_1 = T_{in}$ [S1].
    - Accumulate Gaussian color contribution $\mathbf{C}_{gs}$:
      $$ \mathbf{C}_{gs} = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i $$
      where $N$ is the number of overlapping fragments [S1].

3.  **Process Triangle Fragments:**
    For each triangle fragment $i$ (in depth order):
    - **Sub-pixel Transmittance:** Calculate the transmittance at the sub-pixel level $t_{ji}$ for each sub-pixel $j \in \{1, \dots, M\}$:
      $$ t_{ji} = \prod_{k=1}^{i-1} (1 - o_{jk} \alpha_k) $$
      with $t_{j1} = 1.0$ [S1].
    - **Pixel Coverage:** Compute the effective coverage $O_i$ of the triangle at the pixel level by averaging sub-pixel coverages weighted by sub-pixel transmittance:
      $$ O_i = \frac{1}{M} \sum_{j=1}^{M} o_{ji} t_{ji} $$
      where $o_{ji}$ is the geometric coverage of fragment $i$ on sub-pixel $j$ [S1].
    - **Triangle Color Accumulation:** Compute the color contribution $\mathbf{C}_{tris}$ for the current group of depth-adjacent triangles. The method treats depth-adjacent triangles as a single entity to ensure correct transmittance decay for subsequent blending [S1]:
      $$ \mathbf{C}_{tris} = T_{in} \sum_{i=1}^{N} O_i \alpha_i \mathbf{c}_i $$
      Note: The pixel-level transmittance $T_i$ is updated using $O_i$ for subsequent blending, but is not used in the summation for $\mathbf{C}_{tris}$ itself [S1].
    - **Update Pixel Transmittance:** Update the pixel-level transmittance $T_i$ for the next fragment:
      $$ T_i = \prod_{k=1}^{i-1} (1 - O_k \alpha_k) $$
      This $T_i$ is used for the final pixel color calculation and for blending subsequent fragments, ensuring that transmittance decay accounts for both Gaussians and triangles [S1].

4.  **Final Color Calculation:**
    Combine the triangle contribution, Gaussian contribution (if applicable in the same pass, though the source describes them as separate accumulation paths merged at the end or via the unified transmittance), and background:
    $$ \mathbf{C}_{pixel} = \mathbf{C}_{tris} + T_{bg} \mathbf{C}_{bg} $$
    where $T_{bg} = T_N (1 - \alpha_{bg})$ and $T_N$ is the final transmittance after all fragments [S1].

**Parameters:**
- $M$: Number of sub-pixels for MSAA. Typical value: 4 [S1].

**Edge Cases and Pitfalls:**
- **Color Overflow:** If all triangles overlapping a pixel are treated as a single entity when Gaussians are present between them, the transmittance for deeper triangles does not account for decay from Gaussians. This results in color overflow, manifesting as white artifacts [S1]. To avoid this, only depth-adjacent triangles are treated as a single entity for the $\mathbf{C}_{tris}$ calculation [S1].
- **Sub-pixel Logic:** The transmittance calculation for triangles must be performed at the sub-pixel level to accommodate MSAA, differing from the pixel-level transmittance used for Gaussians [S1].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of fragments (Gaussians and triangles) per pixel. The sub-pixel averaging for triangles adds a constant factor of $M$ (typically 4) to the triangle processing cost [S1].
- **Memory Complexity:** Requires storing sub-pixel coverage data for triangles, increasing memory usage compared to standard rasterization [S1].
- **Strengths:** Enables unified rendering of 3DGS and meshes in a single pass, maintaining high rendering rates while providing correct spatial occlusion and anti-aliasing for both primitives [S1].
- **Limitations:** Requires careful handling of depth-adjacent triangles to avoid artifacts when mixed with Gaussians [S1].

## Background
This method was proposed in "UniMGS: Unifying Mesh and 3D Gaussian Splatting with Single-Pass Rasterization and Proxy-Based Deformation" [S1]. It addresses the limitations of proxy-based 3DGS deformation, which is sensitive to mesh topology quality, by providing a robust rasterization pipeline that links trained 3DGS to proxy meshes with minimal artifacts [S1].

## Sources
[S1] UniMGS: Unifying Mesh and 3D Gaussian Splatting with Single-Pass Rasterization and Proxy-Based Deformation, https://arxiv.org/pdf/2601.19233, fetched 2026-10-26
