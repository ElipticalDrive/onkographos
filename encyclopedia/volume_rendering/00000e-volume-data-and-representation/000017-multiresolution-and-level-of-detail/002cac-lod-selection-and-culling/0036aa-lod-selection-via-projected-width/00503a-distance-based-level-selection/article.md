# Distance-Based Level Selection

Also called: Level Mapping

## Overview
Distance-Based Level Selection is a mechanism that selects the octree resolution level for each anchor point based on its Euclidean distance from the camera, ensuring closer areas use higher detail and distant areas use simplified representations [S1]. It serves as the core LOD selection logic within the Octree-GS system, mapping continuous camera-to-anchor distances to discrete integer resolution levels [S1]. Unlike fixed-distance culling, this method uses a logarithmic relationship to maintain consistent visual quality across varying scene scales [S1]. It builds on [LoD Selection via Projected Width](7837-lod-selection-via-projected-width.md) by using camera distance as the primary metric for detail allocation rather than screen-space footprint [S1].

## How it works
**Inputs:**
- Camera position ($C$)
- Anchor positions ($P_i$)
- Octree branching factor ($fork$)
- Reference distance ($standard\_dist$)
- Bias parameter ($extra\_level$)
- Level mapping strategy ($strategy$)

**Outputs:**
- Integer level ($L_i$) for each anchor $i$
- Optional boolean visibility mask ($M_i$)

**Data Structures:**
- `_anchor`: Tensor of 3D positions, shape $(N, 3)$, where $N$ is the number of anchors [S1].
- `_level`: Tensor of integer levels, shape $(N,)$, storing the selected LOD for each anchor [S1].
- `_anchor_mask`: Boolean visibility mask, shape $(N,)$, indicating which anchors are visible [S1].

**Steps:**
1. **Calculate Distance:** Compute the Euclidean distance $dist_i$ between the camera position $C$ and each anchor position $P_i$:
   $$ dist_i = \| P_i - C \|_2 $$
2. **Compute Floating-Point Level:** Calculate the predicted floating-point level $pred\_level_i$ using the logarithmic formula [S1]:
   $$ pred\_level_i = \frac{\log_2(standard\_dist / dist_i)}{\log_2(fork)} + extra\_level $$
   Where:
   - $standard\_dist$: A reference distance parameter [S1].
   - $dist_i$: The computed Euclidean distance [S1].
   - $fork$: The branching factor of the octree (typically 2) [S1].
   - $extra\_level$: A bias parameter to shift the LOD threshold [S1].
3. **Map to Integer Level:** Convert $pred\_level_i$ to an integer level $L_i$ based on the selected strategy [S1]:
   - **round** (default): $L_i = \text{round}(pred\_level_i)$
   - **floor**: $L_i = \lfloor pred\_level_i \rfloor$
   - **ceil**: $L_i = \lceil pred\_level_i \rceil$
   - **progressive**: Extract integer and fractional parts for blending (see Progressive Level Transitions) [S1].
