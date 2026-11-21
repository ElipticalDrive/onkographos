# Seamless Interpolation

Also called: TileTree seamless interpolation, n-leaf interpolation

## Overview
Seamless Interpolation is a method for rendering smooth surfaces stored in hierarchical tile-based structures (TileTrees) that eliminates pixelated appearance and interpolation seams at tile boundaries. It solves the problem of discontinuous color transitions when sampling from partially covered faces or n-leaves by computing a weighted sum of contributions from up to three adjacent face tiles ($T_x, T_y, T_z$) based on the surface normal. It differs from standard bilinear interpolation by explicitly handling multi-face visibility via binary alpha flags and normal-based dampening to ensure continuity across adaptive resolution boundaries and partial coverage areas.

## How it works
**Inputs and Outputs:**
- Input: A point $P$ in 3D space, the surface normal $\mathbf{n}$ at $P$, and the TileTree data structure containing tiles $T_x, T_y, T_z$.
- Output: A final color $c$ at $P$.

**Data Structures:**
- **Tile:** A block of texture data. Each sample within a tile stores:
  - Color $C$ (RGB).
  - Alpha flag $\alpha$ (binary or interpolated float): Indicates if the sample corresponds to the enclosed surface. Stored in the alpha channel.
- **TileTree:** Hierarchical structure of tiles. Tiles may have different resolutions (adaptive resolution). Neighboring tiles share a one-pixel border replication.

**Steps:**

1. **Identify Contributing Tiles:**
   Determine the three tiles $T_x, T_y, T_z$ corresponding to the faces aligned with the surface normal $\mathbf{n}$. At most three faces contribute, selected by the dominant components of $\mathbf{n}$.

2. **Fetch Samples and Flags:**
   For each contributing tile $T_i$ (where $i \in \{x, y, z\}$), fetch the color $c_i$ and the interpolated alpha flag $\alpha_i$ at the projection of $P$ onto the tile face.
   - If a sample is undefined (e.g., in boundary-leaves or n-leaves interior where the surface does not cover the entire tile), fill the undefined color by reading from the neighboring leaf tile [S1].
   - The flag $\alpha_i$ is interpolated using hardware bilinear filtering. It equals 1 if the tile contains proper samples for the surface and drops to 0 when irrelevant [S1].

3. **Compute Dampening Factors:**
   To ensure robustness and prevent multiple flags from being 1 simultaneously, dampen the flags using the surface normal $\mathbf{n} = (n_x, n_y, n_z)$ [S1]:
   $$
   d_i = \frac{|n_i|}{\max(|n_x|, |n_y|, |n_z|)}
   $$
   where $i \in \{x, y, z\}$.

4. **Compute Weights:**
   Calculate the weights $w_x, w_y, w_z$ using the dampened flags. Add a small epsilon $\epsilon$ to $\alpha_i$ to avoid division by zero [S1]:
   $$
   w_x = \frac{\alpha_x d_x (1 - \alpha_y d_y) (1 - \alpha_z d_z)}{\alpha_x d_x (1 - \alpha_y d_y) (1 - \alpha_z d_z) + \alpha_y d_y (1 - \alpha_z d_z) (1 - \alpha_x d_x) + \alpha_z d_z (1 - \alpha_x d_x) (1 - \alpha_y d_y)}
   $$
   $$
   w_y = \frac{\alpha_y d_y (1 - \alpha_z d_z) (1 - \alpha_x d_x)}{\text{denominator}}
   $$
   $$
   w_z = \frac{\alpha_z d_z (1 - \alpha_x d_x) (1 - \alpha_y d_y)}{\text{denominator}}
   $$
   The denominator is the sum of the numerators.

5. **Compute Final Color:**
   $$
   c = c_x w_x + c_y w_y + c_z w_z
   $$

