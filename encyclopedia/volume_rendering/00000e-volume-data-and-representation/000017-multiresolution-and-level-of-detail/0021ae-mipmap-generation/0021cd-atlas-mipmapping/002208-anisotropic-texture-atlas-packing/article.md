# Anisotropic Texture Atlas Packing

Also called: Anisotropic Rectangular Packing, Shelf-First-Fit-Decreasing Atlas Packing

## Overview
Anisotropic Texture Atlas Packing is a method to pack surfel textures into a fixed-width atlas using anisotropic rectangular tiles sized according to the Nyquist rate of the hash grid, avoiding square padding on thin surfels [S1, unofficial]. It addresses the inefficiency of square padding for non-uniform surfels by aligning tile dimensions with the surfel's tangent axis scales, thereby reducing wasted texture memory and improving cache coherence during rendering [S1, unofficial].

## How it works
**Inputs:**
- Set of surfels $g$, each with center $\mathbf{p}_g$, tangent frame $R$, scales $s_{g,u}, s_{g,v}$, and UV half-extent $e$.
- Hash grid finest voxel size $\delta$.
- Atlas width constraint $W$ (default 4096).
- Resolution bounds $r_{\min}=4, r_{\max}=64$.

**Outputs:**
- A 2D texture atlas of dimensions $W \times H$.
- For each surfel $g$, a rectangular tile of size $r_{g,u} \times r_{g,v}$ and its offset $(u_{off}, v_{off})$ in the atlas.
- For each texel in the tile, the pre-computed RGB value $\rho(\mathbf{x})$.

**Data Structures:**
- **Shelf:** A contiguous horizontal band in the atlas. Defined by its starting row index $y_{start}$, current height $h$, and remaining width $w_{rem}$.
- **Atlas:** A 2D array of size $W \times H$. Initialized to zero or transparent.

**Steps:**

1. **Calculate Anisotropic Resolution:**
   For each surfel $g$ and axis $a \in \{u, v\}$, compute the required resolution $r_{g,a}$:
   $$r_{g,a} = \text{clamp} \left( 2^{\lceil \log_2(4es_{g,a}/\delta) \rceil}, r_{\min}, r_{\max} \right)$$
   where $s_{g,a}$ is the surfel scale on axis $a$, $e$ is the UV half-extent in units of that scale, and $\delta$ is the finest hash voxel size [S1, unofficial]. The factor 4 combines the $2e$ span with Nyquist sampling requirements [S1, unofficial].

2. **Sort Surfels:**
   Sort the surfels in descending order of height $r_{g,v}$, then by width $r_{g,u}$ [S1, unofficial].

3. **Pack into Atlas (Shelf-First-Fit-Decreasing):**
   Initialize atlas height $H = 0$. Initialize a list of open shelves.
   For each surfel $g$ in sorted order:
   - Let tile width $w = r_{g,u}$ and tile height $h = r_{g,v}$.
   - Find the first shelf in the list where $w_{rem} \ge w$.
   - If found:
     - Place the tile at the shelf's current $x$ offset.
     - Update shelf's $w_{rem} \leftarrow w_{rem} - w$.
     - Update shelf's $y_{start}$ if this is the first tile in the shelf (initially $H$).
     - Update shelf's height $h_{shelf} \leftarrow \max(h_{shelf}, h)$.
   - If not found:
     - Create a new shelf starting at row $H$.
     - Place the tile at $x=0$.
     - Set shelf $h_{shelf} = h$.
     - Update $H \leftarrow H + h$.
     - Add new shelf to list.
   - Round $H$ to the next multiple of 64 after placing all surfels [S1, unofficial].

4. **Evaluate and Bake:**
   For each surfel $g$ and each texel $(i, j)$ in its assigned tile ($0 \le i < r_{g,u}, 0 \le j < r_{g,v}$):
   - Compute local coordinates:
     $$u = \frac{i + 0.5}{r_{g,u}} 2e - e$$
     $$v = \frac{j + 0.5}{r_{g,v}} 2e - e$$
     This mapping aligns texel centers with the hardware bilinear sampler to prevent boundary seams [S1, unofficial].
   - Lift to world space:
     $$\mathbf{x} = \mathbf{p}_g + u s_{g,u} \mathbf{R}_{\cdot,0} + v s_{g,v} \mathbf{R}_{\cdot,1}$$
     where $\mathbf{R}_{\cdot,0}$ and $\mathbf{R}_{\cdot,1}$ are the first two columns of the surfel's tangent frame $R$ [S1, unofficial].
   - Evaluate the hash grid and MLP decoder at $\mathbf{x}$ to get RGB value $\rho(\mathbf{x})$.
   - Write $\rho(\mathbf{x})$ to the atlas at the calculated position.

5. **Handle Atlas Limits:**
   If $H$ approaches 65,536 rows (the `cudaArray` limit), increase $W$ (e.g., double it) and repack [S1, unofficial].

**Parameters:**
- $W$: Atlas width. Default 4096 [S1, unofficial].
- $r_{\min}$: Minimum resolution. Default 4 [S1, unofficial].
- $r_{\max}$: Maximum resolution. Default 64 [S1, unofficial].
- $e$: UV half-extent. Typical value 4 [S1, unofficial].
- $\delta$: Finest hash voxel size. Scene-dependent.

**Edge Cases and Pitfalls:**
- **Boundary Seams:** Using integer texel indices $i, j$ directly without the $+0.5$ offset and normalization in Eq. 4 causes misalignment with bilinear filtering, leading to visible seams [S1, unofficial].
- **Wasted Memory:** With $e=4$, a significant portion of texels (e.g., 50.5% in indoor scenes) may lie outside the kernel support and are never fetched, wasting memory [S1, unofficial].
- **Precision:** Logarithmic resolution calculation must use sufficient precision to avoid off-by-one errors in power-of-two sizing [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Sorting is $O(N \log N)$. Packing is $O(N)$ with a simple shelf list. Baking is $O(N \cdot r_{avg}^2)$, where $N$ is the number of surfels and $r_{avg}$ is the average resolution.
- **Memory Complexity:** $O(W \cdot H)$ for the atlas. $H$ grows greedily and is bounded by the number of surfels and their heights.
- **Performance:** Reduces texture memory usage compared to square padding for anisotropic surfels. Improves cache coherence by packing similar-sized tiles together.
- **Strengths:** Efficient packing for non-square surfels. Simple algorithm.
- **Limitations:** Wasted space for large $e$ values. Fixed width may require repacking if height limit is reached.

## Background
Originated in "Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting" [S1, unofficial]. It builds on the concept of atlas mipmapping [2639-atlas-mipmapping] to address undersampling issues by pre-computing textures at appropriate resolutions.

## Sources
[S1] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
