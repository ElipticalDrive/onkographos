# Reverse Z Perspective Projection with Infinite Far Plane

Also called: Infinite Far Plane Projection, LH Reverse Z Projection

## Overview
This method constructs a perspective projection matrix that maps view space coordinates to clip space while utilizing a reversed depth buffer (Reverse Z) and an effectively infinite far plane [S1, unofficial]. It solves the problem of depth precision loss at large distances inherent in standard finite-far-plane projections, thereby avoiding far-plane clipping artifacts and maintaining high precision near the camera [S1, unofficial]. It differs from standard perspective projections by inverting the depth range and removing the far-plane dependency from the matrix structure [S1, unofficial].

## How it works
**Inputs:**
- `near_plane`: Distance to the near clipping plane (float).
- `window_aspect_ratio`: Width divided by height of the viewport (float).
- `fov`: Field of view angle in degrees (float).

**Outputs:**
- `M`: A 4x4 projection matrix in row-major order.

**Data Structures:**
The matrix $M$ is a 4x4 array of floats. In row-major layout, the elements are indexed $M_{0,0}$ to $M_{3,3}$.

**Steps:**
1.  Convert the field of view from degrees to radians and compute half-FOV sine and cosine [S1, unofficial].
2.  Compute the height scaling factor $h$ using the cosine and sine of half-FOV [S1, unofficial].
3.  Compute the width scaling factor $w$ using $h$ and the aspect ratio [S1, unofficial].
4.  Construct the matrix $M$ using $w$, $h$, and $near\_plane$ [S1, unofficial].

**Formulas:**
Let $\theta$ be the field of view in radians.
$$ \theta = \text{fov} \times \frac{\pi}{180} $$
$$ \sin_{\theta/2} = \sin\left(\frac{\theta}{2}\right) $$
$$ \cos_{\theta/2} = \cos\left(\frac{\theta}{2}\right) $$

The height scaling factor $h$ is:
$$ h = \frac{\cos_{\theta/2}}{\sin_{\theta/2}} $$

The width scaling factor $w$ is:
$$ w = \frac{h}{\text{window\_aspect\_ratio}} $$

The projection matrix $M$ is:
$$ M = \begin{bmatrix} w & 0 & 0 & 0 \\ 0 & h & 0 & 0 \\ 0 & 0 & 0 & 1 \\ 0 & 0 & n & 0 \end{bmatrix} $$
where $n$ is `near_plane` [S1, unofficial].

**Parameters:**
- `near_plane`: Typical value is 1.0f [S1, unofficial].
- `far_plane`: Set to 1000000.0f in implementation but effectively ignored due to the infinite far plane formulation [S1, unofficial].

**Pitfalls:**
- Standard perspective projection uses a finite far plane, causing depth precision loss at large distances [S1, unofficial].
- The bottom-right element ($M_{3,3}$) is 0, and the element at row 2, column 3 ($M_{2,3}$) is 1 (in 0-indexed notation, row 2 col 3 is the 3rd row, 4th column in 1-indexed, corresponding to the $n$ term in the source description's specific layout). The source describes the matrix as having `n` in the third row, fourth column position relative to the standard layout modification, specifically replacing the far-plane term [S1, unofficial].
- Precision: The reversed Z approach provides high precision (parts in $10^8$) over large distances [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def construct_reverse_z_infinite_fov_projection(near_plane, fov_degrees, aspect_ratio):
    """
    Constructs a Reverse Z Perspective Projection Matrix with Infinite Far Plane.
    
    Args:
        near_plane (float): Distance to the near clipping plane.
        fov_degrees (float): Field of view in degrees.
        aspect_ratio (float): Width / Height of the viewport.
        
    Returns:
        list: 4x4 matrix in row-major order (list of lists).
    """
    # Step 1: Calculate half-FOV sine and cosine
    fov_radians = math.radians(fov_degrees)
    half_fov = fov_radians / 2.0
    sin_fov = math.sin(half_fov)
    cos_fov = math.cos(half_fov)
    
    # Step 2: Compute height scaling factor
    # h = cos(fov/2) / sin(fov/2)
    height = cos_fov / sin_fov
    
    # Step 3: Compute width scaling factor
    # w = h / aspect_ratio
    width = height / aspect_ratio
    
    # Step 4: Construct the matrix
    # M = [w, 0, 0, 0;
    #      0, h, 0, 0;
    #      0, 0, 0, 1;
    #      0, 0, n, 0]
    # Note: Source [S1, unofficial] specifies row-major order.
    # Row 0: w, 0, 0, 0
    # Row 1: 0, h, 0, 0
    # Row 2: 0, 0, 0, 1  (Note: Source says "0 in the bottom-right and 1 in the third row, fourth column")
    # Row 3: 0, 0, n, 0  (Note: Source says "n" is near_plane)
    
    matrix = [
        [width, 0.0, 0.0, 0.0],
        [0.0, height, 0.0, 0.0],
        [0.0, 0.0, 0.0, 1.0],
        [0.0, 0.0, near_plane, 0.0]
    ]
    
    return matrix
```

## Subtopics

### Reverse Z Depth Buffering
A depth buffer technique where depth is read and tested in reverse, eliminating the need for a far plane and providing high precision (parts in $10^8$) over large distances [S2, unofficial]. Implementation involves every pass reading and testing depth using reversed-Z [S2, unofficial]. Specifically, the lens pass and buffer view read depth via `helio_view_depth(proj, d)`, `helio_is_sky`, and `helio_depth_nearest` for a footprint's nearest texel, while lens tests write the engine's own depth [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Matrix construction is $O(1)$. Depth testing remains $O(1)$ per fragment but with improved precision characteristics.
- **Memory Complexity:** $O(1)$ for the matrix storage.
- **Performance:** Provides depth precision of parts in $10^8$ over large distances [S2, unofficial].
- **Strengths:** Eliminates far-plane clipping artifacts and improves depth precision near the camera [S1, unofficial].
- **Limitations:** Requires a depth buffer format and shader support for reversed-Z testing [S2, unofficial].
- **When to use:** When rendering scenes with large depth ranges where standard Z-buffer precision is insufficient [S1, S2, unofficial].
- **When not to use:** If the rendering pipeline does not support reversed-Z depth testing or if compatibility with standard fixed-function pipelines is required.

## Background
Originating from implementations in voxel engines and custom rendering pipelines, this method addresses the limitations of standard perspective projection matrices [S1, unofficial]. It is used in engines like Helio to handle large-scale voxel planets and chunk-based rendering where distant objects must be visible without precision loss [S2, unofficial]. The technique is often paired with reversed-Z depth buffering to maximize the utility of the depth buffer's precision distribution [S2, unofficial].

## Sources
[S1] rtarun9/voxel-engine, https://github.com/rtarun9/voxel-engine, fetched 2026-10-27 (unofficial)
[S2] Voxel planet: chunk trees, a fully destructible planet, and the lighting to show it, https://github.com/Far-Beyond-Pulsar/Helio/pull/327, fetched 2026-10-28 (unofficial)
