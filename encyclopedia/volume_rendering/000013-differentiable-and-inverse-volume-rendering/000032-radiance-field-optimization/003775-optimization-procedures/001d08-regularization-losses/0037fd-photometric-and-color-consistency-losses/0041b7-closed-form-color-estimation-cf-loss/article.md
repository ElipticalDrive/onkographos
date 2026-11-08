# Closed-Form Color Estimation (CF) Loss

Also called: Closed-form photometric loss, closed-form color estimation method

## Overview
Closed-Form Color Estimation (CF) Loss is a regularization technique for explicit radiance fields (such as Plenoxels and DVGO) that breaks the entanglement between density and color fields to reduce shape-radiance ambiguity [S1, unofficial]. Unlike standard photometric losses that supervise the final rendered image directly, CF loss estimates a "closed-form" color field given the current density field and posed training images, then computes a photometric loss between this estimated color and the ground truth to supervise the density field [S1, unofficial]. It is distinct from distortion loss in that it is sensitive to specific geometric errors and can distinguish objects from background where distortion loss fails [S1, unofficial].

## How it works
**Inputs:**
*   Current density field $\mathbf{F}_v^\sigma$ (voxel grid).
*   Current color field $\mathbf{F}_v^c$ (voxel grid, typically represented by Spherical Harmonics coefficients).
*   Set of $K$ source camera poses and intrinsics.
*   Ground truth images.
*   A mini-batch of rays $R_{cf}$ (size $|R_{cf}|$).

**Outputs:**
*   Scalar loss value $L_{cf}$.
*   Gradients with respect to the density field parameters.

**Data Structures:**
*   **Voxel Grid:** 3D grid where each voxel $v$ stores density $\sigma_v$ and color coefficients. For Plenoxels, color is stored as $3 \times (2+1)^2 = 27$ Spherical Harmonic (SH) coefficients per voxel (SH degree 2) [S1, unofficial].
*   **Ray Batch:** A set of $|R_{cf}|$ rays. For Plenoxels, $|R_{cf}|=25$; for DVGO, $|R_{cf}|=10$ [S1, unofficial].

**Algorithm:**

1.  **Ray Selection:** Select a mini-batch of rays $R_{cf}$ from the training set. A uniform probability distribution $p_d(d) \equiv \frac{1}{4\pi}$ is applied to directions [S1, unofficial].
2.  **Voxel Identification:** Identify all voxels intersected by the rays in $R_{cf}$. Let $M$ be the number of such voxels. If each ray intersects $N$ voxels, $M \approx |R_{cf}| \times N$ [S1, unofficial].
3.  **Closed-Form Color Estimation:** For each intersected voxel $v$, estimate its color $\hat{\mathbf{c}}_v$ by solving a linear system that accounts for occlusion and non-uniform view distribution.
    *   The method computes the contribution of voxel $v$ to the rendered pixels of all $K$ cameras.
    *   Let $\mathbf{A}$ be a matrix where each row corresponds to a camera-ray-voxel interaction, encoding the transmittance and geometric factors.
    *   Let $\mathbf{b}$ be the vector of observed pixel colors from the $K$ cameras for the rays passing through $v$.
    *   The estimated color coefficients $\hat{\mathbf{c}}_v$ are obtained by solving $\mathbf{A}^T \mathbf{A} \hat{\mathbf{c}}_v = \mathbf{A}^T \mathbf{b}$ (or a regularized variant) [S1, unofficial].
    *   *Note on Bias Reduction:* The source mentions a scheme where $\tilde{m} = m$ if $i = \ell$, else $\tilde{m} = i$, to reduce estimation biases, referring to Appendix A for details [S1, unofficial].
