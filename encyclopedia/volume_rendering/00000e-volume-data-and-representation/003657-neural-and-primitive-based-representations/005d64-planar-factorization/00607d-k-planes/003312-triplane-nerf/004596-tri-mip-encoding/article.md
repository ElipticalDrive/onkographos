# Tri-Mip Encoding

Also called: Tri-Mip, Tri-Mip Representation

## Overview
Tri-Mip Encoding is a method to model the pre-filtered 3D feature space by factorizing it into three orthogonal 2D mipmaps, enabling efficient area sampling for anti-aliased rendering [S1]. It decomposes 3D space into three axis-aligned planes (XY, XZ, YZ) and represents each with a mipmap structure, allowing queries to adaptively select resolution based on the sampling area size [S1]. Unlike [TriPlane NeRF](6932-triplane-nerf.md), which uses static feature planes, Tri-Mip Encoding uses mipmaps to explicitly model pre-filtered features, ensuring coherence across different viewing distances and resolutions [S1].

## How it works
**Inputs:**
- Sphere center position $\position \in \mathbb{R}^3$ and radius $\radius \in \mathbb{R}^+$ [S1].
- Three orthogonal mipmaps $\mipmap = \{\mipmap _{XY}, \mipmap _{XZ},\mipmap _{YZ}\}$ [S1].
- Axis Aligned Bounding Box (AABB) of the scene defined by minimum corner $\aabb _{min}$ and maximum corner $\aabb _{max}$ [S1].
- Mipmap base dimensions $H, W$ and channel count $C$ [S1].

**Outputs:**
- Concatenated feature vector $\feat \in \mathbb{R}^{3C}$ [S1].

**Data Structures:**
- **Mipmap Level $M^L_i$**: A 2D tensor of shape $H_i \times W_i \times C$ [S1].
  - Base level $M^L_0$: Shape $H \times W \times C$, randomly initialized and trainable [S1].
  - Higher levels $M^L_i$ ($i=1..N$): Derived by downscaling $M^L_{i-1}$ by 2x along height and width [S1].
  - Storage: Only base levels $M^L_0$ are stored explicitly; higher levels are derived on-the-fly [S1].

**Parameters:**
- $\featradius$: Radius of feature elements in the base level, calculated as:
  $$ \featradius = \sqrt {\frac {(\aabb _{max}-\aabb _{min})_X \cdot (\aabb _{max}-\aabb _{min})_Y}{HW\cdot \pi }} $$
  [S1]

**Steps:**
1. **Decomposition**: Decompose the 3D space into three orthogonal planes: planeXY, planeXZ, and planeYZ [S1].
2. **Projection**: Orthogonally project the sampled sphere $S(\position, \radius)$ onto the three planes to obtain three discs $D = \{D_{XY}, D_{XZ}, D_{YZ}\}$ [S1].
3. **Disc Geometry**: For each disc $D_{XY}$, the 2D center coordinate is the partial coordinate $(x, y)$ of the sphere's center $\position$, and the radius is $\radius$ [S1].
4. **Level Assignment**: Calculate the query level $l$ for each mipmap to match the sphere's radius with the feature element radius:
   $$ l = \log_2\bigg (\frac {\radius }{\featradius }\bigg ) $$
   [S1]
5. **Feature Query**: For each plane (e.g., XY):
   a. Identify the two nearest integer levels $\lfloor l \rfloor$ and $\lceil l \rceil$ [S1].
   b. Project the disc center $(x, y)$ to these two levels [S1].
   c. Find the four nearest neighbors (corners) on each level (8 total neighbors) [S1].
   d. Perform trilinear interpolation on the 8 neighbors to produce the feature vector $f_{XY}$ [S1].
6. **Concatenation**: Concatenate the feature vectors from the three planes $\{f_{XY}, f_{XZ}, f_{YZ}\}$ to form the final feature vector $\feat$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def trimip_encoding(position, radius, mipmap_bases, aabb_min, aabb_max, H, W, C):
    """
    Query Tri-Mip encoding for a sphere defined by position and radius.
    """
    x, y, z = position
    dx = aabb_max[0] - aabb_min[0]
    dy = aabb_max[1] - aabb_min[1]
    featradius = math.sqrt((dx * dy) / (H * W * math.pi))
    l = math.log2(radius / featradius)

    max_level = int(math.log2(min(H, W)))
    l_floor = max(0, math.floor(l))
    l_ceil = min(max_level, math.ceil(l))
    weight = l - l_floor

    features = []
    planes = [('XY', 0, 1), ('XZ', 0, 2), ('YZ', 1, 2)]
    for name, idx_u, idx_v in planes:
        M_base = mipmap_bases[name]
        u = (position[idx_u] - aabb_min[idx_u]) / (aabb_max[idx_u] - aabb_min[idx_u])
        v = (position[idx_v] - aabb_min[idx_v]) / (aabb_max[idx_v] - aabb_min[idx_v])

        w_floor = W / (2 ** l_floor)
        h_floor = H / (2 ** l_floor)
        w_ceil = W / (2 ** l_ceil)
        h_ceil = H / (2 ** l_ceil)

        M_floor = downsample(M_base, l_floor)
        M_ceil = downsample(M_base, l_ceil)

        u_f = u * w_floor
        v_f = v * h_floor
        u_c = u * w_ceil
        v_c = v * h_ceil

        f_floor = trilinear_sample(M_floor, u_f, v_f)
        f_ceil = trilinear_sample(M_ceil, u_c, v_c)
        features.append(lerp(f_floor, f_ceil, weight))

    return concatenate(features)

