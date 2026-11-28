# X-Gaussian

Also called: Radiative Gaussian Splatting for X-Ray, X-Ray Gaussian Splatting

## Overview
X-Gaussian is a 3D Gaussian Splatting (3DGS) framework adapted for X-ray imaging, designed for efficient novel view synthesis (NVS) and sparse-view CT reconstruction [S1]. It replaces the view-dependent RGB appearance model of standard 3DGS with a view-independent radiative intensity model, enabling the rendering of X-ray projections via Differentiable Radiative Rasterization (DRR) [S1]. Unlike its sibling [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md) which focuses on general radiance fields, X-Gaussian specifically addresses the physics of X-ray attenuation and the lack of structure-from-motion (SfM) data in medical scanners by introducing a scanner-parameter-based initialization strategy [S1].

## How it works
X-Gaussian models the scene as a set of $M$ 3D Gaussian kernels $\mathcal{G}_3 = \{G_{3i}\}_{i=1}^M$, inheriting the core data structure from [Radiative and Tomographic Gaussians](4971-radiative-and-tomographic-gaussians.md) [S1]. Each kernel $G_{3i}$ is defined by:
- Position $\mathbf{p}_i \in \mathbb{R}^3$
- Rotation matrix $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$
- Scale matrix $\mathbf{S}_i \in \mathbb{R}^{3 \times 3}$
- Radiative intensity $\alpha_i \in \mathbb{R}$ (scalar, view-independent) [S1]

The covariance matrix $\mathbf{\Sigma}_i$ is computed as:
$$ \mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top $$
[3040-radiative-gaussian-splatting.md]

### Data Structures
The primary data structure is a list of $M$ Gaussian kernels. Each kernel stores:
- $\mathbf{p}_i$: 3 floats (x, y, z)
- $\mathbf{R}_i$: 9 floats (rotation matrix, typically stored as quaternion or 9 floats; sources imply matrix form for covariance calculation)
- $\mathbf{S}_i$: 3 floats (scale along principal axes, assuming diagonal scale matrix for efficiency, or 9 if full; standard 3DGS uses 3 scale parameters and a quaternion for rotation)
- $\alpha_i$: 1 float (radiative intensity)

*Note: Standard 3DGS parameterization uses a quaternion for rotation and 3 scales. The sources describe $\mathbf{R}_i$ and $\mathbf{S}_i$ as matrices for the covariance formula but do not specify the exact storage format (e.g., quaternion vs. 9 floats). Implementers should use the standard 3DGS parameterization (quaternion + 3 scales) and construct $\mathbf{R}_i$ from the quaternion for the covariance calculation [G].*

### Initialization: ACUI Strategy
X-Gaussian uses the ACUI (Analyzer, Camera, Uniform, Initialization) strategy to initialize Gaussians without SfM [S1].
1.  **Input**: X-ray scanner parameters (intrinsic matrix $\mathbf{K}$, extrinsic matrices $\mathbf{T}_j$ for each view $j$).
2.  **Bounding Box**: Define a cuboid enclosing the scanned object.
3.  **Sampling**: Uniformly sample center points $\mathbf{p}_i$ within the cuboid.
4.  **Initial Parameters**: Set initial $\mathbf{R}_i$ to identity, $\mathbf{S}_i$ to a small constant, and $\alpha_i$ to a small constant.

### Rendering: Differentiable Radiative Rasterization (DRR)
Rendering computes the X-ray projection intensity for a given camera view.

1.  **Projection**: For each Gaussian $i$, compute the 2D screen-space covariance $\mathbf{\Sigma}'_i$ using the Jacobian $J$ of the affine projection approximation and the rotation part $W$ of the extrinsic matrix:
    $$ \mathbf{\Sigma}'_i = J W \mathbf{\Sigma}_i W^T J^T $$
    [4525-x-gaussian.md]
    *Note: $W$ is the $3 \times 3$ rotation component of the extrinsic matrix. $J$ is the $2 \times 3$ Jacobian of the perspective projection [4525-x-gaussian.md].*

