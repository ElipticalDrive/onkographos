# Tri-Mip Encoding

Also called: Tri-Mip, Tri-Mip Representation

## Overview
Tri-Mip Encoding is a method to model the pre-filtered 3D feature space by factorizing it into three orthogonal 2D mipmaps, enabling efficient area sampling for anti-aliased rendering [S1]. It decomposes 3D space into three axis-aligned planes (XY, XZ, YZ) and represents each with a mipmap structure, allowing queries to adaptively select resolution based on the sampling area size [S1]. Unlike [TriPlane NeRF](6932-triplane-nerf.md), which uses static feature planes, Tri-Mip Encoding uses mipmaps to explicitly model pre-filtered features, ensuring coherence across different viewing distances and resolutions [S1, 52-name.md].

## How it works
**Inputs:**
- Sphere center position $\position \in \mathbb{R}^3$ and radius $\radius \in \mathbb{R}^+$ [S1].
- Three orthogonal mipmaps $\mipmap = \{\mipmap _{XY}, \mipmap _{XZ},\mipmap _{YZ}\}$ [S1].
- Axis Aligned Bounding Box (AABB) of the scene defined by minimum corner $\aabb _{min}$ and maximum corner $\aabb _{max}$ [S1].
- Mipmap base dimensions $H, W$ and channel count $C$ [S1].

**Outputs:**
- Concatenated feature vector $\feat \in \mathbb{R}^{3C}$ [S1].

**Data Structures:**
- **Mipmap Level $M^L_i$**: A 2D tensor of shape $H_i \times W_i \times C$.
  - Base level $M^L_0$: Shape $H \times W \times C$, randomly initialized and trainable [S1].
  - Higher levels $M^L_i$ ($i=1..N$): Derived by downscaling $M^L_{i-1}$ by 2x along height and width [S1].
  - Storage: Only base levels $M^L_0$ are stored explicitly; higher levels are generated on-the-fly or via standard GPU texture filtering [S1].

**Parameters:**
- $\featradius$: Radius of feature elements in the base level, calculated as:
  $$ \featradius = \sqrt {\frac {(\aabb _{max}-\aabb _{min})_X \cdot (\aabb _{max}-\aabb _{min})_Y}{HW\cdot \pi }} $$
  [S1]
- Typical values for $H, W$: Not given in the sources.
- Typical values for $C$: Not given in the sources.

**Steps:**
1. **Decomposition**: Decompose the 3D space into three orthogonal planes: planeXY, planeXZ, and planeYZ [S1].
2. **Projection**: Orthogonally project the sampled sphere $S(\position, \radius)$ onto the three planes to obtain three discs $D = \{D_{XY}, D_{XZ}, D_{YZ}\}$ [S1].
3. **Disc Geometry**: For each disc $D_{XY}$, the 2D center coordinate is the partial coordinate $(x, y)$ of the sphere's center $\position$, and the radius is $\radius$ [S1].
4. **Level Assignment**: Calculate the query level $l$ for each mipmap to match the sphere's radius with the feature element radius:
   $$ l = log_2\bigg (\frac {\radius }{\featradius }\bigg ) $$
   [S1]
5. **Feature Query**: For each plane (e.g., XY):
   a. Identify the two nearest integer levels $\lfloor l \rfloor$ and $\lceil l \rceil$ [S1].
   b. Project the disc center $(x, y)$ to these two levels [S1].
   c. Find the four nearest neighbors (corners) on each level (8 total neighbors) [S1].
   d. Perform trilinear interpolation on the 8 neighbors based on their distance to the disc center to produce the feature vector $f_{XY}$ [S1].
6. **Concatenation**: Concatenate the feature vectors from the three planes $\{f_{XY}, f_{XZ}, f_{YZ}\}$ to form the final feature vector $\feat$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trimip_encoding(position, radius, mipmap_base, aabb_min, aabb_max, H, W, C):
    """
    Query Tri-Mip encoding for a sphere defined by position and radius.
    
    Args:
        position: Tuple (x, y, z)
        radius: Float
        mipmap_base: Tensor of shape (H, W, C) for each of the 3 planes
        aabb_min: Tuple (xmin, ymin, zmin)
        aabb_max: Tuple (xmax, ymax, zmax)
        H, W, C: Integers for base mipmap dimensions
        
    Returns:
        feature_vector: Tensor of shape (3*C,)
    """
    x, y, z = position
    
    # Calculate feature radius for base level
    dx = aabb_max[0] - aabb_min[0]
    dy = aabb_max[1] - aabb_min[1]
    featradius = math.sqrt((dx * dy) / (H * W * math.pi))
    
    # Calculate query level
    l = math.log2(radius / featradius)
    
    # Determine integer levels for interpolation
    l_floor = math.floor(l)
    l_ceil = math.ceil(l)
    
    # Ensure levels are within valid range [0, max_level]
    # Note: Max level depends on H, W. Assuming infinite levels or clamped.
    # In practice, levels beyond log2(min(H,W)) are constant or zero.
    
    features = []
    planes = [
        ('XY', 0, 1), # indices for x, y in position
        ('XZ', 0, 2),
        ('YZ', 1, 2)
    ]
    
    for plane_name, idx_u, idx_v in planes:
        # Get the base mipmap for this plane
        M_base = mipmap_base[plane_name]
        
        # Project center to 2D coordinates
        u, v = position[idx_u], position[idx_v]
        
        # Query feature from mipmap using trilinear interpolation
        f_plane = query_mipmap_trilinear(M_base, u, v, l_floor, l_ceil, H, W, C)
        features.append(f_plane)
        
    # Concatenate features
    final_feature = concatenate(features, axis=-1)
    return final_feature

