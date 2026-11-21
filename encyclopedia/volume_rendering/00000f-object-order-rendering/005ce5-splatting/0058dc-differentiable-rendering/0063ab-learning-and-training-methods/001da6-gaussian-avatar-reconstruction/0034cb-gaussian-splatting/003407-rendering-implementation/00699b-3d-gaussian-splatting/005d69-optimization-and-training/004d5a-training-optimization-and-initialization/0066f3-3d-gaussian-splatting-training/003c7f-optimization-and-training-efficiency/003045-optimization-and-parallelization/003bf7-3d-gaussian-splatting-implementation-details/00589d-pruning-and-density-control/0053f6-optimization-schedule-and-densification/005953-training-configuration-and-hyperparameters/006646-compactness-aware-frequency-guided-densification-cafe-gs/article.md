# Compactness-Aware Frequency-Guided Densification (CAFe-GS)
Also called: CAFe-GS

## Overview
CAFe-GS is a densification strategy for 3D Gaussian Splatting that replaces the traditional positional-gradient threshold with a per-Gaussian error score modulated by local frequency cues to control the quality-compactness trade-off [S1]. It sets itself apart from its siblings by providing a single, interpretable threshold $\tau$ that allows users to explicitly favor either high fidelity or high compactness, achieving comparable PSNR to standard 3DGS with 2–4× fewer Gaussians, or up to 12–15× fewer Gaussians with a controlled 2–2.5 dB PSNR drop [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussian primitives $g$ with parameters $\{\mu_g, R_g, s_g, \alpha_g, a_{gk}\}$, a set of training images with known camera poses, and a threshold parameter $\tau$.
*   **Outputs:** An updated set of Gaussian primitives after densification (cloning and splitting).

**Data Structures:**
*   **Gaussian Primitive $g$:**
    *   $\mu_g \in \mathbb{R}^3$: Center position.
    *   $q_g \in \mathbb{R}^4$: Quaternion parameterizing rotation.
    *   $s_g \in \mathbb{R}^3_{+}$: Per-axis scale vector.
    *   $\alpha_g \in [0, 1]$: Opacity.
    *   $a_{gk} \in \mathbb{R}^3$: Spherical Harmonics (SH) coefficients for view-dependent color.
    *   $\Sigma_g \in \mathbb{R}^{3\times3}$: Covariance matrix, computed as $\Sigma_g = R_g \text{diag}(s_g^2) R_g^\top$, where $R_g$ is the rotation matrix derived from $q_g$ [S1].
*   **Frequency Map:** A 2D image or tensor representing the Laplacian-of-Gaussian (LoG) response for the current view or aggregated over views.

**Numbered Steps:**

1.  **Initialization:** Structure-from-motion initializes camera parameters and a sparse set of Gaussians. Primitives are projected and rasterized using differentiable splatting [S1].
2.  **Optimization:** All parameters $\{\mu_g, R_g, s_g, \alpha_g, a_{gk}\}$ are optimized to minimize a photometric distortion loss between rendered and ground-truth images [S1].
3.  **Error Score Computation ($E_g$):**
    *   For each pixel $u$ in the training image, compute the pixel-wise distortion (e.g., $L_1$ or $L_2$ loss between rendered color $C_{rendered}(u)$ and ground-truth $C_{GT}(u)$).
    *   Distribute this distortion to the contributing Gaussians. The contribution of Gaussian $g$ to pixel $u$ is weighted by its effective opacity weight (transmittance) $C(g,u)$.
    *   The per-Gaussian error score $E_g$ is the sum of redistributed distortions:
        $$ E_g = \sum_{u \in \text{pixels covered by } g} \text{Distortion}(u) \cdot C(g,u) $$
    *   Here, $C(g,u)$ represents the transmittance of Gaussian $g$ under front-to-back compositing, ensuring visibility modulation [S1].
4.  **Frequency Guidance:**
    *   Compute the Laplacian-of-Gaussian (LoG) response for the image or scene representation. The LoG highlights high-frequency structures (edges, thin contours) and suppresses smooth regions [S1].
    *   Modulate the error score $E_g$ using the LoG response. Let $F_g$ be the average LoG response over the area covered by Gaussian $g$. The modulated score $S_g$ is:
        $$ S_g = E_g \cdot F_g $$
    *   This acts as an importance weighting, emphasizing detail-rich regions and suppressing redundant densification in flat or well-modeled areas [S1].
5.  **Selection:**
    *   Compare the modulated score $S_g$ against the threshold $\tau$.
    *   Select candidate Gaussians for densification if $S_g > \tau$ [S1].
    *   Typical values for $\tau$ are 10 (quality-oriented) and 100 (compactness-oriented) [S1].
6.  **Cloning:**
    *   If a selected Gaussian $g$ is small in all dimensions (i.e., all components of $s_g$ fall below a predefined scale threshold), clone it.
    *   Copy all attributes from $g$ to the child Gaussian $g'$.
    *   Shift the child $g'$ along the dominant axis of the parent's covariance (the axis with the largest scale) [S1].
7.  **Splitting:**
    *   If a selected Gaussian $g$ is not small (i.e., at least one component of $s_g$ is above the scale threshold), split it.
    *   Duplicate attributes from $g$ to the child Gaussian $g'$.
    *   Halve the scale along all axes for both $g$ and $g'$ (i.e., $s_g \leftarrow s_g / 2$, $s_{g'} \leftarrow s_{g'} / 2$).
    *   Slightly displace the child $g'$ from the parent $g$ [S1].
8.  **Periodicity:** Densification is performed every $T$ iterations (e.g., $T=100$ over 15,000 iterations). Optimization typically proceeds for 30,000 iterations, with ADC active for the first 15,000 [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_covariance(quaternion, scales):
    """
    Computes the 3x3 covariance matrix from a quaternion and scale vector.
    quaternion: np.array of shape (4,), unit quaternion [w, x, y, z]
    scales: np.array of shape (3,), positive scale values
    """
    # Convert quaternion to rotation matrix R
    # Standard quaternion-to-matrix conversion
    w, x, y, z = quaternion
    R = np.array([
        [1 - 2*(y**2 + z**2), 2*(x*y - w*z),     2*(x*z + w*y)],
        [2*(x*y + w*z),       1 - 2*(x**2 + z**2), 2*(y*z - w*x)],
        [2*(x*z - w*y),       2*(y*z + w*x),     1 - 2*(x**2 + y**2)]
    ])
    
    # Compute covariance Sigma = R * diag(s^2) * R^T
    S_sq = np.diag(scales**2)
    Sigma = R @ S_sq @ R.T
    return Sigma

def compute_transmittance_weight(gaussian, ray_origin, ray_direction, near, far):
    """
    Computes the effective opacity weight (transmittance) for a Gaussian along a ray.
    This is a simplified approximation for the purpose of error redistribution.
    In practice, this involves integrating the Gaussian along the ray.
    """
    # Placeholder for actual ray-Gaussian intersection and transmittance calculation
    # Returns a scalar weight between 0 and 1
    return 0.5 # [G] Placeholder value; actual implementation requires ray marching or analytical integration

def compute_log_response(image):
    """
    Computes the Laplacian-of-Gaussian response for an image.
    image: np.array of shape (H, W, 3)
    """
    # Convert to grayscale
    gray = np.mean(image, axis=2)
    # Apply Gaussian blur
    from scipy.ndimage import gaussian_filter
    blurred = gaussian_filter(gray, sigma=1.0)
    # Apply Laplacian kernel
    from scipy.ndimage import laplace
    log_response = laplace(blurred)
    return log_response

def densify_gaussians(gaussians, images, cameras, tau, scale_threshold, iteration, total_iterations, T):
    """
    Performs CAFe-GS densification.
    """
    if iteration % T != 0 or iteration > 15000:
        return gaussians

    # 1. Compute Frequency Map (LoG)
    # Aggregate LoG responses over training views or use current view
    # For simplicity, assume we have a pre-computed frequency map F for the scene
    # In practice, this might be view-dependent
    frequency_map = compute_frequency_map(images, cameras) # [G] Implementation detail: aggregate LoG over views

    new_gaussians = []
    gaussians_to_remove = set()

    for g in gaussians:
        # 2. Compute Error Score E_g
        error_score = 0.0
        # Iterate over pixels covered by this Gaussian
        # In practice, use a spatial hash or bounding box to find relevant pixels
        covered_pixels = get_covered_pixels(g, images)
        
        for u in covered_pixels:
            # Compute distortion at pixel u
            distortion = compute_distortion(u, g, images) # [G] L1/L2 loss between rendered and GT
            # Compute transmittance weight
            weight = compute_transmittance_weight(g, ...) # [G] Requires ray info for pixel u
            error_score += distortion * weight
        
        # 3. Modulate by Frequency Guidance
        # Get average frequency response for this Gaussian
        freq_response = get_average_frequency(g, frequency_map)
        modulated_score = error_score * freq_response
        
        # 4. Selection
        if modulated_score > tau:
            # 5. Cloning or Splitting
            if is_small(g, scale_threshold):
                # Clone
                child = copy_gaussian(g)
                shift_along_dominant_axis(child, g)
                new_gaussians.append(child)
            else:
                # Split
                child = copy_gaussian(g)
                halve_scale(g)
                halve_scale(child)
                displace_child(child, g)
                new_gaussians.append(child)
            gaussians_to_remove.add(g)
        else:
            new_gaussians.append(g)

    # Remove old gaussians and add new ones
    final_gaussians = [g for g in gaussians if g not in gaussians_to_remove]
    final_gaussians.extend(new_gaussians)
    
    return final_gaussians

def is_small(gaussian, threshold):
    return np.all(gaussian.scales < threshold)

def shift_along_dominant_axis(child, parent):
    # Find axis with max scale in parent
    dominant_axis = np.argmax(parent.scales)
    # Shift child along this axis
    child.center += parent.scales[dominant_axis] * 0.5 * np.eye(3)[:, dominant_axis]

def displace_child(child, parent):
    # Slight displacement, e.g., random small vector or along principal axis
    child.center += np.random.randn(3) * 0.01
```

**Parameters:**
*   $\tau$ (tau): Threshold for the error-frequency score. Controls the quality-compactness trade-off. Lower values favor compact models (fewer Gaussians), higher values yield finer reconstructions. Typical values: 10 (quality), 100 (compactness) [S1].
*   $T$: Iteration interval for densification. Typical value: 100 [S1].
*   Scale threshold: Threshold for determining if a Gaussian is "small" enough to clone. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Low Resolution:** On datasets with lower input image resolution (e.g., Tanks and Temples), the effectiveness of frequency analysis and error estimation is limited. CAFe-GS may only reach PSNR parity with the gradient-based baseline at the lowest $\tau$ threshold [S1].
*   **Low Frequency Scenes:** In scenes with large low-frequency regions (e.g., sky, ground), the relative advantage of frequency guidance diminishes, though it still maintains consistent benefits at matched model sizes [S1].
*   **Precision:** The covariance matrix must remain symmetric and positive definite. Parameterizing via quaternion and scale ensures this [S1].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** CAFe-GS achieves comparable PSNR to 3DGS using 2–4× fewer Gaussians at matched quality, and up to 12–15× fewer Gaussians at a controlled PSNR trade-off (2–2.5 dB drop) [S1]. This significantly reduces memory footprint.
*   **Time:** The densification process adds overhead to the training loop, but the reduced number of Gaussians can lead to faster rendering times. The exact time complexity of the densification step is not given in the sources.

**Measured Performance:**
*   **Mip-NeRF 360 [1]:**
    *   $\tau=10$: SSIM 0.895, PSNR 29.92 dB, LPIPS 0.119, #G 2.22M [S1].
    *   $\tau=100$: SSIM 0.807, PSNR 27.50 dB, LPIPS 0.238, #G 0.26M [S1].
*   **Tanks and Temples [12]:**
    *   Best quality: PSNR 23.89, LPIPS 0.232, #G 4.5M [S1].
    *   Compactness-first: PSNR 23.93, LPIPS 0.230, #G 2.9M [S1].
*   **3DGS Baseline:** PSNR 21.60, LPIPS 0.405, #G 0.3M [S1].

**Strengths and Limitations:**
*   **Strengths:** Provides a smoother and more predictable quality-compactness trade-off compared to the gradient-based baseline, which suffers from steep quality drops and narrow usable ranges [S1]. Frequency guidance sharpens high-frequency structures (edges, thin contours) and reduces residual blur [S1].
*   **Limitations:** Effectiveness is reduced in low-resolution or low-frequency scenes [S1].

**When to Use:**
*   Use CAFe-GS when model compactness is a priority, or when a controlled trade-off between quality and size is needed. It is particularly effective for scenes with high-frequency details [S1].

## Background
CAFe-GS was proposed by Huar et al. from Inria and InterDigital [S1]. It addresses the limitations of the original 3D Gaussian Splatting (3DGS) densification, which relies on positional gradients and can lead to inefficient Gaussian distribution [S1]. The method builds on the Adaptive Density Control (ADC) mechanism of 3DGS but reorients it toward compactness [S1].

## Sources
[S1] CAFe-GS: Compactness-Aware Frequency-Guided Densification for 3D Gaussian Splatting, https://hal.science/hal-05583323v1, fetched 2026-10-26