6. **Adaptive Resolution Handling:**
   If tiles have different resolutions, force the color of higher-resolution border samples to match the bilinear interpolation of the lower-resolution samples to prevent high-frequency discontinuities [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def seamless_interpolation(point_p, surface_normal_n, tile_tree):
    # 1. Identify contributing tiles based on normal
    # Determine which faces (x, y, z) are relevant.
    # Assume tiles Tx, Ty, Tz are identified.
    tiles = get_contributing_tiles(tile_tree, point_p, surface_normal_n)
    
    # 2. Fetch colors and flags
    # Fetch color and interpolated alpha flag for each tile
    colors = []
    flags = []
    for tile in tiles:
        c, alpha = fetch_tile_sample(tile, point_p)
        # Handle undefined samples by reading from neighbor if necessary
        if c is undefined:
            c = read_neighbor_color(tile, point_p)
        colors.append(c)
        flags.append(alpha)
        
    # 3. Compute dampening factors
    nx, ny, nz = surface_normal_n
    max_abs_n = max(abs(nx), abs(ny), abs(nz))
    if max_abs_n == 0:
        max_abs_n = 1e-6 # Guard against zero normal
        
    damp = [abs(nx)/max_abs_n, abs(ny)/max_abs_n, abs(nz)/max_abs_n]
    
    # 4. Compute weights
    # Add epsilon to flags to avoid zero division
    epsilon = 1e-6
    alpha_damp = [flags[i] * damp[i] + epsilon for i in range(3)]
    
    # Numerators for wx, wy, wz
    # wx = alpha_x_damp * (1 - alpha_y_damp) * (1 - alpha_z_damp)
    # wy = alpha_y_damp * (1 - alpha_z_damp) * (1 - alpha_x_damp)
    # wz = alpha_z_damp * (1 - alpha_x_damp) * (1 - alpha_y_damp)
    
    num_x = alpha_damp[0] * (1 - alpha_damp[1]) * (1 - alpha_damp[2])
    num_y = alpha_damp[1] * (1 - alpha_damp[2]) * (1 - alpha_damp[0])
    num_z = alpha_damp[2] * (1 - alpha_damp[0]) * (1 - alpha_damp[1])
    
    denom = num_x + num_y + num_z
    
    # 5. Compute final color
    if denom == 0:
        # Fallback if all flags are effectively zero (should be rare with epsilon)
        return colors[0] # Or average, or 0
        
    w_x = num_x / denom
    w_y = num_y / denom
    w_z = num_z / denom
    
    final_color = colors[0] * w_x + colors[1] * w_y + colors[2] * w_z
    
    return final_color
```

**Parameters:**
- $\epsilon$: Small constant to prevent division by zero. Value not given in sources [G]. Typical value: $10^{-6}$.
- Tile resolution: $2^k + 1$ pixels per side [S1].

**Edge Cases and Pitfalls:**
- **Multiple Flags = 1:** If multiple flags equal 1, weights become null/undefined. Solved by dampening with surface normal [S1].
- **All Flags = 0:** If all flags equal 0, weights are undefined. Solved by adding epsilon to flags [S1].
- **Discontinuous Normals:** The method assumes a smooth surface normal field. Discontinuities in the normal field will cause discontinuities in the result [S1].
- **Adaptive Resolution Boundaries:** Without specific treatment, boundaries between different resolution tiles produce high-frequency discontinuities. Solved by forcing higher-resolution border samples to match bilinear interpolation of lower-resolution samples [S1].

## Operating characteristics
- **Time Complexity:** Constant time per sample, involving a few multiplications and additions. Relies on hardware bilinear filtering for flag interpolation [S1].
- **Memory Complexity:** Increases space overhead due to one-pixel border replication between neighboring tiles, especially for thick layers [S1].
- **Strengths:** Seamless interpolation over smooth surfaces; handles partial coverage and adaptive resolution; robust against geometric features between tile samples.
- **Limitations:** Limited to smooth surfaces with continuous normal fields; increased memory usage for border replication; requires storing binary flags in alpha channel.

## Background
Seamless Interpolation was introduced in the context of TileTrees, a hierarchical data structure for volume rendering. It was proposed to address the limitations of standard hardware interpolation when dealing with partially covered faces and n-leaves in tile-based volume rendering. The method relies on the graphics hardware interpolation capabilities and extends them with specific handling for tile boundaries and surface normals.

## Sources
[S1] TileTrees, https://doi.org/10.1145/1230100.1230104, fetched 2026-10-26