def query_mipmap_trilinear(M_base, u, v, l_floor, l_ceil, H, W, C):
    """
    Perform trilinear interpolation on a mipmap structure.
    M_base is the base level (H, W, C). Higher levels are derived by 2x downsampling.
    """
    # Helper to get level dimensions
    def get_level_dims(level):
        h = H / (2 ** level)
        w = W / (2 ** level)
        return h, w
    
    # Helper to get level tensor (simulated downsampling)
    def get_level_tensor(level):
        # In implementation, this would be pre-computed or computed via pooling
        # For pseudocode, assume a function that returns the tensor for level
        return downsample(M_base, level)
        
    # Get tensors for floor and ceil levels
    # Clamp levels to valid range if necessary
    h_floor, w_floor = get_level_dims(l_floor)
    h_ceil, w_ceil = get_level_dims(l_ceil)
    
    M_floor = get_level_tensor(l_floor)
    M_ceil = get_level_tensor(l_ceil)
    
    # Normalize u, v to [0, 1] for texture coordinates
    u_norm = u / (aabb_max[0] - aabb_min[0]) # Assuming AABB maps to [0,1] or similar
    v_norm = v / (aabb_max[1] - aabb_min[1])
    
    # Note: The source describes projecting center and finding neighbors.
    # Standard trilinear interpolation on mipmaps involves:
    # 1. Sampling 4 texels on floor level
    # 2. Sampling 4 texels on ceil level
    # 3. Interpolating between them
    
    # Get 4 neighbors on floor level
    # Coordinates in texel space
    u_floor = u_norm * w_floor
    v_floor = v_norm * h_floor
    
    # Get 4 neighbors on ceil level
    u_ceil = u_norm * w_ceil
    v_ceil = v_norm * h_ceil
    
    # Sample 4 values from floor level
    f00_floor = sample_texel(M_floor, u_floor, v_floor)
    f10_floor = sample_texel(M_floor, u_floor + 1, v_floor)
    f01_floor = sample_texel(M_floor, u_floor, v_floor + 1)
    f11_floor = sample_texel(M_floor, u_floor + 1, v_floor + 1)
    
    # Sample 4 values from ceil level
    f00_ceil = sample_texel(M_ceil, u_ceil, v_ceil)
    f10_ceil = sample_texel(M_ceil, u_ceil + 1, v_ceil)
    f01_ceil = sample_texel(M_ceil, u_ceil, v_ceil + 1)
    f11_ceil = sample_texel(M_ceil, u_ceil + 1, v_ceil + 1)
    
    # Interpolate on floor level (bilinear)
    f_floor = bilinear_interpolate(f00_floor, f10_floor, f01_floor, f11_floor, u_floor, v_floor)
    
    # Interpolate on ceil level (bilinear)
    f_ceil = bilinear_interpolate(f00_ceil, f10_ceil, f01_ceil, f11_ceil, u_ceil, v_ceil)
    
    # Interpolate between levels (linear)
    # Weight based on distance to l_floor
    weight = l - l_floor
    f_plane = lerp(f_floor, f_ceil, weight)
    
    return f_plane

def sample_texel(tensor, u, v):
    # Standard bilinear sampling with boundary checks
    # Returns tensor value at (u, v)
    pass

def bilinear_interpolate(f00, f10, f01, f11, u, v):
    # f00 at (0,0), f10 at (1,0), f01 at (0,1), f11 at (1,1)
    # u, v are fractional parts
    top = lerp(f00, f10, u)
    bottom = lerp(f01, f11, u)
    return lerp(top, bottom, v)

def lerp(a, b, t):
    return a + t * (b - a)

def downsample(tensor, level):
    # Apply 2x average pooling 'level' times
    current = tensor
    for _ in range(level):
        current = average_pool_2x(current)
    return current

def average_pool_2x(tensor):
    # Return average of 2x2 blocks
    pass
```

**Edge Cases and Pitfalls:**
- **Precision**: Trilinear interpolation increases effective precision and yields a continuous encoding space, beneficial for training [S1].
- **Boundaries**: The AABB defines the spatial extent. Queries outside the AABB may require clamping or handling undefined behavior, though not explicitly detailed in the sources [S1].
- **Degenerate Input**: Radius $\radius = 0$ leads to $l \to -\infty$, which is invalid. A minimum radius or clamping to the base level is implied but not specified [S1].

## Subtopics

### TriPlane NeRF
Aligns explicit features along three axis-aligned orthogonal feature planes, queried by projecting 3D positions onto these planes. Tri-Mip Encoding builds on this by adding mipmap structures for area sampling [52-name.md].

### MultiPlaneGAN
Uses multiple planes for 3D content generation. Differs in that it does not explicitly model pre-filtered features for anti-aliasing via mipmaps [7441-multiplanegan.md].

### Platonic Solid Projection
Projects 3D space onto platonic solids. Differs in the projection geometry and lack of mipmap-based pre-filtering [8518-platonic-solid-projection.md].

### Spatial Planes
Uses spatially distributed planes. Differs in the orthogonal axis-aligned structure and mipmap representation [9808-spatial-planes.md].

## Operating characteristics
- **Time Complexity**: Querying the mipmap is efficient, leveraging highly optimized GPU texture filtering [S1]. The level assignment involves a logarithm and square root, which are constant time operations [S1].
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
