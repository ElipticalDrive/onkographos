# Image Space Fabric Visualization

Also called: Advection of eigenvector field, Fabric-Like Tensor Field Visualization on Surfaces, Iterative Eigenvector Field Advection, LIC-like advection, Surface Tensor LIC

## Overview
Image Space Fabric Visualization is a method to visualize symmetric tensor fields on arbitrary non-self-intersecting surfaces by projecting tensor data into image space, advecting a noise texture along eigenvectors, and blending with lighting and edge information to create a fabric-like appearance [S1, book]. It differs from CPU-based or multi-pass geometry rendering methods by performing all intermediate calculations in GPU offscreen framebuffer objects (FBOs), leading to large speed gains and real-time performance [S1, book].

## How it works
**Inputs:**
1. Geometry: A non-self-intersecting surface with smooth normals [S1, book].
2. Tensor Field: Symmetric second-order tensors defined on the surface vertices [S1, book].
3. Noise Texture: A scalar noise field (pre-calculated or stored in a file) [S1, book].
4. Camera parameters: For projection and lighting calculation [S1, book].

**Outputs:**
1. Final image: A visualization with a fabric-like texture modulated by lighting and edge information [S1, book].

**Data Structures:**
* Vertex Attributes: Position, normal, and two 3D texture coordinates for tensor information [S1, book]. If tensors are symmetric, 6 floating-point values per vertex are transferred [S1, book].
* Textures: Intermediate textures for FBO passes [S1, book]. Due to hardware limits (e.g., NVIDIA GeForce 8800 GTS), a maximum of four bound textures per rendering pass and active FBO are used [S1, book].
* Edge detection output texture layout: Green channel stores depth buffer value; Blue channel stores unprocessed input noise; Alpha channel stores noise mapped to geometry's surface [S1, book].

**Steps:**
1. **Projection and Pre-computation (Vertex/Fragment Shaders):** Render the geometry into an offscreen FBO [S1, book]. In the vertex shader, map the two main tensor directions to the surface described by the normal $n$ at the current vertex [S1, book]. Calculate each vertex's position in one voxel of the noise texture using equations from Section 2.3 of the source [S1, book]. Pre-calculate Phong light intensity $L(x,y) \in [0,1]$ at every fragment with coordinates $x$ and $y$ [S1, book]. In the fragment shader, perform eigenvalue decomposition and eigenvector calculation *after* interpolation to avoid sign flip artifacts [S1, book]. Project eigenvectors to image space [S1, book]. Normalize eigenvectors for texture storage [S1, book]. Scale eigenvectors $v'_{\lambda_1}$ and $v'_{\lambda_2}$ so values are in $[0,1]$ for texture transportation [S1, book]. Determine the noise texture element for each fragment using the interpolated voxel position calculated in the vertex shader [S1, book].
2. **Edge Detection (Separate Pass):** Apply an edge detection filter to the depth map using a discrete Laplacian filter kernel implemented in a separate offscreen rendering pass [S1, book]. Store results in the edge detection output texture as defined in the layout above [S1, book].
3. **Advection (Iterative Passes):** Advect the previously mapped noise texture using the eigenvector color map from the projection step [S1, book]. The advection step grabs data directly from a single merged texture (depth buffer, edge detection texture, eigenvector color map, and last iteration's output) to reduce texture bindings [S1, book]. Transfer the unprocessed input noise to the advection texture to allow toggling between noise fields during advection, as near-planar geometry does not require complex geometry mapping [S1, book]. Iterate for $i = 0, 1, ...$ until value change exceeds a threshold (reached after a few iterations depending on $k$) [S2, unofficial].
4. **Final Output Processing:** Blend both advected eigenvector fields with light information and geometry edges [S1, book]. Clip fragments using the depth buffer [S1, book]. Use the last output texture from advection, color map with lighting info, and edge detection output texture to create the final visualization [S1, book]. Optional modification: Blend in fractional anisotropy, $\lambda_2$, and implicitly $\lambda_1$ to emphasize additional tensor field features [S1, book]. Blend eigenvector fields with silhouettes and Phong luminance calculated in the projection step [S1, book].

**Formulas:**
* **Phong Shading Normal:** The normal $n$ used for Phong shading to produce a tube-like effect is calculated as [S1, book]:
  $$ n = (1-r)(0,0,1)^T + r b_0 $$
  where $r$ is a parameter describing the ratio between the normal completely on the plane (zero z-component) and the normal completely pointing towards the camera (z-component of one), and $b_0$ is a base vector [S1, book].
* **Eigenvector Normalization:** Since $\lambda_1 \ge \lambda_2$, normalize the vector $(\lambda_1, \lambda_2)$ using the maximum norm [S1, book]. Transfer only the smaller $\lambda_2$ because $\lambda_1$ is always 1 after normalization [S1, book].
* **Advection Iteration:** For each eigenvector field $\lambda \in \{\lambda_1, \lambda_2\}$, initialize input and output fields $P_i^\lambda$ [S2, unofficial]. Iterate:
  $$ P_{i+1}^\lambda = k \cdot \beta_{x,y} + (1-k) \cdot \frac{f_{P_i^\lambda}(x+v'_{\lambda_x}, y+v'_{\lambda_y}) + f_{P_i^\lambda}(x-v'_{\lambda_x}, y-v'_{\lambda_y})}{2} $$
  where $\beta_{x,y}$ is the input noise, $k$ is a scalar factor determining the mix ratio between original and advected noise, and $v'$ are the scaled eigenvectors [S2, unofficial].

**Parameters:**
* $r$: Parameter for Phong shading normal calculation [S1, book].
* $k$: Scalar factor for advection mix ratio (e.g., 0.05 for smooth/blurred, 0.09 for coarse/contrast-rich) [S2, unofficial].
* Texture limit: Maximum four bound textures per rendering pass and active framebuffer object (hardware-dependent, e.g., NVIDIA GeForce 8800 GTS) [S1, book].

**Pitfalls:**
* **Sign Flips:** Eigenvectors are without orientation, causing sign flips between adjacent vertices [S1, book]. If decomposition happens before interpolation, these sign changes render interpolation useless [S1, book]. Solution: Perform eigenvalue decomposition and eigenvector calculation in the fragment shader after interpolation [S1, book].
* **Discontinuities:** The local approach causes discontinuities along tubes and artifacts from blurry input fields where borders are unclear [S1, book]. These are less visible due to the high frequency of the fabric structure [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cross_product(a, b):
    return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])

