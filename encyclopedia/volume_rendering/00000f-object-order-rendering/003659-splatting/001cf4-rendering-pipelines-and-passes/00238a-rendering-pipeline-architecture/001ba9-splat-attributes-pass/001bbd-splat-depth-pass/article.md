# Splat Depth Pass

Also called: Front-Face Depth Pass, Visibility Pass

## Overview
The Splat Depth Pass computes the depth of the front surface of a point-sampled primitive (splat) for each fragment, writing this value to the depth buffer to establish visibility for subsequent blending passes. It differs from sibling passes like [Radix Sort for Splat Depth](1207-radix-sort-for-splat-depth.md) by performing depth calculation per-fragment during rasterization rather than pre-sorting primitives, and from [Tile Binning for Splatting](1214-tile-binning-for-splatting.md) by not explicitly grouping fragments into tiles for early-Z culling. It solves the problem of correctly occluding the back faces of spherical splats while maintaining correct alpha blending order for the front face.

## How it works
**Inputs:** Fragment shader context including `gl_FragCoord` (window-space coordinates), `gl_PointCoord` (texture coordinates within the point sprite, range [0,1]), and vertex attributes `i.radius` (splat radius in world space) and `linearDepth` (a function or attribute providing the depth of the splat center).
**Outputs:** `gl_FragDepth` (the depth value written to the depth buffer).

**Data Structures:**
*   `gl_FragCoord`: `vec4`, `xy` are window coordinates, `z` is linear depth of the center.
*   `gl_PointCoord`: `vec2`, normalized coordinates within the point sprite.
*   `i.radius`: `float`, radius of the splat in world units.

**Steps:**
1.  **Calculate local offset:** Compute the vector from the center of the point sprite to the current fragment in point-space coordinates.
    $$ \mathbf{p}_{local} = \text{gl\_PointCoord} - \begin{bmatrix} 0.5 \\ 0.5 \end{bmatrix} $$
2.  **Discard outside circle:** Calculate the squared magnitude of the local offset. If the fragment lies outside the unit circle (radius 0.5), discard it.
    $$ d^2 = \mathbf{p}_{local} \cdot \mathbf{p}_{local} $$
    $$ \text{if } d^2 > 0.25 \text{ then discard} $$
3.  **Calculate world-space offset:** Scale the local offset to world space using the splat radius.
    $$ \mathbf{p}_{world\_xy} = \mathbf{p}_{local} \times 2 \times i.radius $$
4.  **Calculate depth offset:** Compute the $z$-offset from the center to the front surface of the sphere at the given $(x,y)$ position. This uses the sphere equation $x^2 + y^2 + z^2 = r^2$.
    $$ z_{offset} = \sqrt{i.radius^2 - (\mathbf{p}_{world\_xy} \cdot \mathbf{p}_{world\_xy})} $$
5.  **Compute final depth:** Subtract the depth offset from the center depth to get the front surface depth. Add a small epsilon to prevent z-fighting or gaps between overlapping splats.
    $$ z_{front} = \text{linearDepth}(\text{gl\_FragCoord}.z) - z_{offset} + \epsilon $$
    $$ \text{gl\_FragDepth} = z_{front} $$

**Parameters:**
*   $\epsilon$: Epsilon value added to prevent precision artifacts. Typical value: $0.25$ (derived from `_epsilonMultiplier` in implementation [S1, unofficial]).

**Edge Cases and Pitfalls:**
*   **Z-fighting:** Without the epsilon shift, overlapping splats with similar depths may cause flickering or incorrect occlusion. The epsilon ensures the front face is slightly closer to the camera than the back face of overlapping primitives [S1, unofficial].
*   **Precision:** The square root calculation in step 4 assumes the splat is spherical and aligned with the view. For non-spherical splats or extreme viewing angles, the depth offset calculation may require adjustment [G].
*   **Boundary:** The discard condition in step 2 ensures no fragments are processed outside the splat's circular boundary, optimizing performance.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Logic
def splat_depth_pass(gl_frag_coord, gl_point_coord, i_radius, linear_depth_func, epsilon=0.25):
    # Step 1: Calculate local offset from center (0.5, 0.5)
    from_center_pt_spc = gl_point_coord - vec2(0.5, 0.5)
    
    # Step 2: Calculate squared magnitude
    mag_sq = dot(from_center_pt_spc, from_center_pt_spc)
    
    # Discard if outside the unit circle (radius 0.5, squared 0.25)
    if mag_sq > 0.25:
        discard()
        
    # Step 3: Calculate world-space XY offset
    # Scale by 2 * radius to convert from [0,1] point space to world space
    from_center_wld_spc = from_center_pt_spc * 2.0 * i_radius
    
    # Step 4: Calculate depth offset (z-component of sphere surface)
    # z = sqrt(r^2 - x^2 - y^2)
    dot_xy = dot(from_center_wld_spc, from_center_wld_spc)
    depth_offset = sqrt(i_radius * i_radius - dot_xy)
    
    # Step 5: Calculate final depth
    # Center depth minus offset, plus epsilon for front-face bias
    center_depth = linear_depth_func(gl_frag_coord.z)
    final_depth = center_depth - depth_offset + epsilon
    
    return final_depth
```

## Subtopics

### [Single-Layer Ternary Depth Test](4509-single-layer-ternary-depth-test.md)
A rasterization technique that maintains only a single surface layer per pixel, using a depth band to decide whether to blend, replace, or discard fragments. It simplifies the pipeline by limiting visibility to one depth value per pixel but may introduce occlusion artifacts in complex geometry.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per fragment. The calculation involves basic arithmetic and a square root.
*   **Memory Complexity:** $O(1)$ auxiliary memory. Only local variables are used.
*   **Performance:** Highly efficient as it runs in the fragment shader with minimal branching (only the discard check). The square root operation is the most expensive step but is well-optimized on modern GPUs.
*   **Strengths:** Simple to implement, correct depth ordering for spherical splats, prevents z-fighting with epsilon bias.
*   **Limitations:** Assumes spherical splats. Does not handle non-spherical or elongated splats correctly without modification. Does not perform early-Z culling itself (relies on hardware depth testing).
*   **When to use:** When rendering point clouds or Gaussian splats with spherical kernels where correct front-face depth is required for occlusion.
*   **When not to use:** For non-spherical splats or when pre-sorting is required for other pipeline stages (use [Radix Sort for Splat Depth](1207-radix-sort-for-splat-depth.md) instead).

## Background
The Splat Depth Pass is a standard technique in point-based rendering and Gaussian Splatting pipelines. It is implemented in various open-source point cloud rendering projects, such as the one by arminkz [S1, unofficial]. The method relies on the geometric property of spheres to calculate the front surface depth from the center depth and the fragment's position within the point sprite.

## Sources
[S1] arminkz/PointCloudRendering, https://github.com/arminkz/PointCloudRendering, fetched 2026-10-28 (unofficial)
