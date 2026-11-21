# TR-Gaussians

Also called: TR-3DGS, TR-Gaussian Splatting, Transmission and Reflection 3D Gaussian Splatting, Transmission-Reflection Gaussians

## Overview
TR-Gaussians is a 3D Gaussian Splatting-based representation designed for the high-fidelity, real-time rendering of planar transmission and reflection effects, such as those found in indoor glass panes [S1]. Unlike vanilla 3D Gaussian Splatting, which overfits training images by mixing transmission and reflection into low-opacity artifacts, TR-Gaussians explicitly decomposes the scene into real geometry (transmission) and virtual geometry (reflection) using a learnable reflection plane [S1]. This separation prevents the entanglement of real scene geometry with virtual reflections, ensuring that reflections remain coherent and artifact-free in novel views where the real scene is occluded or unobserved [S1].

## How it works
TR-Gaussians models a scene containing transparent glass by combining standard 3D Gaussians with a parametric reflection plane. The method separates the rendering process into two passes to compute transmission and reflection components independently, blending them via a view-dependent Fresnel model [S1].

### Inputs and Outputs
**Inputs:**
*   Camera pose (view and projection matrices).
*   Scene representation:
    *   Primary Gaussians $\mathcal{G}_p$: Represent the real scene geometry and appearance.
    *   Glass Gaussians $\mathcal{G}_g$: Mark the reflective regions on the glass pane.
    *   Reflection Plane $P$: Defined by position, normal, and base reflectance.
*   Training images (for optimization).

**Outputs:**
*   Final rendered image $C$.
*   Intermediate components: Transmission image $C_t$, Reflection image $C_r$, Reflection strength map $R$.

### Data Structures
The scene is represented by three distinct components [S1]:

1.  **Primary Gaussians ($\mathcal{G}_p$):**
    *   Each Gaussian $i$ has:
        *   Position $\mathbf{u}_i \in \mathbb{R}^3$.
        *   Opacity $\alpha_i \in [0, 1]$.
        *   Spherical Harmonics (SH) coefficients $\mathbf{c}_i$ for view-dependent color $c_i(\mathbf{d})$.
        *   3D Covariance matrix $\Sigma_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top$, where $\mathbf{S}_i$ is the scaling matrix and $\mathbf{R}_i$ is the rotation matrix.
    *   *Note:* These are identical to the Gaussians in vanilla 3DGS [S1].

2.  **Glass Gaussians ($\mathcal{G}_g$):**
    *   A subset of Gaussians (or a separate set) projected onto the reflection plane to define the shape of the reflective surface.
    *   Used to render a binary or soft mask $M$ indicating where the glass pane exists in the screen space.

3.  **Reflection Plane ($P$):**
    *   Defined by a plane equation $\mathbf{n}^\top (\mathbf{x} - \mathbf{p}) = 0$, where $\mathbf{p}$ is a point on the plane and $\mathbf{n}$ is the unit normal vector.
    *   Includes a learnable base reflectance parameter.
    *   Initialized during Stage 1 of optimization [S1].

### Rendering Pipeline
The rendering process consists of two passes [S1].