4.  **Rendering Estimated Color:** Render the scene using the estimated color field $\hat{\mathbf{F}}_v^c$ and the current density field $\mathbf{F}_v^\sigma$ following the standard volume rendering equation (Eq. 1 in [S1, unofficial]):
    $$
    \hat{C}_{cf}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \hat{\mathbf{c}}_i
    $$
    where $\hat{\mathbf{c}}_i$ is the color estimated for the $i$-th voxel along ray $\mathbf{r}$ [S1, unofficial].
5.  **Loss Calculation:** Compute the photometric loss between the ground truth color $C(\mathbf{r})$ and the closed-form estimated color $\hat{C}_{cf}(\mathbf{r})$:
    $$
    L_{cf} = \frac{1}{|R_{cf}|} \sum_{\mathbf{r} \in R_{cf}} ||C(\mathbf{r}) - \hat{C}_{cf}(\mathbf{r})||_2^2
    $$
    [S1, unofficial].
6.  **Regularization:** Add $\lambda L_{cf}$ to the total training loss, where $\lambda$ is a weight factor [S1, unofficial].

**Parameters:**
*   $\lambda$ (Weight factor):
    *   Plenoxels: DTU=10, NeRF synthetic=0.1, LLFF=0.5 [S1, unofficial].
    *   DVGO: DTU coarse=1, fine=0.1; NeRF synthetic coarse=2, fine=0.1; LLFF=0.1 [S1, unofficial].
*   $|R_{cf}|$ (Number of rays): Plenoxels=25, DVGO=10 [S1, unofficial].
*   $K$ (Number of source cameras): DTU=44 or 58, NeRF synthetic=100, LLFF=17 to 54 [S1, unofficial].
*   $N$ (Order of voxels per ray): Not given in the sources.
*   $B$ (Batch size): Typical values 10, 25 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cf_loss(density_field, color_field, cameras, gt_images, ray_batch, lambda_weight):
    """
    Computes the Closed-Form Color Estimation Loss.
    
    Args:
        density_field: 3D tensor of voxel densities.
        color_field: 3D tensor of voxel color coefficients (e.g., SH coefficients).
        cameras: List of camera poses and intrinsics.
        gt_images: Ground truth images corresponding to cameras.
        ray_batch: List of rays to process.
        lambda_weight: Scalar weight for the loss.
        
    Returns:
        loss: Scalar loss value.
    """
    # Step 1: Identify voxels intersected by the ray batch
    intersected_voxels = get_intersected_voxels(density_field, ray_batch)
    
    # Step 2: Estimate closed-form color for each intersected voxel
    estimated_color_field = {}
    for voxel in intersected_voxels:
        # Get all rays passing through this voxel from all K cameras
        voxel_rays = get_rays_through_voxel(voxel, cameras, ray_batch)
        
        # Construct linear system A * c = b
        # A encodes transmittance and geometry from voxel to cameras
        # b encodes observed pixel colors
        A, b = build_linear_system(voxel, voxel_rays, density_field, cameras, gt_images)
        
        # Solve for estimated color coefficients c
        # Using a least-squares solver or regularized inverse
        estimated_c = solve_linear_system(A, b)
        
        estimated_color_field[voxel] = estimated_c
        
    # Step 3: Render the scene using estimated colors and current density
    estimated_rendered_colors = []
    for ray in ray_batch:
        # Get voxels along the ray
        ray_voxels = get_voxels_on_ray(ray, density_field)
        
        # Get estimated colors for these voxels (interpolate if necessary)
        ray_estimated_colors = [estimated_color_field.get(v, default_color) for v in ray_voxels]
        
        # Render using volume rendering equation
        # T_i = exp(-sum(sigma_j * delta_j))
        # C = sum(T_i * (1 - exp(-sigma_i * delta_i)) * c_i)
        rendered_color = volume_render(ray_voxels, ray_estimated_colors, density_field)
        estimated_rendered_colors.append(rendered_color)
        
    # Step 4: Compute photometric loss
    gt_colors = [get_gt_color(ray, gt_images) for ray in ray_batch]
    
    loss = 0.0
    for est_c, gt_c in zip(estimated_rendered_colors, gt_colors):
        loss += sum((est_c - gt_c) ** 2)
        
    loss = loss / len(ray_batch)
    
    # Step 5: Apply weight
    return lambda_weight * loss
