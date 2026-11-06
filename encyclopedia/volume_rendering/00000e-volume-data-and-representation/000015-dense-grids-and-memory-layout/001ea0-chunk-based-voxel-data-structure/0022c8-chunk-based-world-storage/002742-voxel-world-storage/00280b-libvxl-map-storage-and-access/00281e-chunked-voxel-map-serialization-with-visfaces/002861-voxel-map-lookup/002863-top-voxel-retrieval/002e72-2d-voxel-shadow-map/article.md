# 2D Voxel Shadow Map

Also called: Column Shadow Map, Heightmap Shadowing

## Overview
A 2D Voxel Shadow Map is a rendering optimization that stores the highest occupied voxel Y-coordinate for each (X, Z) column in a 2D texture, enabling fast shadow determination without ray marching. It differs from standard depth buffers by storing world-space height values rather than view-space depth, and from ray-marched shadows by using a single texture lookup per fragment instead of iterative sampling. This method builds on [Top Voxel Retrieval](4236-top-voxel-retrieval.md) to provide occlusion data.

## How it works
**Inputs:**
- World-space bounding box of the voxel volume: `minV`, `maxV`.
- Voxel grid dimensions: `SH_MAX_X` (width in voxels), `SH_MAX_Z` (depth in voxels).
- Scaling factor: `root_bitw` (log2 scaling factor, typically 3).

**Outputs:**
- `shadow_map`: A 2D texture (or `uint16_t` array) of size `shadow_map_size[0] x shadow_map_size[1]`, storing the maximum Y height for each (X, Z) column.

**Data Structures:**
- `shadow_map`: `uint16_t*` array of size `shadow_map_size[0] * shadow_map_size[1]`. Each element stores the highest Y-coordinate (in scaled world units) of an occupied voxel at the corresponding (X, Z) column.

**Steps:**
1. **Initialize Shadow Map Dimensions:**
   Calculate the dimensions of the shadow map based on the world bounds and scaling factor:
   $$ \text{width} = (\max[0] \ll \text{root\_bitw}) + (\max[1] \ll \text{root\_bitw}) $$
   $$ \text{depth} = (\max[2] \ll \text{root\_bitw}) $$
   Allocate a `uint16_t` array of size `width * depth`. Initialize all entries to a value indicating "empty" (e.g., 0 or max value, depending on implementation).

2. **Populate Shadow Map:**
   For each occupied voxel at world position `ws = (ws[0], ws[1], ws[2])`:
   a. Calculate the shadow map index:
      $$ \text{index} = (\text{ws}[0] + \text{ws}[1]) + (\text{SH\_MAX\_X} \times \text{ws}[2]) $$
      *Note: This formula shears the coordinate space by adding `ws[1]` to `ws[0]`.*
   b. Update the shadow map entry if the current voxel's Y-coordinate is higher than the stored value:
      $$ \text{shadow\_map}[\text{index}] = \max(\text{shadow\_map}[\text{index}], \text{ws}[1]) $$

3. **Sample Occlusion:**
   To determine if a voxel at `ws` is in shadow:
   a. Calculate the shadow map index for `ws` using the formula above.
   b. Check the stored height and its neighbor:
      $$ \text{is\_shadowed} = (\text{shadow\_map}[\text{index}] \ge \text{ws}[1] + 1) \lor (\text{shadow\_map}[\text{index} + 1] \ge \text{ws}[1] + 1) $$
   c. Return 1 if occluded, 0 otherwise.

**Parameters:**
- `root_bitw`: Log2 scaling factor for world coordinates to shadow map coordinates. Typical value: 3 (scaling factor of 8).

**Edge Cases and Pitfalls:**
- **Index Calculation Shear:** The formula `index = (ws[0] + ws[1]) + (SH_MAX_X * ws[2])` adds `ws[1]` to `ws[0]`, effectively shearing the coordinate space. This requires careful alignment of world bounds and shadow map dimensions to avoid out-of-bounds access.
- **Boundary Conditions:** Ensure that `ws[1] + 1` does not exceed the maximum representable value in `uint16_t` and that `index + 1` is within bounds when checking neighbors.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_shadow_map(max_x, max_y, max_z, root_bitw):
    width = (max_x << root_bitw) + (max_y << root_bitw)
    depth = max_z << root_bitw
    shadow_map = [0] * (width * depth)
    return shadow_map, width, depth

def update_shadow_map(shadow_map, width, ws, SH_MAX_X):
    index = (ws[0] + ws[1]) + (SH_MAX_X * ws[2])
    if index < len(shadow_map):
        shadow_map[index] = max(shadow_map[index], ws[1])

def sample_occlusion(shadow_map, ws, SH_MAX_X):
    index = (ws[0] + ws[1]) + (SH_MAX_X * ws[2])
    if index < len(shadow_map) and (index + 1) < len(shadow_map):
        is_shadowed = (shadow_map[index] >= ws[1] + 1) or (shadow_map[index + 1] >= ws[1] + 1)
        return 1 if is_shadowed else 0
    return 0  # Default to not shadowed if out of bounds
```

## Subtopics

### [Dynamic Orthographic Shadow Map Generation](7386-dynamic-orthographic-shadow-map-generation.md)
Generates a per-frame depth texture from a directional light's perspective to compute shadow occlusion for voxel models. Choose this when dynamic lighting and per-frame updates are required.

## Operating characteristics
- **Time Complexity:** O(N) to populate the shadow map, where N is the number of occupied voxels. O(1) per fragment for occlusion sampling.
- **Memory Complexity:** O(W * D), where W and D are the shadow map dimensions.
- **Measured Performance:** Performance depends on the resolution of the shadow map and the number of voxels. Higher resolution increases memory usage but improves shadow accuracy.
- **Strengths:** Fast shadow determination without ray marching; simple implementation.
- **Limitations:** Sheared coordinate space can lead to out-of-bounds access; limited to voxel grids; static shadow maps require regeneration for dynamic scenes.
- **When to Use:** Suitable for static or slowly changing voxel environments where performance is critical.
- **When Not to Use:** Not ideal for highly dynamic scenes or non-voxel-based geometry.

## Background
The 2D Voxel Shadow Map technique is used in voxel-based rendering systems to optimize shadow calculations. It is particularly relevant in games and simulations involving large voxel worlds, such as [Voxel Plat](https://github.com/kosshi-net/voxplat) [S1, unofficial]. The method leverages the regular structure of voxel grids to simplify shadow determination.

## Sources
[S1] kosshi-net/voxplat, https://github.com/kosshi-net/voxplat, fetched 2026-10-27 (unofficial)
