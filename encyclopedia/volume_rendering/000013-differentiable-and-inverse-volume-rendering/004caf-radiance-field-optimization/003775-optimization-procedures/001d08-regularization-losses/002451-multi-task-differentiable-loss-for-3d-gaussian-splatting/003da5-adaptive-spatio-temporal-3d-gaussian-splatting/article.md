# Adaptive Spatio-Temporal 3D Gaussian Splatting

Also called: AST-3DGS

## Overview
Adaptive Spatio-Temporal 3D Gaussian Splatting (AST-3DGS) is a method for reconstructing dynamic scenes from monocular video using an explicit representation of 3D Gaussians [S1]. It solves the problem of memory inefficiency and slow optimization in dynamic scene reconstruction by maintaining a compact set of primitives through adaptive spatio-temporal densification and using modified Adam optimization with image-based ARAP self-supervision [S1]. It sets itself apart from sibling methods like [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md) by focusing on temporal attribute evolution and motion representation rather than surface-aligned geometry, and from [Gaussian Frosting Layer Creation](12579-gaussian-frosting-layer-creation.md) by modeling volumetric dynamics rather than static fuzzy layers.

## How it works
**Inputs and Outputs:**
- **Input:** A monocular video sequence $\mathcal{I}$ of $N$ calibrated images, where each image $I$ has a unique timestamp $t_I \in [0,1]$ [S1]. Calibration is typically performed using Structure-from-Motion (SfM) [S1].
- **Output:** A set of 3D Gaussian primitives representing the dynamic scene, including their canonical parameters and time-dependent displacement parameters [S1].

**Data Structures:**
Each Gaussian primitive $g$ stores the following parameters [S1]:
- **Canonical Space Parameters:**
  - Center $\mu_g \in \mathbb{R}^3$
  - Covariance $\Sigma_g \in \mathbb{R}^{3 \times 3}$ (symmetric positive definite)
  - Opacity $o_g \in \mathbb{R}$
  - Spherical Harmonics coefficients for color
- **Temporal Parameters:**
  - A sequence of $n_g$ keyframe timestamps $t_g^i$ for $0 \le i < n_g$, where $t_g^0 = 0$ initially [S1].
  - For each keyframe $i$, a translation $\Delta\mu_g^i \in \mathbb{R}^3$ and a rotation quaternion $q_g^i \in \mathbb{R}^4$ [S1].

**Canonical Space:**
The canonical space is the configuration of Gaussians defined by their base centers $\mu_g$ and covariances $\Sigma_g$. This space is learned during training and does not necessarily correspond to any specific input frame [S1].

**Rendering and Image Formation:**
1. For a given time $t$, compute the displacement $\Delta\mu_g(t)$ and rotation $q_g(t)$ for each Gaussian using interpolation between keyframes (see Formulas below).
2. Compute the time-dependent position and covariance:
   $$ \mu_g(t) = \mu_g + \Delta\mu_g(t) $$
   $$ \Sigma_g(t) = R_g(t)\Sigma_g R_g^T(t) $$
   where $R_g(t)$ is the rotation matrix corresponding to $q_g(t)$ [S1].
3. Transform to viewspace using camera rotation $W_I$ and perspective projection Jacobian $J_{g,I}$:
   $$ A_{g,I} = J_{g,I}W_I R_g(t_I) $$
   $$ \Sigma'_{g,I} = A_{g,I}\Sigma_g A_{g,I}^T $$
   Note that $\Sigma_g$ (canonical) is used in the projection, not $\Sigma_g(t)$, as the rotation is applied via the view transformation [S1].
4. Sort primitives by z-depth and render using alpha compositing. The weight for pixel $p$ and Gaussian $g$ is:
   $$ w_{g,p} = a_{g,p}T_{g,p} $$
   where $a_{g,p}$ is opacity and $T_{g,p}$ is transmittance [S1].

