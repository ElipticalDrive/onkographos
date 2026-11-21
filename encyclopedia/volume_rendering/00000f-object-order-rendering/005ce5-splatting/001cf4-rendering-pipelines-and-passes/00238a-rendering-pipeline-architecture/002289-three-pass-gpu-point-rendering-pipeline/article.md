# Three-Pass GPU Point Rendering Pipeline

Also called: Hardware Accelerated Point Splatting Pipeline

## Overview
The Three-Pass GPU Point Rendering Pipeline is a method for hardware-accelerated point splatting that correctly handles transparency and perspective distortion by separating depth testing, color accumulation, and normalization into distinct rendering stages [S1, unofficial]. It differs from single-pass approaches by explicitly accumulating filter weights to enable accurate normalization of the final color, ensuring that the contribution of each splat is weighted by its actual footprint area under perspective projection [S1, unofficial].

## How it works
The pipeline renders a volume represented as a set of points (splats) using standard GPU rasterization capabilities, specifically leveraging `GL_POINTS` primitives [S1, unofficial].

### Inputs and Outputs
*   **Input:** A set of $N$ 3D points $\mathbf{p}_i$, each associated with a color $\mathbf{c}_i$ and a reconstruction kernel (typically Gaussian) with variance $\sigma^2$.
*   **Output:** A 2D image where each pixel contains the normalized color value $\mathbf{C}_{final}(x,y)$.

### Data Structures
The method relies on standard GPU framebuffers:
1.  **Depth Buffer:** Stores the depth $z$ of the closest splat fragment.
2.  **Color Accumulation Buffer (Pass 2):** Stores the weighted sum of colors $\sum \mathbf{c}_i w_i$.
3.  **Weight Accumulation Buffer (Pass 2):** Stores the sum of weights $\sum w_i$.
    *   *Note:* In some implementations, the color and weight buffers may be combined or handled via separate render targets. The core requirement is that the second pass accumulates both values.

### Steps

1.  **Pass 1: Depth Offset Rendering**
    *   Render the scene using `GL_POINTS`.
    *   The vertex shader computes the screen-space position and the depth.
    *   Crucially, the depth value written to the depth buffer is offset slightly along the viewing ray. This offset ensures that during Pass 2, the depth test correctly identifies the front-most splat without self-occlusion artifacts caused by floating-point precision limits at the exact surface [S1, unofficial].
    *   No color or weight accumulation occurs in this pass; only the depth buffer is updated.

2.  **Pass 2: Weighted Accumulation**
    *   Render the splats again using `GL_POINTS`.
    *   **Depth Test:** Enable depth testing. Use the depth value from the splat's center (or a consistent reference point) to compare against the depth buffer populated in Pass 1. Only fragments that pass the depth test (i.e., are front-most) are processed.
    *   **Fragment Shader:**
        *   Compute the 2D footprint of the splat on the screen. For perspective-accurate splatting, this involves computing the variance matrix of the resampling filter in screen space, often denoted as $R_0 + H$ or approximated as $Q_{000}$ [S1, unofficial].
        *   Calculate the weight $w_i$ for the fragment based on the distance from the splat center and the kernel variance.
        *   Accumulate the weighted color: $C_{acc} \leftarrow C_{acc} + \mathbf{c}_i w_i$.
        *   Accumulate the weight: $W_{acc} \leftarrow W_{acc} + w_i$.
    *   **Blending:** Use additive blending (`GL_ADD`) for both color and weight accumulators. The depth buffer is read but not written to (depth mask disabled or depth test only) [S1, unofficial].

