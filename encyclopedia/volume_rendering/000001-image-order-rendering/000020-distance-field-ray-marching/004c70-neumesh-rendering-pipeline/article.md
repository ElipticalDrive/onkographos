# NeuMesh Rendering Pipeline

Also called: NeuMesh

## Overview
NeuMesh is a neural rendering method that represents scenes using a deformable mesh scaffold to decouple geometry and texture, enabling efficient ray marching with high-frequency detail synthesis [S1]. It solves the limitation of standard neural implicit fields (like NeuS) by providing an explicit mesh structure that supports geometry editing while maintaining the rendering quality of neural SDFs [S1]. It differs from its parent [Distance-field ray marching](21-distance-field-ray-marching.md) by replacing the generic SDF evaluation with a mesh-based interpolation of learned codes, and from [Ray Marching with Neural SDF](2409-ray-marching-with-neural-sdf.md) by explicitly maintaining a mesh topology for the implicit field [S1].

## How it works
The method renders a scene by marching rays through a neural implicit field defined on a mesh. The core innovation is the disentanglement of geometry and texture into separate learnable codes associated with mesh vertices, which are interpolated during ray marching.

### Inputs and Outputs
*   **Inputs**: Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), a triangular mesh scaffold (vertices $\mathbf{V}$, faces $\mathbf{F}$), and pre-trained neural network parameters.
*   **Outputs**: Final pixel color $\hat{C}(\mathbf{r})$.

### Data Structures
The model architecture consists of two main components: the Geometry Code and the Texture Code, stored per vertex or interpolated [S1].

1.  **Geometry Code** $\mathbf{l}_g(\mathbf{x})$: A vector of dimension 32 [S1].
2.  **Texture Code** $\mathbf{l}_t(\mathbf{x})$: A vector of dimension 32 [S1].
3.  **Sign Distance** $h(\mathbf{x})$: A scalar value of dimension 1 [S1].
4.  **Interpolated Items**: During ray marching, these are computed for each sample point $\mathbf{x}$:
    *   $\nabla_{\mathbf{x}} s$: Gradient of the SDF, dimension 256 [S1].
    *   $s$: SDF value, dimension 256 [S1].
    *   $\gamma^{(i)}$: Positional encoding or frequency components, dimension 256 or 32 [S1].
    *   $h(\mathbf{x})$: Sign distance, dimension 1 [S1].
    *   $\mathbf{d}$: Auxiliary vector, dimension 160 [S1].
5.  **Decoders**:
    *   **Geometry Decoder**: Multi-Layer Perceptron (MLP) with 3 hidden layers, size 256, using SoftPlus activation [S1]. Outputs SDF-related features.
    *   **Radiance Decoder**: MLP with 4 hidden layers, size 256, using ReLU activation [S1]. Outputs radiance (color) with dimension 3 [S1].

### Algorithm Steps

1.  **Pre-computation of Bounds**:
    For each ray $\mathbf{r}$, compute the near and far bounds by finding the minimum and maximum distances of ray-to-mesh intersections [S1]. This accelerates the marching process by limiting the search volume.

2.  **Coarse Sampling**:
    Sample 64 coarse points along the ray within the pre-computed bounds [S1].

3.  **Fine Sampling (Progressive Up-sampling)**:
    Use a progressive up-sampling strategy (as described in Wang et al. [55], referenced in [S1]) to guide the sampling of 64 fine points [S1]. This results in a total of 128 samples per ray (64 coarse + 64 fine) [S1].

4.  **Neural Field Evaluation**:
    For each sample point $\mathbf{p}$ on the ray:
    *   Interpolate the Geometry Code $\mathbf{l}_g$, Texture Code $\mathbf{l}_t$, and Sign Distance $h$ from the mesh vertices surrounding $\mathbf{p}$ [S1].
    *   Compute the SDF value $s$ and its gradient $\nabla_{\mathbf{x}} s$ using the Geometry Decoder [S1].
    *   Compute the radiance (color) using the Radiance Decoder [S1].

5.  **Volume Rendering**:
    Accumulate color and transmittance using standard volume rendering integration over the 128 samples [S1].

