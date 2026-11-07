# Radiative Gaussian Point Cloud Model

Also called: Radiation Intensity Response Function, Radiative Gaussian, X-Gaussian point cloud model

## Overview
The Radiative Gaussian Point Cloud Model is a scene representation designed for X-ray imaging that replaces the view-dependent Spherical Harmonics (SH) of standard 3D Gaussian Splatting with a view-independent Radiation Intensity Response Function (RIRF) [S1]. It solves the problem of modeling isotropic X-ray attenuation, where radiation intensity depends on radiodensity rather than view direction, thereby avoiding the complex and unnecessary computation of SH functions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N_p$ 3D Gaussian primitives $\mathcal{G}_x$, camera extrinsic matrix $\mathbf{M}_{ext}$, and intrinsic matrix $\mathbf{M}_{int}$ [S1].
*   **Output:** A rendered 2D X-ray image $\mathbf{I}$ [S1].

**Data Structures:**
The scene is represented by a set of Gaussian points $\mathcal{G}_x = \{G_i \mid i = 1, \dots, N_p\}$ [S1]. Each Gaussian $G_i$ is defined by the following fields:
*   $\boldsymbol{\mu}_i \in \mathbb{R}^3$: Center position [S1].
*   $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix [S1].
*   $\alpha_i \in \mathbb{R}$: Opacity [S1].
*   $\mathbf{f}_i \in \mathbb{R}^{N_f}$: Learnable feature vector representing inherent radiative properties [S1].

The dimensionality $N_f$ of the feature vector is a parameter [S1]. The covariance $\boldsymbol{\Sigma}_i$ is typically derived from rotation and scale parameters as in the parent method [3040-radiative-gaussian-splatting.md].

**The Radiation Intensity Response Function (RIRF):**
Standard 3DGS uses SH to model view-dependent color $\mathbf{c}(\mathbf{d})$. The Radiative Gaussian Point Cloud Model replaces this with RIRF to predict a scalar radiation intensity $\mathbf{i}$ that is view-independent [S1].

The RIRF is defined as:
$$ \mathbf{i}(\mathbf{f}) = \text{Sigmoid}(\boldsymbol{\lambda} \odot \mathbf{f}) $$
where:
*   $\mathbf{i} \in \mathbb{R}$ is the predicted radiation intensity [S1].
*   $\mathbf{f} \in \mathbb{R}^{N_f}$ is the learnable feature vector for the Gaussian point [S1].
*   $\boldsymbol{\lambda} \in \mathbb{R}^{N_f}$ is a set of constant (learned) basis weights controlling the importance of each component of $\mathbf{f}$ [S1].
*   $\odot$ denotes element-wise multiplication [S1].
*   $\text{Sigmoid}(x) = \frac{1}{1 + e^{-x}}$ [S1].

This formulation excludes the view direction $\mathbf{d} = (\theta, \phi)$, matching the isotropic nature of X-ray imaging [S1].

**Rendering Pipeline (Differentiable Radiative Rasterization):**
The rendering process, termed Differentiable Radiative Rasterization (DRR), follows these steps [S1]:

1.  **Ray Space Transformation:** Transform 3D Gaussians into ray space to align viewing rays with the coordinate axis [3040-radiative-gaussian-splatting.md].
2.  **Projection:** Project 3D Gaussians to the 2D image plane. Compute the projected 2D covariance $\mathbf{\hat{\Sigma}}_i$ using the Jacobian $J$ of the affine approximation of the projection and the rotation part $W$ of the extrinsic matrix:
    $$ \mathbf{\hat{\Sigma}}_i = J W \boldsymbol{\Sigma}_i W^T J^T $$
    [3040-radiative-gaussian-splatting.md]
3.  **Sorting:** Sort the Gaussians along the viewing ray based on their distance from the camera [3040-radiative-gaussian-splatting.md].
4.  **Alpha Compositing:** Compute the rendered intensity by summing contributions of sorted Gaussians. In radiative models, the contribution $c_i$ is related to the source intensity and attenuation. The standard alpha compositing formula is:
    $$ I_{\text{rendered}} = \sum_{i=1}^M c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    [3040-radiative-gaussian-splatting.md]
    In the context of X-ray imaging, the attenuation is modeled via the scalar radiation intensity $\mathbf{i}$ derived from RIRF [3040-radiative-gaussian-splatting.md].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sigmoid(x):
    """Standard sigmoid activation function."""
    return 1.0 / (1.0 + np.exp(-x))

def compute_radiation_intensity(feature_vector, basis_weights):
    """
    Computes the radiation intensity using the Radiation Intensity Response Function (RIRF).
    
    Args:
        feature_vector (np.ndarray): Learnable feature vector f of shape (N_f,).
        basis_weights (np.ndarray): Constant basis weights lambda of shape (N_f,).
        
    Returns:
        np.ndarray: Scalar radiation intensity i.
    """
    # Element-wise multiplication followed by sigmoid
    # i = Sigmoid(lambda * f)
    intensity = sigmoid(np.sum(basis_weights * feature_vector))
    return intensity