2.  **Opacity Calculation**: The opacity $\alpha_i$ for compositing is derived from the radiative intensity $\alpha_i$ and the Gaussian's spatial extent. In X-ray physics, the intensity $I$ along a ray is $I = I_0 e^{-\int \mu dl}$. The Gaussian models the attenuation coefficient $\mu$. The contribution of a Gaussian to the pixel is proportional to its radiative intensity $\alpha_i$ weighted by the 2D Gaussian density at the pixel location [S1].
    *Specific formula for opacity $\alpha_i$ in DRR is not explicitly given in the sources beyond "derived from radiative intensity and spatial extent" [4525-x-gaussian.md]. Standard alpha compositing is used, where the opacity is typically $\alpha_i \cdot \exp(-\frac{1}{2} \mathbf{x}^T \mathbf{\Sigma}'^{-1}_i \mathbf{x})$ [G].*

3.  **Compositing**: Sort Gaussians by depth. For each pixel, accumulate the radiative intensity using alpha compositing:
    $$ C = \sum_{i \in \text{sorted}} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $c_i$ is the radiative intensity contribution of Gaussian $i$ at the pixel, and $\alpha_i$ is its opacity [4525-x-gaussian.md].

### Training
1.  **Loss**: Minimize the difference between rendered projections and ground truth X-ray projections (e.g., MSE or SSIM) [S1].
2.  **Optimization**: Update $\mathbf{p}_i, \mathbf{R}_i, \mathbf{S}_i, \alpha_i$ using gradient descent. The radiative intensity $\alpha_i$ is optimized directly, excluding view-direction dependence [S1].
3.  **Densification**: Standard 3DGS densification (splitting/cloning) can be applied, though X-Gaussian shows faster convergence and less redundancy [S1].

### Parameters
-   **Training Iterations**: Convergence is observed at 1000, 20000 iterations, and 20s, 60s, 180s of training time [S1].
-   **Typical Values**: Not given in the sources for learning rate or other hyperparameters.

### Edge Cases and Pitfalls
-   **Implementation Complexity**: Requires CUDA (C++) implementation for the rasterization kernel, making debugging harder than PyTorch-only methods [S1].
-   **Precision**: X-ray projections involve high dynamic range attenuation; numerical stability in the exponential decay calculation is critical [G].
-   **Boundary Conditions**: The ACUI initialization requires an accurate bounding box of the object; incorrect bounds lead to wasted Gaussians or incomplete reconstruction [S1].

## Operating characteristics
-   **Time Complexity**: Inference is $73\times$ faster than SOTA NeRF-based methods and $192\times$ faster than RGB NeRF-based methods (TensoRF) [S1]. Training time is $<15\%$ of SOTA methods [S1].
-   **Memory Complexity**: Not given in the sources.
-   **Performance**: Outperforms SOTA methods by over 6.5 dB PSNR on average [S1]. When used to create novel views for sparse-view CT reconstruction (FDK, SART, ASD-POCS), it yields PSNR improvements of 15.19, 13.01, and 13.53 dB respectively, surpassing the next best method (NAF) by 1.32, 2.41, and 2.65 dB [S1].
-   **Scaling**: Mean covariance of 3D Gaussians decreases as the number of training views increases, indicating coarse-to-fine adaptation capable of representing fine-grained structures like small tumors [S1].
-   **Strengths**: Fast convergence (basic shape formed by 1000th iteration), cleaner backgrounds, clearer structural details (ribs, blood vessels) compared to NAF, avoids noisy/redundant Gaussians compared to original 3DGS [S1].
-   **Limitations**: Higher implementation complexity due to CUDA requirement; requires foundational knowledge in computer graphics and 3D vision [S1].

## Background
X-Gaussian was proposed in the paper "Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis" [S1]. It is the first 3DGS-based framework specifically designed for X-ray imaging. It addresses the limitations of NeRF-based methods (slow inference, long training times) and the lack of SfM data in medical X-ray scanners by introducing a radiative model and ACUI initialization [S1]. It is used to generate novel-view projections for learning-free CT reconstruction methods such as FDK, SART, and ASD-POCS [S1].

## Sources
[S1] Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis, https://doi.org/10.1007/978-3-031-73232-4_16, fetched 2026-10-26
