# Image Space Fabric Visualization

Also called: Fabric-Like Tensor Field Visualization on Surfaces, Surface Tensor LIC

## Overview
Image Space Fabric Visualization is a method to visualize symmetric tensor fields on arbitrary non-self-intersecting surfaces by projecting tensor data into image space, advecting a noise texture along eigenvectors, and blending with lighting and edge information to create a fabric-like appearance [S1, unofficial]. It differs from CPU-based or multi-pass geometry rendering methods by performing all intermediate calculations in GPU offscreen framebuffer objects (FBOs), leading to large speed gains and real-time performance [S1, unofficial]. It builds on [Line Integral Convolution (LIC)](7940-line-integral-convolution-lic.md) by adapting the advection process to operate on surface-projected textures rather than volumetric streamlines.

## How it works
**Inputs:**
1.  Geometry: A non-self-intersecting surface with smooth normals.
2.  Tensor Field: Symmetric second-order tensors defined on the surface vertices.
3.  Noise Texture: A scalar noise field (pre-calculated or stored in a file).
4.  Camera parameters: For projection.

**Outputs:**
1.  Final image: A visualization with a fabric-like texture modulated by lighting and edge information.

**Data Structures:**
*   **Vertex Attributes:** Position, normal, and two 3D texture coordinates (for tensor information). If tensors are symmetric, 6 floating-point values per vertex are transferred [S1, unofficial].
*   **Textures:**
    *   Noise texture (input).
    *   Intermediate textures for FBO passes. Due to hardware limits (e.g., NVIDIA GeForce 8800 GTS), a maximum of four bound textures per rendering pass and active FBO are used [S1, unofficial].
    *   Edge detection output texture layout: Green channel stores depth buffer value; Blue channel stores unprocessed input noise; Alpha channel stores noise mapped to geometry's surface [S1, unofficial].

**Steps:**

1.  **Projection and Pre-computation (Vertex/Fragment Shaders):**
    *   Render the geometry into an offscreen FBO.
    *   **Vertex Shader:**
        *   Map the two main tensor directions to the surface described by the normal $n$ at the current vertex [S1, unofficial].
        *   Calculate each vertex's position in one voxel of the noise texture using equations from Section 2.3 of the source (mapping geometry to noise space) [S1, unofficial].
        *   Pre-calculate Phong light intensity $L(x,y) \in [0,1]$ at every fragment with coordinates $x$ and $y$ [S1, unofficial].
    *   **Fragment Shader:**
        *   Perform eigenvalue decomposition and eigenvector calculation *after* interpolation to avoid sign flip artifacts [S1, unofficial].
        *   Project eigenvectors to image space [S1, unofficial].
        *   Normalize eigenvectors for texture storage. Since $\lambda_1 \ge \lambda_2$, normalize the vector $(\lambda_1, \lambda_2)$ using the maximum norm. Transfer only the smaller $\lambda_2$ because $\lambda_1$ is always 1 after normalization [S1, unofficial].
        *   Scale eigenvectors $v'_{\lambda_1}$ and $v'_{\lambda_2}$ so values are in $[0,1]$ for texture transportation [S1, unofficial].
        *   Determine the noise texture element for each fragment using the interpolated voxel position calculated in the vertex shader [S1, unofficial].

2.  **Edge Detection (Separate Pass):**
    *   Apply an edge detection filter to the depth map using a discrete Laplacian filter kernel implemented in a separate offscreen rendering pass [S1, unofficial].
    *   Store results in the edge detection output texture as defined in the layout above [S1, unofficial].