**Pass 1: Transmission and Mask Computation**
1.  **Transmission Image ($C_t$):** Render the primary Gaussians $\mathcal{G}_p$ using standard EWA splatting [S1].
    *   Project each Gaussian $i$ to screen space: $\mathbf{u}'_i = \Pi(\mathbf{u}_i)$, $\Sigma'_i = \Pi(\Sigma_i)$ [S1].
    *   Evaluate Gaussian value at pixel $\mathbf{u}$:
        $$ G_i(\mathbf{u}) = \alpha_i \exp\left(-\frac{1}{2} (\mathbf{u} - \mathbf{u}'_i)^\top (\Sigma'_i)^{-1} (\mathbf{u} - \mathbf{u}'_i)\right) $$
    *   Composite colors and opacities in depth order to produce $C_t$ [S1].

2.  **Reflection Mask ($M$):** Render the glass Gaussians $\mathcal{G}_g$ onto the reflection plane to determine the spatial extent of the glass pane. This produces a mask $M(\mathbf{u})$ where $M(\mathbf{u}) = 1$ if the pixel corresponds to the glass, and $0$ otherwise [S1].

3.  **Reflection Strength Map ($R$):**
    *   Compute the raw reflection strength $R_{raw}$ at each point on the reflection plane using the Fresnel reflectance model. The strength depends on the view direction, surface normal, and base reflectance [S1].
    *   Filter the raw strength with the mask:
        $$ R(\mathbf{u}) = R_{raw}(\mathbf{u}) \times M(\mathbf{u}) $$
    *   $R(\mathbf{u})$ represents the view-dependent ratio of reflected light intensity relative to incident light intensity [S1].

**Pass 2: Reflection Image Computation**
1.  **Mirrored Gaussians ($\hat{\mathcal{G}}_p$):** Generate mirrored Gaussians by reflecting the primary Gaussians $\mathcal{G}_p$ across the reflection plane $P$.
    *   For each primary Gaussian $i$ with center $\mathbf{u}_i$, compute the mirrored center $\hat{\mathbf{u}}_i$:
        $$ \hat{\mathbf{u}}_i = \mathbf{u}_i - 2(\mathbf{n}^\top (\mathbf{u}_i - \mathbf{p})) \mathbf{n} $$
    *   Reflect the covariance matrix to account for the coordinate system flip (if necessary for correct orientation, though standard reflection of position and covariance is sufficient for splatting) [S1].

2.  **Raw Reflection Image ($C_{raw}$):** Render the mirrored Gaussians $\hat{\mathcal{G}}_p$ using standard EWA splatting to produce the raw reflection color $C_{raw}$ [S1].

3.  **Reflection Intensity Map ($A$):** Render the mirrored Gaussians' opacities/depths to compute an intensity map $A$ that weights the reflection based on the visibility and density of the mirrored geometry [S1].

4.  **Reflection Image ($C_r$):** Combine the raw reflection and intensity:
    $$ C_r(\mathbf{u}) = C_{raw}(\mathbf{u}) \times A(\mathbf{u}) $$

**Final Composition**
Blend the transmission and reflection images using the reflection strength map $R$:
$$ C(\mathbf{u}) = (1 - R(\mathbf{u})) \cdot C_t(\mathbf{u}) + R(\mathbf{u}) \cdot C_r(\mathbf{u}) $$
[S1]

### Optimization Strategy
TR-Gaussians uses a three-stage optimization framework to ensure stable decomposition and prevent local optima [S1].

**Stage 1: Initialization (3,000 iterations)**
*   **Goal:** Optimize primary Gaussians as vanilla 3DGS and initialize the reflection plane.
*   **Loss:**
    $$ L = (1 - \lambda)L_1 + \lambda L_{D-SSIM} $$
    where $L_1$ is the L1 loss, $L_{D-SSIM}$ is the D-SSIM loss, and $\lambda = 0.5$ [S1].
*   **Output:** The final image is set to $C = C_t$. The reflection plane $P$ is initialized [S1].

**Stage 2: Reflection Plane Adjustment (3,000 iterations)**
*   **Goal:** Refine the reflection plane and remove "reflection ghosts" from the transmission image.
*   **Loss:** Enable the full rendering equation (Eq. 1) and additional decomposition losses (depth variance loss $L_d$ and gradient conflict loss $L_c$, referenced as Eq. 17 in [S1]).
*   **Gradient Blocking:** Gradients from the reflection image $C_r$ are blocked from back-propagating to the primary Gaussians $\mathcal{G}_p$. This prevents the optimization from coupling reflection image errors with primary Gaussian placement, which would otherwise lead to incorrectly placed reflection planes [S1].

**Stage 3: Joint Optimization (24,000 iterations)**
*   **Goal:** Final joint refinement of all parameters.
*   **Learning Rate:** Reduce the learning rate of the reflection plane's position and normal by a factor of 0.1.
*   **Gradients:** Allow full gradient propagation between all components.
*   **Opacity Perturbation:** Enable periodic noise injection to escape local optima [S1].

**Opacity Perturbation**
To prevent residual reflection components from remaining in the transmission image, noise is added to the opacities of primary Gaussians behind the reflection plane [S1]:
1.  Project primary Gaussians $\mathcal{G}_p$ onto the reflection plane $P$ under all training viewpoints.
2.  Select Gaussians whose center projections fall inside the plane boundaries.
3.  Add uniformly distributed noise $\epsilon \sim \mathcal{U}[-0.4, 0.4]$ to the opacities of selected Gaussians.
4.  Clamp opacities to $[0, 1]$ every 1,000 iterations.
5.  Interleave perturbation periods with clamping periods to avoid conflict with vanilla 3DGS clamping operations [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def render_tr_gaussians(camera, primary_gaussians, glass_gaussians, reflection_plane):
    """
    Renders the final image using the TR-Gaussians two-pass pipeline.
    
    Args:
        camera: Camera pose and projection matrices.
        primary_gaussians: List of Gaussian objects (pos, opacity, SH, cov).
        glass_gaussians: List of Gaussian objects defining the glass pane mask.
        reflection_plane: Object with position p, normal n, and base reflectance.
        
    Returns:
        final_image: Rendered RGB image.
    """
    
    # --- Pass 1: Transmission and Mask ---
    
    # 1. Render Transmission Image (Ct)
    # Standard 3DGS splatting for primary Gaussians
    Ct = render_3dgs(camera, primary_gaussians)
    
    # 2. Render Reflection Mask (M)
    # Render glass Gaussians to determine the spatial extent of the glass
    M = render_mask(camera, glass_gaussians)
    
    # 3. Compute Reflection Strength Map (R)
    # Calculate Fresnel reflectance for each pixel on the plane
    R_raw = compute_fresnel_strength(camera, reflection_plane)
    # Filter by the mask
    R = R_raw * M
    
    # --- Pass 2: Reflection Image ---
    
    # 4. Generate Mirrored Gaussians
    mirrored_gaussians = []
    for g in primary_gaussians:
        # Reflect position across plane P
        # d = dot(n, g.pos - p)
        # g_mirrored.pos = g.pos - 2 * d * n
        g_mirrored = reflect_gaussian_across_plane(g, reflection_plane)
        mirrored_gaussians.append(g_mirrored)
        
    # 5. Render Raw Reflection Image (Craw)
    Craw = render_3dgs(camera, mirrored_gaussians)
    
    # 6. Render Reflection Intensity Map (A)
    # Render opacities/depths of mirrored Gaussians to get intensity weighting
    A = render_intensity(camera, mirrored_gaussians)
    
    # 7. Compute Reflection Image (Cr)
    Cr = Craw * A
    
    # --- Composition ---
    
    # Blend Transmission and Reflection
    final_image = (1 - R) * Ct + R * Cr
    
    return final_image

def compute_fresnel_strength(camera, plane):
    """
    Computes the view-dependent Fresnel reflectance for each pixel.
    """
    # Get view direction for each pixel
    view_dirs = get_view_directions(camera)
    # Get surface normal for each pixel (constant for planar reflection)
    normals = plane.normal
    
    # Compute Fresnel term (simplified Schlick approximation or exact Fresnel)
    # R_raw depends on angle between view_dir and normal, and base reflectance
    R_raw = fresnel_model(view_dirs, normals, plane.base_reflectance)
    return R_raw

def reflect_gaussian_across_plane(gaussian, plane):
    """
    Reflects a Gaussian's position and covariance across a plane.
    """
    p = plane.position
    n = plane.normal
    
    # Reflect position
    # d = dot(n, gaussian.pos - p)
    # new_pos = gaussian.pos - 2 * d * n
    d = torch.dot(n, gaussian.pos - p)
    new_pos = gaussian.pos - 2 * d * n
    
    # Reflect covariance (optional depending on implementation, 
    # but position reflection is critical for correct geometry)
    # For splatting, reflecting position is usually sufficient 
    # if the Gaussian is treated as a point source with spread.
    
    return Gaussian(new_pos, gaussian.opacity, gaussian.sh, gaussian.cov)

def optimize_tr_gaussians(primary_gaussians, glass_gaussians, reflection_plane, images, gt_images):
    """
    Multi-stage optimization process.
    """
    
    # Stage 1: Initialization (3000 iters)
    for i in range(3000):
        # Render only Ct
        Ct = render_3dgs(camera, primary_gaussians)
        loss = (1 - 0.5) * L1_loss(Ct, gt_images) + 0.5 * D_SSIM_loss(Ct, gt_images)
        loss.backward()
        optimizer.step()
        # Initialize reflection plane P here (method specific)
        
    # Stage 2: Reflection Plane Adjustment (3000 iters)
    for i in range(3000):
        # Render full pipeline
        final_image = render_tr_gaussians(camera, primary_gaussians, glass_gaussians, reflection_plane)
        
        # Compute loss including decomposition losses (Ld, Lc)
        loss = compute_loss(final_image, gt_images, primary_gaussians, reflection_plane)
        
        # Block gradients from reflection component to primary Gaussians
        # This is implementation-specific: detach reflection image gradients
        # or mask gradients in the backward pass
        loss.backward()
        optimizer.step()
        
    # Stage 3: Joint Optimization (24000 iters)
    for i in range(24000):
        # Apply opacity perturbation every 1000 iterations
        if i % 1000 == 0:
            apply_opacity_perturbation(primary_gaussians, reflection_plane)
            
        # Render full pipeline
        final_image = render_tr_gaussians(camera, primary_gaussians, glass_gaussians, reflection_plane)
        
        # Compute full loss
        loss = compute_loss(final_image, gt_images, primary_gaussians, reflection_plane)
        
        # Reduce LR for plane position/normal by 0.1
        adjust_learning_rates(reflection_plane, factor=0.1)
        
        loss.backward()
        optimizer.step()
        
        # Clamp opacities
        clamp_opacities(primary_gaussians)

def apply_opacity_perturbation(primary_gaussians, plane):
    """
    Adds noise to opacities of primary Gaussians behind the reflection plane.
    """
    for g in primary_gaussians:
        # Project center to plane
        d = torch.dot(plane.normal, g.pos - plane.position)
        if d < 0: # Behind the plane (assuming normal points out)
            # Check if projection falls inside plane bounds (simplified)
            # In practice, check if projected point is within glass mask M
            noise = torch.rand(1) * 0.8 - 0.4 # Uniform [-0.4, 0.4]
            g.opacity = g.opacity + noise
```

## Operating characteristics
*   **Rendering Speed:** Approximately 225 FPS on the RTR dataset, compared to 338 FPS for vanilla 3DGS [S1]. This is roughly two-thirds the speed of vanilla 3DGS due to the two-pass rendering and additional computations for the reflection plane and mirrored Gaussians [S1].
*   **Training Time:** Moderate training time, totaling 30,000 iterations. This is significantly faster than NeRF-based methods which require hours of training [S1].
*   **Memory:** The representation adds overhead for the reflection plane parameters and the glass Gaussians, but remains comparable to 3DGS in terms of Gaussian count, as it does not rely on excessive low-opacity Gaussians to fit reflections [S1].
*   **Strengths:**
    *   High-fidelity rendering of planar transmission and reflection.
    *   Explicit decomposition prevents "reflection ghosts" and noisy artifacts in novel views.
    *   Real-time rendering capability.
    *   Handles complex view-dependent Fresnel effects [S1].
*   **Limitations:**
    *   Limited to planar reflections (glass panes, mirrors).
    *   Performance is lower than vanilla 3DGS.
    *   Requires careful initialization and multi-stage optimization to avoid local optima [S1].
*   **When to use:** Suitable for indoor scenes with transparent glass panes or mirrors where high-fidelity novel view synthesis is required. Not suitable for non-planar reflections or scenes without significant transmission/reflection components.

## Background
TR-Gaussians was proposed by Yong Liu, Keyang Ye, Tianjia Shao, and Kun Zhou from Zhejiang University [S1]. The method was published in the IEEE Transactions on Visualization and Computer Graphics (TVCG) in 2026 (arXiv preprint 2511.13009v1, Nov 2025) [S1]. It addresses the limitations of vanilla 3DGS in handling glass panes, where the standard method fails to distinguish between transmission and reflection, leading to artifacts in novel views. The work builds upon 3D Gaussian Splatting [S1] and contrasts with methods like MirrorGaussian (which handles only pure reflection) and NeRF-based approaches (which are slower and lack robust decomposition) [S1].

## Sources
[S1] TR-Gaussians: High-Fidelity Real-Time Rendering of Planar Transmission and Reflection With 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3675416, fetched 2026-10-26
