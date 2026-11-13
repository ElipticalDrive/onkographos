# Sparse Voxel Rasterization

Also called: SVR, SVRasterization

## Overview
Sparse Voxel Rasterization (SVRaster) is a real-time volumetric rendering technique that rasterizes explicit sparse voxels to reconstruct radiance fields from multi-view images, avoiding neural networks or 3D Gaussians [S1, S2][S3, unofficial]. It differs from its siblings by using a hybrid primitive-volumetric model where voxels are stored in a flat 1D array without parent-child pointers, and employs a direction-dependent Morton order sorting algorithm to ensure correct front-to-back alpha composition without popping artifacts [S1, S2][S3, unofficial]. Unlike [Intensity-based Sparse Voxel Storage](1775-intensity-based-sparse-voxel-storage.md) which may rely on implicit density fields or different sorting, SVRaster explicitly allocates leaf voxels and sorts them per tile based on ray direction signs [S1, S2]. It outperforms [SVO LOD Management](2990-svo-lod-management.md) and [Voxel Chunk Management](13837-voxel-chunk-management.md) in rendering speed by bypassing ray-casting traversal in favor of direct rasterization [S3, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Camera parameters: origin $\mathbf{r}_o$, direction $\mathbf{r}_d$.
- Scene representation: A set of active sparse voxels. Each voxel $i$ is defined by:
  - Octree level $l \in [1, L]$, where $L=16$ is the maximum level [S1, S2].
  - Voxel index $\mathbf{v} = \{i, j, k\}$ within the global grid [S1].
  - Density parameters $\mathbf{V} \in \mathbb{R}^{2\times2\times2}$ at the eight corners [S1].
  - Spherical Harmonic (SH) coefficients $\mathbf{v}_{\text{sh}} \in \mathbb{R}^{(N_{\text{shd}}+1)^2 \times 3}$ [S1].
- Image grid: Width $W$, Height $H$, tile size $T \times T$ (typically $T=16$) [S1][S3, unofficial].

**Outputs:**
- Rendered color image.
- Optional depth map and normal map [S1].

### Data Structures
The scene is stored in a **1D array** of active voxels. No parent-child pointers or linear Octree structures are maintained; only leaf nodes are stored [S1, S2].
- **Voxel Entry:** Contains level $l$, index $\{i,j,k\}$, density parameters $\mathbf{V}$, and SH coefficients $\mathbf{v}_{\text{sh}}$ [S1, S2].
- **Sorting Key-Value Pair:** Used for tile-based sorting [S1]:
  - **Key:** 64 bits. Upper 32 bits: Tile ID. Lower 32 bits: Morton code (derived from voxel position and ray direction) [S1].
  - **Value:** 32 bits: Voxel ID (index in the 1D array) [S1].

### Parameters
- $L$: Maximum Octree level. Default $L=16$, corresponding to a finest virtual granularity of $65536^3$ [S1, S2][S3, unofficial].
- $K$: Number of sample points inside each voxel for numerical integration. $K=1$ for novel-view synthesis, $K=3$ for mesh reconstruction [S1]. Experiments limit $K \le 3$ [S1].
- $h_{\text{T}}$: Early ray termination threshold. Default $10^{-4}$ [S1].
- $h_{\text{ss}}$: Supersampling scale for anti-aliasing. Default $1.5$ [S1].
- $\tau_{\text{prune}}$: Pruning threshold for voxel removal. Linearly scaled from $0.0001$ to $0.05$ during training [S1].

### Steps

#### 1. Pre-processing
For each active voxel assigned to the current view:
1. **Compute Voxel Geometry:**
   - Voxel size: $\mathbf{v}_s = \mathbf{w}_s \cdot 2^{-l}$ [S1].
   - Voxel center: $\mathbf{v}_c = \mathbf{w}_c - 0.5 \cdot \mathbf{w}_s + \mathbf{v}_s \cdot \mathbf{v}$ [S1].
   - Here, $\mathbf{w}_s$ is the Octree size, $\mathbf{w}_c$ is the Octree center, and $\mathbf{v}=\{i,j,k\}$ is the voxel index [S1].
2. **Compute View-Dependent Color:**
   - Approximate ray direction using the vector from camera to voxel center: $\mathbf{d} = \text{normalize}(\mathbf{v}_c - \mathbf{r}_o)$ [S1].
   - Evaluate SH: $c = \max(0, \text{sh\_eval}(\mathbf{v}_{\text{sh}}, \mathbf{d}))$ [S1].
3. **Compute Voxel Normal:**
   - Compute analytical gradient of density at center $\mathbf{q}_c = (0.5, 0.5, 0.5)$ [S1]:
     $$ \mathbf{n}_{\text{raw}} = 0.25 \cdot \begin{bmatrix} (\mathbf{V}_{100} + \mathbf{V}_{101} + \mathbf{V}_{110} + \mathbf{V}_{111}) - (\mathbf{V}_{000} + \mathbf{V}_{001} + \mathbf{V}_{010} + \mathbf{V}_{011}) \\ (\mathbf{V}_{010} + \mathbf{V}_{011} + \mathbf{V}_{110} + \mathbf{V}_{111}) - (\mathbf{V}_{000} + \mathbf{V}_{001} + \mathbf{V}_{100} + \mathbf{V}_{101}) \\ (\mathbf{V}_{001} + \mathbf{V}_{011} + \mathbf{V}_{101} + \mathbf{V}_{111}) - (\mathbf{V}_{000} + \mathbf{V}_{010} + \mathbf{V}_{100} + \mathbf{V}_{110}) \end{bmatrix} $$
   - Normalize: $\mathbf{n} = \text{normalize}(\mathbf{n}_{\text{raw}})$ [S1].

#### 2. Projection and Tile Assignment
1. Project the eight corners of each voxel to image space [S1][S3, unofficial].
2. Compute the axis-aligned bounding box (AABB) of the projected corners [S1].
3. Assign the voxel to all tiles that overlap with this AABB [S1][S3, unofficial].

#### 3. Sorting
Sort voxels within each tile in near-to-far order using a direction-dependent Morton code [S1][S3, unofficial].
1. Determine the sign pattern of the ray direction $\mathbf{r}_d$ (8 octants) [S1][S3, unofficial].
2. For each voxel, compute its Morton code based on its level $l$ and index $\{i,j,k\}$ [S1].
3. Apply an XOR mask to the Morton code based on the ray direction sign pattern to ensure correct near-to-far traversal [S3, unofficial].
4. Sort the voxels using the modified Morton code as the key [S1][S3, unofficial].

#### 4. Rendering (Alpha Composition)
For each pixel in each tile:
1. Cast ray $\mathbf{r}_o, \mathbf{r}_d$ [S1].
2. Iterate through sorted voxels assigned to the tile [S1].
3. **Ray-AABB Intersection:**
   - Compute entry $a$ and exit $b$ distances:
     $$ c_0 = \frac{\mathbf{v}_c - 0.5 \cdot \mathbf{v}_s - \mathbf{r}_o}{\mathbf{r}_d}, \quad c_1 = \frac{\mathbf{v}_c + 0.5 \cdot \mathbf{v}_s - \mathbf{r}_o}{\mathbf{r}_d} $$
     $$ a = \max(\min(c_0), \min(c_1)), \quad b = \min(\max(c_0), \max(c_1)) $$
   - If $a > b$ or $a \le 0$, skip voxel [S1].
4. **Sample Voxel:**
   - Ray segment length: $l = (b - a) \cdot \|\mathbf{r}_d\|$ [S1].
   - Sample $K$ points evenly along the segment:
     $$ t_k = a + \frac{k - 0.5}{K} \cdot (b - a), \quad \mathbf{p}_k = \mathbf{r}_o + t_k \cdot \mathbf{r}_d $$
   - Local voxel coordinate: $\mathbf{q}_k = (\mathbf{p}_k - (\mathbf{v}_c - 0.5 \cdot \mathbf{v}_s)) / \mathbf{v}_s$ [S1].
   - Trilinear interpolation of density: $v_k = \text{interp}(\mathbf{V}, \mathbf{q}_k)$ [S1].
   - Exponential-linear activation:
     $$ \text{explin}(x) = \begin{cases} x & \text{if } x > 1.1 \\ \exp\left(\frac{x}{1.1} - 1 + \ln 1.1\right) & \text{otherwise} \end{cases} $$
   - Per-sample alpha: $\alpha_k = 1 - \exp\left(-\frac{l}{K} \cdot \text{explin}(v_k)\right)$ [S1].
5. **Composite:**
   - Update transmittance $T$ and color $C$:
     $$ C = C + T \cdot c \cdot \alpha, \quad T = T \cdot (1 - \alpha) $$
     where $\alpha$ is the effective voxel alpha derived from samples (see formula below) [S1].
   - **Voxel Alpha Formula:**
     $$ \alpha = 1 - \exp\left(-\frac{l}{K} \sum_{k=1}^K \text{explin}(v_k)\right) $$
     (Note: Sources [S1] Eq. 4 and A.2 Eq. 14a show $\exp \ln(\text{interp})$ which simplifies to $\text{interp}$ if positive, but the activation is applied after interpolation. The text states "apply the non-linear activation after trilinear interpolation" [S1]. The formula in A.2 Eq. 14a uses $\exp \ln(\text{interp})$ which is redundant if interp is positive, but likely implies the activation function $\text{explin}$ is used. Given Eq. 18 uses $\text{explin}(\text{interp})$, we use $\text{explin}$) [S1].
   - **Early Termination:** If $T < h_{\text{T}}$, stop processing voxels for this pixel [S1].

#### 5. Post-processing
- **Anti-aliasing:** Render at $h_{\text{ss}} \times$ resolution and downsample [S1].
- **Depth/Normal:** Composite depth $D = \sum T_i d_i$ and normals similarly [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def exponential_linear(x):
    """Exponential-linear activation function."""
    if x > 1.1:
        return x
    else:
        return math.exp(x / 1.1 - 1 + math.log(1.1))

def trilinear_interpolate(V, q):
    """
    Trilinear interpolation of density V at local coordinate q.
    V is a 2x2x2 array of density parameters.
    q is a tuple (qx, qy, qz) in [0, 1].
    """
    qx, qy, qz = q
    # Indices i, j, k correspond to x, y, z
    # V[i, j, k]
    v000 = V[0, 0, 0]
    v100 = V[1, 0, 0]
    v010 = V[0, 1, 0]
    v110 = V[1, 1, 0]
    v001 = V[0, 0, 1]
    v101 = V[1, 0, 1]
    v011 = V[0, 1, 1]
    v111 = V[1, 1, 1]
    
    # Interpolation weights
    w000 = (1 - qx) * (1 - qy) * (1 - qz)
    w100 = qx * (1 - qy) * (1 - qz)
    w010 = (1 - qx) * qy * (1 - qz)
    w110 = qx * qy * (1 - qz)
    w001 = (1 - qx) * (1 - qy) * qz
    w101 = qx * (1 - qy) * qz
    w011 = (1 - qx) * qy * qz
    w111 = qx * qy * qz
    
    return (w000 * v000 + w100 * v100 + w010 * v010 + w110 * v110 +
            w001 * v001 + w101 * v101 + w011 * v011 + w111 * v111)

def ray_aabb_intersect(vox_c, vox_s, ro, rd):
    """
    Intersect ray with voxel AABB.
    Returns (a, b, valid).
    a: entry distance, b: exit distance.
    """
    # c0 and c1 are vectors of t-values for min/max faces
    c0 = [(vox_c[d] - 0.5 * vox_s[d] - ro[d]) / rd[d] for d in range(3)]
    c1 = [(vox_c[d] + 0.5 * vox_s[d] - ro[d]) / rd[d] for d in range(3)]
    
    t_min = max(min(c0[d], c1[d]) for d in range(3))
    t_max = min(max(c0[d], c1[d]) for d in range(3))
    
    valid = (t_min <= t_max) and (t_min > 0)
    return t_min, t_max, valid

def compute_voxel_alpha(V, vox_c, vox_s, ro, rd, K=1):
    """
    Compute alpha value for a voxel.
    """
    a, b, valid = ray_aabb_intersect(vox_c, vox_s, ro, rd)
    if not valid:
        return 0.0, 0.0, False # alpha, length, valid
    
    l = (b - a) * math.sqrt(sum(d*d for d in rd)) # Ray segment length
    
    sum_density = 0.0
    for k in range(1, K + 1):
        t_k = a + (k - 0.5) / K * (b - a)
        p_k = [ro[d] + t_k * rd[d] for d in range(3)]
        
        # Local coordinate
        q = [(p_k[d] - (vox_c[d] - 0.5 * vox_s[d])) / vox_s[d] for d in range(3)]
        
        v_k = trilinear_interpolate(V, q)
        sum_density += exponential_linear(v_k)
        
    alpha = 1.0 - math.exp(-sum_density / K * l)
    return alpha, l, True

def render_tile(tile_voxels, ro, rd, tile_id, h_T=1e-4):
    """
    Render a single tile.
    tile_voxels: List of voxels assigned to this tile, sorted by Morton code.
    """
    T = 1.0 # Transmittance
    C = [0.0, 0.0, 0.0] # Color
    
    for voxel in tile_voxels:
        if T < h_T:
            break
            
        alpha, l, valid = compute_voxel_alpha(voxel.V, voxel.v_c, voxel.v_s, ro, rd, K=1)
        
        if valid and alpha > 0:
            # Color c is pre-computed
            C = [C[d] + T * voxel.c[d] * alpha for d in range(3)]
            T *= (1.0 - alpha)
            
    return C, T
```

### Edge Cases and Pitfalls
- **Incorrect Sorting:** Using primitive centers or closest distance to camera causes popping artifacts. Must use direction-dependent Morton order [S1, S2].
- **Multiple Morton Orders:** If a tile contains rays with different direction signs, multiple Morton orders are required. This is a corner case handled in supplementary materials [S1].
- **Exposure Variation:** Scenes with severe exposure variation in training views cause clear boundaries and floaters, slowing rendering [S1].
- **Search Limitations:** The 1D array storage does not support efficient point-to-voxel search (e.g., finding the covering voxel for a point) [S2].

## Operating characteristics
- **Time Complexity:** Sorting dominates preprocessing. Rendering is proportional to the number of voxels per tile. Average 27 primitives per pixel, compared to 63 for 3DGS [S1].
- **Memory Complexity:** Stores only leaf voxels. On 24GB GPU, fails to reach $1024^3$ grid resolution despite pruning [S1]. Model size is larger than 3DGS (1.8GB vs 0.7GB on Mip-NeRF360) [S1].
- **Performance:** Achieves over 10x speedup compared to uniform-grid ray casting [S3, unofficial]. On Mip-NeRF360, achieves 121 FPS (standard), 258 FPS (fast-rend) [S1]. PSNR ~27 dB [S3, unofficial].
- **Strengths:** Real-time rendering, high fidelity, no neural MLPs, no 3D Gaussians, compatible with grid-based processing (Volume Fusion, Marching Cubes) [S2][S3, unofficial].
- **Limitations:** Struggles with severe exposure variation. 1D array limits efficient spatial queries. Maximum level limited to 16 in current implementation [S1, S2].

## Background
Sparse Voxel Rasterization (SVRaster) was introduced in 2025 by the authors of "Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering" [S1]. It builds on the principles of [Voxels rasterized as geometry](27-voxels-rasterized-as-geometry.md) and [Sparse Voxel Storage and Management](8435-sparse-voxel-storage-and-management.md). It was developed to provide a neural-free, explicit voxel-based alternative to 3D Gaussian Splatting and Plenoxels, aiming for high frame rates and scalability [S1][S3, unofficial]. It is used in novel-view synthesis and mesh reconstruction tasks on datasets like Mip-NeRF360, Tanks&Temples, and DTU [S1].

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
[S2] Sparse Voxels Rasterization: Real-Time High-Fidelity Radiance Field Rendering, https://svraster.github.io/, fetched 2026-10-26
[S3] Sparse Voxel Rasterization (SVRaster), https://www.emergentmind.com/topics/sparse-voxel-rasterization-svraster, fetched 2026-10-27 (unofficial)
