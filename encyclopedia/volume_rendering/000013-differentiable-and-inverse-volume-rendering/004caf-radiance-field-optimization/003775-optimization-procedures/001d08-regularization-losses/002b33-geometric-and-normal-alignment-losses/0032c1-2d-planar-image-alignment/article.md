# 2D Planar Image Alignment

Also called: Joint optimization of 2D image content and per-patch homography warps

## Overview
2D Planar Image Alignment is a simplified joint optimization problem that aligns overlapping 2D image patches with learned content [S1][S2, unofficial]. It reconstructs a ground truth image from transformed patches by simultaneously optimizing the image content representation and the geometric transformation parameters [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $L_{2d}$ overlapping image patches $\{I_1, \dots, I_{L_{2d}}\}$ cropped from a ground truth image.
*   **Outputs:** Optimized 2D image content function $F_{2d}: \mathbb{R}^2 \to \mathbb{R}^2$ and a set of homography parameters $P_{2d} = \{P_1, \dots, P_{L_{2d}}\}$ [S2, unofficial].

**Data Structures:**
1.  **Homography Parameters ($P_i$):** Each $P_i \in \mathfrak{sl}(3)$ is parameterized using the Lie algebra of $SL(3)$ [S2, unofficial]. Initialized to zero [S2, unofficial].
2.  **Image Content ($F_{2d}$):** Parameterized by a 2D decomposed low-rank tensor $T_{2d} \in \mathbb{R}^{h \times w}$, where $h, w$ are image dimensions [S2, unofficial].
3.  **Low-Rank Tensor ($T_{2d}$):** Represented via Vector-Matrix (VM) decomposition with rank $R$:
    $$ T_{2d} = \sum_{r=1}^R v_{rX} \otimes v_{rY} $$
    where $v_{rX} \in \mathbb{R}^w$ and $v_{rY} \in \mathbb{R}^h$ are 1D vectors, and $\otimes$ denotes the outer product [S2, unofficial].
4.  **Gaussian Kernel ($N_{2d}$):** A 2D Gaussian kernel used to filter $T_{2d}$ to prevent overfitting [S2, unofficial]. It is constructed as the outer product of two 1D kernels:
    $$ N_{2d}(\sigma) = N_{1d}(\sigma) \otimes N_{1d}(\sigma) $$
    where $N_{1d}(\sigma) \in \mathbb{R}^{L_N}$ is a discrete kernel of size $L_N$ [S2, unofficial].

**Numbered Steps:**
1.  **Initialize:** Set homography parameters $P_i$ to zero (identity transform) [S2, unofficial]. Initialize vectors $v_{rX}, v_{rY}$ for the low-rank tensor [S2, unofficial].
2.  **Construct Kernel:** For a given bandwidth $\sigma$, generate the 1D kernel $N_{1d}(\sigma)$:
    *   If $\sigma > 0.0001$:
        $$ N_{1d}(\sigma)_x = \min\left(1, \frac{1}{\sqrt{2\pi\sigma^2}} e^{-\frac{x^2}{2\sigma^2}}\right) $$
        for $x \in [-L_N/2, L_N/2]$ [S2, unofficial].
    *   If $\sigma < 0.001$: Assign $N_{1d}(\sigma)$ to the discrete impulse function $\delta$ to avoid numerical instability [S2, unofficial].
    *   Construct $N_{2d}(\sigma) = N_{1d}(\sigma) \otimes N_{1d}(\sigma)$ [S2, unofficial].
3.  **Filter Content:** Compute the filtered content function $F_{2d}(x)$ by convolving the tensor $T_{2d}$ with $N_{2d}$:
    $$ F_{2d}(x) = (N_{2d} *_{2d} T_{2d})(x) = \left(N_{2d} *_{2d} \left(\sum_{r=1}^R v_{rX} \otimes v_{rY}\right)\right)(x) $$
    where $*_{2d}$ denotes 2D convolution [S2, unofficial].
4.  **Warp Coordinates:** For each patch $i$ and pixel coordinate $u \in U_{2d}$ (the set of coordinates in the patch), compute the warped coordinate $u'$ using the homography $P_i$:
    $$ u' = W_{2d}(P_i, u) $$
    where $W_{2d}$ maps the coordinate $u$ from the ground truth space to the patch space via the homography parameterized by $P_i \in \mathfrak{sl}(3)$ [S2, unofficial].
5.  **Compute Loss:** Calculate the reconstruction loss $L_{2d}$:
    $$ L_{2d}(F_{2d}, P_{2d}) = \sum_{i=1}^{L_{2d}} \sum_{u \in U_{2d}} \| F_{2d}(W_{2d}(P_i, u)) - I_{iu} \|^2 $$
    where $I_{iu}$ is the color of the pixel at location $u$ in input patch $I_i$ [S2, unofficial].
6.  **Optimize:** Perform gradient-based optimization to minimize $L_{2d}$ with respect to $F_{2d}$ (via $v_{rX}, v_{rY}$) and $P_{2d}$ [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def generate_1d_kernel(sigma, L_N):
    """Generates a 1D Gaussian kernel of size L_N. Handles numerical instability for small sigma."""
    if sigma < 0.001:
        kernel = np.zeros(L_N)
        kernel[L_N // 2] = 1.0
        return kernel
    x = np.arange(-L_N // 2, L_N // 2 + 1)
    pdf = (1.0 / np.sqrt(2 * np.pi * sigma**2)) * np.exp(-x**2 / (2 * sigma**2))
    return np.minimum(1.0, pdf)

def generate_2d_kernel(sigma, L_N):
    """Generates a 2D Gaussian kernel as outer product of two 1D kernels."""
    n1d = generate_1d_kernel(sigma, L_N)
    return np.outer(n1d, n1d)

def reconstruct_and_filter(vectors_x, vectors_y, kernel_2d, h, w):
    """Computes F_2d(x) by reconstructing T_2d and convolving with the 2D Gaussian kernel."""
    R = len(vectors_x)
    T_2d = np.zeros((h, w))
    for r in range(R):
        T_2d += np.outer(vectors_y[r], vectors_x[r])
    # [G] Differentiable convolution is used in the actual implementation.
    from scipy.signal import convolve2d
    return convolve2d(T_2d, kernel_2d, mode='same')

def warp_coordinate_homography(u, H):
    """Warps a 2D coordinate u using homography matrix H."""
    u_h = np.array([u[0], u[1], 1.0])
    u_prime_h = H @ u_h
    return u_prime_h[:2] / u_prime_h[2]

def compute_loss(F_2d, patches, homographies, kernel_sigma, L_N, h, w, vectors_x, vectors_y):
    """Computes the 2D Planar Image Alignment loss."""
    kernel_2d = generate_2d_kernel(kernel_sigma, L_N)
    F_2d = reconstruct_and_filter(vectors_x, vectors_y, kernel_2d, h, w)
    
    total_loss = 0.0
    for i, patch in enumerate(patches):
        H = homographies[i]
        patch_h, patch_w, _ = patch.shape
        for y in range(patch_h):
            for x in range(patch_w):
                u = np.array([x, y])
                u_global = warp_coordinate_homography(u, H)
                # [G] Differentiable grid sampling (e.g., bilinear interpolation) is required here.
                gx, gy = u_global
                val = F_2d[int(gy), int(gx)]
                target = patch[y, x]
                total_loss += np.sum((val - target)**2)
    return total_loss
```

**Parameters:**
*   $\sigma$: Gaussian kernel bandwidth. Typical values not given in sources. [S2, unofficial]
*   $L_N$: Size of the discrete kernel. Not given in sources. [S2, unofficial]
*   $R$: Rank of the low-rank decomposition. Not given in sources. [S2, unofficial]

**Edge Cases and Pitfalls:**
*   **Numerical Instability:** When $\sigma < 0.001$, the Gaussian kernel calculation becomes numerically unstable. The source specifies assigning the 1D kernel to a discrete impulse function $\delta$ in this case [S2, unofficial].
*   **Overfitting:** Naively applying low-rank decomposed tensors without filtering can limit the spectrum of the learned signal. Filtering with a Gaussian kernel is necessary to avoid overfitting [S2, unofficial].

## Subtopics

### 2D Image Field Homography Estimation
A 2D variant of radiance field estimation for homography-based panorama stitching [S3, unofficial]. A 2D coordinate network renders a stitched image, and pixel $p_i$ in training patch $i$ is warped with estimated homography $H_i$ to the rendered image [S3, unofficial]. At every iteration, a reference crop $r$ is randomly selected to align other crops. Pixel $p_i$ from any crop $i$ is warped by estimating composed homography $H'_{ri} = H_r^{-1} \cdot H_i$ to its location $p'_i = H'_{ri} \cdot p_i$ in the reference crop [S3, unofficial]. Color constancy of the backward warps to the reference crop for all surrogate views is penalized with a huber function and minimized alongside the rendering MSE [S3, unofficial]. Homography noise parameters used for evaluation include scale noise $H \in \{0.05, 0.1, 0.15\}$ and fixed translation noise of 0.2 [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides a simplified testbed for joint optimization of geometry and content, sharing computational properties with 3D alignment [S1].
*   **Limitations:** Limited to 2D planar images; does not account for 3D depth or occlusion explicitly [S2, unofficial].
*   **When to use:** When validating differentiable rendering pipelines or joint optimization strategies in a simplified 2D setting [S1][S2, unofficial].
*   **When not to use:** For full 3D scene reconstruction where depth and volume rendering are required [S2, unofficial].

## Background
The method is presented as a simpler example of joint optimization involving overlapping image patches transformed by 2D homography, serving as an analog to 1D and 3D alignment problems [S1][S2, unofficial]. The approach builds on the decomposed low-rank tensor representation (VM-decomposition) proposed by TensoRF [S2, unofficial].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.1609/aaai.v38i2.27859, fetched 2026-10-26
[S2] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
[S3] ACCV 2024 Open Access Repository, https://openaccess.thecvf.com/content/ACCV2024/html/Garg_Direct_Alignment_for_Robust_NeRF_Learning_ACCV_2024_paper.html, fetched 2026-10-29 (unofficial)
