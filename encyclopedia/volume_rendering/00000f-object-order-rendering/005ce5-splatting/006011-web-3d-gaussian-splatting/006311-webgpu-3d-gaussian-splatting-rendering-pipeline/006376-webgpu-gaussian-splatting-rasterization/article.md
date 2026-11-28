# WebGPU Gaussian Splatting Rasterization

Also called: Gaussian Splatting via Quad Rasterization, WebGPU Gaussian Viewer

## Overview
This method renders 3D Gaussian splats by mapping them to standard triangle rasterization, converting each Gaussian into a flat rectangle (quad) facing the camera, and drawing these quads with a fragment shader [S1, unofficial]. It avoids the compute-shader-based pixel accumulation of the original implementation to achieve substantially better framerates on WebGPU [S1, unofficial]. It builds on [WebGPU 3D Gaussian Splatting Rendering Pipeline](18792-webgpu-3d-gaussian-splatting-rendering-pipeline.md).

## How it works
**Inputs:**
- 3D Gaussian parameters: mean $\mu$, covariance $\Sigma$ (or rotation $R$ and scale $S$), opacity $\alpha$, spherical harmonic coefficients for color.
- Camera matrices: View matrix $V$, Projection matrix $P$.
- Camera intrinsics: Focal lengths $f_x, f_y$, half-field-of-view tangents $\tan(\theta_x/2), \tan(\theta_y/2)$.
- Scene scale modifier.

**Outputs:**
- A 2D image with accumulated color and alpha.

**Data Structures:**
1.  **Uniform Buffer (`uniformLayout`):** Contains camera and projection data. The layout is defined as:
    -   `viewMatrix`: `mat4x4<f32>`
    -   `projMatrix`: `mat4x4<f32>`
    -   `cameraPosition`: `vec3<f32>`
    -   `tanHalfFovX`: `f32`
    -   `tanHalfFovY`: `f32`
    -   `focalX`: `f32`
    -   `focalY`: `f32`
    -   `scaleModifier`: `f32`
2.  **Index Buffer:** Size is $6 \times 4 \times N$ bytes, where $N$ is the number of Gaussians. This buffer stores indices for drawing triangles. Each Gaussian is represented by 4 vertices (forming 2 triangles), requiring 6 indices per Gaussian.
3.  **Render Target:** Presentation format `rgba16float` [S1, unofficial].

