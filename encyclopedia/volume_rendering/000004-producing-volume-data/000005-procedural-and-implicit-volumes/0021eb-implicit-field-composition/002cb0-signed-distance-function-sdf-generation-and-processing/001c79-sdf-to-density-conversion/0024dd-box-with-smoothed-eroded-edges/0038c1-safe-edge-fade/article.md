# Safe Edge Fade

Also called: Continuous Edge Protection, Smoothstep Edge Fade

## Overview
Safe Edge Fade is a boundary condition technique that applies a continuous smoothing function to fade volume density to zero at the edges of a bounding box, preventing the appearance of rectangular blank bands or hard discontinuities [S1, unofficial]. It ensures that the outermost voxels are strictly zero while maintaining continuous attenuation across the transition zone [S1, unofficial]. Unlike its parent method, which applies a sigmoid-based erosion to the entire box, Safe Edge Fade specifically targets the perimeter to ensure seamless integration with transparent backgrounds or adjacent volumes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D density field $\rho(x,y,z)$, the volume dimensions, and boundary parameters.
- Output: A modified density field where density is smoothly interpolated to zero near the boundaries.

**Data Structures:**
- The density field is typically stored as a 3D array or texture.
- Parameters are scalar values: `empty_border` and `fade_width`.

**Steps:**
1. For each voxel at position $(x, y, z)$, calculate the distance to the nearest volume boundary in each axis. Let $d_x, d_y, d_z$ be the distances to the left/right, front/back, and bottom/top boundaries respectively.
2. Determine the minimum distance to any boundary: $d_{min} = \min(d_x, d_y, d_z)$.
3. Apply the fade function using the `smoothstep` interpolation:
   $$ fade = \text{smoothstep}\left(\frac{\max(d_{min} - \text{empty\_border}, 0.0)}{\text{fade\_width}}\right) $$
4. Multiply the original density $\rho$ by the fade factor: $\rho_{final} = \rho \times fade$.

**Formula:**
The core smoothing function is the standard `smoothstep`:
$$ \text{smoothstep}(x) = x^2 (3 - 2x) $$
where $x$ is clamped to $[0.0, 1.0]$.

**Parameters:**
- `empty_border`: The distance from the edge within which density is forced to zero.
- `fade_width`: The width of the transition zone where density fades from full to zero.

**Typical Values:**
- For XY boundaries: `SAFE_EMPTY_BORDER_XY` = 0.0, `SAFE_FADE_WIDTH_XY` = 0.0 [S1, unofficial].
- For Z boundaries: `SAFE_EMPTY_BORDER_Z` = 0.0, `SAFE_FADE_WIDTH_Z` = 0.13 [S1, unofficial].
- Alternative generic values cited: Empty border = 0.0, Fade width = 0.045 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def safe_edge_fade(density: np.ndarray, shape: tuple, empty_border: float, fade_width: float) -> np.ndarray:
    """
    Applies Safe Edge Fade to a 3D density array.
    
    Args:
        density: 3D numpy array of density values.
        shape: Tuple (D, H, W) representing the dimensions of the volume.
        empty_border: Distance from edge where density becomes 0.
        fade_width: Width of the smooth transition zone.
        
    Returns:
        Modified density array with edges faded to zero.
    """
    D, H, W = shape
    
    # Create coordinate grids
    z, y, x = np.meshgrid(
        np.arange(D), 
        np.arange(H), 
        np.arange(W), 
        indexing='ij'
    )
    
    # Calculate distance to nearest boundary in each axis
    dist_x = np.minimum(x, W - 1 - x)
    dist_y = np.minimum(y, H - 1 - y)
    dist_z = np.minimum(z, D - 1 - z)
    
    # Minimum distance to any boundary
    dist_to_edge = np.minimum(np.minimum(dist_x, dist_y), dist_z)
    
    # Calculate fade factor
    # Formula: smoothstep(max(dist - empty_border, 0) / fade_width)
    normalized_dist = np.maximum(dist_to_edge - empty_border, 0.0) / fade_width
    
    # Clamp to [0, 1] for smoothstep
    clipped = np.clip(normalized_dist, 0.0, 1.0)
    
    # Apply smoothstep: x^2 * (3 - 2x)
    fade_factor = clipped * clipped * (3.0 - 2.0 * clipped)
    
    # Apply fade to density
    return density * fade_factor
```

**Edge Cases and Pitfalls:**
- If `fade_width` is 0.0, the division by zero must be handled. In the cited implementation, if `fade_width` is 0.0, the fade factor is effectively 1.0 everywhere except where `empty_border` forces it to 0, or the logic defaults to no fade [S1, unofficial]. The pseudocode above assumes `fade_width > 0`. If `fade_width` is 0, the fade factor should be 1.0 if `dist_to_edge > empty_border`, else 0.0.
- Precision issues may arise if `fade_width` is very small relative to voxel size.

## Subtopics

### Spherical Ring Density Fall-off
A boundary condition technique for spherical volumes that creates a ring around the sphere where density smoothly interpolates from the interior value to zero, preventing hard edges [S2, unofficial]. It identifies the spherical boundary, defines a ring region near the surface, and interpolates density within the ring to zero at the outer edge, setting density to zero outside the ring [S2, unofficial]. Density bounds are 0 (full transparency) to 1 (full opacity) [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel is processed once.
- **Memory Complexity:** $O(N)$ for the output density field, or $O(1)$ if applied in-place.
- **Strengths:** Eliminates visible rectangular artifacts at volume boundaries; simple to implement; computationally inexpensive.
- **Limitations:** Only applies to box-shaped volumes (or requires adaptation for other shapes); the fade width must be tuned to avoid excessive blurring or visible transitions.
- **When to use:** When rendering procedural volumes inside bounding boxes where hard edges are undesirable.
- **When not to use:** For volumes with irregular shapes where a box-based fade is inappropriate.

## Background
Safe Edge Fade is used in procedural cloud generation systems, specifically in the Noise-Cloud project, to ensure that volume clouds do not exhibit hard rectangular boundaries when rendered [S1, unofficial]. It is a post-processing step applied to the density field after generation but before ray marching or volume rendering [S1, unofficial]. The technique is part of a broader set of implicit field composition methods used in real-time graphics to create natural-looking volumetric effects.

## Sources
[S1] mingtu149411/Noise-Cloud, https://github.com/mingtu149411/Noise-Cloud, fetched 2026-10-27 (unofficial)
[S2] Volume Rendering Final Project (Longer), https://www.youtube.com/watch?v=j4tECzsy0CE, fetched 2026-10-27 (unofficial)
