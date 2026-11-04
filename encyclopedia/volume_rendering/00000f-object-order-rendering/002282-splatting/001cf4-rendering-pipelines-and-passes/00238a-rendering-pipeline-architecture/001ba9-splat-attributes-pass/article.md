# Splat Attributes Pass

Also called: Blending Pass, G-Buffer Pass

## Overview
The Splat Attributes Pass is a fragment-shader-based rendering step in object-order splatting that computes visibility-weighted attributes (color, normal, and accumulation weight) for each splat fragment and writes them to a G-Buffer. It solves the problem of correctly compositing overlapping translucent splats by discarding fragments occluded by opaque geometry and weighting contributions based on radial distance from the splat center and proximity to the visible surface. It differs from the [Single-Pass Splat Rendering Pipeline](1254-single-pass-splat-rendering-architecture.md) by decoupling attribute accumulation from the final color buffer write, allowing for deferred shading or multi-pass compositing, and from the [Three-Pass GPU Point Rendering Pipeline](2837-three-pass-gpu-point-rendering-architecture.md) by explicitly handling depth-based occlusion against a pre-existing depth buffer within the fragment shader rather than relying solely on hardware depth testing or separate sorting passes for visibility.

## How it works
**Inputs and Outputs:**
*   **Input:** A splat primitive (point sprite) with attributes: center position in view space, radius in world space, color ($C$), and linear depth ($D_{linear}$).
*   **Input:** A depth texture ($g\_depth$) populated by the [Splat Depth Pass](1078-splat-depth-pass.md) or a geometry pass, representing the closest surface depth at each screen coordinate.
*   **Output:** G-Buffer textures:
    *   $g\_normal$: `vec4` at location 0, storing weighted normals in RGB and unused in A.
    *   $g\_color$: `vec4` at location 1, storing weighted color in RGB and accumulated weight in A.

**Data Structures:**
*   **G-Buffer Layout:**
    *   `g_normal`: `vec4` (RGBA32F). RGB stores $\sum w_i \mathbf{n}_i$. A is unused or zero.
    *   `g_color`: `vec4` (RGBA32F). RGB stores $\sum w_i \mathbf{c}_i$. A stores $\sum w_i$.

**Parameters:**
*   $\epsilon$: Small constant for depth-proximity falloff normalization. Typical value: $0.25$ [S1, unofficial].

**Steps:**
1.  **Fragment Position Calculation:** Calculate the offset of the current fragment from the center of the point sprite in screen space.
    $$ \mathbf{p}_{offset} = \text{gl\_PointCoord} - [0.5, 0.5] $$
2.  **Circular Masking:** Calculate the squared magnitude of the offset. Discard the fragment if it lies outside the unit circle.
    $$ m^2 = \mathbf{p}_{offset} \cdot \mathbf{p}_{offset} $$
    If $m^2 > 0.25$, discard.
3.  **World-Space Offset Conversion:** Convert the screen-space offset to a world-space offset using the splat's radius.
    $$ \mathbf{p}_{offset\_world} = \mathbf{p}_{offset} \cdot 2 \cdot r $$
    where $r$ is the splat radius.
4.  **Depth Offset Calculation:** Calculate the depth offset from the splat's spherical surface to the fragment.
    $$ d_{offset} = \sqrt{r^2 - (\mathbf{p}_{offset\_world} \cdot \mathbf{p}_{offset\_world})} $$
5.  **Fragment Depth in View Space:** Compute the fragment's depth in view space by subtracting the surface offset from the linear depth of the splat center.
    $$ D_{frag} = D_{linear} - d_{offset} $$
6.  **Surface Depth Sampling:** Sample the depth buffer at the current screen coordinates to retrieve the closest surface depth.
    $$ \mathbf{uv} = \frac{\text{gl\_FragCoord}.xy}{\text{viewport\_size}} $$
    $$ D_{surface} = \text{linearDepth}(\text{texture}(g\_depth, \mathbf{uv}).x) $$
    *Note: `linearDepth` converts the normalized depth buffer value back to linear view-space depth.*
7.  **Occlusion Test:** Discard the fragment if the splat fragment is behind the existing surface geometry.
    If $D_{frag} > D_{surface}$, discard.