**Interpolation of Motion:**
Translation $\Delta\mu_g(t)$ is computed using linear interpolation (lerp) between keyframe translations [S1]:
$$ \Delta\mu_g(t) = \begin{cases} \Delta\mu_g^0, & t < t_g^0 \\ \operatorname{lerp}\left(\Delta\mu_g^i, \Delta\mu_g^{i+1}, \frac{t-t_g^i}{t_g^{i+1}-t_g^i}\right), & \exists i: t_g^i \le t < t_g^{i+1} \\ \Delta\mu_g^{n_g}, & t > t_g^{n_g} \end{cases} $$
Rotation $q_g(t)$ is computed using spherical linear interpolation (slerp) with the same keyframe interval logic [S1].

**Error Moments and Densification:**
The method uses error moments to guide adaptive spatio-temporal densification (adding Gaussians or keyframes) [S1].

1. **Error Distribution:**
   Compute pixel error $e_p$ using L1 loss (not total photometric loss) for consistency across views [S1]. Distribute error to Gaussian $g$ from pixel $p$:
   $$ e_{g,p} = e_p w_{g,p} $$
   Compute view-level error moments for image $I$:
   $$ w_{g,I} = \sum_{p \in I} w_{g,p} $$
   $$ e_{g,I} = \sum_{p \in I} e_{g,p} $$
   Combine errors over relevant views $\mathcal{I}$ to get final Gaussian error $e_g$:
   $$ e_g = \frac{\sum_{I \in \mathcal{I}} e_{g,I} w_{g,I}}{\sum_{I \in \mathcal{I}} w_{g,I}} $$
   This weighted average incorporates occlusion information and avoids redundant densification from outlier view errors, unlike methods taking the maximum error over views [S1].

2. **Error Moments Calculation:**
   Compute 2D first moment $\mathbf{r}_{g,I}$ and second moment $\mathbf{S}_{g,I}$ for each view:
   $$ \mathbf{r}_{g,I} = \frac{\sum_{p \in I} \mathbf{r}_p e_{g,p}}{e_{g,I}} $$
   $$ \mathbf{S}_{g,I} = \frac{\sum_{p \in I} \mathbf{r}_p \mathbf{r}_p^T e_{g,p}}{e_{g,I}} - \mathbf{r}_{g,I}\mathbf{r}_{g,I}^T $$
   where $\mathbf{r}_p$ is the pixel coordinate [S1].
   Combine moments over views to get 3D error moments $\mathbf{r}_g$ and $\mathbf{S}_g$. The solution for the 3D second moment $\mathbf{S}_g$ is:
   $$ \operatorname{vech}(\mathbf{S}_g) = \left(\sum_{l \in \mathcal{I}} w_{g,l} D_3^T B_{g,l}^T B_{g,l} D_3\right)^{-1} \sum_{l \in \mathcal{I}} w_{g,l} D_3^T B_{g,l}^T \operatorname{vec}(\mathbf{S}_{g,l}) $$
   Then, compute the centered second moment:
   $$ \mathbf{S}_g^c = \mathbf{S}_g - \mathbf{r}_g \mathbf{r}_g^T $$
   These moments are "clamped" based on the shape of the Gaussian to make the process more robust [S1].

3. **Spatial Splitting:**
   If $e_g > \tau_{\text{space}}$, split the Gaussian using a plane passing through its center [S1]. Unlike previous methods that use the plane perpendicular to the Gaussian's longest axis, this method uses the error moments to determine the plane's orientation [S1]. The normal of the splitting plane is:
   $$ \mathbf{n} = \frac{(\mathbf{S}_g^c)^{-1} \mathbf{r}_g}{\left\| (\mathbf{S}_g^c)^{-1} \mathbf{r}_g \right\|} $$
   The plane is parallel to the tangent of the error ellipse in the direction of the first moment [S1].

4. **Temporal Densification:**
   Add more keyframes to existing Gaussians to better capture motion. The keyframe times $t_g^i$ are not optimized but analyzed from motion to determine when to add new keyframes [S1].

