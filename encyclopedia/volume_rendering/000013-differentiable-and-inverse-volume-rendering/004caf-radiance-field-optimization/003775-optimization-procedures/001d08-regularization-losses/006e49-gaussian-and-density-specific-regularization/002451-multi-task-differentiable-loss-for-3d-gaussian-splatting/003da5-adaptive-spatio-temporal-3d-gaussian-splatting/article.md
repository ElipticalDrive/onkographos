# Adaptive Spatio-Temporal 3D Gaussian Splatting

Also called: AST-3DGS

## Overview
Adaptive Spatio-Temporal 3D Gaussian Splatting (AST-3DGS) is a method for reconstructing dynamic scenes from monocular video using an explicit representation of 3D Gaussians [S1]. It solves the problem of efficiently modeling spatially variable non-rigid motion (e.g., foliage) without requiring manual segmentation or prior-based supervision [S1]. It distinguishes itself from sibling methods like [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md) by focusing on temporal attribute evolution and memory-efficient adaptive densification rather than surface-aligned geometry or static frosting layers [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence $\mathcal{I}$ of $N$ calibrated images from a monocular video, each with a unique timestamp $t_I \in [0,1]$ [S1]. Calibration is typically performed using Structure-from-Motion (SfM) [S1].
*   **Output:** A set of 3D Gaussian primitives with parameters optimized to reconstruct the dynamic scene, including canonical positions, covariances, opacities, colors, and time-dependent displacement keyframes [S1].

**Data Structures:**
Each Gaussian primitive $g$ stores:
*   **Canonical Parameters:** Center $\mu_g$, covariance $\Sigma_g$, opacity $o_g$, and spherical harmonics coefficients for color [S1].
*   **Temporal Parameters:** A sequence of $n_g$ keyframe timestamps $t^i_g$ ($0 \le i < n_g$) and, for each keyframe, a translation $\Delta\mu^i_g$ and a rotation quaternion $q^i_g$ [S1].
*   **Error Moments:** First moment $\mathbf{r}_g$ and second moment $\mathbf{S}_g$ used for densification decisions [S1].

**Numbered Steps:**

1.  **Initialization:** Initialize Gaussians from SfM points. Each Gaussian starts with $n_g=1$ keyframe at $t^0_g = 0$ [S1].
2.  **Rendering:** For a given time $t$ and camera view $I$:
    *   Compute the translation $\Delta\mu_g(t)$ using linear interpolation (lerp) between keyframe translations [S1]:
        $$
        \Delta\mu_g(t) = \begin{cases} 
        \Delta\mu_g^0, & t < t_g^0 \\
        \operatorname{lerp}\left(\Delta\mu_g^i, \Delta\mu_g^{i+1}, \frac{t-t_g^i}{t_g^{i+1}-t_g^i}\right), & \exists i: t_g^i \le t < t_g^{i+1} \\
        \Delta\mu_g^{n_g}, & t > t_g^{n_g}
        \end{cases}
        $$
    *   Compute the rotation $q_g(t)$ using spherical linear interpolation (slerp) between keyframe quaternions, following the same interval logic [S1].
    *   Compute the world-space position $\mu_g(t) = \mu_g + \Delta\mu_g(t)$ and covariance $\Sigma_g(t) = R_g(t)\Sigma_g R_g^T(t)$, where $R_g(t)$ is the rotation matrix from $q_g(t)$ [S1].
    *   Transform to viewspace: $\Sigma'_{g,I} = A_{g,I}\Sigma_g A_{g,I}^T$, where $A_{g,I} = J_{g,I}W_I R_g(t_I)$, with $W_I$ as the camera rotation and $J_{g,I}$ as the Jacobian of the perspective projection [S1].
    *   Sort primitives by z-depth and render using alpha blending [S1].
3.  **Loss Computation:**
    *   Compute pixel error $e_p$ using L1 loss (not total photometric loss) for consistency across views [S1].
    *   Compute weight $w_{g,p} = a_{g,p}T_{g,p}$, where $a_{g,p}$ is opacity and $T_{g,p}$ is transmittance [S1].
    *   Distribute error to Gaussian $g$: $e_{g,p} = e_p w_{g,p}$ [S1].
    *   Compute Gaussian view errors $e_{g,I}$ and weights $w_{g,I}$ [S1].
    *   Compute final Gaussian error $e_g$ as a weighted average over relevant views $\mathcal{I}$:
        $$
        e_g = \frac{\sum_{I \in \mathcal{I}} e_{g,I} w_{g,I}}{\sum_{I \in \mathcal{I}} w_{g,I}}
        $$
        This incorporates occlusion information and avoids redundant densification from outlier view errors [S1].
    *   Compute 2D error moments $\mathbf{r}_{g,I}$ and $\mathbf{S}_{g,I}$ for each view [S1].
    *   Compute the 3D second moment $S_g$ by solving:
        $$
        \min_{S_g} \sum_{I \in \mathcal{I}} w_{g,I} \|S_{g,I} - A_{g,I} S_g A_{g,I}^T\|_F^2
        $$
        The solution is computed via the vec operation:
        $$
        \operatorname{vech}(S_g) = \left(\sum_{l \in \mathcal{I}} w_{g,l} D_3^T B_{g,l}^T B_{g,l} D_3\right)^{-1} \sum_{l \in \mathcal{I}} w_{g,l} D_3^T B_{g,l}^T \operatorname{vec}(S_{g,l})
        $$
        Then, compute the centered second moment $S_g^c = S_g - \mathbf{r}_g \mathbf{r}_g^T$ [S1].
    *   Clamp the computed error moments based on the shape of the Gaussian for robustness [S1].
    *   Compute total loss: $\mathcal{L} = \mathcal{L}_{\text{photo}} + \lambda_{\text{geo}} \mathcal{L}_{\text{geo}} + \lambda_{\text{vel}} \mathcal{L}_{\text{vel}}$, with $\lambda_{\text{geo}} = \lambda_{\text{vel}} = 15$ [S1].
4.  **Optimization:** Use modified Adam optimization to update canonical positions, quaternions, and keyframe translations/rotations. Keyframe times $t^i_g$ are not optimized but are added via densification [S1].
5.  **Adaptive Spatio-Temporal Densification:**
    *   **Spatial Splitting:** If $e_g > \tau_{\text{space}}$, split the Gaussian using a plane passing through its center [S1]. The plane's normal is determined by the error moments:
        $$
        \mathbf{n} = \frac{(S_g^c)^{-1} \mathbf{r}_g}{\|(S_g^c)^{-1} \mathbf{r}_g\|}
        $$
        This orients the split based on the error distribution rather than the Gaussian's longest axis [S1].
    *   **Temporal Densification:** Analyze motion to add keyframes to existing Gaussians where motion is complex [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def lerp(a, b, t):
    return a + t * (b - a)

def slerp(q0, q1, t):
    cos_theta = np.dot(q0, q1)
    if cos_theta < 0:
        q1 = -q1
        cos_theta = -cos_theta
    if cos_theta > 0.9995:
        return lerp(q0, q1, t)
    theta = np.arccos(cos_theta)
    sin_theta = np.sin(theta)
    s0 = np.sin((1 - t) * theta) / sin_theta
    s1 = np.sin(t * theta) / sin_theta
    return s0 * q0 + s1 * q1

def quaternion_to_matrix(q):
    x, y, z, w = q
    return np.array([
        [1 - 2*y*y - 2*z*z, 2*x*y - 2*z*w, 2*x*z + 2*y*w],
        [2*x*y + 2*z*w, 1 - 2*x*x - 2*z*z, 2*y*z - 2*x*w],
        [2*x*z - 2*y*w, 2*y*z + 2*x*w, 1 - 2*x*x - 2*y*y]
    ])

def vec(M):
    return M.flatten()

def vech(M):
    return M[np.triu_indices_from(M, k=0)]

def unvec(v, n=3):
    M = np.zeros((n, n))
    M[np.triu_indices_from(M, k=0)] = v
    M += M.T - np.diag(np.diag(M))
    return M

def outer_product(v):
    return np.outer(v, v)

def solve_linear_system(A, b):
    return np.linalg.solve(A, b)

def clamp_moment(moment, sigma):
    # Clamp based on Gaussian shape [G]
    return np.clip(moment, -np.sqrt(np.diag(sigma)), np.sqrt(np.diag(sigma)))

def split_gaussian(g, center, normal):
    # Splits Gaussian g into two along plane with normal through center [G]
    # Returns two new Gaussian objects with adjusted parameters
    pass # Standard splitting algorithm [G]

def compute_gaussian_error(gaussians, views, timestamps):
    for g in gaussians:
        total_weighted_error = 0.0
        total_weight = 0.0
        sum_weighted_D3T_BTB_D3 = np.zeros((6, 6))
        sum_weighted_D3T_B_vec_S = np.zeros(6)
        
        for view_idx, I in enumerate(views):
            t = timestamps[view_idx]
            
            # Find keyframe interval
            i = 0
            while i < len(g.keyframe_times) - 1 and g.keyframe_times[i+1] <= t:
                i += 1
            
            if t < g.keyframe_times[0]:
                delta_mu = g.keyframe_translations[0]
                q = g.keyframe_quaternions[0]
            elif t > g.keyframe_times[-1]:
                delta_mu = g.keyframe_translations[-1]
                q = g.keyframe_quaternions[-1]
            else:
                t_i = g.keyframe_times[i]
                t_next = g.keyframe_times[i+1]
                alpha = (t - t_i) / (t_next - t_i)
                delta_mu = lerp(g.keyframe_translations[i], g.keyframe_translations[i+1], alpha)
                q = slerp(g.keyframe_quaternions[i], g.keyframe_quaternions[i+1], alpha)
            
            mu_t = g.mu + delta_mu
            R_t = quaternion_to_matrix(q)
            Sigma_t = R_t @ g.Sigma @ R_t.T
            
            # Project to viewspace
            J = compute_jacobian_perspective(mu_t, I.camera)
            A = J @ I.camera.R @ R_t
            Sigma_prime = A @ g.Sigma @ A.T
            
            # Render and compute pixel-level error
            rendered_color, transmittance, depths = render_gaussians(gaussians, I.camera, mu_t, Sigma_t, g.opacity, g.colors)
            target_color = I.target_image
            pixel_error = np.abs(rendered_color - target_color)
            
            view_weight = 0.0
            view_error = 0.0
            view_r = np.zeros(2)
            view_S = np.zeros((2, 2))
            
            for p_idx, p in enumerate(I.pixels):
                w_gp = g.opacity * transmittance[p_idx]
                e_gp = pixel_error[p_idx] * w_gp
                
                view_weight += w_gp
                view_error += e_gp
                
                r_p = p.coordinate
                view_r += r_p * e_gp
                view_S += outer_product(r_p) * e_gp
            
            if view_error > 0:
                view_r /= view_error
                view_S /= view_error
            
            # Accumulate for 3D moment S_g
            D3 = get_douglas_matrix()
            B_gl = compute_B_matrix(A, view_idx)
            
            term1 = w_gp * D3.T @ B_gl.T @ B_gl @ D3
            term2 = w_gp * D3.T @ B_gl.T @ vec(view_S)
            
            sum_weighted_D3T_BTB_D3 += term1
            sum_weighted_D3T_B_vec_S += term2
            
            total_weighted_error += view_error * view_weight
            total_weight += view_weight
            
        g.error = total_weighted_error / total_weight
        
        # Compute 3D Second Moment S_g
        M = sum_weighted_D3T_BTB_D3
        rhs = sum_weighted_D3T_B_vec_S
        vech_S_g = solve_linear_system(M, rhs)
        S_g = unvec(vech_S_g)
        
        # Compute First Moment r_g (3D) by unprojecting 2D moments [G]
        # Standard back-projection using camera intrinsics and depth
        r_g_3d = unproject_moments(view_r, view_S, A, I.camera)
        
        g.r = r_g_3d
        g.S = S_g
        g.S_c = S_g - outer_product(r_g_3d)
        
        g.r = clamp_moment(g.r, g.Sigma)
        g.S_c = clamp_moment(g.S_c, g.Sigma)

def densify_gaussians(gaussians, tau_space):
    to_split = [g for g in gaussians if g.error > tau_space]
    for g in to_split:
        inv_S_c = np.linalg.inv(g.S_c)
        normal = inv_S_c @ g.r
        normal = normal / np.linalg.norm(normal)
        new_gaussians = split_gaussian(g, g.mu, normal)
        gaussians.remove(g)
        gaussians.extend(new_gaussians)
```

**Parameters:**
*   $\tau_{\text{space}}$: Spatial densification threshold. Value not given in the sources.
*   $\lambda_{\text{geo}}$: Geometric loss weight. Value: 15 [S1].
*   $\lambda_{\text{vel}}$: Velocity distortion loss weight. Value: 15 [S1].
*   $N$: Total number of calibrated images [S1].
*   $t_I$: Unique timestamp for each image $I$, normalized to $[0,1]$ [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The computation of $S_g$ involves inverting a matrix derived from weighted sums. Numerical stability should be ensured [S1].
*   **Boundaries:** Interpolation for $\Delta\mu_g(t)$ and $q_g(t)$ must handle times outside the range of keyframes by clamping to the first or last keyframe value [S1].
*   **Degenerate Input:** If a Gaussian has zero error or zero weight, moment computation may be undefined. Clamping and checks for zero denominators are required [S1].

## Subtopics

### [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md)
This method extends 3D Gaussian Splatting (GS) by introducing a regularization term based on the gradients of an estimated Signed Distance Function (SDF) to enforce surface-aligned geometry. While standard GS relies on photometric losses that often result in imprecise geometry and noisy normals, this approach enables high-quality surface mesh extraction and improved rendering fidelity, particularly in regions with high curvature or fine detail. It distinguishes itself from sibling methods like [Gaussian Frosting Layer Creation](12579-gaussian-frosting-layer-creation.md) and [Dual-Domain Deformation Model](12804-dual-domain-deformation-model.md) by explicitly supervising the SDF gradient direction against monocular normal estimates to ensure the Gaussians represent a coherent surface rather than a volumetric density field.

### [Gaussian Frosting Layer Creation](12579-gaussian-frosting-layer-creation.md)
Gaussian Frosting Layer Creation is a method for defining a volumetric "frosting" layer around a base surface mesh extracted from 3D Gaussian Splatting, allowing for the representation of fuzzy or translucent materials (e.g., hair, grass) that standard surface extraction misses. It differs from sibling methods like [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md) by explicitly modeling a thickness interval rather than just normal-aligned Gaussians, and from [Dual-Domain Deformation Model](12804-dual-domain-deformation-model.md) by focusing on static layer definition via density isosurfaces rather than deformation fields. It builds on [Adaptive Spatio-Temporal 3D Gaussian Splatting](9535-adaptive-spatio-temporal-3d-gaussian-splatting.md) by providing the geometric bounds for the volumetric components.

### [Dual-Domain Deformation Model](12804-dual-domain-deformation-model.md)
The Dual-Domain Deformation Model (DDDM) is a method for explicitly modeling the time-dependent dynamics of 3D Gaussian particles in 4D scene reconstruction. It solves the problem of representing complex scene deformations across long video footage without requiring separate 3D Gaussian Splatting (3DGS) models per frame or an additional implicit neural field. Unlike its siblings, which may rely on normal information for mesh extraction [12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md] or frosting layers for surface refinement [12579-gaussian-frosting-layer-creation.md], DDDM focuses on temporal attribute evolution by combining polynomial fitting in the time domain with Fourier series fitting in the frequency domain.

### [DN-Splatter](13430-dn-splatter.md)
DN-Splatter is a regularization and initialization strategy for 3D Gaussian Splatting (3DGS) that incorporates depth and normal priors to improve photorealism and geometric accuracy in challenging indoor scenes, enabling direct mesh extraction. It distinguishes itself from standard 3DGS by using an edge-aware depth loss and normal alignment to resolve ambiguities in textureless regions, whereas its sibling [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md) focuses primarily on normal extraction without the specific adaptive depth regularization and sensor-based initialization detailed here.

### [Gaussian Splatting Photometric Stereo](14017-gaussian-splatting-photometric-stereo.md)
Gaussian Splatting Photometric Stereo (GS-PS) is a calibrated photometric stereo method that estimates surface normals and albedo by optimizing 3D Gaussian Splatting parameters to minimize a combined photometric and normal alignment loss. It differs from deep learning-based photometric stereo by optimizing an explicit physical image formation model rather than neural network weights, and from inverse rendering methods using small networks by using Gaussian parameters with a direct geometric interpretation.

### [Rendering-Guided Regularization for SLAM](15093-rendering-guided-regularization-for-slam.md)
This method introduces a regularization term to the mapping loss in 3D Gaussian Splatting SLAM to prevent the model from "forgetting" previously observed geometry when optimizing for the current frame. Unlike original 3D Gaussian Splatting which requires queuing all frames for random selection, this approach allows the use of a small keyframe buffer, making it suitable for online continual mapping. It sets itself apart from siblings like [Dual-Domain Deformation Model](12804-dual-domain-deformation-model.md) by focusing on parameter stability rather than explicit deformation fields, and from [DN-Splatter](13430-dn-splatter.md) by using gradient-based importance weighting rather than noise-specific handling.

### Dynamic Gaussian Deformation with Local Linearity
This variant uses an MLP that takes time $t$ and canonical 3D position $\mathbf{x}_k$ as input to output position offsets $\delta\mathbf{x}_k$, deforming Gaussians from a canonical space [S2]. It deforms Gaussians only to integer time steps and ensures local linearity at intermediate fractional time steps by linearly interpolating positions from the two nearest integer timesteps [S2]. For an integer timestep $i$ and intermediate fractional timestep $j$ between integer steps $i_1$ and $i_2$, the position $\mathbf{x}_j$ is linearly interpolated: $\mathbf{x}_j = (i_2 - j)\mathbf{x}_{i_1} + (j - i_1)\mathbf{x}_{i_2}$ [S2].

### Multi-View Constraint
This regularization technique uses patch matching and homography warping to enforce color consistency between reference and neighborhood views, improving geometry in textureless regions [S3]. It identifies neighborhood frames, renders depth and normal maps, and for a pixel $p$ in a reference view, establishes a spatial plane $(d, n)$ where $n$ is the rendered normal and $d$ is the distance from the camera origin [S3]. It projects pixel $p$ to corresponding pixel $p'$ in a neighborhood view using homography warping $p' = Hp$, constructs a patch $P_i$ by selecting neighboring pixels, and evaluates color consistency using Normalized Cross Correlation (NCC) [S3].

### Regularized Optimization for Continuous Mapping
This optimization technique adds a regularization term to prevent Gaussian parameters from overfitting to the latest frame, which causes degradation of reconstruction quality for previously visited areas (the 'forgetting' problem) [S4]. Unlike prior work which maintains a keyframe buffer and optimizes selected previous frames together with the current frame, this method adds a regularization term to avoid optimizing multiple frames, suitable for online continual mapping [S4].

## Operating characteristics
*   **Time and Memory Complexity:** The method maintains memory efficiency through adaptive spatio-temporal densification, adding Gaussians and keyframes only where needed [S1]. The use of piecewise splines for displacement avoids storing per-frame displacements for all Gaussians [S1].
*   **Measured Performance:** Evaluated on the AmbientGaussians dataset [S1]. The method is tested on an NVIDIA A100 GPU with 40GB of VRAM [S1]. Evaluation metrics include PSNR, SSIM, LPIPS, and VMAF [S1].
*   **Scaling:** The method is designed for large scenes with complex visibility, where other methods like 4D-GS fail to increase primitive count or quality due to sparse gradients [S1].
*   **Strengths:** Fast optimization and rendering, no user interaction or prior-based supervision required, handles oscillatory motion well [S1].
*   **Limitations:** Relies on calibrated input (SfM). The splitting strategy is guided by error moments, which may require careful tuning of $\tau_{\text{space}}$ [S1].
*   **When to use:** Reconstruction of dynamic scenes from monocular video with non-rigid motion (e.g., foliage) [S1].
*   **When not to use:** Scenes requiring explicit surface mesh extraction without additional normal regularization, or scenes with static geometry where standard 3DGS is sufficient.

## Background
Adaptive Spatio-Temporal 3D Gaussian Splatting was introduced in the paper "Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion" [S1]. It builds on the 3D Gaussian Splatting framework [KKLD23] and addresses limitations in existing dynamic scene reconstruction methods that require manual segmentation or prior-based supervision [S1]. The method was developed to handle scenes with large amounts of motion, such as leaves oscillating in the wind, using an explicit representation that maintains memory efficiency [S1].

## Sources
[S1] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
[S2] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
[S3] TraGraph-GS: Trajectory Graph-based Gaussian Splatting for Arbitrary Large-Scale Scene Rendering., https://doi.org/10.1109/tpami.2026.3709368, fetched 2026-10-26
[S4] High-Fidelity SLAM Using Gaussian Splatting with Rendering-Guided Densification and Regularized Optimization, https://core.ac.uk/download/685139838.pdf, fetched 2026-10-31
[S5] 3DGSR: Implicit Surface Reconstruction with 3D Gaussian Splatting, https://doi.org/10.1145/3687952, fetched 2026-10-26
