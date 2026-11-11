# Mipmap Grid Query

Also called: Grid Value Query with Mip-Mapping

## Overview
Mipmap Grid Query retrieves scalar values from a multi-resolution grid hierarchy by selecting a mip level based on the spatial distance of the query point from the center of the base bounding box. It solves the problem of aliasing and resolution mismatch in dense volume rendering by automatically downscaling the query resolution as the point moves further from the center, assuming a geometric scaling relationship between mip levels. It differs from standard trilinear interpolation by dynamically adjusting the grid resolution used for the lookup based on spatial context rather than a fixed user-defined level.

## How it works
**Inputs and Outputs:**
- Input `x`: A tensor of shape $[N, 3]$ containing $N$ query points in world coordinates.
- Input `data`: A tensor of shape $[m, res_x, res_y, res_z]$ containing the grid values for $m$ mip levels. The base level (index 0) has resolution $[res_x, res_y, res_z]$. Subsequent levels are assumed to be scaled by factors of 2.
- Input `base_aabb`: A tensor of shape $[6]$ representing the axis-aligned bounding box of the base level grid, formatted as $[min_x, min_y, min_z, max_x, max_y, max_z]$.
- Output: A tensor of shape $[N]$ containing the interpolated values at the queried points.

**Data Structures:**
The method assumes the `data` tensor is organized such that the first dimension indexes the mip level. The spatial dimensions correspond to the resolution of that specific mip level. The relationship between mip level $l$ and resolution is typically $res_{x,l} = res_x / 2^l$, though the implementation handles this via coordinate scaling.

**Algorithm Steps:**
1.  **Normalize Coordinates:** Transform the query points `x` from world coordinates to the normalized range $[0, 1]$ relative to the `base_aabb`.
    $$ x_{norm} = \frac{x - aabb_{min}}{aabb_{max} - aabb_{min}} $$
    where $aabb_{min} = [min_x, min_y, min_z]$ and $aabb_{max} = [max_x, max_y, max_z]$.

2.  **Compute Distance Metric:** Calculate the maximum absolute deviation of the normalized coordinates from the center $(0.5, 0.5, 0.5)$. This metric determines the required mip level.
    $$ maxval = \max(|x_{norm} - 0.5|) $$
    Clamp $maxval$ to a minimum value of $0.1$ to prevent numerical instability in the subsequent logarithmic operation (specifically, to avoid an exponent of 0 in `frexp`).

3.  **Determine Mip Level:** Compute the mip level index $l$ using the exponent of the floating-point representation of $maxval$.
    $$ l = \lfloor \log_2(maxval) \rfloor + 1 $$
    In practice, this is often implemented using `frexp` or `frexp`-like operations:
    $$ l = \text{exponent}(maxval) + 1 $$
    Clamp $l$ to a minimum of $0$.

4.  **Re-normalize for Selected Mip:** Adjust the normalized coordinates to align with the resolution of the selected mip level $l$. The coordinates are centered around 0.5 and scaled by $2^l$.
    $$ scale = 2^l $$
    $$ x_{unit} = \frac{x_{norm} - 0.5}{scale} + 0.5 $$
    This maps the query point into the coordinate space of the lower-resolution grid.

5.  **Map to Grid Indices:** Convert the re-normalized coordinates $x_{unit}$ to integer grid indices $i_x, i_y, i_z$ using the resolution of the selected mip level.
    $$ resolution_l = \text{shape}(data)[1:] / 2^l $$
    $$ i = \lfloor x_{unit} \cdot resolution_l \rfloor $$
    Clamp the indices $i$ to the valid range $[0, resolution_l - 1]$.

6.  **Query and Return:** Retrieve the value from the `data` tensor at the computed mip level and indices. If the computed mip level $l$ exceeds the number of available levels $m$, the query is invalid. A selector mask is used to handle this.
    $$ selector = (l < m) $$
    $$ result = data[l, i_x, i_y, i_z] \cdot selector $$
    If the selector is false (out of bounds), the result is typically 0 or undefined depending on the specific implementation's handling of invalid queries.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def mipmap_grid_query(x, data, base_aabb):
    """
    Query values from a mipmap grid hierarchy.
    
    Args:
        x: Tensor of shape [N, 3], query points.
        data: Tensor of shape [m, res_x, res_y, res_z], grid values.
        base_aabb: Tensor of shape [6], [min_x, min_y, min_z, max_x, max_y, max_z].
        
    Returns:
        Tensor of shape [N], queried values.
    """
    # 1. Normalize coordinates to [0, 1]
    aabb_min = base_aabb[:3]
    aabb_max = base_aabb[3:]
    x_norm = (x - aabb_min) / (aabb_max - aabb_min)
    
    # 2. Compute maxval: max absolute distance from center (0.5)
    maxval = torch.max(torch.abs(x_norm - 0.5), dim=1).values
    
    # Clamp to avoid frexp exponent 0
    maxval = torch.clamp(maxval, min=0.1)
    
    # 3. Compute mip level using frexp exponent + 1, clamped to min 0
    # torch.frexp returns (mantissa, exponent)
    mantissa, exponent = torch.frexp(maxval)
    mip = exponent + 1
    mip = torch.clamp(mip, min=0)
    
    # 4. Create selector for valid mips
    num_mips = data.shape[0]
    selector = (mip < num_mips)
    
    # Clamp mip to max available index for indexing safety, 
    # though values will be masked by selector later
    mip_clamped = torch.clamp(mip, max=num_mips - 1)
    
    # 5. Re-normalize points to [0, 1] using scale factor 2^mip
    scale = 2.0 ** mip_clamped.unsqueeze(1) # Shape [N, 1]
    x_unit = (x_norm - 0.5) / scale + 0.5
    
    # 6. Map normalized points to grid indices
    # Base resolution
    base_res = torch.tensor(data.shape[1:], device=x.device, dtype=torch.float32)
    # Resolution at current mip level
    res_current = base_res / scale.squeeze(1) # Shape [N, 3]
    
    ix = (x_unit * res_current).long()
    
    # Clamp indices to valid range [0, res - 1]
    ix = torch.clamp(ix, min=0, max=res_current.long() - 1)
    
    # 7. Query values
    # Gather values from data[mip, ix_x, ix_y, ix_z]
    # Note: In PyTorch, advanced indexing with a batch of indices requires careful handling.
    # Assuming data is indexed as [mip, x, y, z]
    
    # Extract individual indices
    ix_x = ix[:, 0]
    ix_y = ix[:, 1]
    ix_z = ix[:, 2]
    
    # Gather values
    values = data[mip_clamped, ix_x, ix_y, ix_z]
    
    # Apply selector
    result = values * selector.float()
    
    return result