8.  **Weight Calculation:** Compute two falloff weights and combine them.
    *   Radial falloff ($w_1$): Decreases from 1 at the center to 0 at the edge.
        $$ w_1 = 1 - \frac{\sqrt{m^2}}{0.5} $$
    *   Depth-proximity falloff ($w_2$): Increases as the splat gets closer to the visible surface, normalized by $\epsilon$.
        $$ w_2 = \frac{D_{surface} - D_{frag}}{\epsilon} $$
    *   Combined weight:
        $$ w = w_1 \cdot w_2 $$
9.  **Normal Calculation:** Compute the view-space normal at the fragment point on the splat sphere.
    $$ \mathbf{n} = \text{normalize}([\mathbf{p}_{offset\_world}.x, \mathbf{p}_{offset\_world}.y, d_{offset}]) $$
10. **Attribute Accumulation:** Accumulate the weighted attributes into the G-Buffer textures using additive blending (or atomic adds if not using blending hardware, but typically done via shader writes to textures with `imageLoad`/`imageStore` or blending).
    *   Normal:
        $$ g\_normal.xyz \leftarrow g\_normal.xyz + w \cdot \mathbf{n} $$
    *   Color:
        $$ g\_color.rgb \leftarrow g\_color.rgb + w \cdot \mathbf{c} $$
    *   Weight:
        $$ g\_color.a \leftarrow g\_color.a + w $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader Logic for Splat Attributes Pass
# Inputs:
#   gl_PointCoord: vec2, fragment position within point sprite [0,1]
#   i.radius: float, splat radius in world space
#   i.linearDepth: float, linear depth of splat center in view space
#   i.color: vec3, splat color
#   g_depth: sampler2D, depth buffer texture
#   viewport_size: vec2, resolution of the viewport
#   epsilon: float, depth proximity falloff constant (e.g., 0.25)
# Outputs:
#   g_normal: vec4, weighted normal accumulation
#   g_color: vec4, weighted color and weight accumulation

void main() {
    // 1. Calculate offset from center in screen space
    vec2 from_center_pt_spc = gl_PointCoord - vec2(0.5, 0.5);
    
    // 2. Calculate squared magnitude
    float mag_sq = dot(from_center_pt_spc, from_center_pt_spc);
    
    // 3. Discard if outside the point circle
    if (mag_sq > 0.25) {
        discard;
    }
    
    // 4. Convert screen-space offset to world-space offset
    vec2 from_center_wld_spc = from_center_pt_spc * 2.0 * i.radius;
    
    // 5. Calculate depth offset from the sphere surface
    float depth_offset = sqrt(i.radius * i.radius - dot(from_center_wld_spc, from_center_wld_spc));
    
    // 6. Calculate fragment depth in view space
    float depth = i.linearDepth - depth_offset;
    
    // 7. Sample the depth buffer to get surface depth
    vec2 tex_coords = gl_FragCoord.xy / viewport_size;
    float surface_depth = linearDepth(texture(g_depth, tex_coords).x);
    
    // 8. Discard if splat is behind the surface
    if (depth > surface_depth) {
        discard;
    }
    
    // 9. Calculate radial falloff weight
    float w1 = 1.0 - (sqrt(mag_sq) / 0.5);
    
    // 10. Calculate depth-proximity falloff weight
    float w2 = (surface_depth - depth) / epsilon;
    
    // 11. Combine weights
    float weight = w1 * w2;
    
    // 12. Compute view-space normal from geometry
    vec3 normal = normalize(vec3(from_center_wld_spc, depth_offset));
    
    // 13. Accumulate weighted normal into g_normal
    // Note: In a real implementation, this requires imageStore or blending
    g_normal.xyz += weight * normal;
    
    // 14. Accumulate weighted color into g_color
    vec3 color = weight * i.color;
    g_color.rgb += color;
    
    // 15. Accumulate weight into g_color alpha
    g_color.a += weight;
}

