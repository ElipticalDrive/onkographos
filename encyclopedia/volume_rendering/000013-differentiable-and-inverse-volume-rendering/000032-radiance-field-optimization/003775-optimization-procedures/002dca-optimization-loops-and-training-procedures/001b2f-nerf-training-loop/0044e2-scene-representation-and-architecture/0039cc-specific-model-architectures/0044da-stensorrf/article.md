# STensoRRF

Also called: StensorR, tensorial able relighting radiance field for single-tensor scene representation

## Overview
STensoRRF is a differentiable volume rendering method that represents a 3D scene using a single 4D tensor to enable high-quality novel view synthesis and relighting [S1, unofficial]. It distinguishes itself from standard NeRF variants by combining tensor decomposition for efficient storage with a direct BRDF estimation network, allowing the scene to be re-illuminated under arbitrary lighting conditions simulated by spherical Gaussians [S1, unofficial]. Unlike methods requiring multiple voxel grids or explicit mesh extraction during training, STensoRRF optimizes a unified tensor field that outputs density, color, normals, and reflectance properties simultaneously [S1, unofficial].

## How it works
**Inputs:** A set of multi-view images of a scene with unknown illumination, along with their corresponding camera poses.
**Outputs:** A trained 4D scene tensor, from which novel view images and relit images can be rendered.

**Data Structures:**
The core representation is a single 4D tensor $\mathbf{G}_S$ [S1, unofficial].
*   **Dimensions:** The first three dimensions correspond to the 3D spatial coordinates $(x, y, z)$ of the scene. The fourth dimension is the feature vector size $D$.
*   **Feature Vector Layout:** The feature vector at any location consists of:
    1.  The first element: Volume density $\sigma$.
    2.  The remaining $D-1$ elements: Appearance features $\mathbf{f}$ [S1, unofficial].
*   **Decomposition:** To manage memory and accelerate convergence, the 4D tensor is decomposed using Vector-Matrix (VM) decomposition into low-rank components [S1, unofficial].

**Parameters:**
*   Number of Spherical Gaussians (SGs) used to fit the environment map: 32 [S1, unofficial].
*   Typical convergence speed: 50% faster than existing relighting radiation field models [S1, unofficial].

**Algorithm:**

1.  **Tensor Decomposition:** Initialize the 4D scene tensor $\mathbf{G}_S$ and decompose it into low-rank vector and matrix components using VM decomposition [S1, unofficial].
2.  **Ray Sampling:** For a given camera pose (origin $\mathbf{o}$, direction $\mathbf{d}$), sample $N$ points $\mathbf{p}_i$ along the ray within the scene bounds [S1, unofficial].
3.  **Feature Retrieval:** For each point $\mathbf{p}_i$, retrieve the density $\sigma_i$ and appearance features $\mathbf{f}_i$ from the decomposed tensor $\mathbf{G}_S$ using trilinear interpolation or direct lookup depending on the decomposition implementation [S1, unofficial].
4.  **Network Decoding:** Pass the appearance features $\mathbf{f}_i$ through three distinct Multi-Layer Perceptrons (MLPs):
    *   **Color Network:** Outputs the base color $\mathbf{c}_i$.
    *   **Normal Network:** Outputs the surface normal $\mathbf{n}_i$.
    *   **BRDF Decoder:** Outputs the BRDF value $\mathbf{b}_i$, which represents the amount of light reflected from the point [S1, unofficial].
