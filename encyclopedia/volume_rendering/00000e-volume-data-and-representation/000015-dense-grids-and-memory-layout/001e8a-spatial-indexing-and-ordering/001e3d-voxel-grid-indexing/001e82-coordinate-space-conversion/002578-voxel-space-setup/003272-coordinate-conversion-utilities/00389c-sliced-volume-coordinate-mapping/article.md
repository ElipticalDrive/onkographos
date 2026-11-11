# Sliced Volume Coordinate Mapping

Also called: 2D Texture Volume Representation, 3D to 2D Tiled Texture Coordinate Conversion, Flattened 3D Volume to 2D Texture Mapping, Slice Stacking Texture Layout, XYZ to XY Tile Mapping

## Overview
Sliced Volume Coordinate Mapping converts 3D voxel coordinates into 2D texture coordinates for volumetric data stored as a tiled grid of 2D slices within a single 2D image [S2, unofficial]. This technique enables GPU texture sampling of 3D data using standard 2D texture units, which is necessary when hardware 3D texture support is unavailable or inefficient [S1, unofficial]. It differs from standard 3D texture sampling by flattening the volume into a 2D grid of slices, requiring specific coordinate mapping logic to resolve 3D coordinates to 2D image pixels [S2, unofficial].

## How it works
**Inputs:**
- 3D voxel coordinates $(x, y, z)$, typically normalized or integer-based depending on the implementation.
- Volume dimensions: Width $W$, Height $H$ (per slice), and Depth $D$ (number of slices).
- Tiling parameters: Number of tile columns $C$ and tile rows $R$ required to arrange $D$ slices in the 2D texture. $C = \lceil \sqrt{D} \rceil$, $R = \lceil D / C \rceil$.
- Texture dimensions: Total Width $W_{tex} = C \times W$, Total Height $H_{tex} = R \times H$.

**Outputs:**
- 2D texture coordinates $(u, v)$ for the primary slice.
- 2D texture coordinates $(u', v')$ for the adjacent slice (for interpolation).
- Interpolation weight $w$ between the two slices.

**Data Structures:**
The volume is stored in a row-major 2D image. Each slice occupies a $W \times H$ block.
- Slice $k$ (where $0 \le k < D$) is mapped to tile position $(t_x, t_y)$ where:
  - $t_x = k \bmod C$
  - $t_y = \lfloor k / C \rfloor$
- The pixel offset within the texture for slice $k$ is $(t_x \cdot W, t_y \cdot H)$.

**Steps:**

1. **Normalize and Clamp Input:**
   Ensure $z$ is within valid bounds $[0, D-1]$. If $z$ is continuous (floating point), it represents a position between slices.
   $$ z_{clamped} = \min(D-1, \max(0, z)) $$
   [S2, unofficial] uses a specific clamping formula: `_slice = min(1, max(_z, 0)) * ZSlices`. This implies `_z` might be normalized to $[0,1]$ or needs scaling. We assume $z$ is in voxel units $[0, D-1]$ for the general case, but the source [S2, unofficial] suggests a normalized approach. Let's stick to the integer slice index logic derived from [S2, unofficial] which handles continuous $z$.

2. **Compute Continuous Slice Index:**
   Calculate the continuous slice index $\sigma$ from the $z$ coordinate.
   $$ \sigma = z $$
   [S2, unofficial] defines `_slice` as the continuous slice index.

3. **Determine Integer Slice Indices:**
   Identify the two adjacent slices for interpolation.
   $$ s_1 = \lfloor \sigma \rfloor $$
   $$ s_2 = s_1 + 1 $$
   Clamp $s_2$ to $D-1$ if it exceeds the volume depth.

4. **Calculate Tile Positions:**
   For each slice $s_i$ ($i \in \{1, 2\}$), calculate its tile coordinates $(tx_i, ty_i)$ in the 2D texture grid.
   $$ tx_i = s_i \bmod C $$
   $$ ty_i = \lfloor s_i / C \rfloor $$
   To prevent rounding crossover issues at tile boundaries, add a small epsilon $\epsilon$ (e.g., 0.001) to the row calculation [S2, unofficial]:
   $$ ty_i = \lfloor s_i / C + \epsilon \rfloor $$

5. **Compute Normalized UV Coordinates:**
   Map the 3D coordinates $(x, y)$ to normalized texture coordinates $(u, v)$ for each slice.
   First, clamp $x$ and $y$ to $[0, W-1]$ and $[0, H-1]$ respectively, or normalize them to $[0, 1]$ if the texture coordinates are expected to be in $[0, 1]$. The source [S2, unofficial] outputs normalized coordinates.
   
   Let $clip(v)$ be the clamping function for coordinate $v$ to the valid range $[0, \text{size}-1]$.
   
   For slice 1:
   $$ u_1 = \frac{clip(x) + tx_1}{C} $$
   $$ v_1 = \frac{clip(y) + ty_1}{R} $$
   
   For slice 2:
   $$ u_2 = \frac{clip(x) + tx_2}{C} $$
   $$ v_2 = \frac{clip(y) + ty_2}{R} $$
   
   Note: The source [S2, unofficial] divides by `TileColumns` ($C$) and `NumRows` ($R$). This assumes the final output coordinates are normalized to $[0, 1]$. If integer pixel coordinates are needed, multiply by $W_{tex}$ and $H_{tex}$.