// Helper function to convert normalized depth to linear view depth
float linearDepth(float z) {
    // Implementation depends on projection matrix and near/far planes
    // Typically: z_linear = (2.0 * near * far) / (far + near - z * (far - near))
    // For simplicity, assuming linear depth is already provided or standard conversion
    return z; 
}
```

**Edge Cases and Pitfalls:**
*   **Precision:** Linear depth conversion must match the depth buffer format. Mismatches cause incorrect occlusion tests [S1, unofficial].
*   **Boundaries:** The radial falloff $w_1$ goes to 0 at the edge, ensuring smooth blending but requiring careful handling of floating-point precision near the boundary to avoid artifacts.
*   **Degenerate Input:** If $r=0$, the splat has no area. The code handles this via the discard condition, but rendering zero-radius splats may cause division by zero or undefined behavior if not guarded.

## Subtopics

### Tile Binning for Splatting
Tile Binning for Splatting is a preprocessing step that groups depth-sorted splats into tile buckets (typically 16x16 pixel blocks) to improve cache locality during the compositing phase [S2, unofficial]. It uses a count-then-scatter two-pass approach: Pass 1 initializes bin offsets and counts the number of splats per tile based on their bounding boxes, then performs a prefix sum to determine start indices. Pass 2 scatters splat indices into a flat array organized by tile [S2, unofficial]. This method is chosen when memory bandwidth is a bottleneck during compositing, as it ensures that splats contributing to a specific tile are processed contiguously in memory. It differs from standard global sorting by reducing the scope of memory accesses during the attribute accumulation and final blending passes.

### Front-to-Back Blending Setup
Front-to-Back Blending Setup configures the GPU state for the subsequent compositing pass that uses the G-Buffer data [S3, unofficial]. It involves clearing the color buffer to RGBA(0,0,0,0) to ensure zero initial alpha, disabling depth testing, and enabling blending with specific factors: RGB uses `GL_ONE_MINUS_DST_ALPHA` / `GL_ONE`, and Alpha uses `GL_ONE_MINUS_DST_ALPHA` / `GL_ONE`, with `GL_FUNC_ADD` [S3, unofficial]. This setup ensures that splats are composited in front-to-back order (enforced by prior sorting) to correctly accumulate color and alpha. It is chosen when the rendering pipeline requires hardware-accelerated alpha blending for the final image generation, rather than manual accumulation in a compute shader.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of splat fragments. Each fragment performs a constant number of arithmetic operations and texture samples.
*   **Memory Complexity:** $O(W \times H)$ for the G-Buffer textures, where $W$ and $H$ are the viewport dimensions.
*   **Measured Performance:** Performance is heavily dependent on fill rate and texture bandwidth. On typical hardware (e.g., NVIDIA RTX 30-series), rendering 200k splats at 1080p takes approximately 10-20ms for the attribute pass, depending on the complexity of the depth buffer sampling and the fill rate [S2, unofficial].
*   **Strengths:** Explicit control over occlusion and weighting allows for high-quality rendering of translucent surfaces. Decoupling attributes from the final color buffer enables deferred shading techniques.
*   **Limitations:** Requires a pre-populated depth buffer, adding a dependency on the [Splat Depth Pass](1078-splat-depth-pass.md). The fragment shader logic is more complex than simple point sprites, potentially causing pipeline stalls if the depth buffer sampling is not cached efficiently.
*   **When to Use:** Use when high-quality translucent rendering is required and the pipeline supports multi-pass rendering with G-Buffers. Avoid when hardware resources are extremely limited and a single-pass solution is mandatory.

## Background
The Splat Attributes Pass originates from the field of point-based rendering and Gaussian Splatting, where 3D primitives are projected onto the 2D screen and rendered as fragments. The specific technique of using depth-based occlusion and radial/depth-proximity weighting for attribute accumulation was detailed in unofficial implementations such as `arminkz/PointCloudRendering` [S1, unofficial]. It builds on the principles of object-order rendering, where splats are processed in screen space order, and has been adapted for real-time rendering of large point clouds and Gaussian splats. The method is commonly used in real-time viewers for 3D Gaussian Splatting and point cloud visualization tools.

## Sources
[S1] arminkz/PointCloudRendering, https://github.com/arminkz/PointCloudRendering, fetched 2026-10-28 (unofficial)
[S2] Tsplat – Run Gaussian splatting in your terminal, https://github.com/darshanmakwana412/tsplat, fetched 2026-10-26 (unofficial)
[S3] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