def normalize(v):
    mag = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return (v[0]/mag, v[1]/mag, v[2]/mag)

def compute_surface_normal(vertex):
    return normalize(cross_product(vertex.edge1, vertex.edge2))

def map_to_noise_space(position, noise_texture):
    return (position[0], position[1], position[2])

def eigen_decompose(tensor):
    eigenvalues = [1.0, 0.5, 0.1]
    eigenvectors = [(1,0,0), (0,1,0), (0,0,1)]
    return eigenvalues, eigenvectors

def scale_to_0_1(vector):
    return ((vector[0]+1)/2, (vector[1]+1)/2, (vector[2]+1)/2)

def sample_noise(texture, coords):
    return texture[coords]

def interpolate(attributes):
    return attributes

def compute_phong_lighting(n, camera_params):
    return 0.5

def create_empty_texture_like(template):
    return {}

class Framebuffer:
    def __init__(self):
        self.depth_buffer = {}
        self.alpha_channel = {}
        self.fragments = []

class Texture:
    def __init__(self):
        self.green = {}
        self.blue = {}
        self.alpha = {}
        self.grid = []

def image_space_fabric_visualization(geometry, tensor_field, noise_texture, camera_params):
    fbo1 = Framebuffer()
    fbo1.fragments = geometry.fragments
    
    for vertex in geometry.vertices:
        vertex.surface_normal = compute_surface_normal(vertex)
        vertex.noise_coords = map_to_noise_space(vertex.position, noise_texture)
        vertex.light_intensity = 0.0 
        
    for fragment in fbo1.fragments:
        interpolated_normal = interpolate(fragment.vertex_normals)
        interpolated_noise_coords = interpolate(fragment.vertex_noise_coords)
        tensor = interpolate(fragment.vertex_tensors)
        
        eigenvalues, eigenvectors = eigen_decompose(tensor)
        lambda_1, lambda_2 = eigenvalues[0], eigenvalues[1]
        v_1, v_2 = eigenvectors[0], eigenvectors[1]
        
        normalized_lambda_2 = lambda_2 / max(lambda_1, lambda_2)
        v_1_scaled = scale_to_0_1(v_1)
        v_2_scaled = scale_to_0_1(v_2)
        
        noise_value = sample_noise(noise_texture, interpolated_noise_coords)
        
        r = 0.5
        b_0 = (0, 0, 1)
        n_phong = ((1-r)*0 + r*b_0[0], (1-r)*0 + r*b_0[1], (1-r)*1 + r*b_0[2])
        light_intensity = compute_phong_lighting(n_phong, camera_params)
        
        fragment.data = {
            'eigenvector_1': v_1_scaled,
            'eigenvector_2': v_2_scaled,
            'lambda_2': normalized_lambda_2,
            'noise_value': noise_value,
            'light_intensity': light_intensity,
            'depth': fragment.depth
        }

    edge_texture = Texture()
    edge_texture.green = fbo1.depth_buffer
    edge_texture.blue = noise_texture
    edge_texture.alpha = fbo1.alpha_channel

    advected_noise = fbo1.alpha_channel
    k = 0.05
    for i in range(10):
        new_advected = create_empty_texture_like(advected_noise)
        for x, y in advected_noise.grid:
            pos_plus = (x + v_2_scaled[0], y + v_2_scaled[1])
            pos_minus = (x - v_2_scaled[0], y - v_2_scaled[1])
            sample_plus = sample_noise(advected_noise, pos_plus)
            sample_minus = sample_noise(advected_noise, pos_minus)
            beta = edge_texture.blue[x, y]
            new_advected[x, y] = k * beta + (1 - k) * (sample_plus + sample_minus) / 2
        advected_noise = new_advected
        merged_texture = Texture()
        merged_texture.green = fbo1.depth_buffer
        merged_texture.blue = edge_texture.blue
        merged_texture.alpha = advected_noise
        advected_noise = merged_texture.alpha

    final_image = advected_noise * fbo1.light_intensity
    final_image = final_image
    final_image = final_image
    
    return final_image
