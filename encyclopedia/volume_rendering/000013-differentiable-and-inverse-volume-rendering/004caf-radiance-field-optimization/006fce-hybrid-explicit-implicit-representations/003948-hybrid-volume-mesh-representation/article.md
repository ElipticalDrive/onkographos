# Hybrid Volume-Mesh Representation

Also called: VMesh

## Overview
VMesh is a hybrid representation that depicts an object using a textured triangular mesh for primary geometry and an auxiliary sparse volume for subtle structures, obtained from multi-view images via an implicit signed distance-density field [S1]. It sets itself apart from sibling methods by using the mesh and volume to represent separate geometries rather than using the mesh merely as guidance for volume rendering, allowing the mesh to be directly used for rendering while retaining the ability to model thin structures via the volumetric counterpart [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Multi-view images of an object [S1].
*   **Output:** A hybrid asset consisting of a triangular mesh, texture maps, and a sparse volume optimized for real-time rendering [S1].

**Data Structures:**
1.  **Implicit Fields (Training):**
    *   Signed Distance Field (SDF) $f_{sdf}$: A neural network outputting scalar distance $d$ [S1].
    *   Density Field $\sigma$: A neural network outputting volume density $\sigma_{vol}$ [S1].
    *   Color/Texture Fields: MLPs or look-up tables predicting diffuse color $\mathbf{c}_d$, specular tint $\mathbf{s}$, metallic property $m$, and basis weights [S1].
2.  **Explicit Assets (Rendering):**
    *   **Mesh:** Triangular mesh with vertex attributes (position, normal, UV coordinates) [S1].
    *   **Sparse Volume:** A sparse voxel grid storing density $\sigma_{vol}$ and color features [S1].
    *   **RefBasis Texture:** Look-up tables or small networks for specular basis colors $\{\mathbf{c}_{s1}, \dots, \mathbf{c}_{sN}\}$ and attenuation factor $A(\theta, m)$ [S1].

**Numbered Steps:**
1.  **Contiguous Stage (Joint Optimization):** Initialize implicit neural representations for the SDF $f_{sdf}$ and density $\sigma_{vol}$ [S1]. Jointly optimize these fields using volume rendering from multi-view images [S1]. The hybrid density is computed as $\sigma_{hyb} = \sigma_{surf} + \sigma_{vol}$, where $\sigma_{surf}$ is derived from the SDF [S1]. The hybrid opacity is $\alpha_{hyb} = 1 - (1 - \alpha_{surf})(1 - \alpha_{vol})$ [S1].
2.  **Discretization Stage (Quantization):** Convert the optimized implicit fields into explicit representations through quality-preserving quantization [S1]. Extract the triangular mesh from the SDF $f_{sdf}$ [S1]. Extract the sparse volume from the density field $\sigma_{vol}$ [S1].
3.  **Texture Representation (RefBasis):** Model view-dependent specular color using the RefBasis representation [S1]. This involves predicting a set of $N$ base specular colors $\{\mathbf{c}_{s1}, \dots, \mathbf{c}_{sN}\}$ for each reflected ray direction $\omega_r$ [S1]. Predicting an $N$-dimensional weighting vector $\mathbf{w}$ for each point [S1]. Computing specular color $\mathbf{c}_s$ as a weighted sum [S1]. Predicting an attenuation factor $A(\theta, m)$ based on incident angle $\theta$ and metallic property $m$ to model Fresnel effects [S1]. Final color $\mathbf{c}$ is $\mathbf{c} = \gamma (\mathbf{c}_d + \mathbf{s} \odot A(\theta, m)\mathbf{c}_s(\hat{\omega}_r))$ [S1].

**Formulas and Definitions:**
*   **Sigmoid Mapping:** Maps signed distance $x$ to opacity [S1].
    $$ \Phi_s(x) = (1 + e^{-sx})^{-1} $$
    where $s$ is a scene-wise learnable parameter [S1].
*   **Discretized Volume Opacity:**
    $$ \alpha_i = \max\left(\frac{\Phi_s(d_i) - \Phi_s(d_{i+1})}{\Phi_s(d_i)}, 0\right) $$
    where $d_i$ and $d_{i+1}$ are signed distance values at sample points $t_i$ and $t_{i+1}$ along a ray, and $\Phi_s$ is the Sigmoid function [S1].
*   **Normal Computation:**
    $$ \mathbf{n} = \nabla f_{sdf}(\mathbf{x}) $$
    The normal $\mathbf{n}$ is the gradient of the SDF [S1].
*   **Reflected Ray Direction:**
    $$ \omega_r = 2(\omega_o \cdot \mathbf{n}) - \omega_o $$
    where $\omega_o$ is the incident/viewing direction [S1].
*   **Loss Functions:**
    *   **Eikonal Loss:** Regularizes the SDF gradient magnitude [S1].
        $$ L_{eik} = \sum_i (\|\nabla f_{sdf}(\mathbf{x}(t_i))\| - 1)^2 $$
    *   **Mask Loss:** Binary cross entropy between estimated opacity $\hat{M} = \sum_i \omega_i$ and ground truth mask $M$ [S1].
        $$ L_{mask} = \text{BCE}(\hat{M}, M) $$
    *   **Normal Smoothness Loss:** Encourages predicted normal $\hat{\mathbf{n}}$ to align with analytic normal $\mathbf{n}$ [S1].
        $$ L_{norm} = 1 - \hat{\mathbf{n}} \cdot \mathbf{n} $$
    *   **Photometric Loss:** Applies to both hybrid and surface-only renders to prioritize surface usage [S1].
        $$ L_{pm} = \sum_i \left( \|\hat{\mathbf{C}}_i^{hyb} - \mathbf{C}_i\|^2 + \|\hat{\mathbf{C}}_i^{surf} - \mathbf{C}_i\|^2 \right) $$
    *   **Sparsity Loss:** Penalizes volume density to encourage sparsity [S1].
        $$ L_{sp} = \sum_i |1 - \exp(-\lambda \sigma_i)| $$

**Parameters:**
*   $s$: Scene-wise learnable parameter for the Sigmoid function. $1/s$ represents surface uncertainty; $1/s=0$ is equivalent to surface rendering [S1].
*   $N$: Number of learned basis functions for specular color [S1].
*   $\lambda$: Density sparsity regularization coefficient [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x, s):
    return 1.0 / (1.0 + math.exp(-s * x))

def compute_opacity_from_sdf(d_i, d_i_plus_1, s):
    phi_i = sigmoid(d_i, s)
    phi_i_plus_1 = sigmoid(d_i_plus_1, s)
    if phi_i == 0.0:
        return 0.0
    alpha = (phi_i - phi_i_plus_1) / phi_i
    return max(alpha, 0.0)

def dot_product(v1, v2):
    return sum(a * b for a, b in zip(v1, v2))

def vector_norm(v):
    return math.sqrt(sum(x * x for x in v))

def normalize(v):
    n = vector_norm(v)
    if n == 0.0:
        return [0.0, 0.0, 0.0]
    return [x / n for x in v]

def compute_normal(sdf_fn, x, eps=1e-4):
    grad = [0.0, 0.0, 0.0]
    for i in range(3):
        x_plus = list(x)
        x_minus = list(x)
        x_plus[i] += eps
        x_minus[i] -= eps
        grad[i] = (sdf_fn(x_plus) - sdf_fn(x_minus)) / (2.0 * eps)
    return normalize(grad)

def compute_reflected_direction(omega_o, n):
    dot = dot_product(omega_o, n)
    return [2.0 * dot * n[i] - omega_o[i] for i in range(3)]

def compute_specular_color(omega_r, basis_colors, weights):
    c_s = [0.0, 0.0, 0.0]
    for i in range(len(basis_colors)):
        c_s[0] += weights[i] * basis_colors[i][0]
        c_s[1] += weights[i] * basis_colors[i][1]
        c_s[2] += weights[i] * basis_colors[i][2]
    return c_s

def compute_attenuation(theta, m):
    # Standard Fresnel approximation [G]
    return math.pow(1.0 - math.cos(theta), 5.0) * (1.0 - m) + m

def render_color(x, omega_o, sdf_fn, density_fn, 
                 diffuse_fn, specular_tint_fn, metallic_fn,
                 weights_fn, basis_colors_fn, s_param, tone_mapping, use_volume=True):
    d = sdf_fn(x)
    sigma_vol = density_fn(x) if use_volume else 0.0
    
    alpha_surf = sigmoid(d, s_param)
    step_size = 0.01
    alpha_vol = 1.0 - math.exp(-sigma_vol * step_size) if use_volume else 0.0
    
    alpha_hyb = 1.0 - (1.0 - alpha_surf) * (1.0 - alpha_vol)
    
    n = compute_normal(sdf_fn, x)
    omega_r = compute_reflected_direction(omega_o, n)
    
    c_d = diffuse_fn(x)
    s_tint = specular_tint_fn(x)
    m = metallic_fn(x)
    weights = weights_fn(x)
    
    c_s_basis = basis_colors_fn(omega_r)
    c_s = compute_specular_color(omega_r, c_s_basis, weights)
    
    theta = math.acos(max(-1.0, min(1.0, dot_product(normalize(omega_o), n))))
    A = compute_attenuation(theta, m)
    
    c = tone_mapping([c_d[i] + s_tint[i] * A * c_s[i] for i in range(3)])
    return c, alpha_hyb, alpha_surf

def composite(colors, alphas):
    c_out = [0.0, 0.0, 0.0]
    T = 1.0
    for c, a in zip(colors, alphas):
        for j in range(3):
            c_out[j] += T * a * c[j]
        T *= (1.0 - a)
    return c_out

def train_step(images, masks, ray_origins, ray_directions, sample_ts, 
               sdf_fn, density_fn, color_fns, basis_fn, s_param, lambda_param):
    total_loss = 0.0
    for img, mask, ray_o, ray_d in zip(images, masks, ray_origins, ray_directions):
        points = [[ray_o[i] + ray_d[i] * t for i in range(3)] for t in sample_ts]
        
        d_vals = [sdf_fn(p) for p in points]
        sigma_vol_vals = [density_fn(p) for p in points]
        
        alphas = []
        for i in range(len(points) - 1):
            alpha = compute_opacity_from_sdf(d_vals[i], d_vals[i+1], s_param)
            alphas.append(alpha)
        
        c_hyb_list, c_surf_list, alpha_hyb_list, alpha_surf_list = [], [], [], []
        for i, p in enumerate(points):
            c_h, a_h, a_s = render_color(p, ray_d, sdf_fn, density_fn, 
                                         color_fns, basis_fn, s_param, lambda_param, use_volume=True)
            c_s, _, _ = render_color(p, ray_d, sdf_fn, density_fn, 
                                     color_fns, basis_fn, s_param, lambda_param, use_volume=False)
            c_hyb_list.append(c_h)
            c_surf_list.append(c_s)
            alpha_hyb_list.append(a_h)
            alpha_surf_list.append(a_s)
        
        C_hyb = composite(c_hyb_list, alpha_hyb_list)
        C_surf = composite(c_surf_list, alpha_surf_list)
        
        L_pm = sum((C_hyb[j] - img[j])**2 + (C_surf[j] - img[j])**2 for j in range(3))
        
        M_hat = sum(alpha_hyb_list)
        L_mask = -(mask * math.log(M_hat + 1e-8) + (1 - mask) * math.log(1 - M_hat + 1e-8))
        
        L_eik = sum((vector_norm(compute_normal(sdf_fn, p)) - 1.0)**2 for p in points)
        
        L_sp = sum(abs(1.0 - math.exp(-lambda_param * sigma_vol_vals[i])) for i in range(len(sigma_vol_vals)))
        
        total_loss += L_pm + L_mask + L_eik + L_sp
    return total_loss
```

**Edge Cases and Pitfalls:**
*   **Precision:** The Sigmoid parameter $s$ controls surface uncertainty. If $s$ is too small, the surface becomes too thick; if too large, it may become numerically sensitive [S1].
*   **Boundaries:** The hybrid opacity combination assumes independence between surface and volume opacity. The `max(..., 0)` guard in the opacity formula handles negative density transitions [S1].
*   **Normal Quality:** The analytic normal from the SDF can be noisy [S1]. The normal smoothness loss $L_{norm}$ is crucial to stabilize normals during early training [S1].

## Subtopics

### VoISDF Volume Rendering and Adaptive Sampling
Uses adaptive sampling for volume rendering [G]. [821-voisdf-volume-rendering-and-adaptive-sampling.md]

### Gaussian Head Avatar (GHA)
Uses Gaussians for head avatars [G]. [1478-gaussian-head-avatar-gha.md]

### MAtCha
Multi-view stereo [G]. [1487-matcha.md]

### Explicit Neural Feature Volume
Uses feature grids [G]. [1838-explicit-neural-feature-volume.md]

### Joint Learning with Shared MLPs
Shares MLPs across features [G]. [2086-joint-learning-with-shared-mlps.md]

### Per-Primitive Color Representations
Assigns color per primitive [G]. [2582-per-primitive-color-representations.md]

### 2D Feature Fusion
Fuses 2D features [G]. [2907-2d-feature-fusion.md]

### Structure-aware primitive learning
Learns primitives aware of structure [G]. [3157-structure-aware-primitive-learning.md]

### Splat the Net
Splatting-based rendering [G]. [3241-splat-the-net.md]

### Neural Primitive Splatting
Neural primitives with splatting [G]. [3300-neural-primitive-splatting.md]

### K-planes
Planar factorization [G]. [3317-k-planes.md]

### Neural Geometric Level of Detail
LOD for neural fields [G]. [3484-neural-geometric-level-of-detail.md]

### MLP Neural Fields
Pure MLP fields [G]. [3854-mlp-neural-fields.md]

### Hash Grid Neural Fields
Hash grid encoding [G]. [3855-hash-grid-neural-fields.md]

### VEG
Vector encoding grid [G]. [4089-veg.md]

### Forplane
Planar features [G]. [5177-forplane.md]

### Feature Grid as Radiance Field
Grid-based radiance [G]. [5508-feature-grid-as-radiance-field.md]

### Atlas-Net
Atlas-based representation [G]. [6961-atlas-net.md]

### Dynamic MLP Maps
Dynamic maps [G]. [7360-dynamic-mlp-maps.md]

### Neural Voxel Fields
Voxel-based neural fields [G]. [7824-neural-voxel-fields.md]

### Viewshed Fields (VF) Learning
View-dependent fields [G]. [7849-viewshed-fields-vf-learning.md]

### Primitive-bound feature embedding
Embeds features per primitive [G]. [9161-primitive-bound-feature-embedding.md]

### Masked Wavelet Representation
Wavelet-based [G]. [9620-masked-wavelet-representation.md]

### Marching Neurons
Marching cubes on neurons [G]. [9793-marching-neurons.md]

### Neural Network 3D Object Representation
General NN 3D rep [G]. [11278-neural-network-3d-object-representation.md]

### Neural Graph Mapping
Graph-based mapping [G]. [11959-neural-graph-mapping.md]

### Monte Carlo Volumetric Estimation
MC estimation [G]. [12926-monte-carlo-volumetric-estimation.md]

### Patch-Grid
Patch-based grid [G]. [14449-patch-grid.md]

## Operating characteristics
*   **Time Complexity:** Real-time rendering at 2K resolution and 60FPS on common consumer devices (laptops, tablets, mobile phones) [S1].
*   **Memory Complexity:** Compact storage, occupying moderate disk space (<50MB) [S1].
*   **Measured Performance:** Achieves comparable view synthesis quality to existing real-time NeRF variants on the NeRF-Synthetic dataset [S1].
*   **Scaling:** Efficient in storage and processing, suitable for integration into modern graphics engines [S1].
*   **Strengths:** Retains advantages of mesh-based assets (efficient rendering, compact storage, easy editing) while incorporating the ability to represent subtle geometric structures provided by the volumetric counterpart [S1].
*   **Limitations:** Does not assume a strict physically-based rendering model, which may limit physical accuracy compared to inverse rendering methods [S1].
*   **When to use:** When real-time view synthesis with high fidelity and compact storage is required, especially for objects with subtle structures [S1].
*   **When not to use:** When strict physical accuracy is required, or when the scene is too complex for mesh representation [S1].

## Background
*   **Origin:** VMesh was proposed by Guo et al. in the paper "VMesh: Hybrid Volume-Mesh Representation for Efficient View Synthesis" [S1].
*   **History and Development:** The work draws inspiration from recent implicit neural surface reconstruction methods where signed distance values can be converted to volume densities [S1]. It aims to address the high rendering costs and difficulty in editing associated with pure neural radiance fields [S1].
*   **Where it is used:** Real-time immersive applications, web-based rendering via WebGL2, and integration with modern graphics engines [S1].

## Sources
[S1] VMesh: Hybrid Volume-Mesh Representation for Efficient View Synthesis, https://core.ac.uk/download/560406650.pdf, fetched 2026-10-27
