# R2-Gaussian
Also called: Rectified Radiative Gaussians

## Overview
R2-Gaussian is a tomographic reconstruction method that adapts 3D Gaussian Splatting (3DGS) to sparse-view X-ray imaging by rectifying an integration bias inherent in standard 3DGS splatting [S2]. It replaces the appearance-based 3DGS kernel with a radiative Gaussian kernel parameterized by density, position, and covariance, and derives new X-ray rendering functions to ensure accurate volume retrieval [S2]. Unlike siblings that use 3DGS only for data augmentation or replace rasterization with simulators, R2-Gaussian maintains efficient differentiable rasterization while enabling direct 3D density volume extraction [S2].

## How it works
**Inputs:** Sparse-view X-ray projections (sinogram data), camera poses, and initial Gaussian parameters.
**Outputs:** A set of optimized 3D Gaussian parameters representing the density field, and a reconstructed 3D volume.

**Data Structures:**
The core representation is a set of $N$ 3D Gaussians. Each Gaussian $i$ is defined by:
1.  **Position:** $\mu_i \in \mathbb{R}^3$
2.  **Covariance:** $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (symmetric positive definite)
3.  **Central Density:** $\sigma_i \in \mathbb{R}$ (isotropic attenuation coefficient)
4.  **Opacity/Alpha:** $\alpha_i \in [0, 1]$ (standard 3DGS parameter, though the radiative kernel focuses on density)

**Step 1: Initialization**
Initialize the Gaussian parameters using the analytical Filtered Back-Projection (FBP) or Feldkamp-Davis-Kress (FDK) method [S2]. This provides a coarse density estimate to bootstrap the optimization.

**Step 2: Forward Projection (X-ray Rasterization)**
For each view, render the projection by mapping 3D Gaussians to 2D Gaussians on the detector plane.
1.  **Local Affine Approximation:** For a ray $r(t) = o + td$, approximate the 3D Gaussian transformation to the 2D image plane using a local affine approximation [S1, unofficial].
2.  **Rectification:** Standard 3DGS splatting overlooks a covariance-related scaling factor when projecting a 3D Gaussian to 2D, leading to integration bias [S2]. R2-Gaussian derives new X-ray rendering functions that correct this bias. The rendered pixel value $I(r)$ in logarithmic space is computed as the line integral of the density $\sigma(r(t))$:
    $$
    I(r) = \int_{t_n}^{t_f} \sigma(r(t)) dt
    $$
    where $t_n, t_f$ are the near and far bounds of the ray segment covered by the Gaussian [S2].
3.  **Alpha Compositing:** The 2D Gaussians are composited using standard alpha blending to produce the final projection image $\hat{p}$ [S1, unofficial].

**Step 3: Loss Computation**
Compute the photometric loss between the measured X-ray projection $p$ and the rendered projection $\hat{p}$. The raw pixel value $I'(r)$ follows the Beer-Lambert Law:
$$
I'(r) = I_0 \exp\left(-\int_{t_n}^{t_f} \sigma(r(t)) dt\right)
$$
Tomography transforms this to logarithmic space for linearization:
$$
I(r) = \log I_0 - \log I'(r) = \int_{t_n}^{t_f} \sigma(r(t)) dt \quad \text{[S2]}
$$
The loss $\mathcal{L}$ is typically the Mean Squared Error (MSE) between $I(r)$ and the measured log-intensity.

**Step 4: Optimization**
Optimize the Gaussian parameters ($\mu_i, \Sigma_i, \sigma_i$) using gradient descent (e.g., Adam) [S2].
1.  Compute the gradient of the loss with respect to the rendered projections: $\frac{\partial \mathcal{L}}{\partial \hat{p}}$.
2.  Back-propagate through the differentiable rasterizer to obtain gradients for Gaussian parameters [S2].
3.  **Voxel-based Regularization:** Use a CUDA-based differentiable voxelizer to extract a 3D volume from the Gaussians during training. This allows the application of voxel-based regularizers (e.g., Total Variation) to the extracted volume, improving reconstruction quality [S2].

**Step 5: Volume Extraction**
After optimization, extract the final 3D density volume using the CUDA-based differentiable voxelizer [S2].

**Pitfalls:**
*   **Integration Bias:** Using standard 3DGS splatting without rectification leads to inconsistent volumetric properties queried from different views, hampering accurate density retrieval [S2].
*   **Affine Approximation Degradation:** The local affine approximation used in splatting can degrade quantitative accuracy and complicate nonlinear geometric corrections [S1, unofficial].

## Subtopics

### Tomography Application of Volumetrically Consistent Rasterizer
A tomography reconstruction setup that uses the volumetrically consistent rasterizer to recover the underlying 3D density field $\sigma(x)$ from 2D images [S3]. It requires no changes to the alpha computation process and avoids the affine approximation used in standard 3DGS splatting, providing a more accurate forward rendering model for tomography [S3].

## Operating characteristics
*   **Speed:** R2-Gaussian is significantly faster than NeRF-based solutions. On the IntraTomo dataset, it achieves 34.56 dB PSNR in 3 minutes (10k iterations) and 35.01 dB in 10 minutes (30k iterations) [S2].
*   **Comparison:** It is 12x faster than the most efficient NeRF-based solution (NAF) and converges to optimal results in 15 minutes, improving PSNR by 0.6 dB compared to SOTA methods [S2].
*   **Performance Metrics:**
    *   IntraTomo: 33.19 dB, 138 min [S2]
    *   NAF: 33.99 dB, 32 min [S2]
    *   SAX-NeRF: 34.45 dB, 792 min [S2]
    *   R2-Gaussian (10k iters): 34.56 dB, 3 min [S2]
    *   R2-Gaussian (30k iters): 35.01 dB, 10 min [S2]
*   **Strengths:** High efficiency, direct 3D density volume extraction, bias-free training pipeline [S2].
*   **Limitations:** Relies on the accuracy of the local affine approximation for splatting, which may degrade quantitative accuracy in complex geometries [S1, unofficial].

## Background
R2-Gaussian was introduced in 2024 as the first 3DGS-based framework for sparse-view tomographic reconstruction [S2]. It addresses the under-explored potential of 3DGS in volumetric reconstruction tasks like X-ray CT [S2]. The method was developed to overcome the limitations of previous approaches that either treated 3DGS solely as data augmentation for traditional algorithms (e.g., FDK) or replaced efficient rasterization with CT simulators [S2].

## Sources
[S1] Radioactive 3D Gaussian Ray Tracing for Tomographic Reconstruction, https://arxiv.org/abs/2602.01057v1, fetched 2026-10-26 (unofficial)
[S2] R$^2$-Gaussian: Rectifying Radiative Gaussian Splatting for Tomographic Reconstruction, https://arxiv.org/abs/2405.20693v2, fetched 2026-10-26
[S3] Volumetrically Consistent 3D Gaussian Rasterization, https://arxiv.org/abs/2412.03378v3, fetched 2026-10-26