6. **Calculate Interpolation Weight:**
   The weight for linear interpolation between slice $s_1$ and $s_2$ is the fractional part of $\sigma$.
   $$ w = \sigma - s_1 $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sliced_volume_mapping(x, y, z, W, H, D, C, R):
    """
    Maps 3D voxel coordinates to 2D texture coordinates for a sliced volume.
    
    Args:
        x, y, z: Input 3D coordinates (float or int).
        W, H: Width and height of a single slice.
        D: Total number of slices (depth).
        C: Number of tile columns in the 2D texture.
        R: Number of tile rows in the 2D texture.
        
    Returns:
        u1, v1: Normalized UV coordinates for slice 1.
        u2, v2: Normalized UV coordinates for slice 2.
        w: Interpolation weight between slice 1 and slice 2.
    """
    # Step 1: Clamp z to valid range [0, D-1]
    z_clamped = max(0.0, min(float(D - 1), z))
    
    # Step 2: Continuous slice index
    sigma = z_clamped
    
    # Step 3: Integer slice indices
    s1 = int(sigma)
    s2 = s1 + 1
    if s2 >= D:
        s2 = D - 1
        
    # Step 4: Tile positions with epsilon to avoid rounding crossover
    epsilon = 0.001
    tx1 = s1 % C
    ty1 = int(s1 / C + epsilon)
    
    tx2 = s2 % C
    ty2 = int(s2 / C + epsilon)
    
    # Step 5: Normalize x and y to [0, W-1] and [0, H-1] for pixel indexing
    # Assuming x, y are in voxel space [0, W-1], [0, H-1]
    x_clamped = max(0.0, min(float(W - 1), x))
    y_clamped = max(0.0, min(float(H - 1), y))
    
    # Step 6: Compute normalized UV coordinates
    # Output is normalized to [0, 1] based on source [S2, unofficial]
    u1 = (x_clamped + tx1) / C
    v1 = (y_clamped + ty1) / R
    
    u2 = (x_clamped + tx2) / C
    v2 = (y_clamped + ty2) / R
    
    # Step 7: Interpolation weight
    w = sigma - s1
    
    return u1, v1, u2, v2, w
```

**Edge Cases and Pitfalls:**
- **Slice Count Dependency:** The mapping assumes a fixed number of slices $D$ and tiling configuration $(C, R)$. Changing the slice count without updating $C$ and $R$ breaks the mapping logic [S1, unofficial].
- **Filtering Artifacts:** If bilinear or trilinear filtering is used, the texture sampler may interpolate across slice boundaries, blending data from adjacent slices that are spatially distant in the 3D volume but adjacent in the 2D texture [S1, unofficial]. Using nearest-neighbor filtering avoids this but causes banding artifacts [S2, unofficial]. Linear interpolation between the two calculated slices ($s_1$ and $s_2$) reduces banding [S2, unofficial].
- **Rounding Crossover:** At tile boundaries, floating-point precision issues can cause incorrect tile selection. Adding a small epsilon (0.001) to the row calculation prevents this [S2, unofficial].
- **Out-of-Bounds Access:** The $z$ coordinate must be clamped to prevent out-of-bounds access when calculating slice indices [S2, unofficial].

## Subtopics

### [Voxel-to-Image Slicing (Tiled EXR)](8289-voxel-to-image-slicing-tiled-exr.md)
A specific implementation for rendering 3D voxel data in raster-based renderers like Cycles by converting volumetric data into a 2D tiled EXR image. It uses linear interpolation between adjacent slices to reduce banding artifacts, differing from nearest-neighbor sampling.

## Operating characteristics
- **Time Complexity:** $O(1)$ per voxel/sample. The mapping involves a constant number of arithmetic operations (modulo, floor, division, addition).
- **Memory Complexity:** $O(W \times H \times D)$ for the volume data, but stored as a 2D texture of size $W_{tex} \times H_{tex}$. The overhead is minimal, primarily the tiling structure.
- **Performance:** Enables efficient GPU sampling using 2D texture units, which are widely supported and optimized. Avoids the need for 3D texture hardware support, which may be slower or unavailable on some GPUs [S1, unofficial].
- **Strengths:** Compatible with standard 2D texture pipelines; allows use of 2D image formats (like EXR) for volumetric data; supports linear interpolation for smoother rendering [S2, unofficial].
- **Limitations:** Requires careful handling of filtering to avoid cross-slice blending artifacts; tiling configuration must match the volume dimensions; not suitable for dynamic volumes with changing depth without remapping [S1, unofficial].

## Background
The technique is used in computer graphics to render volumetric data on hardware that lacks efficient 3D texture support or when integrating volumetric data into 2D-based rendering pipelines. It is notably used in the Blender Cycles renderer for handling voxel data [S2, unofficial]. The approach of stacking slices into a 2D texture is a common workaround for 3D texture limitations [S1, unofficial].

## Sources
[S1] JINKEJIN/CGprject-VolumeRendering, https://github.com/JINKEJIN/CGprject-VolumeRendering, fetched 2026-10-26 (unofficial)
[S2] Voxel data in Cycles, https://blender.stackexchange.com/questions/14995, fetched 2026-10-30 (unofficial)
