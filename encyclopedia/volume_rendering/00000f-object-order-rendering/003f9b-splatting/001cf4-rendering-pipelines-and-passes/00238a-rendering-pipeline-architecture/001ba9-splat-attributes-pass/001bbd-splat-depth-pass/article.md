# Splat Depth Pass

Also called: Front-Face Depth Pass, Visibility Pass

## Overview
The Splat Depth Pass computes the depth of the front surface of a spherical splat for each fragment and writes it to the depth buffer to establish visibility for subsequent blending passes [S1, unofficial]. It performs this calculation per-fragment during rasterization, differing from [Radix Sort for Splat Depth](1207-radix-sort-for-splat-depth.md) which pre-sorts primitives, and from [Tile Binning for Splatting](1214-tile-binning-for-splatting.md) which groups fragments into tiles [G].

## How it works
**Inputs:** Fragment shader context including `gl_FragCoord` (window-space coordinates), `gl_PointCoord` (texture coordinates within the point sprite, range [0,1]), and vertex attributes `i.radius` (splat radius in world space) and `linearDepth` (a function or attribute providing the depth of the splat center) [S1, unofficial].
**Outputs:** `gl_FragDepth` (the depth value written to the depth buffer) [S1, unofficial].

**Data Structures:**
*   `gl_FragCoord`: `vec4`, `xy` are window coordinates, `z` is linear depth of the center [S1, unofficial].
*   `gl_PointCoord`: `vec2`, normalized coordinates within the point sprite [S1, unofficial].
*   `i.radius`: `float`, radius of the splat in world units [S1, unofficial].

**Steps:**
1.  **Calculate local offset:** Compute the vector from the center of the point sprite to the current fragment in point-space coordinates [S1, unofficial].
    $$ \mathbf{p}_{local} = \text{gl\_PointCoord} - \begin{bmatrix} 0.5 \\ 0.5 \end{bmatrix} $$
2.  **Discard outside circle:** Calculate the squared magnitude of the local offset [S1, unofficial].
    $$ d^2 = \mathbf{p}_{local} \cdot \mathbf{p}_{local} $$
    $$ \text{if } d^2 > 0.25 \text{ then discard} $$
3.  **Calculate world-space offset:** Scale the local offset to world space using the splat radius [S1, unofficial].
    $$ \mathbf{p}_{world\_xy} = \mathbf{p}_{local} \times 2 \times i.radius $$
4.  **Calculate depth offset:** Compute the $z$-offset from the center to the front surface of the sphere at the given $(x,y)$ position [S1, unofficial].
    $$ z_{offset} = \sqrt{i.radius^2 - (\mathbf{p}_{world\_xy} \cdot \mathbf{p}_{world\_xy})} $$
5.  **Compute final depth:** Subtract the depth offset from the center depth to get the front surface depth [S1, unofficial].
    $$ z_{front} = \text{linearDepth}(\text{gl\_FragCoord}.z) - z_{offset} + \epsilon $$
    $$ \text{gl\_FragDepth} = z_{front} $$

**Parameters:**
*   $\epsilon$: Epsilon value added to prevent precision artifacts. Typical value: $0.25$ (derived from `_epsilonMultiplier` in implementation [S1, unofficial]).

**Edge Cases and Pitfalls:**
*   **Z-fighting:** Without the epsilon shift, overlapping splats with similar depths may cause flickering or incorrect occlusion [S1, unofficial].
*   **Precision:** The square root calculation assumes the splat is spherical and aligned with the view [G].
*   **Boundary:** The discard condition ensures no fragments are processed outside the splat's circular boundary [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splat_depth_pass(gl_frag_coord, gl_point_coord, i_radius, linear_depth_func, epsilon=0.25):
    # Step 1: Calculate local offset from center (0.5, 0.5)
    from_center_pt_spc = gl_point_coord - vec2(0.5, 0.5)
    
    # Step 2: Calculate squared magnitude
    mag_sq = dot(from_center_pt_spc, from_center_pt_spc)
    
    # Discard if outside the unit circle (radius 0.5, squared 0.25)
    if mag_sq > 0.25:
        discard_fragment()
        
    # Step 3: Calculate world-space XY offset
    from_center_wld_spc = from_center_pt_spc * 2.0 * i_radius
    
    # Step 4: Calculate depth offset (z-component of sphere surface)
    dot_xy = dot(from_center_wld_spc, from_center_wld_spc)
    depth_offset = sqrt(i_radius * i_radius - dot_xy)
    
    # Step 5: Calculate final depth
    center_depth = linear_depth_func(gl_frag_coord.z)
    final_depth = center_depth - depth_offset + epsilon
    
    return final_depth

def discard_fragment():
    # Standard fragment shader operation to skip further processing
    raise StopIteration
```

## Subtopics

### [Single-Layer Ternary Depth Test](4509-single-layer-ternary-depth-test.md)
A rasterization technique for point-sampled surfaces that maintains only a single surface layer per pixel, using a depth band to decide whether to blend, replace, or discard incoming fragments. It differs from multi-layer approaches by strictly limiting visibility to one depth value per pixel, which introduces visibility and occlusion artifacts in complex geometry. Choose it when rendering simple point clouds where memory is constrained and complex occlusion is not required.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The method is implemented in point cloud rendering pipelines using standard fragment shaders to compute front-face depth for spherical splats [S1, unofficial]. Specific historical origins or earlier papers are not given in the sources.

## Sources
[S1] arminkz/PointCloudRendering, https://github.com/arminkz/PointCloudRendering, fetched 2026-10-28 (unofficial)