**Numbered Steps:**
1.  **Quad Generation (Vertex Shader):** For each Gaussian, compute the 2D bounding box in screen space. Generate a quad (two triangles) that tightly bounds the projected Gaussian ellipse. The quad must face the camera. The vertex shader transforms the Gaussian's 3D center and covariance into screen space to determine the quad's vertices [S1, unofficial].
2.  **Depth Sorting:** Sort the generated quads by their depth (distance from camera) on the CPU [S1, unofficial]. This ensures correct alpha blending order.
3.  **Index Buffer Update:** Copy the sorted indices into the GPU index buffer. The indices map to the quad vertices generated in step 1.
4.  **Rasterization and Blending:** Draw the sorted quads using `triangle-list` topology. The fragment shader evaluates the Gaussian density at each pixel covered by the quad.
5.  **Blending:** Accumulate colors using the blend equation:
    $$color_{new} = src.color \times src.alpha + dst.color \times (1 - src.alpha)$$
    This is achieved by setting the WebGPU blend state to `srcFactor: "one-minus-dst-alpha"` and `dstFactor: "one"` with `operation: "add"` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_gaussians(gaussians, view_matrix, proj_matrix, camera_position, 
                     tan_half_fov_x, tan_half_fov_y, focal_x, focal_y, 
                     scale_modifier, width, height):
    """
    Renders 3D Gaussians using WebGPU rasterization.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
            - mean: vec3
            - rotation: quat or mat3
            - scale: vec3
            - opacity: f32
            - sh_coeffs: array of f32 (spherical harmonics)
        view_matrix: mat4x4
        proj_matrix: mat4x4
        camera_position: vec3
        tan_half_fov_x: f32
        tan_half_fov_y: f32
        focal_x: f32
        focal_y: f32
        scale_modifier: f32
        width: int
        height: int
        
    Returns:
        Image buffer with accumulated color and alpha.
    """
    # Step 1: Generate quads for each Gaussian
    quads = []
    for i, g in enumerate(gaussians):
        # Transform Gaussian center to camera space
        center_cam = view_matrix * vec4(g.mean, 1.0)
        
        # Project to screen space to get 2D covariance
        # Note: Exact projection of covariance matrix involves Jacobian
        # J = d(p)/d(x) where p is screen position, x is 3D position
        # Sigma_2d = J * Sigma_3d * J^T
        # For simplicity, we assume the vertex shader handles the full transformation
        # and we just need the bounding quad here for sorting.
        
        # Compute 2D bounding box in clip space
        # This requires projecting the covariance ellipse
        # For implementation, we generate 4 vertices for the quad
        # The quad is axis-aligned in screen space for simplicity in this method
        # or oriented along the major axis of the projected ellipse.
        
        # Calculate depth for sorting (z-coordinate in camera space)
        depth = center_cam.z
        
        # Generate quad vertices (4 vertices per Gaussian)
        # The vertex shader will handle the actual screen-space positioning
        # based on the Gaussian parameters passed as attributes.
        # Here we just store the index and depth for sorting.
        quad_index = i
        quads.append({
            'index': quad_index,
            'depth': depth,
            'vertices': generate_quad_vertices(g, view_matrix, proj_matrix, 
                                               tan_half_fov_x, tan_half_fov_y, 
                                               focal_x, focal_y, scale_modifier, 
                                               width, height)
        })
    
    # Step 2: Sort quads by depth (back to front)
    # The source notes mention CPU sorting is a bottleneck [S1, unofficial]
    quads.sort(key=lambda q: q['depth'], reverse=True)
    
    # Step 3: Prepare index buffer
    # Each quad is 2 triangles, 6 indices. 4 vertices per quad.
    # Indices are 0-3 for each quad.
    num_gaussians = len(quads)
    index_buffer_data = []
    for quad in quads:
        base_index = quad['index'] * 4
        # Two triangles: (0,1,2) and (0,2,3) or similar winding
        indices = [base_index, base_index + 1, base_index + 2,
                   base_index, base_index + 2, base_index + 3]
        index_buffer_data.extend(indices)
    
    # Upload index buffer to GPU
    gpu_index_buffer = create_gpu_buffer(index_buffer_data)
    
    # Step 4: Set up render pass
    # Create bind group with uniform buffer and Gaussian attributes
    uniform_data = {
        'viewMatrix': view_matrix,
        'projMatrix': proj_matrix,
        'cameraPosition': camera_position,
        'tanHalfFovX': tan_half_fov_x,
        'tanHalfFovY': tan_half_fov_y,
        'focalX': focal_x,
        'focalY': focal_y,
        'scaleModifier': scale_modifier
    }
    uniform_buffer = create_gpu_buffer(uniform_data)
    
    # Create bind group layout and bind group
    bind_group = create_bind_group(uniform_buffer, gaussians_attributes_buffer)
    
    # Step 5: Encode draw commands
    encoder = create_command_encoder()
    render_pass = encoder.begin_render_pass(
        color_attachments=[{
            'view': target_texture_view,
            'load_op': 'clear',
            'store_op': 'store',
            'clear_value': (0.0, 0.0, 0.0, 0.0),
            'blend_state': {
                'color': {
                    'src_factor': 'one-minus-dst-alpha',
                    'dst_factor': 'one',
                    'operation': 'add'
                },
                'alpha': {
                    'src_factor': 'one-minus-dst-alpha',
                    'dst_factor': 'one',
                    'operation': 'add'
                }
            }
        }]
    )
    
    render_pass.set_pipeline(rasterization_pipeline)
    render_pass.set_bind_group(0, bind_group)
    render_pass.set_index_buffer(gpu_index_buffer, format='uint32')
    
    # Draw all quads
    render_pass.draw_indexed(6 * num_gaussians, 1, 0, 0, 0)
    render_pass.end()
    
    # Submit commands
    command_buffer = encoder.finish()
    queue.submit([command_buffer])
    
    # Read back result
    return read_texture(target_texture)