**Optimization:**
- Use modified Adam optimization [S1].
- Loss function:
  $$ \mathcal{L} = \mathcal{L}_{\text{photo}} + \lambda_{\text{geo}} \mathcal{L}_{\text{geo}} + \lambda_{\text{vel}} \mathcal{L}_{\text{vel}} $$
  where $\mathcal{L}_{\text{photo}}$ is photometric reconstruction loss, $\mathcal{L}_{\text{geo}}$ is geometric loss, and $\mathcal{L}_{\text{vel}}$ is total velocity distortion loss averaged over all pixels of an image [S1].
- Parameters: $\lambda_{\text{geo}} = \lambda_{\text{vel}} = 15$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_gaussians(sfm_points):
    """Initialize Gaussians from SfM points with 1 keyframe each."""
    gaussians = []
    for point in sfm_points:
        g = Gaussian()
        g.mu = point.position
        g.Sigma = point.covariance
        g.opacity = point.opacity
        g.sh_coeffs = point.color
        g.keyframes = [Keyframe(time=0.0, translation=vec3(0,0,0), rotation=quat(1,0,0,0))]
        gaussians.append(g)
    return gaussians

def interpolate_motion(gaussian, time_t):
    """Compute displacement and rotation at time t using lerp/slerp."""
    keyframes = gaussian.keyframes
    n = len(keyframes)
    
    # Translation
    if time_t <= keyframes[0].time:
        delta_mu = keyframes[0].translation
    elif time_t >= keyframes[n-1].time:
        delta_mu = keyframes[n-1].translation
    else:
        # Find interval
        for i in range(n-1):
            if keyframes[i].time <= time_t < keyframes[i+1].time:
                t0 = keyframes[i].time
                t1 = keyframes[i+1].time
                frac = (time_t - t0) / (t1 - t0)
                delta_mu = lerp(keyframes[i].translation, keyframes[i+1].translation, frac)
                break
    
    # Rotation
    if time_t <= keyframes[0].time:
        q = keyframes[0].rotation
    elif time_t >= keyframes[n-1].time:
        q = keyframes[n-1].rotation
    else:
        for i in range(n-1):
            if keyframes[i].time <= time_t < keyframes[i+1].time:
                t0 = keyframes[i].time
                t1 = keyframes[i+1].time
                frac = (time_t - t0) / (t1 - t0)
                q = slerp(keyframes[i].rotation, keyframes[i+1].rotation, frac)
                break
                
    return delta_mu, q

def compute_error_moments(gaussian, images):
    """Compute error moments for a Gaussian across relevant views."""
    total_weight = 0
    total_error_weighted = 0
    r_sum = vec3(0,0,0)
    S_sum = mat3(0,0,0, 0,0,0, 0,0,0)
    
    for I in images:
        # Compute pixel errors and weights
        view_weight = 0
        view_error = 0
        view_r = vec3(0,0,0)
        view_S = mat3(0,0,0, 0,0,0, 0,0,0)
        
        for pixel in I.pixels:
            # Render Gaussian at pixel
            w_gp = gaussian.opacity * transmittance
            e_p = l1_loss(rendered_color, ground_truth_color)
            e_gp = e_p * w_gp
            
            view_weight += w_gp
            view_error += e_gp
            
            # Accumulate moments
            view_r += pixel_coord * e_gp
            view_S += outer_product(pixel_coord, pixel_coord) * e_gp
            
        if view_error > 0:
            r_g_I = view_r / view_error
            S_g_I = (view_S / view_error) - outer_product(r_g_I, r_g_I)
            
            # Combine into global moments
            total_weight += view_weight
            total_error_weighted += view_error * view_weight
            # Note: Exact combination formula for r_g and S_g from views requires unprojection
            # as described in Eq 20-22 of source. Simplified here for structure.
            # In practice, this involves D_3 and B_{g,l} matrices.
            
    # Final error
    e_g = total_error_weighted / total_weight if total_weight > 0 else 0
    
    # Clamp moments based on Gaussian shape [G]
    # Implementation detail: clamp eigenvalues or norms to prevent degenerate splits
    
    return e_g, r_g, S_g

