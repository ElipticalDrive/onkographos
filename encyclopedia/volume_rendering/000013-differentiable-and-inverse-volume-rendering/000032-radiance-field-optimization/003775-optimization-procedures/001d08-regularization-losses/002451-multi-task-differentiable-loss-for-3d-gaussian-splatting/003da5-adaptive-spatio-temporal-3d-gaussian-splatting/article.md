# Adaptive Spatio-Temporal 3D Gaussian Splatting

Also called: AST-3DGS

## Overview
Adaptive Spatio-Temporal 3D Gaussian Splatting (AST-3DGS) is a method for reconstructing dynamic scenes from monocular video using an explicit representation of 3D Gaussians [S1]. It solves the problem of memory inefficiency and slow optimization in dynamic scene reconstruction by maintaining a compact set of primitives through adaptive spatio-temporal densification and using modified Adam optimization with image-based ARAP self-supervision [S1]. It distinguishes itself from sibling methods like [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md) by focusing on temporal attribute evolution and error-driven splitting rather than surface-aligned geometry or frosting layers [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A monocular video sequence $\mathcal{I}$ of $N$ calibrated images, where each image $I$ has a unique timestamp $t_I \in [0,1]$ [S1]. Calibration is typically performed using Structure-from-Motion (SfM) [S1].
*   **Output:** A set of 3D Gaussian primitives representing the dynamic scene, optimized for rendering and motion reconstruction [S1].

**Data Structures:**
Each Gaussian primitive $g$ stores the following parameters [S1]:
*   **Canonical Space Parameters:**
    *   Center: $\mu_g \in \mathbb{R}^3$
    *   Covariance: $\Sigma_g \in \mathbb{R}^{3 \times 3}$ (symmetric positive definite)
    *   Opacity: $o_g \in \mathbb{R}$
    *   Spherical Harmonics coefficients: per-color-channel
*   **Temporal Motion Parameters:**
    *   A sequence of $n_g$ keyframes $t^i_g$ for $0 \le i < n_g$, serving as nodes of a spline [S1].
    *   For each keyframe $i$:
        *   Translation: $\Delta\mu^i_g \in \mathbb{R}^3$
        *   Rotation: $q^i_g \in \mathbb{R}^4$ (quaternion)

**Initialization:**
1.  Initialize Gaussians from SfM points [S1].
2.  Assign 1 keyframe to each Gaussian, where $t^0_g = 0$ [S1].

**Rendering and Error Computation:**
1.  **Transform to Viewspace:** For a given image $I$ at time $t_I$, compute the world-to-view rotation $W_I$ and the Jacobian of the perspective projection $J_{g,I}$ at the Gaussian's center [S1].
2.  **Apply Temporal Displacement:**
    *   Compute translation $\Delta\mu_g(t_I)$ using linear interpolation (lerp) between keyframe translations [S1]:
        $$
        \Delta\mu_g(t) = \begin{cases} 
        \Delta\mu_g^0, & t < t_g^0 \\
        \operatorname{lerp}\left(\Delta\mu_g^i, \Delta\mu_g^{i+1}, \frac{t-t_g^i}{t_g^{i+1}-t_g^i}\right), & \exists i: t_g^i \le t < t_g^{i+1} \\
        \Delta\mu_g^{n_g}, & t > t_g^{n_g}
        \end{cases}
        $$
    *   Compute rotation $q_g(t_I)$ using spherical linear interpolation (slerp) between keyframe quaternions, following the same interval logic [S1].
    *   Convert $q_g(t_I)$ to rotation matrix $R_g(t_I)$ [S1].
3.  **Compute Transformed Covariance:**
    *   Compute the combined transformation matrix $A_{g,I} = J_{g,I} W_I R_g(t_I)$ [S1].
    *   Compute the viewspace covariance: $\Sigma'_{g,I} = A_{g,I} \Sigma_g A_{g,I}^T$ [S1].
4.  **Project and Sort:** Project centers to screen space and sort Gaussians by z-depth [S1].
5.  **Rasterize:** Compute pixel colors using alpha blending [S1].
6.  **Compute Pixel Error:** Calculate the pixel error $e_p$ using the L1 loss between the rendered and ground truth colors [S1].
7.  **Distribute Error to Gaussians:**
    *   Compute weight for pixel $p$ and Gaussian $g$: $w_{g,p} = a_{g,p} T_{g,p}$, where $a_{g,p}$ is opacity and $T_{g,p}$ is transmittance [S1].
    *   Distribute error: $e_{g,p} = e_p w_{g,p}$ [S1].
8.  **Compute Gaussian View Errors:**
    *   Sum weights and errors over all pixels $p$ in image $I$:
        $$w_{g,I} = \sum_{p \in I} w_{g,p}, \quad e_{g,I} = \sum_{p \in I} e_{g,p}$$
    *   Combine errors over all relevant views $\mathcal{I}$ using weighted average:
        $$e_g = \frac{\sum_{I \in \mathcal{I}} e_{g,I} w_{g,I}}{\sum_{I \in \mathcal{I}} w_{g,I}}$$
        This avoids redundant densification from outlier view errors compared to taking the maximum error [S1].
9.  **Compute Error Moments:**
    *   For each view $I$, compute the 2D first moment $\mathbf{r}_{g,I}$ and second moment $\mathbf{S}_{g,I}$ of the projected error distribution [S1]:
        $$\mathbf{r}_{g,I} = \frac{\sum_{p \in I} \mathbf{r}_p e_{g,p}}{e_{g,I}}, \quad \mathbf{S}_{g,I} = \frac{\sum_{p \in I} \mathbf{r}_p \mathbf{r}_p^T e_{g,p}}{e_{g,I}}$$
        where $\mathbf{r}_p$ is the pixel coordinate [S1].
    *   Combine moments over views $\mathcal{I}$ to get global moments $\mathbf{r}_g$ and $\mathbf{S}_g$ [S1].
    *   Compute the centered second moment: $\mathbf{S}_g^c = \mathbf{S}_g - \mathbf{r}_g \mathbf{r}_g^T$ [S1].
    *   **Clamp:** Clamp the computed error moments based on the shape of the Gaussian to make the process more robust [S1].

**Optimization:**
*   Use modified Adam optimization to update canonical positions, quaternions, and keyframe translations/rotations [S1].
*   Loss function:
    $$\mathcal{L} = \mathcal{L}_{\text{photo}} + \lambda_{\text{geo}} \mathcal{L}_{\text{geo}} + \lambda_{\text{vel}} \mathcal{L}_{\text{vel}}$$
    where $\mathcal{L}_{\text{photo}}$ is photometric reconstruction loss, $\mathcal{L}_{\text{geo}}$ is geometric loss, and $\mathcal{L}_{\text{vel}}$ is total velocity distortion loss averaged over all pixels [S1].
*   Parameters: $\lambda_{\text{geo}} = \lambda_{\text{vel}} = 15$ [S1].

**Adaptive Spatio-Temporal Densification:**
1.  **Spatial Splitting:**
    *   If Gaussian error $e_g > \tau_{\text{space}}$, split the Gaussian [S1].
    *   Use a plane passing through the Gaussian center [S1].
    *   Determine plane normal using error moments:
        $$\mathbf{n} = \frac{(\mathbf{S}_g^c)^{-1} \mathbf{r}_g}{\left\| (\mathbf{S}_g^c)^{-1} \mathbf{r}_g \right\|}$$
        This differs from methods using the Gaussian's longest axis [S1].
    *   Place new Gaussians on either side of the plane to minimize difference with the original [S1].
2.  **Temporal Densification:**
    *   Analyze motion to add more keyframes to existing Gaussians [S1].
    *   The keyframe times $t^i_g$ are not optimized but determined by motion analysis [S1].

**Evaluation Setup:**
*   Hold out every 8th segment (each segment consists of four consecutive frames) for the test set [S1].
*   Metrics: PSNR, SSIM, LPIPS, and VMAF [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.spatial.transform import Rotation

def lerp(a, b, t):
    return a + t * (b - a)

def slerp(q1, q2, t):
    # Simplified slerp for unit quaternions
    dot = np.dot(q1, q2)
    if dot < 0:
        q2 = -q2
        dot = -dot
    if dot > 0.9995:
        return lerp(q1, q2, t)
    theta_0 = np.arccos(dot)
    theta = theta_0 * t
    sin_theta_0 = np.sin(theta_0)
    s0 = np.cos(theta) - dot * np.sin(theta) / sin_theta_0
    s1 = np.sin(theta) / sin_theta_0
    return s0 * q1 + s1 * q2

def compute_keyframe_displacement(keyframes, t):
    """
    keyframes: list of dicts with keys 't', 'delta_mu', 'q'
    t: current time
    Returns: (delta_mu, q)
    """
    if not keyframes:
        return np.zeros(3), np.array([1, 0, 0, 0])
    
    # Find interval
    if t <= keyframes[0]['t']:
        return keyframes[0]['delta_mu'], keyframes[0]['q']
    if t >= keyframes[-1]['t']:
        return keyframes[-1]['delta_mu'], keyframes[-1]['q']
    
    for i in range(len(keyframes) - 1):
        t_i = keyframes[i]['t']
        t_i1 = keyframes[i+1]['t']
        if t_i <= t < t_i1:
            frac = (t - t_i) / (t_i1 - t_i)
            delta_mu = lerp(keyframes[i]['delta_mu'], keyframes[i+1]['delta_mu'], frac)
            q = slerp(keyframes[i]['q'], keyframes[i+1]['q'], frac)
            return delta_mu, q

def render_gaussian(gaussian, camera, time):
    # 1. Get temporal displacement
    delta_mu, q = compute_keyframe_displacement(gaussian['keyframes'], time)
    
    # 2. Compute world position and rotation
    mu_world = gaussian['mu'] + delta_mu
    R_world = Rotation.from_quat(q).as_matrix()
    
    # 3. Transform to viewspace
    # W_I is camera rotation matrix, J_g,I is Jacobian of perspective projection
    # A_g,I = J_g,I * W_I * R_world
    A = camera.jacobian * camera.rotation_matrix @ R_world
    
    # 4. Compute viewspace covariance
    Sigma_viewspace = A @ gaussian['Sigma'] @ A.T
    
    # 5. Project and rasterize (simplified)
    # ... standard 3DGS rasterization ...
    return rendered_color

def compute_error_moments(gaussians, images, ground_truth):
    """
    Computes error moments for all Gaussians across all images.
    """
    moments = {}
    for g in gaussians:
        moments[g['id']] = {'r': np.zeros(2), 'S': np.zeros((2,2)), 'w_sum': 0, 'e_sum': 0}
        
    for I, gt_image in enumerate(ground_truth):
        rendered_image = render_scene(gaussians, images[I])
        # Compute L1 loss per pixel
        l1_loss = np.abs(rendered_image - gt_image)
        
        for g in gaussians:
            # Get weights and errors for this Gaussian in this image
            # w_g,p = a_g,p * T_g,p
            # e_g,p = e_p * w_g,p
            # ... compute pixel-wise contributions ...
            
            # Accumulate moments
            # r_g,I = sum(r_p * e_g,p) / e_g,I
            # S_g,I = sum(r_p * r_p^T * e_g,p) / e_g,I
            pass
            
    # Combine moments over views
    for g in gaussians:
        # e_g = sum(e_g,I * w_g,I) / sum(w_g,I)
        # r_g = sum(r_g,I * w_g,I) / sum(w_g,I)
        # S_g = sum(S_g,I * w_g,I) / sum(w_g,I)
        # S_g_c = S_g - r_g * r_g^T
        pass
    return moments

def split_gaussian(gaussian, moments, threshold):
    if gaussian['error'] > threshold:
        # Compute plane normal from error moments
        r = moments['r']
        S_c = moments['S_c']
        normal = np.linalg.solve(S_c, r)
        normal = normal / np.linalg.norm(normal)
        
        # Split Gaussian using plane through center with normal
        # ... place new Gaussians ...
        pass
```

## Subtopics

### [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md)
This method extends 3D Gaussian Splatting (GS) by introducing a regularization term based on the gradients of an estimated Signed Distance Function (SDF) to enforce surface-aligned geometry. While standard GS relies on photometric losses that often result in imprecise geometry and noisy normals, this approach enables high-quality surface mesh extraction and improved rendering fidelity, particularly in regions with high curvature or fine detail. It distinguishes itself from sibling methods like [Gaussian Frosting Layer Creation](12579-gaussian-frosting-layer-creation.md) and [Dual-Domain Deformation Model](12804-dual-domain-deformation-model.md) by explicitly supervising the SDF gradient direction against monocular normal estimates to ensure the Gaussians represent a coherent surface rather than a volumetric density field.

### [Gaussian Frosting Layer Creation](12579-gaussian-frosting-layer-creation.md)
Gaussian Frosting Layer Creation is a method for defining a volumetric "frosting" layer around a base surface mesh extracted from 3D Gaussian Splatting, allowing for the representation of fuzzy or translucent materials (e.g., hair, grass) that standard surface extraction misses. It differs from sibling methods like [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md) by explicitly modeling a thickness interval rather than just normal-aligned Gaussians, and from [Dual-Domain Deformation Model](12804-dual-domain-deformation-model.md) by focusing on static layer definition via density isosurfaces rather than deformation fields. It builds on [Adaptive Spatio-Temporal 3D Gaussian Splatting](9535-adaptive-spatio-temporal-3d-gaussian-splatting.md) by providing the geometric bounds for the volumetric components.

### [Dual-Domain Deformation Model](12804-dual-domain-deformation-model.md)
The Dual-Domain Deformation Model (DDDM) is a method for explicitly modeling the time-dependent dynamics of 3D Gaussian particles in 4D scene reconstruction. It solves the problem of representing complex scene deformations across long video footage without requiring separate 3D Gaussian Splatting (3DGS) models per frame or an additional implicit neural field. Unlike its siblings, which may rely on normal information for mesh extraction [12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md] or frosting layers for surface refinement [12579-gaussian-frosting-layer-creation.md], DDDM focuses on temporal attribute evolution by combining polynomial fitting in the time domain with Fourier series fitting in the frequency domain.

### Dynamic Gaussian Deformation with Local Linearity
This variant uses an MLP that takes time $t$ and canonical 3D position $\mathbf{x}_k$ as input to output position offsets $\delta\mathbf{x}_k$, deforming Gaussians from a canonical space [S2]. It deforms Gaussians only to integer time steps and ensures local linearity at intermediate fractional time steps by linearly interpolating positions from the two nearest integer timesteps [S2]. For an integer timestep $i$ and intermediate fractional timestep $j$ between integer steps $i_1$ and $i_2$, the position $\mathbf{x}_j$ is linearly interpolated: $\mathbf{x}_j = (i_2 - j)\mathbf{x}_{i_1} + (j - i_1)\mathbf{x}_{i_2}$ [S2].

## Operating characteristics
*   **Time and Memory Complexity:** Memory efficiency is maintained through adaptive spatio-temporal densification, adding Gaussians and keyframes only where needed [S1]. The method allows for fast optimization and rendering [S1].
*   **Measured Performance:** Evaluated on the AmbientGaussians dataset [S1]. For example, on Forest Scene #5, it achieved a PSNR of 25.25, SSIM of 0.83, LPIPS of 0.18, and VMAF of 70.97, with 1.04M Gaussians and a training time of 1h 56m on an NVIDIA A100 GPU with 40GB VRAM [S1].
*   **Scaling:** 4D-GS densification threshold tuning fails to increase primitive count or quality in large scenes due to sparse gradients and suboptimal Gaussian placement under complex visibility [S1].
*   **Strengths:** Memory efficient, fast optimization, handles large scenes with oscillatory motion (e.g., foliage) without user interaction [S1].
*   **Limitations:** Relies on calibrated images from SfM [S1].

## Background
AST-3DGS was introduced in "Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion" [S1]. It addresses the limitations of previous explicit methods for monocular videos that required manual clicking on dynamic elements for segmentation, which is impractical for scenes with many moving objects like leaves [S1]. It builds on the 3D Gaussian Splatting framework [KKLD23] and adapts gradient-based densification strategies to a spatiotemporal context using error moments [S1].

## Sources
[S1] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
[S2] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
