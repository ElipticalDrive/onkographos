# Mipmap Filtering

Also called: Mipmap Interpolation, Texture Filtering, Trilinear/Quadrilinear Filtering

## Overview
Mipmap Filtering is a pre-filtering technique that avoids aliasing artifacts when a projected texture's resolution exceeds the sample resolution, and mitigates block-like discontinuities when the projected resolution is lower than the target resolution [S1, unofficial]. It sets itself apart from its siblings by explicitly managing the transition between different levels of detail (LOD) in the mip-chain during the sampling phase, whereas [Atlas Mipmapping](2639-atlas-mipmapping.md) focuses on storage layout, [Voxel Pre-integration](2956-voxel-pre-integration.md) on radiative transfer approximation, and [GPU 3D Mip Chain Generation](5067-gpu-3d-mip-chain-generation.md) on the creation process itself [G].

## How it works
**Inputs and Outputs:**
* Input: A mip-map hierarchy (a set of downsampled grids/volumes) and a sampling point $p$ in texture space. [G]
* Output: A single interpolated value (color, density, etc.) at $p$. [G]

**Data Structures:**
The method operates on a mip-map hierarchy. Each level $l$ contains a grid of texels/voxels. The layout is standard for each level (e.g., 2D array for textures, 3D array for volumes). The hierarchy is indexed by level $l \in [0, L_{max}]$, where $l=0$ is the highest resolution (original) and $l=L_{max}$ is the lowest. [G]

**Steps:**

1. **Determine LOD:** Calculate the appropriate mip level $l$ based on the projected resolution of the sampling point relative to the texture resolution. The goal is to select the texture level that best matches the target resolution after projection [S1, unofficial].
2. **Minification (Downsampling):** If the projected texture resolution exceeds the sample resolution (minification), select the two suitable mip levels that bracket the ideal resolution. Interpolate between these two levels using a first-degree polynomial (linear interpolation) to eliminate discontinuities between adjacent texels sampled from different mipmaps [S1, unofficial]. This is referred to as a linear filter for minification [S1, unofficial].
3. **Magnification (Upsampling):** If the projected texture resolution is lower than the target resolution (magnification), sample the adjacent texels relative to the sampling point $p$. Interpolate the values with respect to the distance from $p$ and the texel centers. This is a bi-linear filter for 2D textures [S1, unofficial].
4. **Combination (Trilinear/Quadlinear):**
    * For 2D textures: Combine the minification (linear interpolation between levels) and magnification (bi-linear interpolation within a level) to form a **tri-linear filter** [S1, unofficial].
    * For 3D volumes: Apply a tri-linear filter for magnification (interpolation within the voxel) and a linear filter for minification (interpolation between mip levels) to form a **quad-linear filter** [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def mipmap_filter_3d(volume_hierarchy, p, lod_bias=0.0):
    """
    Performs quad-linear filtering on a 3D volume mip-chain.
    
    Args:
        volume_hierarchy: List of 3D arrays, where hierarchy[0] is the highest resolution.
        p: Tuple (x, y, z) representing the sampling point in texture coordinates.
        lod_bias: Optional bias to adjust the selected LOD.
        
    Returns:
        Interpolated value at p.
    """
    # 1. Determine LOD
    # LOD calculation depends on projection context (e.g., ray marching step size or screen-space derivatives) [G].
    lod = 0.0  # Placeholder for context-dependent LOD calculation [G]
    lod = lod + lod_bias
    
    # Clamp LOD to valid range
    lod = max(0.0, min(float(len(volume_hierarchy) - 1), lod))
    
    # 2. Select Mip Levels
    lod_floor = int(math.floor(lod))
    lod_ceil = lod_floor + 1
    
    # Handle edge case where lod is exactly at the max level
    if lod_ceil >= len(volume_hierarchy):
        lod_ceil = len(volume_hierarchy) - 1
        
    level_low = volume_hierarchy[lod_floor]
    level_high = volume_hierarchy[lod_ceil]
    
    # 3. Magnification: Trilinear Interpolation within each level
    # Interpolate within the low-res level
    val_low = trilinear_interpolate(level_low, p)
    # Interpolate within the high-res level
    val_high = trilinear_interpolate(level_high, p)
    
    # 4. Minification: Linear Interpolation between levels
    # Interpolate between the two levels based on the fractional part of lod
    frac = lod - lod_floor
    result = lerp(val_low, val_high, frac)
    
    return result

def trilinear_interpolate(grid, p):
    """
    Performs trilinear interpolation on a 3D grid.
    """
    x, y, z = p
    
    # Get integer coordinates
    ix = int(math.floor(x))
    iy = int(math.floor(y))
    iz = int(math.floor(z))
    
    # Get fractional coordinates
    fx = x - ix
    fy = y - iy
    fz = z - iz
    
    # Sample the 8 surrounding voxels
    # Boundary checks should be implemented here [G]
    v000 = grid[ix, iy, iz]
    v100 = grid[ix+1, iy, iz]
    v010 = grid[ix, iy+1, iz]
    v110 = grid[ix+1, iy+1, iz]
    v001 = grid[ix, iy, iz+1]
    v101 = grid[ix+1, iy, iz+1]
    v011 = grid[ix, iy+1, iz+1]
    v111 = grid[ix+1, iy+1, iz+1]
    
    # Interpolate along x
    v00 = lerp(v000, v100, fx)
    v01 = lerp(v010, v110, fx)
    v10 = lerp(v001, v101, fx)
    v11 = lerp(v011, v111, fx)
    
    # Interpolate along y
    v0 = lerp(v00, v01, fy)
    v1 = lerp(v10, v11, fy)
    
    # Interpolate along z
    return lerp(v0, v1, fz)

def lerp(a, b, t):
    return a + t * (b - a)
```

**Parameters:**
* `lod_bias`: A value added to the calculated LOD to shift the sampling point up or down the mip chain. Not given in the sources for general volume rendering. [G]
* `lod`: The floating-point level of detail index. [G]

**Edge Cases and Pitfalls:**
* **Discontinuities:** Without interpolation between mip levels, visible seams appear when adjacent texels are sampled from different mipmaps [S1, unofficial].
* **Block-like Artifacts:** Without magnification filtering, block-like discontinuities appear when the projected resolution is lower than the target resolution [S1, unofficial].
* **Precision:** Standard floating-point arithmetic is assumed for interpolation weights. [G]

## Subtopics

### Stochastic Mip Filtering
A feature that samples the two closest mip levels stochastically, similar to Virtual Textures optimizations, to effectively cut the number of texture samples in half and improve performance [S2]. Without this feature, a scene with five 'Aerial Explosion' instances streaming 2GB/s IO Bandwidth would need approximately 25GB/s without the bandwidth limiter and without scaling according to projected screen space size [S2].

## Operating characteristics
Not given in the sources.

## Background
Mipmap filtering was introduced as a solution to texture aliasing in computer graphics, based on the Nyquist–Shannon sampling theorem [S1, unofficial]. The concept has been extended to 3D volumes and sparse voxel structures, such as Sparse Volume Textures (SVT) in Unreal Engine [S2].

## Sources
[S1] PDF On sparse voxel DAGs and memory efficient compression of surface ..., https://research.chalmers.se/publication/528795/file/528795_Fulltext.pdf, fetched 2026-10-27 (unofficial)
[S2] Sparse Volume Textures in Unreal Engine | Unreal Engine 5.8 ... - Epic Dev, https://dev.epicgames.com/documentation/en-us/unreal-engine/sparse-volume-textures-in-unreal-engine, fetched 2026-10-28
