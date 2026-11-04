# Splat Depth Pass

Also called: Front-Face Depth Pass, Visibility Pass

## Overview
The Splat Depth Pass computes the front-facing depth of each splat fragment to populate the depth buffer, ensuring correct visibility ordering for subsequent compositing [S1, unofficial]. It differs from the parent [Splat Attributes Pass](1066-splat-attributes-pass.md) by focusing exclusively on geometric occlusion rather than color or opacity accumulation, and from [Radix Sort for Splat Depth](1207-radix-sort-for-splat-depth.md) by performing depth calculation per-fragment in the shader rather than sorting primitives in a pre-pass [G].

## How it works
**Inputs and Outputs:**
- Input: Fragment position in screen space (`gl_FragCoord.z`), splat radius (`i.radius`), and a small epsilon value (`epsilon`) [S1, unofficial].
- Output: Updated depth buffer value (`gl_FragDepth`) [S1, unofficial].

**Data Structures:**
- The splat attributes are passed via vertex/point shader varying `i`, containing at least:
  - `radius`: float, the radius of the splat in screen space [S1, unofficial].

**Steps:**
1. **Local Coordinate Calculation:** Compute the vector from the center of the point sprite in screen space [S1, unofficial].
   $$ \text{from\_center\_pt\_spc} = \text{gl\_PointCoord} - \begin{bmatrix} 0.5 \\ 0.5 \end{bmatrix} $$
2. **Circular Boundary Check:** Calculate the squared magnitude of the offset. If the fragment lies outside the unit circle (radius 0.5), discard it [S1, unofficial].
   $$ \text{mag\_sq} = \|\text{from\_center\_pt\_spc}\|^2 $$
   $$ \text{if } \text{mag\_sq} > 0.25 \text{ then discard} $$
3. **World-Space Offset:** Scale the local offset by the splat's screen-space radius to determine the offset in world space (assuming linear depth mapping for the offset calculation) [S1, unofficial].
   $$ \text{from\_center\_wld\_spc} = \text{from\_center\_pt\_spc} \times 2 \times i.\text{radius} $$
4. **Depth Offset Calculation:** Compute the z-offset from the front surface of the sphere to the current fragment position on the sphere's surface. This uses the Pythagorean theorem on the splat's radius [S1, unofficial].
   $$ \text{depth\_offset} = \sqrt{i.\text{radius}^2 - \text{dot}(\text{from\_center\_wld\_spc}, \text{from\_center\_wld\_spc})} $$
5. **Final Depth Computation:** Shift the linear depth of the splat's center (derived from `gl_FragCoord.z`) by the negative depth offset (moving towards the camera) and add an epsilon bias [S1, unofficial].
   $$ \text{depth} = \text{depthSample}(\text{linearDepth}(\text{gl\_FragCoord}.z) - \text{depth\_offset} + \epsilon) $$
   Where `depthSample` converts the linear depth back to the clip-space depth required by the framebuffer, and `linearDepth` converts clip-space Z to linear view-space Z [G].
6. **Write Depth:** Write the computed depth to the fragment shader output [S1, unofficial].
   $$ \text{gl\_FragDepth} = \text{depth} $$

**Parameters:**
- `epsilon`: A small bias added to prevent z-fighting and ensure correct blending of overlapping splats. Typical value: $0.25$ (scaled by an `_epsilonMultiplier` if applicable) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Z-fighting/Gaps:** Without the epsilon shift, splats at similar depths may fail depth tests incorrectly, causing gaps or flickering [S1, unofficial].
- **Precision:** The calculation assumes `gl_FragCoord.z` can be accurately converted to linear depth [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Splat Depth Pass Fragment Shader Logic
# Inputs: gl_FragCoord (vec4), gl_PointCoord (vec2), i_radius (float), epsilon (float)
# Outputs: gl_FragDepth (float)
# Note: vec2, dot, sqrt are standard vector/math operations [G].

class ShaderDiscard(Exception):
    """Raised to terminate fragment processing for discarded pixels [G]."""
    pass

def linearDepth(clip_z):
    """Converts clip-space Z to linear view-space Z [G]."""
    # Standard OpenGL depth conversion
    return (2.0 * clip_z * 0.1 * 100.0) / (100.0 + 0.1 - clip_z * (100.0 - 0.1))

def depthSample(linear_z):
    """Converts linear view-space Z back to clip-space Z for gl_FragDepth [G]."""
    return (linear_z * (100.0 - 0.1) + 100.0 + 0.1) / (200.0)

def splat_depth_pass(gl_FragCoord, gl_PointCoord, i_radius, epsilon):
    # 1. Calculate distance from center in point sprite coordinates
    from_center_pt_spc = gl_PointCoord - vec2(0.5, 0.5)
    
    # 2. Discard if outside circle
    mag_sq = dot(from_center_pt_spc, from_center_pt_spc)
    if mag_sq > 0.25:
        raise ShaderDiscard()
        
    # 3. Calculate world-space offset (scaled by radius)
    from_center_wld_spc = from_center_pt_spc * 2.0 * i_radius
    
    # 4. Calculate depth offset (z-distance from front of sphere to current fragment)
    depth_offset = sqrt(i_radius * i_radius - dot(from_center_wld_spc, from_center_wld_spc))
    
    # 5. Calculate final depth
    linear_center_depth = linearDepth(gl_FragCoord.z)
    final_linear_depth = linear_center_depth - depth_offset + epsilon
    depth = depthSample(final_linear_depth)
    
    # 6. Write Depth
    gl_FragDepth = depth
    return depth
```

## Subtopics

### Single-Layer Ternary Depth Test
A prior approach [Weyrich et al. 2007] cited in [S2, unofficial] that maintains only a single surface layer per pixel. It applies a ternary depth test: if an incoming fragment's depth falls within an $\epsilon$-band of the stored value, it is blended into the running EWA sum; otherwise, it is replaced or discarded [S2, unofficial]. This method produces hard, opaque fringes at silhouettes because normalized output is always fully opaque where any surfel contributes, and exacerbates self-occlusion artifacts at concavities by incorrectly rejecting deeper surfels [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per fragment. The calculation involves basic arithmetic and square roots [G].
- **Memory Complexity:** $O(1)$ auxiliary memory. Only the depth buffer is written [G].
- **Strengths:** Simple to implement in hardware shaders; provides correct visibility for front-facing surfaces [G].
- **Limitations:** Does not handle multiple surface layers per pixel (see [Single-Layer Ternary Depth Test](#single-layer-ternary-depth-test) pitfall) [S2, unofficial]; relies on accurate depth buffer precision [G].

## Background
The method is derived from point-sprite rendering techniques for point clouds [S1, unofficial]. The specific implementation details, including the epsilon bias and depth offset calculation, are documented in the [PointCloudRendering](https://github.com/arminkz/PointCloudRendering) repository [S1, unofficial]. The comparison with the single-layer ternary depth test is detailed in the 3DSS paper [S2, unofficial].

## Sources
[S1] arminkz/PointCloudRendering, https://github.com/arminkz/PointCloudRendering, fetched 2026-10-28 (unofficial)
[S2] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
