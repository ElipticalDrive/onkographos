# Sliced Wasserstein-based Gaussian Cloning Update

Also called: Slice-based Scale Update, Sliced Wasserstein Cloning

## Overview
A method to determine the scale update for cloned Gaussians by minimizing the difference between the integrated color of the new and old Gaussians along arbitrary 1D slices passing through their centers, ensuring rasterization integral conservation [S1]. It sets itself apart from standard densification by enforcing a specific integral constraint on 1D projections rather than relying solely on position gradient magnitudes, leading to improved PSNR, SSIM, and lower LPIPS compared to base 3DGS [S1].

## How it works
**Inputs:**
*   A set of $N$ cloned Gaussians with new opacity $o^{new}$ and sliced covariance $\Sigma^{new}$.
*   The corresponding old Gaussians with opacity $o^{old}$ and sliced covariance $\Sigma^{old}$.
*   A set of arbitrary 1D slices $\mathcal{S}$ passing through the Gaussian centers.

**Outputs:**
*   Updated scale parameters for the cloned Gaussians that satisfy the integral conservation constraint.

**Data Structures:**
*   Gaussians are represented by their mean $\mu$, covariance $\Sigma$, opacity $o$, and spherical harmonics coefficients (for color). For the derivation, the mean is assumed to be zero ($\mu=0$) without loss of generality [S1].
*   The 'sliced' covariance $\Sigma$ is the projection of the 3D covariance onto the 1D slice direction [S1].

**Steps:**
1.  **Define the Objective:** Minimize the squared L2 norm of the difference between the integrals of the new and old color functions $\mathbf{C}$ over the entire real line for all slices $\mathcal{S}$ [S1]:
    $$ \left\| \int_{-\infty}^{\infty} \mathbf{C}^{new}(x) dx - \int_{-\infty}^{\infty} \mathbf{C}^{old}(x) dx \right\|_2^2, \quad \forall \mathcal{S}. $$
    This ensures that the rasterization integral remains equal before and after cloning for any arbitrary slice [S1].

2.  **Derive Color Function for New Gaussians:**
    The color function $\mathbf{C}^{new}(x)$ along the slice is derived from the standard 3DGS rasterization equation. For $N$ Gaussians, it is given by:
    $$ \mathbf{C}^{new}(x) = \sum_{i=1}^N o^{new} \exp\left(-\frac{x^2}{2\Sigma^{new}}\right) \prod_{j=1}^{i-1} \left(1 - o^{new} \exp\left(-\frac{x^2}{2\Sigma^{new}}\right)\right). $$
    This can be simplified to:
    $$ \mathbf{C}^{new}(x) = \sum_{i=1}^N o^{new} \exp\left(-\frac{x^2}{2\Sigma^{new}}\right) \left(1 - o^{new} \exp\left(-\frac{x^2}{2\Sigma^{new}}\right)\right)^{i-1}. $$

3.  **Apply Binomial Expansion:**
    Using the binomial expansion for $p < 1$ where $(1-p)^N = \sum_{k=0}^N \binom{N}{k} (-1)^k (p)^k$, the term $\left(1 - o^{new} \exp\left(-\frac{x^2}{2\Sigma^{new}}\right)\right)^{i-1}$ is expanded. Let $p = o^{new} \exp\left(-\frac{x^2}{2\Sigma^{new}}\right)$. The color function becomes:
    $$ \mathbf{C}^{new}(x) = \sum_{i=1}^N \sum_{k=0}^{i-1} \binom{i-1}{k} (-1)^k (o^{new})^{k+1} \exp\left(-\frac{(k+1)x^2}{2\Sigma^{new}}\right). $$

4.  **Compute Integrals:**
    The integral of $\mathbf{C}^{new}(x)$ over $x \in (-\infty, \infty)$ can be computed analytically because it involves integrals of Gaussian functions of the form $\exp\left(-\frac{(k+1)x^2}{2\Sigma^{new}}\right)$. Similarly, the integral for $\mathbf{C}^{old}(x)$ is computed using its parameters.

5.  **Solve for Scale Update:**
    The equation $\int_{-\infty}^{\infty} \mathbf{C}^{new}(x) dx = \int_{-\infty}^{\infty} \mathbf{C}^{old}(x) dx$ provides a simple analytical solution for updating the scales (embedded in $\Sigma^{new}$) of the cloned Gaussians [S1]. The exact update equation referenced in the source is denoted as equation (9) in the original paper, which is not explicitly written out in the provided notes but is derived from the above integral equality [S1].

**Parameters:**
*   Number of Gaussians $N$: Varies per scene and training stage.
*   Opacity $o^{new}$: Typically initialized or updated based on the cloning strategy.
*   Sliced Covariance $\Sigma^{new}$: Derived from the 3D covariance matrix projected onto the slice direction.

**Edge Cases and Pitfalls:**
*   The binomial expansion assumes $p < 1$, i.e., $o^{new} \exp\left(-\frac{x^2}{2\Sigma^{new}}\right) < 1$. This is generally true for valid Gaussian splats, but extreme opacity values or very small covariances could violate this assumption locally.
*   The derivation assumes the mean $\mu$ is zero. In practice, the slice is defined relative to the Gaussian center, so this is a coordinate transformation.
*   The method relies on the assumption that integrating over 1D slices is sufficient to preserve the rasterization integral. This is an approximation inspired by sliced Wasserstein methods [S1].

## Operating characteristics
*   **Time Complexity:** The analytical solution for the integral is efficient, but the binomial expansion involves a double summation up to $N$, which could be costly for large $N$. However, the source notes that the resulting equation has a "simple analytical form" [S1], suggesting the final update step is computationally inexpensive.
*   **Memory Complexity:** No additional significant memory overhead compared to standard 3DGS, as it operates on existing Gaussian parameters.
*   **Performance:** On the NeRF Synthetic dataset with the same number of Gaussians, the method achieves higher PSNR, SSIM, and lower LPIPS than base 3DGS [S1]. For example, on the 'Mic' scene, PSNR is 37.29 vs 35.35 for 3DGS. Average PSNR is 33.80 for the proposed method vs 33.42 for 3DGS [S1].
*   **Strengths:** Improves reconstruction quality (PSNR, SSIM, LPIPS) without increasing the number of Gaussians.
*   **Limitations:** The method is specific to the cloning step and does not replace the entire training loop. It relies on the validity of the 1D slice approximation.

## Background
The method is introduced in "3D Gaussian Splatting as Markov Chain Monte Carlo" [S1]. It draws inspiration from sliced Wasserstein methods [S1] to enforce integral conservation during the cloning operation in 3D Gaussian Splatting training.

## Sources
[S1] 3D Gaussian Splatting as Markov Chain Monte Carlo, https://ubc-vision.github.io/3dgs-mcmc/, fetched 2026-10-26