```

**Parameters:**
- `x`: Tensor of shape $[N, 3]$.
- `data`: Tensor of shape $[m, res_x, res_y, res_z]$.
- `base_aabb`: Tensor of shape $[6]$.

**Edge Cases and Pitfalls:**
- **Precision:** The use of `frexp` and floating-point division can introduce precision errors, especially for points very close to the boundaries or the center. Clamping `maxval` to $0.1$ prevents `frexp` from returning an exponent that would lead to an invalid mip level calculation for points near the center.
- **Boundaries:** Points outside the `base_aabb` will have normalized coordinates outside $[0, 1]$. The re-normalization step may map these to invalid grid indices. Clamping indices to $[0, res-1]$ handles this, but the semantic meaning of the query is undefined for out-of-bounds points.
- **Degenerate Input:** If `base_aabb` has zero volume (min equals max), division by zero occurs. This must be checked and handled before normalization.

## Subtopics

### Ripmap Encoding with Tetra-Linear Interpolation
A 4D interpolation method used in Rip-NeRF to query feature values from Ripmaps (R) based on position $(p_x, p_y)$ and level $(l_x, l_y)$. It uses tetra-linear interpolation over 16 neighboring vertices in the 4D space. Given a query point $(p_x, p_y, l_x, l_y)$, the 16 neighboring vertices are identified with binary indices $i, j, k, m \in \{0, 1\}$. The feature value $f$ is computed as a weighted sum: $f = \sum_{i=0}^1 \sum_{j=0}^1 \sum_{k=0}^1 \sum_{m=0}^1 w_{ijklm} f_{ijklm}$, where $f_{ijklm} = R_{l_x^k, l_y^m}(p_x^i, p_y^j)$ is the feature value at the neighboring vertex, and $w_{ijklm} = \prod_{d \in \{x,y\}} \prod_{t \in \{p,l\}} (1 - |t_d - t_{d,i,j}|)$ is the interpolation weight. The Ripmap R has a shape of $512 \times 512 \times 16$, corresponding to 10 unparalleled icosahedron faces. This method allows for efficient querying while considering the anisotropic nature of the Gaussian footprint [S3].

## Operating characteristics
- **Time Complexity:** $O(N)$ for $N$ query points, with constant overhead per point for coordinate transformations and indexing.
- **Memory Complexity:** $O(m \cdot res_x \cdot res_y \cdot res_z)$ to store the mipmap grid.
- **Performance:** Efficient for large-scale volume rendering as it reduces the number of active voxels queried for distant points. The use of integer indexing and simple arithmetic operations makes it fast on GPU hardware.
- **Strengths:** Automatic level-of-detail selection based on spatial context; reduces aliasing; simple implementation.
- **Limitations:** Assumes a strict 2x scaling relationship between mip levels; may not be optimal for scenes with non-uniform detail distribution; boundary handling requires careful clamping.
- **When to use:** Dense volume rendering with hierarchical grid data where spatial coherence is high and detail varies with distance from the center.
- **When not to use:** Scenes with highly anisotropic or non-uniform detail distribution that does not follow the 2x scaling assumption; when precise control over the level of detail is required independent of spatial position.

## Background
Mipmap Grid Query is a technique used in modern neural radiance field (NeRF) implementations, particularly those involving dense grid representations and anti-aliasing. It is closely related to the mipmapping strategies used in computer graphics to manage texture resolution. The specific implementation described here is found in the `nerfacc` library [S1, unofficial] and the `CNC` repository [S2, unofficial]. The concept of using distance-based mip selection is also adapted in Rip-NeRF for Ripmap encoding [S3].

## Sources
[S1] nerfstudio-project/nerfacc, https://github.com/nerfstudio-project/nerfacc, fetched 2026-10-26 (unofficial)
[S2] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
[S3] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
