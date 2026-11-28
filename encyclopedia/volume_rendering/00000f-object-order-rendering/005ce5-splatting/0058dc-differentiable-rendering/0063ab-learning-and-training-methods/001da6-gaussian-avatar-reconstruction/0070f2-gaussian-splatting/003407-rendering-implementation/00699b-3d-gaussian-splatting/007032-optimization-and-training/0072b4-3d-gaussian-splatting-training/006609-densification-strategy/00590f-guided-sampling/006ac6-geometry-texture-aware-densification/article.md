# Geometry-Texture-Aware Densification

Also called: GeoTexDensifier

## Overview
Geometry-Texture-Aware Densification (GeoTexDensifier) is a densification strategy for 3D Gaussian Splatting that utilizes geometry priors (monocular depth and normal maps) to guide splitting in weakly textured regions and texture gradients to control densification in fully textured areas, thereby avoiding the over-splitting of floaters in textureless regions [S1]. It distinguishes itself from sibling methods like Mini-Splatting-D by replacing the static contribution-area threshold with a texture-weighted statistic and validating split positions against geometric consistency, ensuring splats conform to actual surface tangents rather than drifting into background noise [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Training images $\{I_t\}$ and their camera poses $\{M_t\}$ estimated by COLMAP [S1].
*   Sparse Structure-from-Motion (SfM) points used to initialize initial Gaussian splats [S1].
*   Monocular depth maps $\{D_t\}$ estimated by ZoeDepth [S1].
*   Normal maps $\{N_t\}$ computed from the depth maps $\{D_t\}$ [S1].
*   Current set of Gaussian splats $\mathcal{G}$, each with position $\mu$, covariance $\Sigma$, opacity $\alpha$, color $c$, and attributes $V^{(a)}$ (reference view ID) and $w^{(a)}$ (maximal rendering weight) [S1].
*   Optimization iteration counter $l$ [S1].

**Outputs:**
*   Updated set of Gaussian splats $\mathcal{G}'$ with new splats generated via splitting, where invalid children are discarded and parents are retained [S1].

### Data Structures
Each Gaussian splat $G_i$ maintains the following attributes relevant to densification:
*   $\mu_i \in \mathbb{R}^3$: 3D position.
*   $V^{(a)}_i \in \mathbb{Z}$: Image ID of the reference view where $G_i$ provides maximal rendering weight.
*   $w^{(a)}_i \in \mathbb{R}$: Maximal rendering weight $w_{V^{(a)}}(G_i)$ in $\alpha$-blending.
*   $V^{(b)}_i \in \mathbb{Z}$: Initial reference view ID (view with smallest reprojection error for the initial SfM point).
*   $\hat{R}^{(b)}_i \in \mathbb{R}$: Initial depth ratio $R^{V^{(b)}}_i(G_i)$ calculated at initialization.

### Parameters
*   $\alpha_s = 20$: Linear regression coefficient for texture weight activation [S1].
*   $\beta_s = 0.16$: Linear regression coefficient for texture weight activation [S1].
*   $T_s = 40$: Start threshold value for adaptive threshold calculation [S1].
*   $T_e = 4$: End threshold value for adaptive threshold calculation [S1].
*   $\delta_p = 0.1$: Threshold for discarding child splats based on relative depth ratio change [S1].
*   $l_s, l_e$: Start and end iteration numbers for densification [S1].

### Initialization
1.  **Camera Pose and SfM Initialization:** Run COLMAP to estimate camera poses $\{M_t\}$ for all input views. Initialize Gaussian splats from sparse SfM feature map points [S1].
2.  **Initial Splat Filtering:** Filter out SfM map points with track lengths less than 3 or reprojection errors exceeding 1 pixel to ensure correct initial locations [S1].
3.  **Geometric Prior Estimation:** Estimate monocular depth maps $D_t$ for each training image $I_t$ using ZoeDepth. Compute normal maps $N_t$ from the depth maps $D_t$ [S1].
4.  **Initial Reference View Selection:** For each initial Gaussian $G_a$ derived from an SfM point, choose the image view $V^{(b)}_a$ with the smallest reprojection error as its initial reference view. Record the initial depth ratio $\hat{R}^{(b)}_a(G_a)$ [S1].

### Texture-Aware Densification
The method replaces the fixed contribution area threshold with a texture-weighted statistic to identify splats that require splitting.

1.  **Texture Weight Calculation:** For each training image $I_t$, calculate the pixel-wise texture weight $s_t(\bar{x})$ using the normalized image gradient $\nabla I_t(\bar{x})$ (gray values normalized from $[0, 255]$ to $[0, 1]$):
    $$
    s_t(\bar{x}) = \frac{\tanh(\alpha_s (\nabla I_t(\bar{x}) + \beta_s)) + 1}{2}
    $$
    This weight reflects the likelihood that a Gaussian contributes to a textured region [S1].

2.  **Adaptive Threshold:** Define an adaptive threshold $\hat{T}$ that decreases over iterations to allow smaller splats to split later:
    $$
    \hat{T} = T_s + (T_e - T_s) \frac{l - l_s}{l_e - l_s}
    $$
    where $l$ is the current iteration number [S1].

3.  **Contribution Area Redefinition:** For each Gaussian $G_i$ and image $I_t$, calculate the maximal contribution area $\hat{S}^t_i$ by summing texture weights only for pixels where $G_i$ has the maximal rendering contribution:
    $$
    \hat{S}^t_i = \sum_{\bar{x}=(1,1)}^{(W,H)} s_t(\bar{x}) \delta(I_i(\bar{x}) = I_{max}(\bar{x}))
    $$
    where $I_i(\bar{x})$ is the contribution of $G_i$ at pixel $\bar{x}$, and $I_{max}(\bar{x})$ is the maximal contribution at that pixel. Fully textured areas yield higher statistics, enforcing splitting, while weakly textured areas remain sparse [S1].

4.  **Blur Splat Identification:** Identify blur splats $G^b_t$ for image $I_t$ as those where $\hat{S}^t_i > \hat{T}$ [S1].

### Geometry-Aware Splitting
For each identified blur splat $G_a$, perform splitting with geometric validation.

1.  **Reference View Selection:** Choose the reference view $V^{(a)}$ where $G_a$ provides the maximal rendering weight $w_{V^{(a)}}(G_a)$ in $\alpha$-blending. If another view provides a higher weight during optimization, update $V^{(a)}$ and $w_{V^{(a)}}(G_a)$ [S1].

2.  **Parent Validation (VDRC for Parent):** Before splitting, validate the parent $G_a$ itself. Project $G_a$ to its initial reference view $V^{(b)}_a$ to calculate the current depth ratio $R^{V^{(b)}_a}(G_a)$. Compare it with the initial depth ratio $\hat{R}^{(b)}_a(G_a)$ using the Validation of Depth Ratio Change (VDRC). If the change exceeds $\delta_p$, the parent is considered unreliable and may be pruned or handled according to the parent's specific policy (the text implies invalid parents generate invalid children, so validation prevents this cascade) [S1].

3.  **Child Generation and Positioning:**
    *   Generate a child splat $G_c$ with a random position $\tilde{\mu}_c$ sampled from the parent's distribution [S1].
    *   Project both the parent $G_a$ and the random child $\tilde{\mu}_c$ to the reference view $V^{(a)}$. Let the projection pixels be $\bar{x}_a$ and $\bar{x}_c$ respectively.
    *   Calculate the projection pixel coordinates:
        $$
        \bar{x}_a = \pi(M_{V^{(a)}} \mu_a), \quad \bar{x}_c = \pi(M_{V^{(a)}} \tilde{\mu}_c)
        $$
        where $\pi(x, y, z) = (f_u \frac{x}{z} + c_u, f_v \frac{y}{z} + c_v)$, with $f_u, f_v$ as focal lengths and $(c_u, c_v)$ as the optical center [S1].
    *   Calculate the perpendicular projection $\mu^\perp$ of $\tilde{\mu}_c$ onto the tangential plane of $G_a$ at $\bar{x}_a$:
        $$
        \mu^\perp = \tilde{\mu}_c - ((\tilde{\mu}_c - \mu_a) \cdot N_{V^{(a)}}(\bar{x}_a)) N_{V^{(a)}}(\bar{x}_a)
        $$
        where $N_{V^{(a)}}(\bar{x}_a)$ is the normal map value at pixel $\bar{x}_a$ [S1].
    *   Determine the optimal child position $\mu_c$ using the image gradient $\nabla I_{V^{(a)}}(\bar{x}_c)$ at the child's pixel:
        $$
        \mu_c = (\tilde{\mu}_c - \mu^\perp) \nabla I_{V^{(a)}}(\bar{x}_c) + \mu^\perp
        $$
        This normal-guided positioning ensures the child stays close to the geometric surface in textureless areas [S1].

4.  **Depth Ratio Change Validation (VDRC):**
    *   Calculate the projection depth of the parent $G_a$ to $V^{(a)}$:
        $$
        Z_{V^{(a)}}(G_a) = [M_{V^{(a)}} \mu_a]_z
        $$
    *   Obtain the monocular depth $D_{V^{(a)}}(\bar{x}_a)$ from the ZoeDepth map at pixel $\bar{x}_a$.
    *   Calculate the depth ratio for the parent:
        $$
        R_{V^{(a)}}(G_a) = \frac{D_{V^{(a)}}(\bar{x}_a)}{Z_{V^{(a)}}(G_a)}
        $$
    *   Similarly, calculate the depth ratio for the child $G_c$ at its projected pixel $\bar{x}_c$:
        $$
        R_{V^{(a)}}(G_c) = \frac{D_{V^{(a)}}(\bar{x}_c)}{Z_{V^{(a)}}(G_c)}
        $$
    *   Compute the relative depth ratio change:
        $$
        P_{V^{(a)}}(G_a, G_c) = \frac{|R_{V^{(a)}}(G_a) - R_{V^{(a)}}(G_c)|}{R_{V^{(a)}}(G_a)}
        $$
    *   If $P_{V^{(a)}}(G_a, G_c) > \delta_p$, discard the child splat $G_c$ and retain the parent $G_a$ to take its place. Otherwise, accept $G_c$ [S1].

5.  **Child Inheritance:** For a valid child splat $G_c$, inherit the parent's initial reference view $V^{(b)}_c = V^{(a)}$ and set its initial depth ratio $\hat{R}^{(b)}_c(G_c) = R_{V^{(a)}}(G_c)$ for future verification steps [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def calculate_texture_weight(image_gradient, alpha_s=20, beta_s=0.16):
    """
    Calculate texture weight s_t(x) for a pixel.
    image_gradient: normalized gradient magnitude in [0, 1].
    """
    return (np.tanh(alpha_s * (image_gradient + beta_s)) + 1) / 2

def calculate_adaptive_threshold(l, l_s, l_e, T_s=40, T_e=4):
    """
    Calculate adaptive threshold T_hat.
    l: current iteration.
    """
    if l_e == l_s:
        return T_s
    return T_s + (T_e - T_s) * (l - l_s) / (l_e - l_s)

def project_to_view(point_3d, projection_matrix, focal_lengths, optical_center):
    """
    Project 3D point to 2D pixel coordinates.
    point_3d: [x, y, z]
    projection_matrix: 4x4 matrix M
    focal_lengths: [fu, fv]
    optical_center: [cu, cv]
    """
    # Homogeneous coordinates
    point_h = np.append(point_3d, 1.0)
    projected_h = projection_matrix @ point_h
    x, y, z = projected_h[:3]
    
    # Avoid division by zero
    if z == 0:
        return None, 0.0
    
    u = focal_lengths[0] * (x / z) + optical_center[0]
    v = focal_lengths[1] * (y / z) + optical_center[1]
    return (u, v), z

def calculate_depth_ratio(depth_map, pixel_coords, projection_depth):
    """
    Calculate depth ratio R = D / Z.
    depth_map: 2D array of monocular depths.
    pixel_coords: (u, v)
    projection_depth: Z from camera projection.
    """
    # Interpolate depth map at pixel_coords
    d = depth_map[pixel_coords[1], pixel_coords[0]]
    return d / projection_depth

def validate_split(parent, child, reference_view_id, depth_maps, normal_maps, 
                   projection_matrices, focal_lengths, optical_center, delta_p=0.1):
    """
    Validate a child splat against the parent using VDRC.
    Returns True if child is valid, False otherwise.
    """
    # 1. Get reference view projection matrix and depth/normal maps
    M_ref = projection_matrices[reference_view_id]
    D_ref = depth_maps[reference_view_id]
    N_ref = normal_maps[reference_view_id]
    
    # 2. Project parent to reference view
    pixel_a, z_a = project_to_view(parent.mu, M_ref, focal_lengths, optical_center)
    if pixel_a is None:
        return False
        
    # 3. Calculate parent depth ratio
    R_a = calculate_depth_ratio(D_ref, pixel_a, z_a)
    
    # 4. Project child to reference view
    pixel_c, z_c = project_to_view(child.mu, M_ref, focal_lengths, optical_center)
    if pixel_c is None:
        return False
        
    # 5. Calculate child depth ratio
    R_c = calculate_depth_ratio(D_ref, pixel_c, z_c)
    
    # 6. Calculate relative depth ratio change
    if R_a == 0:
        return False # Avoid division by zero
        
    P = abs(R_a - R_c) / R_a
    
    # 7. Check threshold
    return P <= delta_p

def normal_guided_positioning(parent, random_child_mu, reference_view_id, 
                              normal_maps, image_gradients, projection_matrices, 
                              focal_lengths, optical_center):
    """
    Determine optimal child position mu_c using normal guidance.
    """
    M_ref = projection_matrices[reference_view_id]
    N_ref = normal_maps[reference_view_id]
    I_grad = image_gradients[reference_view_id]
    
    # Project parent
    pixel_a, _ = project_to_view(parent.mu, M_ref, focal_lengths, optical_center)
    
    # Get normal at parent pixel
    normal_vec = N_ref[pixel_a[1], pixel_a[0]]
    normal_vec = normal_vec / np.linalg.norm(normal_vec)
    
    # Project random child
    pixel_c, _ = project_to_view(random_child_mu, M_ref, focal_lengths, optical_center)
    
    # Perpendicular projection mu_perp
    diff = random_child_mu - parent.mu
    mu_perp = random_child_mu - np.dot(diff, normal_vec) * normal_vec
    
    # Image gradient at child pixel
    grad_vec = I_grad[pixel_c[1], pixel_c[0]]
    # Note: The formula uses grad_vec as a vector/scalar multiplier. 
    # In the source, it implies a directional push. Assuming grad_vec is normalized or scaled appropriately.
    # mu_c = (mu_tilde - mu_perp) * grad + mu_perp
    # This part requires careful interpretation of "image gradient" as a vector field.
    # We assume grad_vec is the gradient vector at pixel_c.
    
    mu_c = (random_child_mu - mu_perp) * grad_vec + mu_perp
    
    return mu_c

def densify_step(gaussians, images, depth_maps, normal_maps, projection_matrices, 
                 focal_lengths, optical_center, l, l_s, l_e):
    """
    Main densification step.
    """
    new_gaussians = []
    
    for G_a in gaussians:
        # 1. Determine Reference View V^(a)
        # Find view with max rendering weight w
        ref_view_id = G_a.V_a
        # (In practice, this is updated during optimization, assumed known here)
        
        # 2. Calculate Texture Weights for all images
        # Precompute texture weights for the reference view image I_ref
        I_ref = images[ref_view_id]
        grad_ref = compute_gradient(I_ref) # Normalize to [0,1]
        s_ref = calculate_texture_weight(grad_ref)
        
        # 3. Calculate Contribution Area Statistic S_hat
        # This requires rendering G_a to all views or just the reference view?
        # Source says "inside the contribution area of Gi on It".
        # We assume we iterate over training views to find max S_hat.
        max_S_hat = 0
        for t, I_t in enumerate(images):
            grad_t = compute_gradient(I_t)
            s_t = calculate_texture_weight(grad_t)
            # Calculate S_hat_t for G_i
            # This requires knowing which pixels G_i contributes maximally.
            # Simplified: Assume we have a mask or contribution map for G_i on I_t.
            # contribution_map = render_contribution(G_i, I_t)
            # max_contribution = max(contribution_map)
            # S_hat_t = sum(s_t * (contribution_map == max_contribution))
            # For pseudocode, we assume a helper function get_texture_statistic(G_i, t)
            S_hat_t = get_texture_statistic(G_a, t, s_t)
            if S_hat_t > max_S_hat:
                max_S_hat = S_hat_t
        
        # 4. Check Adaptive Threshold
        T_hat = calculate_adaptive_threshold(l, l_s, l_e)
        
        if max_S_hat > T_hat:
            # 5. Split
            # Generate random child position
            random_mu_c = sample_from_gaussian(G_a)
            
            # Normal-guided positioning
            mu_c = normal_guided_positioning(G_a, random_mu_c, ref_view_id, 
                                             normal_maps, images, projection_matrices, 
                                             focal_lengths, optical_center)
            
            # Create child splat
            child = Gaussian(mu=mu_c, ...) # Initialize other attributes
            
            # 6. Validate with VDRC
            if validate_split(G_a, child, ref_view_id, depth_maps, normal_maps, 
                              projection_matrices, focal_lengths, optical_center):
                # Inherit initial reference view and depth ratio
                child.V_b = G_a.V_a
                # Calculate initial depth ratio for child at its inherited reference view
                # This requires re-projecting child to G_a.V_a
                M_init = projection_matrices[G_a.V_a]
                pixel_c_init, z_c_init = project_to_view(mu_c, M_init, focal_lengths, optical_center)
                D_init = depth_maps[G_a.V_a]
                R_c_init = calculate_depth_ratio(D_init, pixel_c_init, z_c_init)
                child.R_b = R_c_init
                
                new_gaussians.append(child)
            else:
                # Discard child, keep parent (parent is already in gaussians list)
                pass
        else:
            # Keep parent
            new_gaussians.append(G_a)
            
    return new_gaussians
```

### Edge Cases and Pitfalls
*   **Depth Scale Ambiguity:** ZoeDepth provides monocular depth without true scale. The method uses relative depth ratios ($R = D/Z$) to mitigate this, as both numerator and denominator are affected by scale in a consistent way for nearby points, but large depth discrepancies still trigger VDRC [S1].
*   **Noisy Outliers:** SfM points with high reprojection errors or short track lengths are filtered during initialization to prevent bad initial splats from propagating errors [S1].
*   **Parent Drift:** Parents may drift to improper locations during optimization. VDRC is applied to parents before splitting to validate their current position against their initial reference view [S1].
*   **Textureless Regions:** In weakly textured areas, image gradients are small, leading to low texture weights $s_t$. This prevents over-splitting, but requires normal guidance to place children correctly on the surface [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Memory:** On the 'Bonsai' case from Mip-NeRF 360, GeoTexDensifier uses 3.75GB peak GPU memory, compared to 6.61GB for Mini-Splatting-D [S1].
*   **Time:** On the 'Bonsai' case, GeoTexDensifier takes 42m 31s for training, compared to 28m 16s for Mini-Splatting-D [S1].
*   **Rendering:** GeoTexDensifier achieves 177 FPS, compared to 132 FPS for Mini-Splatting-D [S1].

### Strengths and Limitations
*   **Strengths:** Reduces floaters in textureless regions by preventing over-splitting. Improves rendering quality in textured areas by enforcing splitting based on texture gradients. Lower memory usage and higher FPS due to fewer unnecessary splats [S1].
*   **Limitations:** Relies on the accuracy of monocular depth and normal estimates (ZoeDepth). Incorrect priors can influence splitting, especially if the true depth scale is unavailable or priors are noisy [S1].

### When to Use
*   Use when high-quality photorealistic reconstruction is required, especially in scenes with mixed texture density (some textureless, some highly textured) [S1].
*   Use when memory and rendering speed are constraints, and floaters in textureless regions are a common issue [S1].

### When Not to Use
*   Avoid if monocular depth estimation is unreliable for the scene (e.g., highly reflective or transparent surfaces where ZoeDepth fails) [S1].
*   Avoid if strict real-time training speed is the only priority and slight quality loss in textureless regions is acceptable [S1].

## Background

### Origin and History
GeoTexDensifier was introduced in the paper "GeoTexDensifier: Geometry-Texture-Aware Densification for High-Quality Photorealistic 3D Gaussian Splatting" [S1]. It addresses limitations in original 3D Gaussian Splatting (3DGS) and subsequent densification strategies like Mini-Splatting-D, which split splats based solely on contribution area size, leading to over-splitting in textureless regions [S1]. The method builds on geometry prior-based approaches like MonoSDF, DNSplatter, and DNGaussian, but integrates these priors directly into the densification and splitting process rather than just as loss constraints [S1].

### Applications
The method is used for high-quality 3D Gaussian Splatting reconstruction, enabling photorealistic rendering and VR navigation, as demonstrated on the Oculus Quest II using the Gaussian Splatting VR Viewer Unity Plugin [S1].

## Sources

[S1] GeoTexDensifier: Geometry-Texture-Aware Densification for High-Quality Photorealistic 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3644697, fetched 2026-10-26