6.  **Background Handling (Optional)**:
    If the scene has an unbounded background, apply binary cross-entropy loss using foreground masks during training [S1]. For rendering, if using NeuMesh++, blend the foreground mesh rendering with a background hash grid model (see Subtopics) [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray(ray_origin, ray_direction, mesh, geometry_decoder, radiance_decoder, 
               geometry_codes, texture_codes, sign_distances):
    """
    Renders a single ray using the NeuMesh pipeline.
    """
    # Step 1: Pre-compute near-far bounds
    # Count min/max distances of ray-to-mesh intersections
    t_min, t_max = compute_ray_mesh_bounds(ray_origin, ray_direction, mesh)
    
    # Step 2: Sample 64 coarse points
    coarse_samples = sample_linear(ray_origin, ray_direction, t_min, t_max, num_samples=64)
    
    # Step 3: Progressive up-sampling for 64 fine points
    # Guided by coarse samples (Wang et al. [55] strategy)
    fine_samples = progressive_upsample(coarse_samples, num_fine=64)
    
    # Total 128 samples
    all_samples = coarse_samples + fine_samples
    
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_transmittance = 1.0
    
    for t in all_samples:
        p = ray_origin + t * ray_direction
        
        # Interpolate codes from mesh vertices at p
        l_g = interpolate_geometry_code(p, mesh, geometry_codes)
        l_t = interpolate_texture_code(p, mesh, texture_codes)
        h = interpolate_sign_distance(p, mesh, sign_distances)
        
        # Evaluate Geometry Decoder
        # Inputs: l_g, l_t, h, and other interpolated items (gradient, gamma, etc.)
        # Output: SDF value s and gradient grad_s
        s, grad_s = geometry_decoder(l_g, l_t, h)
        
        # Evaluate Radiance Decoder
        # Output: Color c (dim 3)
        c = radiance_decoder(l_g, l_t, h, grad_s)
        
        # Volume rendering step
        # Assume alpha is derived from s (e.g., sigmoid of s * beta)
        alpha = compute_alpha(s)
        weight = accumulated_transmittance * alpha
        accumulated_color += weight * c
        accumulated_transmittance *= (1.0 - alpha)
        
        # Early termination if transmittance is negligible
        if accumulated_transmittance < 1e-4:
            break
            
    return accumulated_color

def compute_ray_mesh_bounds(origin, direction, mesh):
    """
    Computes the intersection interval [t_min, t_max] of a ray with a mesh.
    Returns the minimum and maximum t values where the ray enters and exits the mesh volume.
    """
    # Implementation detail: Standard ray-triangle intersection logic
    # Collect all valid t values, sort them, and take min/max
    # If no intersection, return (0, infinity) or handle appropriately
    pass

def interpolate_geometry_code(point, mesh, codes):
    """
    Interpolates the geometry code (dim 32) for a point inside a mesh triangle.
    Uses barycentric coordinates.
    """
    # Find containing triangle
    # Compute barycentric weights
    # Return weighted sum of vertex codes
    pass
```

### Parameters
*   **Batch Size**: 512 rays [S1].
*   **Optimizer**: Adam [S1].
*   **Learning Rate**: Initial 0.0005 [S1].
*   **Scheduler**: Cosine annealing with 5000 warm-up steps [S1].
*   **Hardware**: Single Nvidia RTX3090-24G GPU [S1].
*   **Training Time**: Approximately 16 hours per model [S1].

### Edge Cases and Pitfalls
*   **Unbounded Backgrounds**: Standard NeuMesh assumes bounded scenes. For unbounded backgrounds, foreground masks must be used with binary cross-entropy loss during training to prevent background artifacts [S1].
*   **Geometry Editing Consistency**: When deforming the mesh, the local consistency of learnable signed distances must be maintained. Rotations of surface normals must be compensated in the learnable signed indicator $h(\mathbf{x})$ to ensure the interpolated SDF remains consistent [S1].
*   **Precision**: The SDF gradient $\nabla_{\mathbf{x}} s$ is used in the radiance decoder. Numerical instability in gradient computation can affect shading quality.

## Subtopics

### NeuMesh++
A rendering technique for unbounded scenes that combines a foreground mesh-based neural representation (NeuMesh++) with an additional implicit hash grid for the background, blending them based on transmittance [S2]. It extracts a foreground mesh enclosed within a unit cube (e.g., from Neuralangelo) and learns an implicit hash grid for the background following the NeuS approach [S2]. The final color is computed as:
$$ \hat{C}(\mathbf{r})_{\text{comp}} = \hat{C}(\mathbf{r}) * T + \hat{C}(\mathbf{r})_{\text{bg}} * (1 - T) $$
where $\hat{C}(\mathbf{r})$ is the NeuMesh++ foreground color, $\hat{C}(\mathbf{r})_{\text{bg}}$ is the background model color, and $T$ is the accumulated transmittance of NeuMesh++ [S2]. It outperforms BakedSDF on datasets like Mip-NeRF 360, particularly for high-frequency details like sharp grass and detailed textures [S2].

## Operating characteristics
*   **Time Complexity**: Rendering involves 128 samples per ray (64 coarse + 64 fine) [S1]. Pre-computing ray-mesh bounds accelerates the marching loop [S1].
*   **Memory Complexity**: Stores geometry codes (dim 32), texture codes (dim 32), and sign distances per vertex, plus the MLP parameters for the decoders (3-4 layers, 256 hidden units) [S1].
*   **Performance**: Trains in ~16 hours on a single RTX3090 [S1]. Rendering quality shows better appearance details than NeuS and NeuTex on DTU and NeRF Synthetic datasets [S1].
*   **Strengths**: Explicit mesh scaffold allows for geometry editing; disentangled codes improve high-frequency detail synthesis; efficient ray marching via pre-computed bounds [S1, S2].
*   **Limitations**: Requires a pre-existing mesh scaffold; unbounded scenes require hybrid background modeling (NeuMesh++) [S1, S2].

## Background
NeuMesh was introduced to address the limitations of neural implicit fields in supporting geometry editing and efficient rendering [S1]. It builds on the principles of neural SDFs (like NeuS) but introduces a mesh-based representation to decouple geometry and texture [S1]. NeuMesh++ extends this to unbounded scenes by integrating hash grid backgrounds [S2]. The method is used in novel view synthesis and 3D editing applications [S1, S2].

## Sources
[S1] NeuMesh: Learning Disentangled Neural Mesh-Based Implicit Field for Geometry and Texture Editing, https://doi.org/10.1007/978-3-031-19787-1_34, fetched 2026-10-26
[S2] NeuMesh++: Toward Versatile and Efficient Volumetric Editing With Disentangled Neural Mesh-Based Implicit Field, https://doi.org/10.1109/TPAMI.2025.3600473, fetched 2026-10-29