def render_radiative_gaussians(gaussians, camera_matrix, image_shape):
    """
    Renders an X-ray image from a set of radiative Gaussians.
    
    Args:
        gaussians (list): List of Gaussian objects, each containing:
            - mu: Position (3,)
            - Sigma: Covariance (3,3)
            - alpha: Opacity (scalar)
            - f: Feature vector (N_f,)
        camera_matrix (dict): Contains 'M_ext' (4,4) and 'M_int' (3,3).
        image_shape (tuple): (height, width) of the output image.
        
    Returns:
        np.ndarray: Rendered X-ray image (height, width).
    """
    # Note: The specific implementation of DRR (Differentiable Radiative Rasterization)
    # involves ray-space transformation and projection which are handled by the 
    # parent method's pipeline [3040-radiative-gaussian-splatting.md].
    # This pseudocode focuses on the RIRF integration.
    
    # 1. Initialize output image
    rendered_image = np.zeros(image_shape)
    
    # 2. For each pixel, cast a ray and intersect with Gaussians
    # (Detailed rasterization logic is assumed to be implemented in DRR as per [S1])
    # For each Gaussian i:
    for g in gaussians:
        # Compute radiation intensity using RIRF
        # lambda is a constant weight vector, assumed to be shared or pre-defined
        # In practice, lambda might be learned per Gaussian or global. 
        # The source describes lambda as "a set of constant weights" [S1].
        # We assume basis_weights is available.
        intensity = compute_radiation_intensity(g.f, g.basis_weights)
        
        # 3. Project Gaussian to 2D and compute alpha contribution
        # (Projection logic: Sigma_hat = J * W * Sigma * W^T * J^T [3040-radiative-gaussian-splatting.md])
        # 4. Composite into image
        # I_rendered = sum(c_i * alpha_i * prod(1 - alpha_j))
        # In radiative models, c_i is related to intensity and attenuation [3040-radiative-gaussian-splatting.md]
        pass 
        
    return rendered_image
```

**Parameters:**
*   $N_f$: Dimensionality of the feature vector $\mathbf{f}$ [S1]. Value not given in the sources.
*   $\boldsymbol{\lambda}$: Basis weights vector [S1]. Value not given in the sources.

**Edge Cases and Pitfalls:**
*   **Initialization:** Standard Structure-from-Motion (SfM) initialization is not suitable for X-ray imaging due to lower contrast and overlapping layers [S1]. The method uses a customized "Angle-pose Cuboid Uniform Initialization" (ACUI) strategy, though details are not provided in the source text [S1].
*   **Isotropy Assumption:** The model assumes X-ray attenuation is isotropic. If anisotropic effects are significant, this model may not capture them accurately [S1].

## Subtopics

### Radiative Gaussian Splatting
Uses central density $\rho_i$ for isotropic density fields in CT reconstruction [3040-radiative-gaussian-splatting.md]. Choose when reconstructing volumetric density from CT data. [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md)

### X-Gaussian
Uses direct radiation intensity $I_i$ as the primitive scalar [4525-x-gaussian.md]. Choose when direct intensity mapping is preferred over feature-based RIRF. [X-Gaussian](4525-x-gaussian.md)

### Volumetric Primitives Radiance Field
Uses volumetric primitives for radiance field representation [5900-volumetric-primitives-radiance-field.md]. Choose for general radiance field applications beyond X-ray. [Volumetric Primitives Radiance Field](5900-volumetric-primitives-radiance-field.md)

## Operating characteristics
*   **Time Complexity:** Faster than original 3DGS because it avoids the complex computation of Spherical Harmonics (SH) functions [S1].
*   **Memory Complexity:** Depends on the dimensionality $N_f$ of the feature vector $\mathbf{f}$ and the basis weights $\boldsymbol{\lambda}$ [S1].
*   **Strengths:** Matches the isotropic nature of X-ray imaging; faster forward and backward processes compared to original 3DGS [S1].
*   **Limitations:** Not suitable for view-dependent phenomena; requires specialized initialization (ACUI) due to the unsuitability of SfM for X-ray images [S1].
*   **When to use:** X-ray novel view synthesis (NVS) and reconstruction [S1].
*   **When not to use:** Natural light imaging where view-dependent color is important [S1].

## Background
The Radiative Gaussian Point Cloud Model was proposed in the paper "Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis" [S1]. It addresses the limitations of standard 3DGS in X-ray imaging, where natural light assumptions (reflection, anisotropy) do not hold. X-rays penetrate objects, and attenuation depends on radiodensity, making the radiation intensity isotropic [S1]. The method was developed to replace the view-dependent SH with the view-independent RIRF and to handle the challenges of X-ray image characteristics such as low contrast and overlapping layers [S1].

## Sources
[S1] Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis, https://doi.org/10.1007/978-3-031-73232-4_16, fetched 2026-10-26