def generate_quad_vertices(gaussian, view_matrix, proj_matrix, 
                           tan_half_fov_x, tan_half_fov_y, 
                           focal_x, focal_y, scale_modifier, 
                           width, height):
    """
    Generates 4 vertices for the quad representing a Gaussian.
    This is a simplified placeholder. The actual implementation
    likely computes the bounding box of the projected 2D Gaussian
    in screen space.
    """
    # Transform center to camera space
    center_cam = view_matrix * vec4(gaussian.mean, 1.0)
    
    # Project center to NDC
    center_ndc = proj_matrix * center_cam
    center_ndc /= center_ndc.w
    
    # Compute 2D covariance in screen space
    # This involves the Jacobian of the projection
    # For a 3D Gaussian with covariance Sigma, the 2D covariance is:
    # Sigma_2d = T * Sigma * T^T
    # where T is the Jacobian of the projection transformation.
    
    # Approximate bounding box in screen space
    # The width and height of the quad are determined by the standard deviations
    # of the projected Gaussian, scaled by a factor (e.g., 3 sigma).
    
    # Calculate scale factors
    # The scale modifier is applied to the Gaussian's scale
    effective_scale = gaussian.scale * scale_modifier
    
    # Project the scale to screen space
    # This is non-trivial and depends on the orientation of the Gaussian
    # For a simple axis-aligned quad, we might use the max extent
    
    # For the purpose of this article, we assume the vertex shader
    # receives the Gaussian parameters and computes the quad vertices
    # based on the uniform parameters.
    
    # Return placeholder vertices
    return [vec2(0,0), vec2(1,0), vec2(1,1), vec2(0,1)]
```

**Edge Cases and Pitfalls:**
-   **Sorting Performance:** Sorting quads by depth on the CPU is a significant performance bottleneck compared to GPU-based sorting (e.g., CUDA implementations) [S1, unofficial]. It is recommended to replace this with a compute shader sort if possible.
-   **Precision:** The use of `rgba16float` for the render target provides sufficient precision for most scenes, but extreme dynamic ranges may require higher precision formats.
-   **Degenerate Gaussians:** Gaussians with very small scales or zero opacity should be culled to avoid rendering artifacts and wasted computation.

## Operating characteristics
-   **Time Complexity:** Dominated by the CPU sorting step, which is $O(N \log N)$ where $N$ is the number of Gaussians. The rasterization step is $O(P)$ where $P$ is the number of pixels covered by the quads.
-   **Memory Complexity:** $O(N)$ for storing Gaussian parameters and index buffers.
-   **Measured Performance:** Yields substantially better framerates than compute-shader-based implementations on WebGPU [S1, unofficial].
-   **Strengths:** Leverages standard rasterization hardware, which is highly optimized. Simple to implement in WebGPU.
-   **Limitations:** CPU sorting is slow. Quad approximation may introduce artifacts for highly anisotropic Gaussians.
-   **When to Use:** When targeting WebGPU and prioritizing simplicity and compatibility over maximum performance.
-   **When Not to Use:** When high performance is critical and GPU-based sorting is available.

## Background
This method was developed as part of the "WebGPU Viewer for 3D Gaussian Splatting Real-Time Radiance Field Rendering" project [S1, unofficial]. It was created to enable real-time rendering of 3D Gaussian Splatting in web browsers using WebGPU, avoiding the limitations of compute shaders for this specific use case.

## Sources
[S1] WebGPU Viewer for 3D Gaussian Splatting Real-Time Radiance Field Rendering, https://github.com/cvlab-epfl/gaussian-splatting-web, fetched 2026-10-26 (unofficial)