```

## Operating characteristics
* **Time Complexity:** The bottleneck is the hardware's ability to render large and triangle-rich geometry [S1, book]. All further steps (advection, blending) can be done in constant time per pixel, assuming fixed iteration counts for advection [S1, book].
* **Memory Complexity:** Limited by GPU texture memory [S1, book]. On NVIDIA GeForce 8800 GTS, a maximum of four bound textures per rendering pass and active FBO are used [S1, book]. Data merging techniques are employed to reduce texture bindings [S1, book].
* **Measured Performance:** Frame rates are high enough for real-time user interaction [S1, book].
* **Strengths:** Real-time performance, ability to handle arbitrary non-self-intersecting surfaces, GPU-based implementation avoids CPU calculations and multiple geometry renders [S1, book].
* **Limitations:** Local approach causes discontinuities along tubes and artifacts from blurry input fields [S1, book]. Sign flip artifacts must be handled by performing decomposition in the fragment shader [S1, book].
* **When to use:** When visualizing tensor fields on surfaces derived from scalar metrics (e.g., fractional anisotropy), hyper-stream surfaces, wrapped streamlines, or domain-dependent dissection-like surfaces, especially in medical or mechanical datasets [S1, book].

## Background
* **Origin:** The method was introduced in the paper "Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space" [S1, book].
* **History and Development:** It builds on ideas from [17] to transform the algorithm into image space, similar to the method introduced in [13] [S1, book].
* **Usage:** Applied to medical diffusion tensor data sets to visualize fiber directions, modulating color coding based on fractional anisotropy (major direction in RGB, secondary in grayscale below primary) [S1, book]. Also applied to mechanical datasets including earthquake simulations (concrete pile) and analytical strain tensor fields (single point load) [S1, book].

## Sources
[S1] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (book)
[S2] Image-Space Tensor Field Visualization Using a LIC-like Method, https://doi.org/10.1007/978-3-642-21608-4_11, fetched 2026-10-27 (unofficial)
