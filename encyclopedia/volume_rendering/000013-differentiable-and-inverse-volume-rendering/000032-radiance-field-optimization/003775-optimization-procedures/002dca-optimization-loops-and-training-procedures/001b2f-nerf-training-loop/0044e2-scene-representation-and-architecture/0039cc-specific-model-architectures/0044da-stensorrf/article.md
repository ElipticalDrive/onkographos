# STensoRRF

Also called: StensorR, tensorial able relighting radiance field for single-tensor scene representation

## Overview
STensoRRF is a differentiable volume rendering method that represents a 3D scene using a single 4D tensor, enabling both novel view synthesis and relighting [S1, unofficial]. Unlike standard NeRF variants that predict color directly, STensoRRF decodes appearance features into a BRDF value and normal vector, allowing the scene to be re-illuminated by changing the incident light field [S1, unofficial]. It sets itself apart from sibling architectures by using a single 4D tensor decomposed via vector-matrix (VM) decomposition, rather than multiple voxel meshes or separate tensors, which accelerates convergence by 50% compared to existing relighting radiation field models [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of multi-view 2D images of a scene with unknown illumination; camera poses $(o, \mathbf{d})$; a new environment map (lighting) for relighting.
*   **Outputs:** Rendered pixel colors for new viewpoints; rendered pixel colors under new illumination (relighting); surface normals; BRDF values.

**Data Structures:**
The core representation is a single 4D tensor $\mathbf{G}_S$ [S1, unofficial].
*   **Dimensions:** The first three dimensions correspond to spatial coordinates $(x, y, z)$. The fourth dimension is the feature vector.
*   **Feature Vector Layout:** The vector at each $(x, y, z)$ contains:
    1.  The first element: Volume density $\sigma$.
    2.  Remaining elements: Appearance features $\mathbf{f}$ [S1, unofficial].
*   **Decomposition:** To manage memory and computation, the 4D tensor is decomposed into several low-rank tensor components using Vector-Matrix (VM) decomposition [S1, unofficial]. This allows efficient retrieval of $\sigma$ and $\mathbf{f}$ given a 3D coordinate.

**Parameters:**
*   Number of Spherical Gaussians (SGs) used to fit the environment map: 32 [S1, unofficial].
*   Positional Encoding: As per the parent [Parent], raw coordinates are mapped to a higher-dimensional space using sinusoidal functions before being indexed into the tensor or processed by MLPs.

**Algorithm Steps:**

1.  **Ray Sampling:** For a given camera ray $\mathbf{r}(t) = o + t\mathbf{d}$, sample $N$ points $\mathbf{x}_i$ along the ray within the scene bounds.
2.  **Feature Retrieval:** For each point $\mathbf{x}_i$, retrieve the volume density $\sigma_i$ and appearance feature vector $\mathbf{f}_i$ from the decomposed 4D tensor $\mathbf{G}_S$ [S1, unofficial].
3.  **Feature Decoding:** Pass $\mathbf{f}_i$ through distinct Multi-Layer Perceptrons (MLPs) to decode:
    *   Color value $\mathbf{c}_i$ (using a color network).
    *   Normal vector $\mathbf{n}_i$ (using a normal network).
    *   BRDF value $\mathbf{b}_i$ (using a BRDF decoder) [S1, unofficial].
    *   *Note:* The "BRDF value" is defined as the amount of light reflected from a specific point, estimated directly by the network [S1, unofficial].
4.  **Volume Rendering (Training):** Compute the pixel color $C(\mathbf{r})$ using standard volumetric rendering:
    $$
    C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \text{where } \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\delta_i$ is the distance between samples [Parent].
5.  **Lighting Simulation:** Represent the incident light field $L_i(\omega)$ using 32 Spherical Gaussians [S1, unofficial].
6.  **Relighting Rendering:** After training convergence (or during specific relighting phases), identify surface points on the ray (e.g., using depth estimation from density $\sigma$) [S1, unofficial]. For these surface points, compute the reflected color using the BRDF value $\mathbf{b}_i$, normal $\mathbf{n}_i$, and the simulated incident light field:
    $$
    C_{relight} = \int_{\Omega} \mathbf{b}_i \cdot L_i(\omega) \, d\omega \approx \sum_{k=1}^{32} \mathbf{b}_i \cdot \text{SG}_k(\omega)
    $$
    *Note:* The source describes combining the BRDF value with the "secondary reflected light field" (simulated by SGs) to approximate the integral [S1, unofficial].
7.  **Optimization:** Optimize the parameters of the 4D tensor $\mathbf{G}_S$ (and potentially the MLPs) by minimizing the difference between the rendered pixel colors $C(\mathbf{r})$ and the ground truth pixel values from the input images [S1, unofficial]. The supervision may include both the standard volume rendering loss and the relighting loss if ground truth relighting data is available, though the primary optimization is against the captured images [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def retrieve_features(tensor_decomposed, x, y, z):
    """
    Retrieve density and appearance features from the decomposed 4D tensor.
    Uses VM decomposition logic to reconstruct the feature vector at (x,y,z).
    """
    # Implementation detail: VM decomposition allows efficient lookup.
    # The first element of the reconstructed vector is density.
    # The rest are appearance features.
    feature_vector = decomposed_tensor_lookup(tensor_decomposed, x, y, z)
    density = feature_vector[0]
    appearance_features = feature_vector[1:]
    return density, appearance_features

def decode_features(appearance_features, color_mlp, normal_mlp, brdf_mlp):
    """
    Decode appearance features into color, normal, and BRDF value.
    """
    color = color_mlp(appearance_features)
    normal = normal_mlp(appearance_features)
    brdf_value = brdf_mlp(appearance_features)
    return color, normal, brdf_value

def render_volume(rays, tensor_decomposed, color_mlp, normal_mlp, brdf_mlp, num_samples=128):
    """
    Standard volume rendering for training supervision.
    """
    pixel_colors = []
    for ray in rays:
        o, d = ray.origin, ray.direction
        t_vals = np.linspace(ray.near, ray.far, num_samples)
        points = o + t_vals[:, None] * d[None, :]
        
        densities = []
        colors = []
        
        for i in range(num_samples):
            x, y, z = points[i]
            sigma, features = retrieve_features(tensor_decomposed, x, y, z)
            c, _, _ = decode_features(features, color_mlp, normal_mlp, brdf_mlp)
            densities.append(sigma)
            colors.append(c)
            
        densities = np.array(densities)
        colors = np.array(colors)
        
        # Volumetric rendering formula
        deltas = np.diff(t_vals, append=ray.far)
        alphas = 1.0 - np.exp(-densities * deltas)
        transmittance = np.exp(-np.cumsum(densities * deltas, axis=0))
        weights = alphas * transmittance
        pixel_color = np.sum(weights[:, None] * colors, axis=0)
        pixel_colors.append(pixel_color)
        
    return np.array(pixel_colors)

def render_relighting(rays, tensor_decomposed, color_mlp, normal_mlp, brdf_mlp, env_map_sgs):
    """
    Relighting rendering using Spherical Gaussians.
    """
    pixel_colors = []
    for ray in rays:
        o, d = ray.origin, ray.direction
        # Sample points to find surface
        t_vals = np.linspace(ray.near, ray.far, 256) # Higher res for surface finding
        points = o + t_vals[:, None] * d[None, :]
        
        densities = []
        for i in range(len(t_vals)):
            x, y, z = points[i]
            sigma, _ = retrieve_features(tensor_decomposed, x, y, z)
            densities.append(sigma)
            
        densities = np.array(densities)
        # Estimate surface depth (e.g., first point where transmittance is low)
        # [G] Standard NeRF depth estimation: find t where cumulative alpha > threshold
        alphas = 1.0 - np.exp(-densities * np.diff(t_vals, append=ray.far))
        transmittance = np.exp(-np.cumsum(densities * np.diff(t_vals, append=ray.far), axis=0))
        surface_idx = np.argmax(transmittance < 0.1) # Example heuristic
        
        if surface_idx < len(t_vals):
            x, y, z = points[surface_idx]
            sigma, features = retrieve_features(tensor_decomposed, x, y, z)
            _, normal, brdf_value = decode_features(features, color_mlp, normal_mlp, brdf_mlp)
            
            # Integrate BRDF * Light (simulated by 32 SGs)
            # [G] Approximate integral sum over SGs
            relight_color = np.zeros(3)
            for sg in env_map_sgs: # 32 SGs
                # Evaluate SG at normal direction
                # SG_k(omega) = a_k * exp(b_k * (omega . c_k))
                # [G] Simplified: assume sg provides value at normal
                light_val = sg.evaluate(normal)
                relight_color += brdf_value * light_val
                
            pixel_colors.append(relight_color)
        else:
            pixel_colors.append(np.zeros(3))
            
    return np.array(pixel_colors)
```

## Subtopics

### Tensor Decomposition (VM)
A technique used within STensoRRF to decompose the 4D tensor into low-rank components. It sets itself apart from full 4D tensor storage by reducing memory and accelerating optimization. Choose when memory is constrained or training speed is critical. See [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md).

### Spherical Gaussians
A method for representing the environment map (incident light field) using a sum of spherical Gaussian functions. It sets itself apart from discrete environment map sampling by providing a smooth, differentiable, and compact representation. Choose when efficient integration of lighting is required. See [Neural Light Field Representations](9273-neural-light-field-representations.md).

## Operating characteristics
*   **Time Complexity:** Training convergence is 50% faster than existing relighting radiation field models [S1, unofficial].
*   **Memory Complexity:** Uses a single 4D tensor decomposed via VM decomposition, which is more memory-efficient than methods using multiple voxel meshes or tensors [S1, unofficial]. Exact memory usage depends on the tensor resolution and the rank of the VM decomposition, which is not given in the sources.
*   **Performance:** Rendering and relighting speed is approximately 1.5x faster than previous state-of-the-art models [S1, unofficial].
*   **Strengths:** High-quality novel view synthesis; enables relighting without complex geometric surface properties; faster convergence and rendering than prior relighting NeRFs.
*   **Limitations:** Relies on the assumption that the scene can be well-represented by a single 4D tensor with VM decomposition; quality depends on the accuracy of the BRDF value estimation network.
*   **When to use:** When relighting capability is required alongside novel view synthesis, and computational efficiency (convergence/rendering speed) is a priority.
*   **When not to use:** When high-fidelity, physically accurate BRDFs are required (STensoRRF uses a learned "BRDF value" rather than a full analytical BRDF); when the scene has extreme geometric complexity that a single 4D tensor cannot capture efficiently.

## Background
STensoRRF (Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation) was proposed in a preprint published in January 2024 [S1, unofficial]. It addresses the limitations of previous radiance field methods that either relied on multiple voxel meshes/tensors or combined reflectance equations with separate lighting estimation models. By integrating the reflectance equation with tensor decomposition and a direct BRDF value estimation network, it aims to accelerate convergence and enable efficient relighting [S1, unofficial].

## Sources
[S1] STensoRRF:Tensorial able Relighting Radiance Field for Single-Tensor Scene Representation, https://doi.org/10.20944/preprints202401.1085.v1, fetched 2026-10-30 (unofficial)
