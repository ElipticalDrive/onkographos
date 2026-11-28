# Single-Scattering Differentiable Volume Rendering

Also called: Ours

## Overview
Single-Scattering Differentiable Volume Rendering is a differentiable renderer that reconstructs smoke density fields using a single-scattering assumption, enabling analytical gradient computation with respect to density $\rho$ [S1]. It solves the inverse problem of recovering volumetric density from multi-view images by computing gradients directly via the differential form of the Radiative Transfer Equation (RTE) [S1]. It sets itself apart from Differential Ray Tracing (DRT) by pre-computing view-independent radiance terms, achieving a 50-100x speedup, and from full multiple-scattering methods by trading absolute accuracy for interactive reconstruction speeds [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of reference images captured from different viewpoints, a volume grid of resolution $R_x \times R_y \times R_z$, and a scattering coefficient $\sigma_t$ [S1].
*   **Output:** A reconstructed density field $\rho(x)$ and its analytical gradients $\frac{\partial L_{out}}{\partial \rho}$ with respect to the rendered image loss [S1].

**Data Structures:**
*   **Density Field:** A 3D grid of floating-point values representing $\rho(x)$ at voxel centers, accessed via trilinear interpolation during ray marching [S1].
*   **Pre-computed Radiance:** A set of radiance values computed for $N_d$ directional samples, stored as view-independent terms to accelerate the inverse pass [S1].

**Parameters:**
*   **Learning Rate ($\alpha$):** 0.01 for gradient descent optimization [S1].
*   **Directional Samples ($N_d$):** 30, determined as the optimal balance between accuracy and computational efficiency [S1].
*   **Iterations:** 200 iterations for reconstruction experiments [S1].
*   **Scattering Coefficient ($\sigma_t$):** Constant, relating optical thickness to density via $\kappa_t(x) = \sigma_t \rho(x)$ [S1].

**Algorithm Steps:**
1.  **Pre-computation:** For each of the $N_d$ directional light samples, compute the radiance contribution assuming single scattering and store these terms. This step scales linearly with $N_d$ because radiance is computed for each sample direction independently [S1].
2.  **Primal Rendering (Forward Pass):** For each pixel in each view, cast a ray through the volume, perform ray marching using trilinear interpolation to sample $\rho(x)$, and compute the transmitted radiance using the single-scattering approximation and the pre-computed directional terms [S1].
3.  **Gradient Computation (Inverse Pass):** Compute the loss $L$ between the rendered image $L_{out}$ and the reference image, then compute the analytical gradient $\frac{\partial L_{out}}{\partial \rho}$ using the differential form of the RTE [S1]. The density directly influences optical properties through $\kappa_t(x) = \sigma_t \rho(x)$, allowing the gradient to be expressed analytically [S1].
4.  **Optimization:** Update the density field $\rho$ using gradient descent: $\rho_{new} = \rho_{old} - \alpha \frac{\partial L}{\partial \rho}$ [S1]. Repeat for the specified number of iterations (e.g., 200) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def single_scattering_differentiable_rendering(reference_images, volume_resolution, sigma_t, num_directional_samples=30, learning_rate=0.01, num_iterations=200):
    # Initialize density field [G]
    rho = [[[0.0 for _ in range(volume_resolution[2])] for _ in range(volume_resolution[1])] for _ in range(volume_resolution[0])]

    # Step 1: Pre-computation of radiance terms
    # Pre-computation scales linearly with directional samples [S1]
    precomputed_radiance = []
    for i in range(num_directional_samples):
        direction = compute_direction(i, num_directional_samples)
        radiance_i = compute_single_scattering_radiance(rho, volume_resolution, sigma_t, direction)
        precomputed_radiance.append(radiance_i)

    # Step 2 & 3: Optimization Loop
    for iteration in range(num_iterations):
        total_loss = 0.0
        total_gradient = [[[0.0 for _ in range(volume_resolution[2])] for _ in range(volume_resolution[1])] for _ in range(volume_resolution[0])]

        for view_idx, ref_image in enumerate(reference_images):
            # Primal Rendering
            rendered_image = render_view(rho, precomputed_radiance, view_idx, volume_resolution, sigma_t)

            # Compute Loss
            loss = compute_loss(rendered_image, ref_image)
            total_loss += loss

            # Inverse Rendering: Compute Gradient
            gradient = compute_gradient(rendered_image, ref_image, rho, precomputed_radiance, view_idx, volume_resolution, sigma_t)
            for x in range(volume_resolution[0]):
                for y in range(volume_resolution[1]):
                    for z in range(volume_resolution[2]):
                        total_gradient[x][y][z] += gradient[x][y][z]

        # Update Density
        for x in range(volume_resolution[0]):
            for y in range(volume_resolution[1]):
                for z in range(volume_resolution[2]):
                    rho[x][y][z] -= learning_rate * total_gradient[x][y][z]

    return rho

def compute_direction(index, total):
    # Generate uniformly distributed directions [G]
    angle = index * 2.399963229728653
    return (math.sin(angle), math.cos(angle), 0.0)

def compute_single_scattering_radiance(rho, resolution, sigma_t, direction):
    # The sources do not provide the exact physics implementation for single-scattering radiance calculation [S1].
    return 0.0

def trilinear_interpolate(rho, x, y, z, resolution):
    # Clamp coordinates to volume bounds
    x = max(0.0, min(float(resolution[0] - 1), x))
    y = max(0.0, min(float(resolution[1] - 1), y))
    z = max(0.0, min(float(resolution[2] - 1), z))
    x0, y0, z0 = int(math.floor(x)), int(math.floor(y)), int(math.floor(z))
    x1, y1, z1 = min(x0 + 1, resolution[0] - 1), min(y0 + 1, resolution[1] - 1), min(z0 + 1, resolution[2] - 1)
    dx, dy, dz = x - x0, y - y0, z - z0
    return (1 - dx) * (1 - dy) * (1 - dz) * rho[x0][y0][z0] + \
           dx * (1 - dy) * (1 - dz) * rho[x1][y0][z0] + \
           (1 - dx) * dy * (1 - dz) * rho[x0][y1][z0] + \
           dx * dy * (1 - dz) * rho[x1][y1][z0] + \
           (1 - dx) * (1 - dy) * dz * rho[x0][y0][z1] + \
           dx * (1 - dy) * dz * rho[x1][y0][z1] + \
           (1 - dx) * dy * dz * rho[x0][y1][z1] + \
           dx * dy * dz * rho[x1][y1][z1]

def render_view(rho, precomputed_radiance, view_idx, resolution, sigma_t):
    # Cast rays for the view and perform ray marching [S1]
    # Uses trilinear interpolation to sample rho [S1]
    # Combines with precomputed radiance terms
    # The sources do not provide the exact ray marching loop implementation [S1].
    return []

def compute_loss(rendered_image, ref_image):
    # Computes Mean Squared Error between rendered and reference image [G]
    return sum((r - ref) ** 2 for r, ref in zip(rendered_image, ref_image)) / len(rendered_image)

def compute_gradient(rendered_image, ref_image, rho, precomputed_radiance, view_idx, resolution, sigma_t):
    # Computes the analytical gradient of the loss with respect to density
    # Uses the differential form of the RTE [S1]
    # The density influences optical properties through kappa_t(x) = sigma_t * rho(x) [S1]
    # The sources do not provide the exact gradient computation implementation [S1].
    return [[[0.0 for _ in range(resolution[2])] for _ in range(resolution[1])] for _ in range(resolution[0])]
```

**Edge Cases and Pitfalls:**
*   **Gradient Noise:** The method exhibits higher gradient noise compared to Finite Difference (FD) and Differential Ray Tracing (DRT) due to ray marching being a biased sampling technique [S1]. Gradient standard deviations are 0.000269 for this method vs. 0.000175 (FD) and 0.000202 (DRT) [S1].
*   **Spatial Bias:** Reconstruction fidelity is higher in outer volume regions (e.g., $z=84$) than in inner regions (e.g., $z=76$) because image-based reconstruction provides stronger constraints for regions directly influencing pixel appearances [S1].
*   **Grid Artifacts:** Regular patterns may appear due to the discrete voxel representation combined with trilinear interpolation. The gradient scattering process preserves the voxel-grid structure rather than promoting smooth transitions [S1].
*   **Flat Density Regions:** Optically thin areas may exhibit unnaturally flat density distributions due to insufficient gradient information from the single-scattering model (where multiple scattering would contribute detail) and the under-constrained nature of the inverse problem [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Pre-computation:** Scales linearly with the number of directional samples $N_d$ and cubically with volume resolution $O(n^3)$ due to the voxel count [S1].
*   **Primal Rendering:** Scales favorably with resolution; consistently fast (21-28 ms for 256³ resolution) regardless of directional sampling count [S1].
*   **Inverse Rendering:** Stable time (91-95 ms for 256³ resolution) due to direct gradient computation [S1].
*   **Memory:** Reasonable usage even for high-resolution volumes (e.g., 2.73 GB for 256³ resolution) [S1].

**Measured Performance:**
*   **Hardware:** NVIDIA GeForce RTX 2080 Ti [S1].
*   **Resolution:** Experiments conducted at 128³ and 256³ resolutions [S1].
*   **Speedup:** 6x speedup over DRT for single-view at 512 samples per pixel, and 27x for multi-view [S1]. Overall 50-100x speedup compared to DRT [S1].
*   **Accuracy:** MSE ranges from 3.1 to 8.88 (pixel intensities 0-255). SSIM values are above 0.8 for all views, with some close to 0.95 [S1].

**Strengths and Limitations:**
*   **Strengths:** High computational efficiency, enabling interactive reconstruction workflows. Suitable for moderately dense smoke and scenarios prioritizing speed over absolute accuracy [S1].
*   **Limitations:** Decreased absolute accuracy compared to full multiple-scattering methods. Susceptible to grid artifacts and spatial bias [S1].

**When to Use:**
*   Moderately dense smoke reconstruction [S1].
*   Scenarios prioritizing reconstruction speed over absolute accuracy [S1].
*   Applications where approximate density distributions are sufficient for downstream processing [S1].

**When Not to Use:**
*   Scenarios requiring high-fidelity reconstruction of optically thick or highly scattering media where multiple scattering effects are significant [S1].
*   Applications where grid artifacts are unacceptable [S1].

## Background
**Origin:**
The method is described in "A High-Efficiency Physics-Based Differentiable Renderer for Smoke Reconstruction" [S1]. It is based on the differential form of the RTE derived in Section 3.2 of the source [S1].

**History and Development:**
The method validates its gradient computation approach against Finite Difference and Differential Ray Tracing [S1]. It is compared against Laplacian Projection, a physics-based tomographic reconstruction method [S1].

**Usage:**
The method is used for smoke reconstruction using the ScalarFlow dataset [S1]. Reconstructed density fields are relit using a standard volumetric path tracker to demonstrate adaptability across diverse lighting conditions [S1].

## Sources
[S1] A High-Efficiency Physics-Based Differentiable Renderer for Smoke Reconstruction, https://doi.org/10.26599/cvm.2026.9450546, fetched 2026-10-27
