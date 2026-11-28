# 3D Distance-Aware SSIM
Also called: 3D distance-aware D-SSIM loss, 3D distance-aware Gaussian filter

## Overview
3D Distance-Aware SSIM is a regularization loss used in differentiable volume rendering (specifically 3D Gaussian Splatting) to mitigate invalid structural similarity penalties between adjacent pixels in 2D image space that originate from different viewpoints [S1, unofficial]. It sets itself apart from standard SSIM by computing the structural similarity metric in 3D world space rather than 2D image space, ensuring that pixels adjacent in the image but distant in 3D space do not force structural similarity [S1, unofficial].

## How it works
**Inputs:**
- Rendered image $I_1$ and ground truth image $I_2$ (dimensions $H \times W \times C$).
- Camera projection matrix and predicted depth map to construct $M_{\text{pixel} \to \text{world}}$.
- Kernel width parameter $\sigma$.
- Constants $C_1, C_2$.

**Outputs:**
- Scalar loss value $L_{\text{D-SSIM}}$.

**Data Structures:**
- $M_{\text{pixel} \to \text{world}}$: A $4 \times 4$ transformation matrix mapping pixel coordinates $(u, v)$ to world coordinates $(X, Y, Z)$. This matrix is obtained using the predicted depth from the rasterizer [S1, unofficial].

**Steps:**
1. **Coordinate Transformation:** For each pixel $(u, v)$ in the image grid, compute the corresponding 3D world point $(X, Y, Z)$ using the transformation matrix $M_{\text{pixel} \to \text{world}}$. The homogeneous coordinate is defined as $(X, Y, Z, 1)^T = M_{\text{pixel} \to \text{world}} (u, v, 1)^T$ [S1, unofficial].
2. **Kernel Definition:** Define the 3D distance-aware Gaussian kernel $K_\sigma^*$ centered at the origin in world space. The kernel weight for a pixel $(u, v)$ is determined by its 3D Euclidean distance from the kernel center [S1, unofficial].
3. **Local Statistics Calculation:** Compute the local means and variances for $I_1$ and $I_2$ using the 3D distance-aware kernel $K_\sigma^*$ instead of the standard 2D kernel.
4. **SSIM Computation:** Calculate the SSIM index using the computed statistics.
5. **Loss Calculation:** Compute the final loss as $1 - \text{SSIM}$.

**Formulas:**
The D-SSIM loss is defined as [S1, unofficial]:
$$
D\text{-SSIM}(I_1, I_2) = 1 - \text{SSIM}(I_1, I_2)
$$

The SSIM index is defined as [S1, unofficial]:
$$
\text{SSIM}(I_1, I_2) = \frac{(2\mu_1\mu_2 + C_1)(2\tau_{12} + C_2)}{(\mu_1^2 + \mu_2^2 + C_1)(\tau_1^2 + \tau_2^2 + C_2)}
$$

Where the local statistics are computed using the Frobenius inner product $\langle \cdot, \cdot \rangle_F$ and element-wise product $\circ$ [S1, unofficial]:
$$
\mu_1 = \langle I_1, K_\sigma^* \rangle_F, \quad \mu_2 = \langle I_2, K_\sigma^* \rangle_F
$$
$$
\tau_1^2 = \langle I_1 \circ I_1, K_\sigma^* \rangle_F - \mu_1^2
$$
$$
\tau_2^2 = \langle I_2 \circ I_2, K_\sigma^* \rangle_F - \mu_2^2
$$
$$
\tau_{12} = \langle I_1 \circ I_2, K_\sigma^* \rangle_F - \mu_1\mu_2
$$

The 3D distance-aware Gaussian kernel $K_\sigma^*$ is defined as [S1, unofficial]:
$$
K_\sigma^*(u, v) = \frac{1}{2\pi\sigma^2} \exp\left(-\frac{X^2 + Y^2 + Z^2}{2\sigma^2}\right)
$$
where $(X, Y, Z, 1)^T = M_{\text{pixel} \to \text{world}} (u, v, 1)^T$ [S1, unofficial].

For comparison, the standard 2D Gaussian kernel $K_\sigma$ used in conventional SSIM is [S1, unofficial]:
$$
K_\sigma(u, v) = \frac{1}{2\pi\sigma^2} \exp\left(-\frac{u^2 + v^2}{2\sigma^2}\right)
$$

**Parameters:**
- $\sigma$: Kernel width. Not given in the sources.
- $C_1, C_2$: Predefined constants for numerical stability. Not given in the sources.
- $M_{\text{pixel} \to \text{world}}$: Derived from predicted depth and camera intrinsics/extrinsics.

**Edge Cases and Pitfalls:**
- **Depth Prediction Accuracy:** The method relies on $M_{\text{pixel} \to \text{world}}$ being obtained from predicted depth. Inaccurate depth predictions will lead to incorrect 3D distances and thus incorrect kernel weights [S1, unofficial].
- **Viewpoint Discontinuity:** The primary motivation is to handle cases where adjacent pixels in 2D space originate from different viewpoints (e.g., at object boundaries or in multi-view training). Standard 2D SSIM would incorrectly penalize these pixels for structural dissimilarity despite them being distant in 3D [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Mitigates invalid structural similarity penalties between adjacent pixels originating from different viewpoints, making it compatible with multi-view training in 3D Gaussian Splatting [S1, unofficial].
- **Limitations:** Requires accurate predicted depth to construct the transformation matrix. Not given in the sources.
- **When to use:** When training 3D Gaussian Splatting models with multi-view data where adjacent pixels may correspond to different 3D locations due to viewpoint differences [S1, unofficial].
- **When not to use:** When single-view rendering or when depth prediction is unreliable.

## Background
The method was proposed in the context of efficient multi-view training for 3D Gaussian Splatting (3DGS) [S1, unofficial]. Standard 3DGS uses D-SSIM loss for training along with photometric $\ell_1$ loss. However, the conventional densification method is not compatible with multi-view training, and standard D-SSIM can penalize adjacent pixels from different viewpoints incorrectly. The 3D distance-aware D-SSIM loss was devised to resolve this by computing the loss in 3D space [S1, unofficial].

## Sources
[S1] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