```

**Edge Cases and Pitfalls:**
*   **Highly Reflective Objects:** The closed-form color field has difficulty perfectly recovering highly reflective objects, resulting in less reflective surfaces compared to ground truth [S1, unofficial].
*   **CUDA Thread Limits:** High computational overhead if excessive threads are used due to hardware limits on maximum CUDA threads, preventing guaranteed parallel execution [S1, unofficial].
*   **Bias:** Non-uniformly distributed views can cause estimation biases. The source mentions a bias reduction strategy in Appendix A [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   The total number of voxels involved is of order $M = |R_{cf}| \times N$.
*   For each voxel, density values along the ray from the voxel to $K$ source cameras need to be calculated.
*   The total computation is of order $K \times M \times N = K \times |R_{cf}| \times N^2$ without optimization.
*   With CUDA parallelization (one block per voxel, one thread per camera), the theoretical overhead is reduced to order $N$ [S1, unofficial].
*   Memory usage is $O(n)$ for the ray batch processing, avoiding storage of intermediate states for all pixels [S5, unofficial].

**Measured Performance:**
*   **DTU Scan 24:** DVGO PSNR 27.78, IMRC 17.45. DVGO + CF loss PSNR 28.64, IMRC 18.53 [S1, unofficial].
*   **Mic Scene (NeRF synthetic):** DVGO PSNR 28.41, IMRC 8.09. DVGO + CF loss PSNR 34.01, IMRC 19.83 [S1, unofficial].
*   **Training Time (Plenoxels, 9 epochs):**
    *   Batch size 10: DTU ~24 min, NeRF synthetic ~20.5 min, LLFF ~29.4 min [S1, unofficial].
    *   Batch size 25: DTU ~32 min, NeRF synthetic ~25.5 min, LLFF ~34.6 min [S1, unofficial].
    *   Note: DTU dataset requires longer training time because it is trained for 12 epochs in the source, while NeRF synthetic is trained for 9 epochs [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Complements distortion loss; more effective at correcting specific geometric errors like floaters; distinguishes objects from background (e.g., wire vs background) where distortion loss fails [S1, unofficial].
*   **Limitations:** Struggles with highly reflective objects; sensitive to view distribution; computational overhead can be affected by batch size due to hardware limits [S1, unofficial].

**When to Use:**
*   Use with explicit NeRFs (Plenoxels, DVGO) when shape-radiance ambiguity is a concern.
*   Use when distortion loss alone fails to correct specific geometric errors.
*   Not recommended for scenes with highly reflective objects unless SH degree is increased or other mitigations are applied [S1, unofficial].

## Background
**Origin:**
*   Proposed in "Reducing Shape-Radiance Ambiguity in Radiance Fields with a Closed-Form Color Estimation Method" [S1, unofficial].
*   The paper addresses the shape-radiance ambiguity problem in NeRFs, where wrong geometry can be learned and poor novel-views rendered even with simple backgrounds and abundant training images [S1, unofficial].

**History and Development:**
*   The method was implemented using CUDA kernels for explicit NeRFs [S1, unofficial].
*   It was tested on DTU, NeRF synthetic, and LLFF datasets [S1, unofficial].
*   The method uses Spherical Harmonics (SH) degree 2 for color representation in Plenoxels [S1, unofficial].

**Usage:**
*   Applied to explicit NeRFs: Plenoxels and DVGO [S1, unofficial].
*   Used in conjunction with distortion loss for best results on DVGO [S1, unofficial].

## Sources
[S1] Reducing Shape-Radiance Ambiguity in Radiance Fields with a Closed-Form Color Estimation Method, https://doi.org/10.52202/075280-2570, fetched 2026-10-26 (unofficial)