3.  **Advection (Iterative Passes):**
    *   Advect the previously mapped noise texture using the eigenvector color map from the projection step [S1, unofficial].
    *   The advection step grabs data directly from a single merged texture (depth buffer, edge detection texture, eigenvector color map, and last iteration's output) to reduce texture bindings [S1, unofficial].
    *   Transfer the unprocessed input noise to the advection texture to allow toggling between noise fields during advection, as near-planar geometry does not require complex geometry mapping [S1, unofficial].
    *   Repeat advection iterations until the fabric structure is sufficiently formed [S1, unofficial].

4.  **Final Output Processing:**
    *   Blend both advected eigenvector fields with light information and geometry edges [S1, unofficial].
    *   Clip fragments using the depth buffer [S1, unofficial].
    *   Use the last output texture from advection, color map with lighting info, and edge detection output texture to create the final visualization [S1, unofficial].
    *   **Optional Modification:** Blend in fractional anisotropy, $\lambda_2$, and implicitly $\lambda_1$ to emphasize additional tensor field features. Blend eigenvector fields with silhouettes and Phong luminance calculated in the projection step [S1, unofficial].

**Formulas:**
*   **Phong Shading Normal:** The normal $n$ used for Phong shading to produce a tube-like effect is calculated as [S1, unofficial]:
    $$ n = (1-r)(0,0,1)^T + r b_0 $$
    where $r$ is a parameter describing the ratio between the normal completely on the plane (zero z-component) and the normal completely pointing towards the camera (z-component of one), and $b_0$ is a base vector [S1, unofficial].
*   **Eigenvector Normalization:** The vector $(\lambda_1, \lambda_2)$ is normalized using the maximum norm, ensuring $\lambda_1 = 1$ and storing only $\lambda_2$ [S1, unofficial].

**Parameters:**
*   $r$: Parameter for Phong shading normal calculation [S1, unofficial].
*   Texture limit: Maximum four bound textures per rendering pass and active framebuffer object (hardware-dependent, e.g., NVIDIA GeForce 8800 GTS) [S1, unofficial].

**Pitfalls:**
*   **Sign Flips:** Eigenvectors are without orientation, causing sign flips between adjacent vertices. If decomposition happens before interpolation, these sign changes render interpolation useless. Solution: Perform eigenvalue decomposition and eigenvector calculation in the fragment shader after interpolation [S1, unofficial].
*   **Discontinuities:** The local approach causes discontinuities along tubes and artifacts from blurry input fields where borders are unclear. These are less visible due to the high frequency of the fabric structure [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Image Space Fabric Visualization
# Assumes GLSL-like shader logic translated to Python-like steps for clarity

def image_space_fabric_visualization(geometry, tensor_field, noise_texture, camera_params):
    """
    Main pipeline for Image Space Fabric Visualization.
    """
    # Step 1: Projection and Pre-computation
    # Render geometry into FBO1
    fbo1 = render_geometry(geometry, camera_params)
    
    # Vertex Shader Logic (per vertex)
    for vertex in geometry.vertices:
        # Map tensor directions to surface normal n
        # n is derived from geometry normal
        vertex.surface_normal = compute_surface_normal(vertex)
        
        # Map vertex position to noise texture voxel coordinates
        vertex.noise_coords = map_to_noise_space(vertex.position, noise_texture)
        
        # Pre-calculate Phong light intensity L(x,y)
        # This is done per fragment in the actual shader, but conceptually linked here
        vertex.light_intensity = 0.0 # Placeholder for fragment shader calculation

    # Fragment Shader Logic (per fragment)
    for fragment in fbo1.fragments:
        # Interpolate vertex attributes
        interpolated_normal = interpolate(fragment.vertex_normals)
        interpolated_noise_coords = interpolate(fragment.vertex_noise_coords)
        
        # Get tensor at fragment (interpolated from vertex tensors)
        tensor = interpolate(fragment.vertex_tensors)
        
        # Eigenvalue decomposition in fragment shader to avoid sign flip artifacts
        eigenvalues, eigenvectors = eigen_decompose(tensor)
        lambda_1, lambda_2 = eigenvalues
        v_1, v_2 = eigenvectors
        
        # Normalize eigenvectors for texture storage
        # Since lambda_1 >= lambda_2, normalize (lambda_1, lambda_2) with max norm
        # lambda_1 becomes 1, store lambda_2
        normalized_lambda_2 = lambda_2 / max(lambda_1, lambda_2)
        
        # Scale eigenvectors to [0, 1]
        v_1_scaled = scale_to_0_1(v_1)
        v_2_scaled = scale_to_0_1(v_2)
        
        # Determine noise texture element
        noise_value = sample_noise(noise_texture, interpolated_noise_coords)
        
        # Calculate Phong light intensity L(x,y)
        # Using formula: n = (1-r)(0,0,1)^T + r*b_0
        r = 0.5 # Example parameter
        b_0 = (0, 0, 1) # Example base vector
        n_phong = (1-r) * (0,0,1) + r * b_0
        light_intensity = compute_phong_lighting(n_phong, camera_params)
        
        # Store in FBO1
        fragment.data = {
            'eigenvector_1': v_1_scaled,
            'eigenvector_2': v_2_scaled,
            'lambda_2': normalized_lambda_2,
            'noise_value': noise_value,
            'light_intensity': light_intensity,
            'depth': fragment.depth
        }

    # Step 2: Edge Detection
    # Apply discrete Laplacian filter to depth map
    edge_texture = apply_edge_detection(fbo1.depth_buffer)
    # Layout: Green=depth, Blue=unprocessed noise, Alpha=geometry-mapped noise
    edge_texture.green = fbo1.depth_buffer
    edge_texture.blue = noise_texture # Unprocessed input noise
    edge_texture.alpha = fbo1.alpha_channel # Noise mapped to geometry

    # Step 3: Advection
    # Initialize advection texture with geometry-mapped noise
    advected_noise = fbo1.alpha_channel
    
    # Iterative advection
    num_iterations = 10 # Example
    for i in range(num_iterations):
        # Advect noise along eigenvectors
        # Use eigenvector color map from FBO1
        advected_noise = advect_noise(advected_noise, fbo1.eigenvector_1, fbo1.eigenvector_2)
        
        # Merge data into single texture for next pass if needed
        # To reduce texture bindings, merge depth, edge detection, and current advection output
        merged_texture = merge_textures(fbo1.depth_buffer, edge_texture, advected_noise)
        
        # Grab data from merged texture for next iteration
        # Note: In actual implementation, this is done via shader texture bindings
        advected_noise = extract_advected_noise(merged_texture)

    # Step 4: Final Output Processing
    # Blend advected fields with light and edges
    final_image = blend_advected_with_lighting(advected_noise, fbo1.light_intensity)
    
    # Clip using depth buffer
    final_image = clip_with_depth(final_image, fbo1.depth_buffer)
    
    # Blend with edge detection output
    final_image = blend_with_edges(final_image, edge_texture)
    
    # Optional: Blend in fractional anisotropy, lambda_2, lambda_1
    # final_image = blend_tensor_features(final_image, fbo1.lambda_2, fbo1.lambda_1)
    
    return final_image

def advect_noise(noise_texture, eigenvector_1, eigenvector_2):
    """
    Advect noise texture along eigenvectors.
    This is a simplified representation of the LIC advection step.
    """
    # In GLSL, this would be a fragment shader that samples noise
    # at positions displaced along the eigenvectors
    # For pseudocode, we assume a function that performs this sampling
    advected = sample_noise_along_vectors(noise_texture, eigenvector_1, eigenvector_2)
    return advected

def sample_noise_along_vectors(noise_texture, v1, v2):
    """
    Sample noise texture along directions v1 and v2.
    """
    # This is a placeholder for the actual LIC advection logic
    # which involves integrating along streamlines in the texture space
    # defined by the eigenvectors.
    # Since the exact integration step is not detailed in the source,
    # we note that it uses the eigenvector color map from the projection step.
    return noise_texture # Placeholder

def compute_phong_lighting(n, camera_params):
    """
    Compute Phong light intensity.
    """
    # Simplified Phong lighting calculation
    # Actual implementation depends on light sources and camera position
    return 0.5 # Placeholder

def scale_to_0_1(vector):
    """
    Scale vector components to [0, 1].
    """
    # Assuming vector components are in [-1, 1]
    return (vector + 1) / 2

def map_to_noise_space(position, noise_texture):
    """
    Map geometry position to noise texture voxel coordinates.
    """
    # Equations from Section 2.3 of the source
    # This maps the 3D position to a 3D texture coordinate
    return position # Placeholder

def eigen_decompose(tensor):
    """
    Perform eigenvalue decomposition on a symmetric 3x3 tensor.
    """
    # Standard eigenvalue decomposition
    eigenvalues = [1.0, 0.5, 0.1] # Placeholder
    eigenvectors = [(1,0,0), (0,1,0), (0,0,1)] # Placeholder
    return eigenvalues, eigenvectors

def interpolate(attributes):
    """
    Interpolate vertex attributes to fragment.
    """
    # Barycentric interpolation
    return attributes # Placeholder

def render_geometry(geometry, camera_params):
    """
    Render geometry into an FBO.
    """
    # Standard rasterization
    return FBO() # Placeholder

def apply_edge_detection(depth_buffer):
    """
    Apply discrete Laplacian filter to depth buffer.
    """
    # Convolution with Laplacian kernel
    return Texture() # Placeholder

def merge_textures(depth, edge, advected):
    """
    Merge multiple textures into one for efficient binding.
    """
    # Combine channels
    return Texture() # Placeholder

def extract_advected_noise(merged_texture):
    """
    Extract advected noise from merged texture.
    """
    return merged_texture.alpha # Placeholder

def blend_advected_with_lighting(advected, light_intensity):
    """
    Blend advected noise with lighting.
    """
    return advected * light_intensity # Placeholder

def clip_with_depth(image, depth_buffer):
    """
    Clip fragments using depth buffer.
    """
    # Discard fragments behind other geometry
    return image # Placeholder

def blend_with_edges(image, edge_texture):
    """
    Blend image with edge detection output.
    """
    # Use edge texture to enhance edges
    return image # Placeholder

def blend_tensor_features(image, lambda_2, lambda_1):
    """
    Blend in fractional anisotropy and eigenvalues.
    """
    # Modulate color based on tensor features
    return image # Placeholder
```

## Operating characteristics
*   **Time Complexity:** The bottleneck is the hardware's ability to render large and triangle-rich geometry. All further steps (advection, blending) can be done in constant time per pixel, assuming fixed iteration counts for advection [S1, unofficial].
*   **Memory Complexity:** Limited by GPU texture memory. On NVIDIA GeForce 8800 GTS, a maximum of four bound textures per rendering pass and active FBO are used [S1, unofficial]. Data merging techniques are employed to reduce texture bindings [S1, unofficial].
*   **Measured Performance:** Frame rates are high enough for real-time user interaction [S1, unofficial].
*   **Strengths:** Real-time performance, ability to handle arbitrary non-self-intersecting surfaces, GPU-based implementation avoids CPU calculations and multiple geometry renders [S1, unofficial].
*   **Limitations:** Local approach causes discontinuities along tubes and artifacts from blurry input fields. Sign flip artifacts must be handled by performing decomposition in the fragment shader [S1, unofficial].
*   **When to use:** When visualizing tensor fields on surfaces derived from scalar metrics (e.g., fractional anisotropy), hyper-stream surfaces, wrapped streamlines, or domain-dependent dissection-like surfaces, especially in medical or mechanical datasets [S1, unofficial].
*   **When not to use:** When global integration along streamlines is required to avoid discontinuities, or when the input tensor field is very blurry and borders are unclear [S1, unofficial].

## Background
*   **Origin:** The method was introduced in the paper "Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space" [S1, unofficial].
*   **History and Development:** It builds on ideas from [17] to transform the algorithm into image space, similar to the method introduced in [13] [S1, unofficial].
*   **Usage:** Applied to medical diffusion tensor data sets to visualize fiber directions, modulating color coding based on fractional anisotropy (major direction in RGB, secondary in grayscale below primary) [S1, unofficial]. Also applied to mechanical datasets including earthquake simulations (concrete pile) and analytical strain tensor fields (single point load) [S1, unofficial].

## Sources
[S1] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (unofficial)
