# STensoRRF

Also called: StensorR, tensorial able relighting radiance field for single-tensor scene representation

## Overview
STensoRRF is a differentiable volume rendering method that represents a 3D scene using a single 4D tensor to enable high-quality novel view synthesis and relighting [S1, unofficial]. Unlike standard NeRF architectures that use coupled MLPs for density and color, STensoRRF decouples appearance features from density within a tensor structure, allowing for the direct estimation of BRDF values and normals to simulate lighting changes [S1, unofficial]. It sets itself apart from sibling architectures by replacing the multi-tensor or multi-voxel mesh representations of previous relighting methods with a single decomposed 4D tensor, accelerating convergence by 50% and rendering speed by 1.5x [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view 2D images of a scene with unknown illumination; camera poses $(o, d)$; a target environment map (lighting).
*   **Outputs:** Rendered pixel colors for new viewpoints; relit images under new illumination; surface normals and BRDF values for geometry.

**Data Structures:**
The core representation is a single 4D tensor $\mathbf{G}_S$ [S1, unofficial].
*   **Dimensions:** The first three dimensions correspond to 3D spatial coordinates $(x, y, z)$. The fourth dimension is the feature vector.
*   **Feature Vector Layout:** The first element of the feature vector is the volume density $\sigma$. The remaining elements constitute the appearance features $\mathbf{f}$ [S1, unofficial].
*   **Decomposition:** To manage memory and accelerate optimization, the 4D tensor is decomposed using Vector-Matrix (VM) decomposition into several low-rank tensor components [S1, unofficial].

**Network Topology:**
Distinct Multi-Layer Perceptrons (MLPs) decode the appearance features $\mathbf{f}$ retrieved from the tensor:
1.  **Color Network:** Decodes $\mathbf{f}$ to predict base color $\mathbf{c}$ [S1, unofficial].
2.  **Normal Network:** Decodes $\mathbf{f}$ to predict the surface normal vector $\mathbf{n}$ [S1, unofficial].
3.  **BRDF Decoder:** Decodes $\mathbf{f}$ to predict a "BRDF value" $\beta$. Note: This is not a full analytical BRDF function but a scalar/vector estimate of the reflected light amount at that point, conceptually aligned with BRDF output [S1, unofficial].

**Lighting Model:**
The incident light field is simulated using Spherical Gaussians (SG). The environment map is fitted using 32 spherical Gaussians [S1, unofficial].

**Rendering Integration:**
1.  **Volume Rendering (Training):** Rays are sampled. Density $\sigma$ and color $\mathbf{c}$ are retrieved. Pixel color $C(\mathbf{r})$ is computed via standard volumetric rendering:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\delta_i$ is the distance between samples [S1, unofficial].
2.  **Relighting (Inference/Post-Training):** Surface points are identified (e.g., via density thresholds). The normal $\mathbf{n}$ and BRDF value $\beta$ are decoded. The reflected light is calculated by combining $\beta$, $\mathbf{n}$, and the SG-simulated incident light field [S1, unofficial].

**Optimization Loop:**
The 4D tensor is optimized iteratively by minimizing the difference between rendered pixel colors and real image pixel values [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def decompose_4d_tensor(tensor_4d, ranks):
    """
    Decomposes a 4D tensor into low-rank components using Vector-Matrix (VM) decomposition.
    tensor_4d: The initial 4D scene tensor.
    ranks: List of ranks for the decomposition components.
    Returns: List of low-rank factor matrices/vectors.
    """
    # Implementation of VM decomposition specific to the paper's structure.
    # Typically involves unfolding the tensor and performing SVD or CP decomposition.
    # [G] Standard tensor decomposition algorithm (e.g., CP or Tucker) is assumed here
    # as the paper cites [10, 11, 20] for the technique but does not provide code.
    factors = []
    for i, rank in enumerate(ranks):
        # Placeholder for actual decomposition logic
        factor = decompose_step(tensor_4d, rank)
        factors.append(factor)
    return factors

def retrieve_features(factors, x, y, z):
    """
    Retrieves density and appearance features from the decomposed tensor at coordinate (x,y,z).
    """
    # Reconstruct the feature vector at (x,y,z) from the low-rank factors
    feature_vector = reconstruct_from_factors(factors, x, y, z)
    
    # Layout: First element is density, rest are appearance features
    density = feature_vector[0]
    appearance_features = feature_vector[1:]
    
    return density, appearance_features

def decode_features(appearance_features):
    """
    Decodes appearance features into color, normal, and BRDF value using distinct MLPs.
    """
    color_network = MLP() # Initialized with random weights
    normal_network = MLP()
    brdf_decoder = MLP()
    
    color = color_network(appearance_features)
    normal = normal_network(appearance_features)
    brdf_value = brdf_decoder(appearance_features)
    
    return color, normal, brdf_value

def fit_environment_map(images, num_sg=32):
    """
    Fits the environment map using Spherical Gaussians.
    """
    # [G] Standard SG fitting algorithm (e.g., using least squares to fit 
    # sum of N SGs to the HDR environment map).
    sg_params = fit_sg_to_hdr(num_sg)
    return sg_params

def render_volume(rays, factors, color_network, normal_network, brdf_decoder):
    """
    Renders a ray using volume rendering.
    """
    total_color = np.zeros(3)
    transmittance = 1.0
    
    for point in rays:
        x, y, z = point.coords
        density, features = retrieve_features(factors, x, y, z)
        
        # Skip if density is negligible
        if density < 1e-4:
            continue
            
        color, _, _ = decode_features(features)
        
        alpha = 1.0 - np.exp(-density * point.delta)
        weight = transmittance * alpha
        
        total_color += weight * color
        transmittance *= (1.0 - alpha)
        
        if transmittance < 1e-4:
            break
            
    return total_color

def render_relit(surface_points, sg_params, normal_network, brdf_decoder):
    """
    Renders a relit image using surface points, normals, BRDF values, and SG lighting.
    """
    relit_image = np.zeros_like(surface_points.image)
    
    for point in surface_points:
        features = retrieve_features(point.tensor_factors, point.x, point.y, point.z)[1]
        normal, brdf_value = decode_features(features)
        
        # Calculate incident light from SG environment map at the normal's direction
        incident_light = evaluate_sg_environment(sg_params, normal)
        
        # Approximate reflected light using the estimated BRDF value
        # [G] The paper describes combining BRDF value, normal, and secondary reflected light field.
        # A standard approximation for this "BRDF value" output is:
        reflected_color = brdf_value * incident_light
        
        relit_image[point.u, point.v] = reflected_color
        
    return relit_image

def train_stensorrf(images, poses, num_iterations):
    """
    Main training loop.
    """
    # Initialize 4D tensor
    tensor_4d = initialize_4d_tensor()
    factors = decompose_4d_tensor(tensor_4d, ranks=[...])
    
    # Fit lighting
    sg_params = fit_environment_map(images)
    
    # Initialize decoders
    color_net = MLP()
    normal_net = MLP()
    brdf_dec = MLP()
    
    for i in range(num_iterations):
        # Sample rays from training images
        rays = sample_rays(images, poses)
        
        # Render
        rendered_colors = []
        for ray in rays:
            color = render_volume(ray, factors, color_net, normal_net, brdf_dec)
            rendered_colors.append(color)
            
        # Compute loss against ground truth
        loss = compute_loss(rendered_colors, images)
        
        # Backpropagate to update tensor factors and MLP weights
        update_parameters(factors, color_net, normal_net, brdf_dec, loss)
        
    return factors, sg_params, color_net, normal_net, brdf_dec
```

**Parameters:**
*   Number of Spherical Gaussians for environment map fitting: 32 [S1, unofficial].
*   Typical values for other hyperparameters (learning rate, tensor resolution, ranks) are Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Surface Point Identification:** The method relies on calculating surface points after a certain number of iterations to decode normals and BRDF values. If the density field is noisy or under-converged, surface estimation may fail [S1, unofficial].
*   **BRDF Value Interpretation:** The "BRDF value" is a network output, not a physical function. It must be interpreted as a direct estimate of reflected light intensity rather than a full directional reflectance distribution [S1, unofficial].

## Subtopics

### Tensor Decomposition for NeRF
A family of methods that compress NeRF representations using tensor decomposition (CP, Tucker, VM) to reduce memory and accelerate training. STensoRRF uses VM decomposition on a single 4D tensor. Choose when memory efficiency and training speed are critical for large scenes. See [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md).

### Relighting Radiance Fields
Methods that extend NeRF to support dynamic lighting changes. STensoRRF differs by using a single tensor and direct BRDF value estimation rather than complex reflectance equations or multiple tensors. Choose when high-quality relighting is required with faster convergence. See [Appearance and Lighting Decomposition](11061-appearance-and-lighting-decomposition.md).

## Operating characteristics
*   **Time Complexity:** Convergence rate is 50% faster compared to existing relighting radiation field models [S1, unofficial].
*   **Memory Complexity:** Reduced by using a single 4D tensor with low-rank decomposition instead of multiple voxel meshes or tensors [S1, unofficial].
*   **Performance:** Rendering and relighting speed is approximately 1.5x faster than previous state-of-the-art models [S1, unofficial].
*   **Strengths:** High-quality novel view synthesis; efficient relighting; simplified training data requirements (no complex geometric surface properties needed); faster convergence.
*   **Limitations:** Relies on the accuracy of the "BRDF value" estimation network; performance depends on the quality of the Spherical Gaussian fit for the environment map.
*   **When to use:** When relighting capability is required alongside novel view synthesis, and computational efficiency (training speed and rendering speed) is a priority.
*   **When not to use:** When full physical BRDF accuracy is required (as STensoRRF uses an estimated scalar/vector value rather than a full function), or when the scene has complex, time-varying illumination that cannot be well-approximated by 32 Spherical Gaussians.

## Background
STensoRRF (Tensorial Able Relighting Radiance Field for Single-Tensor Scene Representation) was proposed in a preprint published in January 2024 [S1, unofficial]. It addresses the limitations of previous radiance field methods that relied on multiple voxel meshes or tensors for scene representation, which were computationally expensive. By combining the reflectance equation with tensor decomposition, it aims to accelerate convergence and enable efficient relighting. The method builds on prior work in tensor decomposition for scene representation [10, 11, 20] and relighting via reflectance equations [17–19, 27].

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
