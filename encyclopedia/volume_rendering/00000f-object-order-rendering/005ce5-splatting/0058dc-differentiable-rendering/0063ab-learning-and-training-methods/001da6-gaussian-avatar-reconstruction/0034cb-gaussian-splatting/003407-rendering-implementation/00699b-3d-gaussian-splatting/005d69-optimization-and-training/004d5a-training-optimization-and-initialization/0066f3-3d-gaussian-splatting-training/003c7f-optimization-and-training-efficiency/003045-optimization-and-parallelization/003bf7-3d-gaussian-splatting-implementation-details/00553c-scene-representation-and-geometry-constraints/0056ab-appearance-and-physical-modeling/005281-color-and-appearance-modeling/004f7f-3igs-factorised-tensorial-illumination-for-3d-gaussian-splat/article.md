# 3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)


## Overview
3iGS enhances 3D Gaussian Splatting (3DGS) by modeling outgoing radiance as a function of a continuous local illumination field and individual Gaussian BRDF attributes, rather than optimizing outgoing radiance in isolation [S1]. It solves the problem of inaccurate specular view-dependent effects in standard 3DGS, which lacks physical material definitions and fails to capture lighting interactions across viewing angles [S1]. Unlike vanilla 3DGS, which uses spherical harmonics for view-dependent color, 3iGS separates scene lighting (via a factorised tensor grid) from material properties (via BRDF features) to model specular effects [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 3D Gaussians $\{G_i\}_{i=1}^N$ initialized with means $\boldsymbol{\mu}_i$, covariance matrices $\boldsymbol{\Sigma}_i$ (derived from rotation $\mathbf{R}_i$ and scale $\mathbf{S}_i$), opacity $\alpha_i$, and initial BRDF features. A scene bounding box defining the initial illumination grid boundary.
*   **Outputs:** A rendered image $\hat{\mathbf{C}}$ and optimized parameters: Gaussian means, opacities, covariances, diffuse colors, BRDF features, and the illumination grid tensors.

**Data Structures:**
1.  **3D Gaussian $G_i$:**
    *   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Mean coordinate.
    *   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix (orthonormal).
    *   $\mathbf{S}_i \in \mathbb{R}^3$: Scale vector.
    *   $\alpha_i \in [0, 1]$: Opacity.
    *   $\mathbf{c}_{d,i} \in \mathbb{R}^3$: Diffuse color.
    *   $\boldsymbol{\rho}_i$: BRDF features. Repurposed from 3DGS: 16x3 Spherical Harmonic (SH) coefficients (48 values) plus 4 additional parameters for base color and roughness used in Integrated Directional Encoding (IDE) [S1].
2.  **Illumination Grid $\mathcal{G}_l$:**
    *   Represented using Vector-Matrix (VM) spatial factorisation [S1].
    *   Resolution: Coarse grid of $150^3$ voxels [S1].
    *   Structure: Sum of rank-$R_L$ components. Each component consists of feature tensors $\mathbf{A}_{L,r}^X, \mathbf{A}_{L,r}^Y, \mathbf{A}_{L,r}^Z$ and feature vectors $\mathbf{b}_{3r-2}, \mathbf{b}_{3r-1}, \mathbf{b}_{3r}$ [S1].
    *   $\mathbf{A}_{L,r}^X \in \mathbb{R}^{N_X \times R_L}$, $\mathbf{A}_{L,r}^Y \in \mathbb{R}^{N_Y \times R_L}$, $\mathbf{A}_{L,r}^Z \in \mathbb{R}^{N_Z \times R_L}$ (where $N_X, N_Y, N_Z$ are grid dimensions).
    *   $\mathbf{b} \in \mathbb{R}^{3 R_L}$: Appearance feature mode vectors [S1].

**Numbered Steps:**

1.  **Covariance Transformation:**
    For each Gaussian $i$, compute the 3D covariance $\boldsymbol{\Sigma}_i$ from learned scale $\mathbf{S}_i$ and rotation $\mathbf{R}_i$:
    $$ \boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T $$
    Transform to camera coordinates using viewing transformation $\mathbf{W}$ and affine approximation of projective transformation $\mathbf{J}$:
    $$ \boldsymbol{\Sigma}'_i = \mathbf{J} \mathbf{W} \boldsymbol{\Sigma}_i \mathbf{W}^T \mathbf{J}^T $$

2.  **Illumination Query:**
    For each Gaussian $i$ with mean $\mathbf{x}_i$, compute incoming illumination $L_i$ by interpolating from the factorised tensor illumination grid $\mathcal{G}_l$:
    $$ \mathcal{G}_l : \{ \mathbf{x}_i \} \mapsto L_i $$
    The grid value is calculated as:
    $$ \mathcal{G}_l = \sum_{r=1}^{R_L} \mathbf{A}_{L,r}^X \circ \mathbf{b}_{3r-2} + \mathbf{A}_{L,r}^Y \circ \mathbf{b}_{3r-1} + \mathbf{A}_{L,r}^Z \circ \mathbf{b}_{3r} $$
    where $\circ$ denotes the appropriate tensor contraction/interpolation based on the coordinate $\mathbf{x}_i$ [S1].

3.  **Shading (Specular Color):**
    Compute the specular color $\mathbf{c}_{s,i}$ for viewing direction $\omega_o$ using a neural network $\mathcal{F}$:
    $$ \mathcal{F} : \{ \boldsymbol{\rho}_i, L_i, \omega_o \} \mapsto \mathbf{c}_{s,i} $$
    The network $\mathcal{F}$ takes the Gaussian BRDF features $\boldsymbol{\rho}_i$, the queried illumination $L_i$, and the viewing direction $\omega_o$ (encoded with IDE) [S1].

4.  **Radiance Composition:**
    Compute the final outgoing radiance $\mathbf{c}_i(\omega_o)$ by linearly adding the diffuse color $\mathbf{c}_{d,i}$ and the specular color $\mathbf{c}_{s,i}$:
    $$ \mathbf{c}_i(\omega_o) = \mathbf{c}_{d,i} + \mathbf{c}_{s,i} $$

5.  **Rendering:**
    Render the scene using the standard 3DGS rasterization pipeline. For a pixel $\mathbf{u}$, alpha blend the contributions of $N$ ordered Gaussians:
    $$ \hat{\mathbf{C}} = \sum_{i \in N} T_i g_i(\mathbf{u} | \boldsymbol{\mu}'_i, \boldsymbol{\Sigma}'_i) \alpha_i \mathbf{c}_i(\omega_o) $$
    $$ T_i = \prod_{j=1}^{i-1} (1 - g_j(\mathbf{u} | \boldsymbol{\mu}'_j, \boldsymbol{\Sigma}'_j) \alpha_j) $$
    where $g_i$ is the 2D projected Gaussian density [S1].

6.  **Optimization Training Strategy:**
    *   **Phase 1 (Iterations 0–3,000):** Train the model using only the diffuse color $\mathbf{c}_{d,i}$. Optimize means, opacities, covariances, and diffuse colors.
    *   **Phase 2 (Iterations 3,000–30,000):** Add specular colors. Optimize BRDF features $\boldsymbol{\rho}_i$ and the illumination grid $\mathcal{G}_l$ jointly with other parameters [S1].
    *   **Grid Boundary Management:** Define an initial boundary encapsulating the scene bounding box. Midway through training (approx. 15,000 iterations), shrink the illumination grid to fit the Gaussians and resample the grid with the same number of voxels [S1].

7.  **Loss Function:**
    Minimize the combined loss:
    $$ \mathcal{L} = (1 - \lambda)\mathcal{L}_1 + \lambda \mathcal{L}_{\text{D-SSIM}} $$
    with $\lambda = 0.2$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_covariance(R, S):
    """
    Compute 3D covariance from rotation R and scale S.
    R: (3, 3) rotation matrix
    S: (3,) scale vector
    """
    S_matrix = np.diag(S)
    return R @ S_matrix @ S_matrix.T @ R.T

def transform_covariance_to_camera(Sigma, W, J):
    """
    Transform covariance to camera coordinates.
    Sigma: (3, 3) 3D covariance
    W: (3, 3) viewing transformation matrix
    J: (3, 3) affine approximation of projective transformation
    """
    return J @ W @ Sigma @ W.T @ J.T

def query_illumination_grid(x_i, grid_tensors, grid_vectors, grid_dims):
    """
    Query the factorised tensor illumination grid.
    x_i: (3,) mean coordinate of Gaussian
    grid_tensors: List of tuples (A_X, A_Y, A_Z) for each rank r
    grid_vectors: (3 * R_L,) feature vectors b
    grid_dims: (3,) dimensions of the grid (N_X, N_Y, N_Z)
    """
    R_L = len(grid_tensors)
    illumination = np.zeros(3) # Assuming 3D illumination vector
    
    for r in range(R_L):
        A_X, A_Y, A_Z = grid_tensors[r]
        # Extract vectors for this rank
        b_idx = 3 * r
        b_x = grid_vectors[b_idx]
        b_y = grid_vectors[b_idx + 1]
        b_z = grid_vectors[b_idx + 2]
        
        # Interpolate A tensors at x_i
        # Note: Actual interpolation depends on specific VM implementation details
        # Here we assume a helper function for trilinear interpolation of a tensor
        val_X = interpolate_tensor(A_X, x_i, grid_dims[0])
        val_Y = interpolate_tensor(A_Y, x_i, grid_dims[1])
        val_Z = interpolate_tensor(A_Z, x_i, grid_dims[2])
        
        # VM Factorisation combination
        # A_X is (N_X, R_L), A_Y is (N_Y, R_L), A_Z is (N_Z, R_L)
        # The formula is sum_r (A_X_r * b_{3r-2} + A_Y_r * b_{3r-1} + A_Z_r * b_{3r})
        # Assuming val_X, val_Y, val_Z are the interpolated feature vectors of size R_L
        illumination += val_X * b_x + val_Y * b_y + val_Z * b_z
        
    return illumination

def shade_gaussian(brdf_features, illumination, view_direction):
    """
    Compute specular color using neural network F.
    brdf_features: (48 + 4,) concatenated SH coefficients and base/roughness
    illumination: (3,) incoming light
    view_direction: (3,) viewing direction omega_o
    """
    # IDE encoding of view direction
    ide_encoding = integrated_directional_encoding(view_direction)
    
    # Concatenate inputs for neural network F
    input_features = np.concatenate([brdf_features, illumination, ide_encoding])
    
    # Neural network forward pass
    specular_color = neural_network_F(input_features)
    return specular_color

def render_gaussians(gaussians, camera_params, illumination_grid):
    """
    Render scene using 3DGS rasterization.
    """
    # 1. Project Gaussians to 2D
    projected_gaussians = []
    for g in gaussians:
        Sigma_3d = compute_covariance(g.R, g.S)
        Sigma_2d = transform_covariance_to_camera(Sigma_3d, camera_params.W, camera_params.J)
        
        # Query illumination
        L_i = query_illumination_grid(g.mu, illumination_grid['tensors'], 
                                      illumination_grid['vectors'], illumination_grid['dims'])
        
        # Shade
        c_d = g.c_diffuse
        c_s = shade_gaussian(g.brdf_features, L_i, camera_params.view_direction)
        c_total = c_d + c_s
        
        projected_gaussians.append({
            'mu_2d': project_mean(g.mu, camera_params),
            'Sigma_2d': Sigma_2d,
            'alpha': g.alpha,
            'color': c_total,
            'original_index': g.index
        })
    
    # 2. Rasterize (Alpha Blending)
    # Sort by depth, iterate over pixels, accumulate color
    # This is a simplified representation of the rasterization pipeline
    image = np.zeros((height, width, 3))
    for g in sorted(projected_gaussians, key=lambda k: k['mu_2d'].z):
        # Evaluate 2D Gaussian at pixel centers covered by bounding box
        # Accumulate into image buffer
        pass
        
    return image

def optimize_step(gaussians, illumination_grid, images, targets, iteration):
    """
    Perform one optimization step.
    """
    # Phase 1: Only diffuse color for first 3000 iterations
    if iteration < 3000:
        # Compute loss using only diffuse colors
        # Backpropagate to optimize mu, alpha, R, S, c_diffuse
        pass
    else:
        # Phase 2: Include specular colors
        # Compute loss using total radiance
        # Backpropagate to optimize mu, alpha, R, S, c_diffuse, brdf_features, illumination_grid tensors/vectors
        
        # Grid Boundary Management: Shrink grid midway
        if iteration == 15000:
            illumination_grid = shrink_and_resample_grid(illumination_grid, gaussians)
            
    return loss
```

## Subtopics

### [Physically-Based 3D Gaussian Splatting](16738-physically-based-3d-gaussian-splatting.md)
Physically-Based 3D Gaussian Splatting (PBR 3DGS) is a framework that assigns physically interpretable rendering attributes to Gaussian primitives and evaluates the rendering equation to produce per-Gaussian PBR colors, replacing the view-dependent spherical harmonics used in vanilla 3DGS with a physical Bidirectional Reflectance Distribution Function (BRDF). It solves the problem of inverse rendering and relighting by enabling the simulation of complex lighting interactions, such as indirect illumination and shadows, which vanilla 3DGS cannot natively represent due to its lack of physical material definitions. Unlike vanilla 3DGS, which uses $k$-order spherical harmonics for view-dependent effects that lack physical interpretability, this method provides physical interpretability by explicitly modeling albedo, roughness, and metalness.

### [Variational Quantum Circuit (VQC) for View-Dependent Encoding](17531-variational-quantum-circuit-vqc-for-view-dependent-encoding.md)
This method replaces or modulates classical Spherical Harmonics (SH) with a Variational Quantum Circuit (VQC) to model view-dependent radiance and opacity for 3D Gaussians, specifically targeting high-frequency specular effects and reducing artifacts like "floaters". It sets itself apart from its parent [3iGS](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md) by using quantum-inspired entanglement and Bloch sphere mapping for directional encoding, and from its sibling [Physically-Based 3D Gaussian Splatting](16738-physically-based-3d-gaussian-splatting.md) by relying on learned quantum parameterizations rather than explicit physical scattering models.

### [Appearance-Aware 3D Gaussian Splatting](18281-appearance-aware-3d-gaussian-splatting.md)
AAGS reconstructs static scenes from unconstrained, in-the-wild photo collections by decoupling intrinsic object properties from extrinsic appearance conditions (lighting and post-processing) and explicitly removing transient objects. Unlike NeRF-based methods (e.g., NeRF-W, Ha-NeRF) which model transients explicitly with high overhead, or SWAG which requires additional Gaussian training for transients, AAGS leverages 3D Gaussian Splatting's fast rendering to use a lightweight 2D visibility mask for transient exclusion.

## Operating characteristics
*   **Time Complexity:** Rendering speed is comparable to 3DGS because the illumination field is pre-calculated and stored in volumes/light probes, allowing fast evaluation via interpolation [S1]. The neural network $\mathcal{F}$ adds minor overhead per Gaussian but is efficient due to the compact grid representation [S1].
*   **Memory Complexity:** The illumination field uses 87.5% less voxels compared to TensoRF on synthetic datasets [S1]. The memory footprint is low compared to the number of optimized Gaussians, which is often an order of magnitude higher [S1].
*   **Performance:** On the Shiny Blender dataset, 3iGS achieves an average PSNR of 27.51, SSIM of 0.930, and LPIPS of 0.045 [S1]. It surpasses 3DGS quantitatively and qualitatively on synthetic datasets (NeRF Blender, Shiny Blender) and real-world scenarios (Tanks and Temples) [S1].
*   **Strengths:** Significantly enhances rendering quality of specular view-dependent effects; maintains rapid training and rendering speeds; models incident illumination locally rather than globally [S1].
*   **Limitations:** Requires careful tuning of the training schedule (diffuse-only phase); grid boundary management is critical for performance [S1].
*   **When to use:** Scenes with reflective surfaces where surface colors change dramatically across viewing angles [S1].
*   **When not to use:** Scenes where global lighting effects (e.g., complex indirect illumination from distant sources) are more important than local specular highlights, or where computational resources for neural network evaluation per Gaussian are constrained.

## Background
3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting) was introduced to address the limitations of vanilla 3DGS in modeling specular view-dependent effects. Vanilla 3DGS optimizes outgoing radiance in isolation using spherical harmonics, which fails to capture the physical interdependencies of scene lighting and material properties [S1]. 3iGS draws inspiration from conventional computer graphics engines that use baking techniques and light probes for fast rendering [S1]. It utilizes Tensorial Factorisation (inspired by TensoRF) to represent the continuous local illumination field compactly [S1]. The method was evaluated on synthetic datasets (NeRF Blender, Shiny Blender) and real-world datasets (Tanks and Temples) [S1].

## Sources
[S1] 3iGS: Factorised Tensorial Illumination for 3D Gaussian Splatting, https://core.ac.uk/works/391741134, fetched 2026-10-29