def downsample(tensor, level):
    current = tensor
    for _ in range(level):
        current = average_pool_2x(current)
    return current

def average_pool_2x(tensor):
    H, W, C = tensor.shape
    new_H, new_W = H // 2, W // 2
    result = [[[0.0] * C for _ in range(new_W)] for _ in range(new_H)]
    for i in range(new_H):
        for j in range(new_W):
            for c in range(C):
                result[i][j][c] = (tensor[2*i][2*j][c] + tensor[2*i+1][2*j][c] +
                                   tensor[2*i][2*j+1][c] + tensor[2*i+1][2*j+1][c]) / 4.0
    return result

def trilinear_sample(tensor, u, v):
    u_floor = math.floor(u)
    v_floor = math.floor(v)
    u_frac = u - u_floor
    v_frac = v - v_floor

    H, W, C = tensor.shape
    u_floor = max(0, min(W - 2, u_floor))
    v_floor = max(0, min(H - 2, v_floor))

    f00 = get_value(tensor, u_floor, v_floor)
    f10 = get_value(tensor, u_floor + 1, v_floor)
    f01 = get_value(tensor, u_floor, v_floor + 1)
    f11 = get_value(tensor, u_floor + 1, v_floor + 1)

    top = lerp(f00, f10, u_frac)
    bottom = lerp(f01, f11, u_frac)
    return lerp(top, bottom, v_frac)

def get_value(tensor, u, v):
    H, W, C = tensor.shape
    u = max(0, min(W - 1, u))
    v = max(0, min(H - 1, v))
    return tensor[int(v)][int(u)]

def lerp(a, b, t):
    return [a[i] + t * (b[i] - a[i]) for i in range(len(a))]

def concatenate(vectors):
    return [val for vec in vectors for val in vec]
```

**Edge Cases and Pitfalls:**
- **Precision**: Trilinear interpolation increases effective precision and yields a continuous encoding space, beneficial for training [S1].
- **Boundaries**: The AABB defines the spatial extent. Queries outside the AABB require coordinate clamping to valid texel ranges [G].
- **Level Bounds**: The query level $l$ must be clamped to the valid mipmap range $[0, \lfloor \log_2(\min(H,W)) \rfloor]$ to prevent out-of-bounds access [G].

## Subtopics

### TriPlane NeRF
Aligns explicit features along three axis-aligned orthogonal feature planes, queried by projecting 3D positions onto these planes [S1].

### MultiPlaneGAN
Uses multiple planes for 3D content generation [S1].

### Platonic Solid Projection
Projects 3D space onto platonic solids [G].

### Spatial Planes
Uses spatially distributed planes [G].

## Operating characteristics
- **Time Complexity**: Querying the mipmap is efficient, leveraging highly optimized GPU texture filtering [S1].
- **Memory Complexity**: Storage is compact, requiring only three 2D feature maps (base levels) of shape $H \times W \times C$ [S1]. Higher levels are derived by downscaling, so they do not add to storage cost [S1].
- **Measured Performance**: Tri-MipRF achieves reconstruction in around ten minutes, significantly faster than methods taking days, while providing higher rendering quality than Instant-NGP, which suffers from blurring or aliasing [S1].
- **Scaling**: The method scales with the resolution $H, W$ and channel count $C$ of the base mipmap [S1].
- **Strengths**: Enables instant reconstruction and anti-aliased high-fidelity rendering; efficient area sampling; compact model size [S1].
- **Limitations**: Relies on orthographic projection, which may not capture all 3D details as effectively as perspective-aware methods in some contexts, though the cone-casting technique mitigates this for rendering [S1].
- **When to use**: When efficient reconstruction and anti-aliased rendering are required for neural radiance fields [S1].
- **When not to use**: When extreme geometric detail beyond the pre-filtered representation is needed, or when the scene is highly dynamic and requires 4D extensions not covered in the base method [S1].

## Background
Tri-Mip Encoding was proposed in "Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields" [S1]. It is inspired by the factorization of 3D space into three orthogonal planes used in 3D content generation methods like EG3D [S1]. Unlike EG3D, which uses tri-planes for generation, Tri-Mip uses mipmaps to model pre-filtered features for area sampling, addressing the aliasing issues in methods like Instant-NGP [S1]. The method is part of the hybrid representation family, combining Tri-Mip encoding with a tiny MLP [S1].

## Sources
[S1] Tri-MipRF: Tri-Mip Representation for Efficient Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01811, fetched 2026-10-26
