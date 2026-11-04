# Gaussian Splatting Volume Rendering

Also called: 3D Gaussian Splatting (3DGS) Rendering, Volume Splatting

## Overview
Gaussian Splatting Volume Rendering is a differentiable rendering technique that synthesizes images by projecting 3D Gaussian distributions onto the image plane and alpha-compositing them along viewing rays [S1][S2, unofficial]. It solves the problem of high-fidelity, real-time novel view synthesis by replacing traditional triangle meshes with probabilistic volume primitives, allowing for efficient tile-based rasterization and differentiable optimization of scene parameters [S2, unofficial]. Unlike standard rasterization which uses geometric primitives, this method materializes smooth ellipses in 2D from 3D probability distributions, capturing view-dependent effects via spherical harmonics [S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   A set of $N$ 3D Gaussians, each defined by a centroid, covariance (parameterized by scale and rotation), opacity, and spherical harmonic coefficients [S1][S2, unofficial].
*   Camera parameters (view and projection matrices) [S2, unofficial].
*   Image resolution $W \times H$ [S1].

**Outputs:**
*   A synthesized RGB image $I_f$ [S1].

### Data Structures
The scene is stored as an array of Gaussian primitives. A typical layout for a single Gaussian splat is [S2, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct GaussianSplat {
    glm::vec3 centroid;           // World-space position (3 floats)
    float opacity;                // Scalar opacity (1 float)
    std::array<float, 48> sphericalHarmonics; // SH coefficients: 16 coefficients * 3 channels (48 floats)
    std::array<float, 3> scale;   // Log-space scale factors (3 floats)
    std::array<float, 4> rotation;// Quaternion rotation (4 floats)
};
```

*   **Scale:** Stored in log-space to ensure positivity. The diagonal scaling matrix $S$ is obtained by $S = \text{diag}(\exp(\text{scale}))$ [S2, unofficial].
*   **Rotation:** Stored as a unit quaternion $q$, converted to a rotation matrix $R$ [S2, unofficial].
*   **Spherical Harmonics (SH):** Coefficients $c_{\ell m}$ for degree $L=3$ (16 coefficients per channel) allow view-dependent color representation [S2, unofficial].

### Numbered Steps

1.  **Load Scene:** Parse the Gaussian data (typically from a `.ply` file) into an array of `GaussianSplat` objects [S2, unofficial].
2.  **Project Gaussians:** For each Gaussian $i$, project its 3D centroid $\mu_i$ and covariance $\Sigma_i$ into 2D screen space.
    *   The 3D covariance is factorized as $\Sigma_i = R_i S_i S_i^T R_i^T$, where $R_i$ is the rotation matrix and $S_i$ is the diagonal scaling matrix [S2, unofficial].
    *   The 2D covariance $\tilde{\Sigma}_i$ is computed using the Jacobian of the projection transformation [S2, unofficial].
3.  **Sort Gaussians:** Sort all Gaussians that intersect the view frustum by their depth (distance from the camera) along each viewing ray [S1].
4.  **Compute Color:** For each Gaussian $i$ and a viewing direction $\mathbf{v}$, compute the color $c_i$ using spherical harmonics:
    $$ c_i = \text{clamp}\left( \mathbf{0.5} + \sum_{k=0}^{15} \text{shCoeff}_{i,k} \cdot \text{SHBasis}_k(\mathbf{v}), 0.0, 1.0 \right) $$
    where $\text{shCoeff}_{i,k}$ are the SH coefficients and $\text{SHBasis}_k$ are the basis functions evaluated at the viewing direction [S2, unofficial].
5.  **Alpha-Composite:** For each pixel, iterate through the sorted Gaussians $i=1 \dots N$ and accumulate color and opacity using the emission-absorption model [S1]:
    $$ I_f \approx \sum_{i=1}^N c_i \tilde{\sigma}_i \tilde{\mathcal{G}}_i(\mathbf{p}) \prod_{j=1}^{i-1} \left(1 - \tilde{\sigma}_j \tilde{\mathcal{G}}_j(\mathbf{p})\right) $$
    where:
    *   $c_i$ is the color of Gaussian $i$.
    *   $\tilde{\sigma}_i$ is the opacity of Gaussian $i$.
    *   $\tilde{\mathcal{G}}_i(\mathbf{p})$ is the normalized 2D Gaussian value at pixel $\mathbf{p}$, derived from the projected covariance $\tilde{\Sigma}_i$ [S1].
    *   The product term represents the transmittance from all preceding Gaussians $j < i$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_gaussian_splatting(scene, camera, width, height):
    """
    Renders a scene of 3D Gaussians using tile-based rasterization.
    scene: List of GaussianSplat objects.
    camera: Object containing view and projection matrices.
    width, height: Image dimensions.
    Returns: Image buffer of shape (height, width, 3).
    """
    image = np.zeros((height, width, 3), dtype=np.float32)
    
    # 1. Project and Filter Gaussians
    # Project 3D Gaussians to 2D screen space
    projected_gaussians = []
    for g in scene:
        # Project centroid
        center_2d = project_point(g.centroid, camera.view_matrix, camera.proj_matrix, width, height)
        
        # Project covariance
        # Sigma_3d = R * diag(exp(scale)) * diag(exp(scale)) * R^T
        S_diag = np.exp(g.scale)
        R = quaternion_to_rotation_matrix(g.rotation)
        Sigma_3d = R @ np.diag(S_diag**2) @ R.T
        
        # Compute 2D covariance using Jacobian of projection
        # J is the Jacobian of the perspective projection
        J = compute_projection_jacobian(g.centroid, camera)
        Sigma_2d = J @ Sigma_3d @ J.T
        
        # Ensure positive semi-definite and valid
        if is_valid_gaussian(Sigma_2d):
            projected_gaussians.append({
                'center': center_2d,
                'cov': Sigma_2d,
                'opacity': g.opacity,
                'sh_coeffs': g.sphericalHarmonics,
                'depth': g.centroid.z # Depth in camera space
            })
            
    # 2. Sort by Depth
    projected_gaussians.sort(key=lambda x: x['depth'])
    
    # 3. Rasterize (Tile-based)
    # For each pixel, accumulate color
    for y in range(height):
        for x in range(width):
            color = np.array([0.5, 0.5, 0.5]) # Bias for SH
            transmittance = 1.0
            
            for g in projected_gaussians:
                # Check if Gaussian covers this pixel (e.g., within 3 sigma)
                if not covers_pixel(g['center'], g['cov'], x, y):
                    continue
                
                # Evaluate 2D Gaussian at pixel
                dx = x - g['center'][0]
                dy = y - g['center'][1]
                # Mahalanobis distance
                dist_sq = np.array([dx, dy]) @ np.linalg.inv(g['cov']) @ np.array([dx, dy])
                
                # Gaussian value (unnormalized for efficiency, opacity handles scaling)
                # G(p) = exp(-0.5 * dist_sq)
                g_val = np.exp(-0.5 * dist_sq)
                
                if g_val < 1e-4:
                    continue
                
                # Compute Color using SH
                # Direction vector from camera to centroid
                view_dir = normalize(g['centroid'] - camera.position)
                sh_color = spherical_harmonics_color(g['sh_coeffs'], view_dir)
                
                # Alpha compositing step
                alpha = g['opacity'] * g_val
                alpha = np.clip(alpha, 0, 1)
                
                # Update color and transmittance
                # I_f += c_i * sigma_i * G_i * Product(1 - sigma_j * G_j)
                color += sh_color * alpha * transmittance
                transmittance *= (1 - alpha)
                
                if transmittance < 1e-4:
                    break
            
            image[y, x] = color
            
    return image

def spherical_harmonics_color(sh_coeffs, direction):
    """
    Computes RGB color from SH coefficients and viewing direction.
    sh_coeffs: array of 48 floats (16 coeffs * 3 channels)
    direction: normalized 3D vector
    Returns: RGB vector in [0, 1]
    """
    # Bias is 0.5
    rgb = np.array([0.5, 0.5, 0.5])
    
    # Evaluate SH basis functions for degree 3 (16 terms)
    # This is a simplified loop; in practice, explicit basis functions are used
    for i in range(16):
        basis_val = evaluate_sh_basis(i, direction)
        # Coefficients are stored as [R0, G0, B0, R1, G1, B1, ...]
        # Or interleaved? Standard 3DGS stores R, G, B separately or interleaved.
        # Assuming interleaved: coeff[3*i] is R, 3*i+1 is G, 3*i+2 is B
        rgb[0] += sh_coeffs[3*i] * basis_val
        rgb[1] += sh_coeffs[3*i+1] * basis_val
        rgb[2] += sh_coeffs[3*i+2] * basis_val
        
    return np.clip(rgb, 0.0, 1.0)
```

### Parameters
*   **Number of Gaussians:** Typically 200,000–500,000 for complex scenes [S1].
*   **SH Degree:** Degree 3, resulting in 16 coefficients per channel (48 total floats per Gaussian) [S2, unofficial].
*   **Bias:** A constant bias of 0.5 is added to the SH output to center the color around mid-grey before clamping [S2, unofficial].

### Edge Cases and Pitfalls
*   **Covariance Validity:** Directly updating the entries of the covariance matrix $\Sigma$ during training does not guarantee it remains positive semi-definite (PSD). Using scale and rotation parameters ensures the reconstructed $\Sigma$ is always a valid covariance matrix [S2, unofficial].
*   **Aliasing:** High-frequency content can cause aliasing artifacts. A dilation-based filter is often applied to reduce these, though it may fail with varying focal lengths or camera distances [S1].
*   **Lighting Limitations:** The method uses an emission-absorption model, baking lighting into SH coefficients. This limits performance on highly reflective surfaces and prevents dynamic relighting [S1].
*   **Isotropic vs Anisotropic:** If the renderer only supports isotropic volumes (e.g., some volume rendering implementations), it may use base colors instead of SH coefficients, resulting in dimmer, less vibrant images that lack viewpoint-dependent color shifts [S3, unofficial].

## Subtopics

### Volumetric Gaussian Splatting (AVIS)
A variant that treats Gaussians as volumetric primitives rather than surface emitters, allowing for the rendering of participating media and smoke [S3, unofficial]. It differs from standard 3DGS by integrating density and color fields into a volume grid, often requiring additional steps like bounding box calculation and volume clipping [S3, unofficial].

### Gaussian Splatting for Microscopy
Adapts the splatting technique for microscopy data, often involving specific initialization policies and density control to handle the unique characteristics of biological samples [S3301].

### Direct Gaussian Splatting
A variant that may alter the initialization or optimization pipeline to directly map input data to Gaussians without intermediate steps [S3555].

### Efficiency Variants (EffGS, UltraGauss)
Methods designed to reduce the memory and computational cost of rendering large numbers of Gaussians, often through pruning, merging, or optimized data structures [S3365, S5170].

### Surface Normal Rendering via External Channel
A technique to render surface normals by treating them as an extra attribute in an external channel, since Gaussian ellipsoids do not natively contain surface normal information [S4, unofficial]. Normals are computed externally (e.g., via PyTorch3D) and interpolated into the image, allowing for subsequent lighting integration using standard diffuse functions [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ per pixel in the worst case, but tile-based rasterization and early-z culling reduce the effective complexity significantly. Sorting is $O(N \log N)$ [S1].
*   **Memory Complexity:** $O(N)$, where $N$ is the number of Gaussians. Each Gaussian stores position, scale, rotation, opacity, and 48 SH coefficients [S2, unofficial].
*   **Performance:** Capable of real-time frame rates (60+ FPS) on modern GPUs for scenes with hundreds of thousands of Gaussians [S1].
*   **Strengths:** High visual fidelity, real-time rendering, differentiable optimization, compact representation compared to NeRFs [S1][S2, unofficial].
*   **Limitations:** High memory usage for large scenes, artifacts at varying focal lengths, inability to handle dynamic lighting or reflections natively [S1][S3, unofficial].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 as a method for real-time 3D scene reconstruction and novel view synthesis [S1]. It builds on the concept of volume rendering and splatting, adapting it for differentiable optimization. The method has since inspired numerous variants and extensions, including volumetric splatting for media and surface normal rendering [S3, S4, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] 3D Gaussian Splatting in a Weekend, https://bfeldman.me/3dgs-weekend/, fetched 2026-10-26 (unofficial)
[S3] ShengzhiWu/taichi-volume-renderer, https://github.com/ShengzhiWu/taichi-volume-renderer, fetched 2026-10-26 (unofficial)
[S4] VoGE: A Differentiable Volume Renderer using Gaussian Ellipsoids for Analysis-by-Synthesis, https://doi.org/10.48550/arxiv.2205.15401, fetched 2026-10-26 (unofficial)