def split_gaussian(gaussian, plane_normal):
    """Split Gaussian using a plane through its center."""
    # Use algorithm from [FCC*24] to place Gaussians on either side
    # minimizing difference with original Gaussian
    new_gaussians = []
    # ... splitting logic ...
    return new_gaussians

def optimize_step(gaussians, images, loss_weights):
    """Perform one optimization step."""
    loss = 0
    for I in images:
        # Render image
        rendered = render(gaussians, I.camera, I.time)
        
        # Compute losses
        L_photo = l1_loss(rendered, I.gt_color)
        L_geo = geometric_loss(gaussians, I) # e.g., ARAP self-supervision
        L_vel = velocity_distortion_loss(gaussians, I)
        
        total_loss = L_photo + loss_weights['geo'] * L_geo + loss_weights['vel'] * L_vel
        loss += total_loss
        
        # Compute gradients and update parameters using modified Adam
        update_parameters(gaussians, gradients)
        
    return loss

def densify_gaussians(gaussians, tau_space):
    """Adaptive spatio-temporal densification."""
    to_split = []
    for g in gaussians:
        e_g, r_g, S_g = compute_error_moments(g, relevant_images)
        if e_g > tau_space:
            # Compute plane normal from error moments
            S_c = S_g - outer_product(r_g, r_g)
            normal = normalize(inv(S_c) @ r_g)
            to_split.append((g, normal))
            
    new_gaussians = []
    for g, normal in to_split:
        new_gaussians.extend(split_gaussian(g, normal))
    for g in gaussians:
        if g not in [item[0] for item in to_split]:
            new_gaussians.append(g)
            
    # Temporal densification: add keyframes where needed [G]
    # Analyze motion residuals to insert new keyframes
    
    return new_gaussians
```

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

### Dynamic Gaussian Deformation with Local Linearity
This variant uses an MLP that takes time $t$ and canonical 3D position $\mathbf{x}_k$ as input to output position offsets $\delta\mathbf{x}_k$, deforming Gaussians from a canonical space [S2]. It deforms Gaussians only to integer time steps and ensures local linearity at intermediate fractional time steps by linearly interpolating positions from the two nearest integer timesteps [S2]. For an integer timestep $i$ and intermediate fractional timestep $j$ between integer steps $i_1$ and $i_2$, the position $\mathbf{x}_j$ is linearly interpolated: $\mathbf{x}_j = (i_2 - j)\mathbf{x}_{i_1} + (j - i_1)\mathbf{x}_{i_2}$ [S2].

## Operating characteristics
- **Time and Memory Complexity:** Memory efficiency is maintained through adaptive spatio-temporal densification, which limits the number of Gaussians and keyframes [S1]. The method uses modified Adam optimization for fast optimization and rendering [S1].
- **Measured Performance:** Experiments were conducted on an NVIDIA A100 GPU with 40GB of VRAM [S1]. Evaluation metrics include PSNR, SSIM, LPIPS, and VMAF [LLLK13] (which accounts for cross-frame consistency) [S1].
- **Scaling:** The method is designed for large scenes with complex visibility, where other methods like 4D-GS densification threshold tuning fail to increase primitive count or quality due to sparse gradients and suboptimal Gaussian placement [S1].
- **Strengths and Limitations:** Strengths include memory efficiency, fast optimization, and no requirement for user interaction or prior-based supervision [S1]. Limitations include the complexity of error moment computation and the need for calibrated input images [S1].

## Background
Adaptive Spatio-Temporal 3D Gaussian Splatting was introduced to address the impracticality of manual segmentation for dynamic elements in monocular videos, such as leaves [S1]. It builds on the 3D Gaussian Splatting framework [KKLD23] and adapts it for dynamic scenes by introducing piecewise spline representations for motion and error-moment-guided densification [S1]. The method was developed to enable reconstruction of dynamic scenes from monocular video using an explicit representation that maintains memory efficiency [S1].

## Sources
[S1] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
[S2] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