3.  **Pass 3: Normalization**
    *   Render a full-screen quad (or use a post-processing pass).
    *   For each pixel, retrieve the accumulated color $C_{acc}$ and accumulated weight $W_{acc}$.
    *   Compute the final color:
        $$ \mathbf{C}_{final} = \frac{C_{acc}}{W_{acc}} $$
    *   Handle the edge case where $W_{acc} \approx 0$ (no splats covered the pixel) by setting $\mathbf{C}_{final}$ to a background color or zero [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pass 1: Depth Offset
def pass1_depth_offset(points, view_matrix, proj_matrix):
    """
    Renders points to update only the depth buffer with an offset.
    """
    gl.depth_mask(True)
    gl.color_mask(False, False, False, False)
    gl.enable(gl.DEPTH_TEST)
    
    for point in points:
        # Transform point to clip space
        clip_pos = proj_matrix * view_matrix * point.position
        
        # Compute depth offset along viewing ray
        # The offset is small, e.g., epsilon * depth
        depth_offset = compute_viewing_ray_offset(clip_pos)
        
        # Write depth with offset
        gl.depth_func(gl.LEQUAL)
        draw_point(clip_pos.position.xy, depth=clip_pos.z + depth_offset)

# Pass 2: Accumulation
def pass2_accumulate(points, view_matrix, proj_matrix):
    """
    Renders points to accumulate weighted color and weights.
    """
    gl.depth_mask(False) # Do not update depth buffer
    gl.color_mask(True, True, True, True)
    gl.enable(gl.BLEND)
    gl.blend_func(gl.ONE, gl.ONE) # Additive blending
    
    # Separate render targets for Color Accum and Weight Accum
    # Or use a single texture with RGB for color, A for weight
    
    for point in points:
        # Compute screen-space footprint (EWA or similar)
        # This requires inverting the projection matrix to get kernel variance
        # Q_000 approximation or full variance matrix R_0 + H
        variance_matrix = compute_screen_space_variance(point, view_matrix, proj_matrix)
        
        # Determine if splat is visible (passed depth test from Pass 1)
        # The GPU depth test handles this automatically if depth buffer is intact
        if is_visible(point):
            # Compute weight w based on fragment distance and variance
            w = compute_kernel_weight(fragment_uv, point.center, variance_matrix)
            
            # Fragment shader accumulates:
            # color_accum += point.color * w
            # weight_accum += w
            draw_point(point.position, weight=w)

# Pass 3: Normalization
def pass3_normalize():
    """
    Post-process the accumulated buffers to produce final image.
    """
    gl.depth_mask(False)
    gl.color_mask(True, True, True, True)
    gl.disable(gl.BLEND)
    
    # Render a full-screen quad
    # Fragment shader:
    # C_final = color_accum / weight_accum
    # if weight_accum < epsilon: C_final = background_color
    
    render_fullscreen_quad(normalize_fragment_shader)
```

### Parameters and Edge Cases
*   **Depth Offset:** The offset in Pass 1 must be small enough to not cause visible depth errors but large enough to prevent z-fighting in Pass 2. Typical values are a small fraction of the near plane distance [S1, unofficial].
*   **Variance Matrix Inversion:** Computing the screen-space kernel variance requires inverting the projective mapping. This can be numerically ill-conditioned if the splat degenerates into a line segment (viewing direction parallel to the splat normal) [S1, unofficial].
*   **Zero Weight:** Division by zero in Pass 3 must be guarded against. If no splats cover a pixel, the weight is zero [S1, unofficial].

## Subtopics

### Standard Point Splatting Assembly
This variant refers to the core assembly logic often used in simpler or CPU-based implementations, or as a baseline for GPU pipelines. It involves initializing canvases (depth, color, label, index), sorting points by depth, and iterating to write colors to pixels where the point's depth is less than the current depth at that pixel. A written mask is created to identify valid pixels, and invalid pixels are cleared [S2, unofficial].

### [Point Cloud Rendering](6750-point-cloud-rendering.md)
Point Cloud Rendering projects a set of 3D points into screen space and aggregates their contributions. It is distinguished by its differentiability, allowing gradients to flow from the rendered image back to the 3D point positions and attributes, which is useful for optimization tasks.

### [Deferred Splatting with Temporal Coherency and Bi-Bi-1 Visibility Pass](12274-deferred-splatting-with-temporal-coherency-and-bi-bi-1-visib.md)
This method reduces performance costs by performing expensive Elliptical Weighted Average (EWA) splatting only on visible surfels. It uses a multi-pass approach similar to deferred shading, separating visibility determination from color accumulation. It employs temporal coherency to accelerate the visibility pass and a secondary visibility pass ($B_i - B_{i-1}$) to fix artifacts caused by hidden points becoming visible. This allows expensive filtering to be skipped for hidden points.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for $N$ points, assuming constant-time kernel evaluation and rasterization. The three-pass structure adds a constant overhead factor of 3 to the rendering pipeline.
*   **Memory Complexity:** Requires additional framebuffers for color and weight accumulation. Typically, two additional textures (or one texture with multiple channels) are needed.
*   **Strengths:** Correctly handles perspective distortion and transparency. Leverages standard GPU hardware for rasterization and depth testing.
*   **Limitations:** Requires multiple passes, which can be inefficient on hardware with limited render target support or high overhead for state changes. Numerical instability in variance matrix inversion can occur for grazing angles [S1, unofficial].
*   **When to Use:** When high-quality perspective-accurate point splatting is required and the GPU supports multiple render targets or efficient post-processing.
*   **When Not to Use:** For simple orthographic projections where single-pass methods suffice, or on hardware with severe limitations on framebuffer operations.

## Background
The three-pass approach for hardware-accelerated point splatting was developed to address the limitations of earlier methods that struggled with perspective accuracy and transparency. It builds on the principles of splatting, where volumetric data is decomposed into primitives and rasterized [S1, unofficial]. The specific technique of using a depth offset pass and separate accumulation/normalization passes was introduced to ensure correct weighting under perspective projection, as described in "Perspective Accurate Splatting" [S1, unofficial].

## Sources
[S1] Perspective Accurate Splatting, http://infoscience.epfl.ch/record/149345, fetched 2026-10-26 (unofficial)
[S2] Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, https://github.com/Luhao03/Point-Splatting-Multi-View-2D-Rendering-Pipeline, fetched 2026-10-28 (unofficial)