5.  **Volume Rendering (Training):** Compute the pixel color $C$ using standard volume rendering integration, combining the density $\sigma_i$, color $\mathbf{c}_i$, and ray position [S1, unofficial].
6.  **Lighting Simulation:** Fit the environment map (incident light field) using 32 Spherical Gaussians [S1, unofficial].
7.  **Relighting Rendering:** After a certain number of training iterations, identify surface points on the ray (e.g., using depth estimation or density thresholds). For these points, calculate the reflected light integral by combining the BRDF value $\mathbf{b}_i$, the normal $\mathbf{n}_i$, and the secondary reflected light field simulated by the spherical Gaussians [S1, unofficial].
8.  **Optimization:** Iteratively optimize the 4D tensor parameters by minimizing the difference between the rendered pixel colors (from step 5) and the real image pixel values. The relighting loss may also be used to supervise the tensor field jointly [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def stensorrrf_render(ray_origin, ray_direction, scene_tensor, sg_params, num_samples):
    """
    Renders a single ray using the STensoRRF representation.
    
    Args:
        ray_origin: 3D vector, camera origin.
        ray_direction: 3D vector, camera viewing direction.
        scene_tensor: Decomposed 4D tensor object supporting query(p).
        sg_params: List of 32 Spherical Gaussian parameters (coefficients, centers, exponents).
        num_samples: Integer, number of points to sample along the ray.
        
    Returns:
        color: 3D vector, rendered pixel color.
        depth: Float, estimated depth.
        brdf_value: 3D vector, BRDF value at the surface point (for relighting).
        normal: 3D vector, surface normal at the surface point.
    """
    
    # 1. Sample points along the ray
    # Assuming uniform sampling between near and far bounds
    t_vals = linspace(ray_near, ray_far, num_samples)
    points = ray_origin + outer_product(t_vals, ray_direction)
    
    # 2. Retrieve features from the 4D tensor
    # The tensor is decomposed, so we query the low-rank components
    densities = []
    features = []
    for p in points:
        sigma, f = scene_tensor.query(p)
        densities.append(sigma)
        features.append(f)
        
    densities = array(densities)
    features = array(features)
    
    # 3. Decode features using MLPs
    # Color Network
    colors = color_mlp(features)
    
    # Normal Network
    normals = normal_mlp(features)
    
    # BRDF Decoder
    brdf_values = brdf_mlp(features)
    
    # 4. Volume Rendering for Training Supervision
    # Compute transmittance and accumulated color
    # alpha = 1 - exp(-sigma * delta_t)
    # C = sum(alpha_i * color_i * prod(1 - alpha_j) for j < i)
    deltas = diff(t_vals, prepend=ray_near)
    alphas = 1.0 - exp(-densities * deltas)
    transmittance = cumprod(1.0 - alphas, axis=0)
    # Handle the first point's transmittance (should be 1.0)
    transmittance = concatenate([ones(1), transmittance[:-1]])
    
    weights = alphas * transmittance
    rendered_color = sum(weights * colors, axis=0)
    rendered_depth = sum(weights * t_vals, axis=0)
    
    # 5. Identify Surface Point for Relighting
    # Use the depth from volume rendering or a threshold on density
    # Here we assume we pick the point with the highest weight (dominant surface)
    surface_idx = argmax(weights)
    surface_point = points[surface_idx]
    surface_brdf = brdf_values[surface_idx]
    surface_normal = normals[surface_idx]
    
    # 6. Relighting Calculation (if requested)
    # Simulate incident light using Spherical Gaussians
    incident_light = evaluate_spherical_gaussians(surface_normal, sg_params)
    
    # Calculate reflected light integral
    # Simplified: BRDF value * Incident Light
    # Note: The source describes combining BRDF, normal, and secondary reflected light field
    relit_color = surface_brdf * incident_light 
    
    return rendered_color, rendered_depth, relit_color, surface_normal, surface_brdf

def optimize_stensorrrf(images, poses, scene_tensor, sg_params, num_iterations):
    """
    Optimizes the 4D tensor and SG parameters.
    """
    for i in range(num_iterations):
        loss = 0
        for img, pose in zip(images, poses):
            # Render all pixels for this pose
            rendered = render_image(pose, scene_tensor, sg_params)
            
            # Compute loss against real image
            pixel_loss = mse(rendered, img)
            loss += pixel_loss
            
        # Backpropagate to update scene_tensor and sg_params
        gradient = compute_gradient(loss)
        update(scene_tensor, sg_params, gradient)
        
    return scene_tensor, sg_params
```

**Edge Cases and Pitfalls:**
*   **Precision:** The use of a single 4D tensor requires careful handling of the feature vector size $D$. If $D$ is too small, the network may not have sufficient capacity to encode complex BRDF variations; if too large, memory usage increases despite decomposition [S1, unofficial].
*   **Boundaries:** Ray sampling must be bounded correctly to avoid querying outside the tensor's defined spatial domain.
*   **Degenerate Input:** The method assumes multi-view captured images. Single-view input is not supported as described [S1, unofficial].
*   **Lighting Estimation:** The accuracy of relighting depends heavily on the fit of the 32 Spherical Gaussians to the true environment map. Poor fitting leads to incorrect incident light estimation [S1, unofficial].

## Subtopics

### Tensor Decomposition
A technique used within STensoRRF to decompose the 4D tensor into low-rank components (Vector-Matrix decomposition) to accelerate convergence and reduce memory [S1, unofficial]. See [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md).

### Spherical Gaussians
A method for approximating the environment map (incident light field) using a sum of spherical Gaussian functions, used here with 32 components [S1, unofficial]. See [Neural Light Field Representations](9273-neural-light-field-representations.md).

## Operating characteristics
*   **Time Complexity:** The method achieves a 50% faster convergence rate compared to existing relighting radiation field models [S1, unofficial].
*   **Memory Complexity:** Uses a single 4D tensor with VM decomposition, which is more memory-efficient than methods using multiple voxel meshes or tensors [S1, unofficial]. Exact memory footprint depends on the tensor dimensions and decomposition ranks, which are not specified in the sources.
*   **Performance:** Rendering and relighting speed is approximately 1.5x faster than previous state-of-the-art models [S1, unofficial].
*   **Strengths:** High-quality novel view synthesis, efficient relighting, faster convergence, and simplified training data requirements (no complex geometric surface properties needed) [S1, unofficial].
*   **Limitations:** Relies on the assumption that the environment map can be well-approximated by 32 Spherical Gaussians. The quality of relighting is dependent on this approximation.
*   **When to use:** When relighting capability is required alongside novel view synthesis, and computational efficiency is a priority.
*   **When not to use:** When the scene has complex, non-Lambertian lighting interactions that cannot be captured by the simplified BRDF value and SG approximation, or when explicit mesh geometry is required.

## Background
STensoRRF (Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation) was proposed in a preprint published in January 2024 [S1, unofficial]. It addresses the limitations of previous radiance field methods that either used multiple voxel meshes/tensors or combined the reflectance equation with complex geometric surface properties. The method builds on the concept of neural scene representations and tensor decomposition techniques [S1, unofficial]. It is used for 3D reconstruction, novel view synthesis, and relighting in computer vision and graphics [S1, unofficial].

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