4. **Create Visibility Mask (Optional):** Generate a boolean mask $M_i$ where $M_i = \text{True}$ if the anchor is within the valid LOD range for rendering, filtering out anchors that are too far or too close for the selected resolution [S1].
5. **Filter Anchors:** Use the mask to pass only visible anchors to the renderer [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def distance_based_level_selection(
    camera_position: np.ndarray,
    anchor_positions: np.ndarray,
    fork: float = 2.0,
    standard_dist: float = 1.0,
    extra_level: float = 0.0,
    strategy: str = "round"
) -> tuple:
    """
    Selects LOD levels for anchors based on camera distance.
    
    Args:
        camera_position: 3D vector of camera coordinates.
        anchor_positions: Nx3 array of anchor coordinates.
        fork: Branching factor of the octree.
        standard_dist: Reference distance for level calculation.
        extra_level: Bias parameter for level shifting.
        strategy: 'round', 'floor', 'ceil', or 'progressive'.
        
    Returns:
        levels: Nx1 array of integer levels.
        mask: Nx1 boolean array of visible anchors.
    """
    # Step 1: Calculate Euclidean distances
    # Shape: (N,)
    distances = np.linalg.norm(anchor_positions - camera_position, axis=1)
    
    # Guard against division by zero
    distances = np.maximum(distances, 1e-6)
    
    # Step 2: Compute floating-point level
    # Formula: log2(standard_dist / dist) / log2(fork) + extra_level
    # Using change of base for log2 if necessary, but numpy has log2
    pred_levels = (np.log2(standard_dist / distances) / np.log2(fork)) + extra_level
    
    # Step 3: Map to integer level based on strategy
    if strategy == "round":
        levels = np.round(pred_levels).astype(int)
    elif strategy == "floor":
        levels = np.floor(pred_levels).astype(int)
    elif strategy == "ceil":
        levels = np.ceil(pred_levels).astype(int)
    elif strategy == "progressive":
        # For progressive, we keep the float for blending later, 
        # but the integer level is the floor
        levels = np.floor(pred_levels).astype(int)
    else:
        raise ValueError(f"Unknown strategy: {strategy}")
        
    # Step 4: Create visibility mask
    # Assuming valid levels are >= 0. Specific min/max bounds depend on octree depth.
    # [G] A common guard is to ensure levels are within the octree's defined depth range.
    # Let's assume a max_depth is known or inferred from data. 
    # For this implementation, we assume any non-negative level is potentially visible,
    # but typically a max_level is enforced.
    mask = levels >= 0
    
    return levels, mask

def progressive_blend_properties(
    properties_level_n: dict,
    properties_level_n_plus_1: dict,
    fractional_part: float
) -> dict:
    """
    Blends Gaussian properties (specifically opacity) between adjacent levels.
    
    Args:
        properties_level_n: Properties for integer level n.
        properties_level_n_plus_1: Properties for integer level n+1.
        fractional_part: The blending factor (0.0 to 1.0).
        
    Returns:
        Blended properties.
    """
    # [G] Assuming properties is a dict with 'opacity' key.
    # Opacity is blended linearly.
    opacity_n = properties_level_n['opacity']
    opacity_n1 = properties_level_n_plus_1['opacity']
    
    blended_opacity = opacity_n * (1 - fractional_part) + opacity_n1 * fractional_part
    
    return {
        'opacity': blended_opacity,
        # [G] Other properties like position/scale might also need blending 
        # depending on the specific Gaussian Splatting implementation details not fully specified.
        # The source specifically mentions opacity.
    }
```

**Parameters:**
- `fork`: The branching factor of the octree tree, determining subdivisions per level. Typical value: 2 [S1].
- `extra_level`: A bias parameter added to the calculated level to shift the LOD selection threshold [S1].
- `standard_dist`: A reference distance used in the logarithmic calculation [S1].
- `base_layer`: The coarsest layer of the octree, corresponding to LOD 0 [S1].

**Edge Cases and Pitfalls:**
- **Zero Distance:** If an anchor is at the camera position, $dist=0$ causes division by zero. A small epsilon (e.g., $10^{-6}$) should be added to distances [G].
- **Negative Levels:** If $dist > standard\_dist \cdot fork^{extra\_level}$, $pred\_level$ becomes negative. These anchors should be masked out or clamped to the coarsest level [G].
- **Precision:** Floating-point precision in logarithmic calculations can lead to jitter in level selection for anchors at similar distances. The `progressive` strategy mitigates visual artifacts but requires careful handling of fractional parts [S1].

## Subtopics

### Automatic Level Adaptation
A mechanism to automatically determine the optimal number of octree levels based on the scene's distance distribution relative to the camera, avoiding manual tuning [S1]. It calculates distance statistics for each camera position, determines appropriate min and max distances ($dist\_min$, $dist\_max$), and computes the optimal number of levels using the formula: $levels = \frac{\log_2(dist\_max / dist\_min)}{\log_2(fork)} + 1$ [S1]. This ensures the system works effectively across a wide range of scenes without manual tuning [S1].

### Anchor Masking
A process that creates a boolean visibility mask for anchor points based on their computed LOD level, filtering out anchors that are too far or too close for the selected resolution [S1]. It computes the distance from each anchor to the camera, converts distances to level values, creates a boolean mask where True indicates the anchor should be visible, and filters anchors by this mask before passing them to the renderer [S1]. Data structures include `_anchor` (Tensor of 3D positions), `_level` (Tensor of integer levels), and `_anchor_mask` (Boolean visibility mask) [S1].

### Progressive Level Transitions
A technique to avoid sudden visual changes during LOD switching by blending Gaussian properties (specifically opacity) between adjacent integer levels using the fractional part of the computed level [S1]. When `dist2level` is set to 'progressive', the system extracts the integer part (floor) of the computed floating-point level to select the main level, and the fractional part as a blending factor [S1]. It selects Gaussian properties for the integer level and blends them with properties from the next level (integer + 1) using the fractional part as the blend factor, applying the blend factor specifically to the opacity of Gaussians at transition boundaries to create continuous transitions [S1].

## Operating characteristics
- **Time Complexity:** $O(N)$ for distance calculation and level mapping, where $N$ is the number of anchors [S1].
- **Memory Complexity:** $O(N)$ to store anchor positions, levels, and masks [S1].
- **Performance:** The method is efficient as it relies on simple arithmetic operations and logarithms [S1].
- **Strengths:** Provides smooth LOD transitions with the 'progressive' strategy; automatically adapts to scene scale with automatic level adaptation [S1].
- **Limitations:** Requires careful tuning of `standard_dist` and `extra_level` for optimal visual quality; may not account for occlusion or screen-space detail requirements as effectively as projected width methods [S1].
- **When to Use:** Suitable for scenes with varying camera distances where consistent detail allocation is desired [S1].
- **When Not to Use:** Not ideal for scenes with static cameras or where screen-space detail is more critical than world-space distance [S1].

## Background
Distance-Based Level Selection is part of the Level of Detail (LOD) system in Octree-GS, a method for 3D Gaussian Splatting [S1]. It was introduced in the Octree-GS project by city-super [S1]. The system integrates with the Gaussian rendering pipeline to provide level-appropriate Gaussians for rendering [S1]. The method is documented in the Octree-GS source code and related documentation [S1].

## Sources
[S1] Level of Detail System | city-super/Octree-GS | DeepWiki, https://deepwiki.com/city-super/Octree-GS/2.3-level-of-detail-system, fetched 2026-10-28
