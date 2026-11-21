# Reference-Based Non-Photorealistic Radiance Fields
Also called: Ref-NPR

## Overview
Ref-NPR is a controllable scene stylization method that transfers the style of a single 2D reference image to a 3D scene represented by a radiance field, ensuring semantic consistency across novel views [S1]. Unlike standard NeRF optimization which minimizes photometric error against ground-truth photos, Ref-NPR optimizes a non-photorealistic (NP) radiance field using explicit supervision from a stylized reference view and implicit supervision for occluded regions [S1]. It sets itself apart from sibling methods like [Neural Reconstruction in the Wild](6202-neural-reconstruction-in-the-wild.md) by explicitly mapping 2D stylized pixels to 3D space via ray registration to establish semantic correspondence, rather than relying solely on global style loss or 2D warping [S1].

## How it works
Ref-NPR builds on the differentiable volume rendering principles of [Neural Radiance Fields](4972-neural-radiance-fields.md). It optimizes a new NP radiance field $\omega_{NP}$ using a pre-trained photorealistic radiance field $\omega_P$ as a geometric prior [S1].

**Inputs:**
1.  Pre-trained photorealistic radiance field $\omega_P$ (parameters $\sigma, \mathbf{c}$).
2.  Set of training camera poses $\Phi = \{\phi_1, \dots, \phi_K\}$ and their corresponding content images $I_T$.
3.  A specific reference camera pose $\phi_R \in \Phi$ (or arbitrary) and its content image $I_R$.
4.  A stylized version of the reference image $S_R$, obtained via 2D stylization (e.g., Gatys [11] or AdaIN [20]) or manual editing [S1].

**Outputs:**
1.  Optimized non-photorealistic radiance field $\omega_{NP}$.
2.  Stylized renderings $S_T$ for arbitrary target camera poses $\phi_T$ [S1].

**Data Structures:**
*   **Radiance Field $\omega$**: A neural network mapping 3D position $\mathbf{x}$ and view direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$.
*   **Reference Dictionary $\mathcal{D}$**: A collection of rays from the reference camera $\phi_R$, each associated with a 3D intersection point and a stylized color.
*   **Pseudo-Ray Supervision $\Gamma_R$**: A set of rays from training views $\Phi$ mapped to the reference dictionary, carrying stylized color targets [S1].

**Algorithm Steps:**

1.  **Render Reference View:** Render the content image $I_R$ from camera $\phi_R$ using $\omega_P$ to obtain the reference view [S1].
2.  **Stylize Reference:** Apply a 2D stylization algorithm to $I_R$ to generate the stylized reference view $S_R$ [S1].
3.  **Estimate Pseudo-Depth:** For each ray $r$ in the reference view, estimate the depth $l(r)$ where the ray intersects the scene geometry. The intersection point is defined as:
    $$
    x(r) = o + l(r)\mathbf{d}
    $$
    where $o$ is the camera origin, $\mathbf{d}$ is the ray direction, and $l(r)$ is the depth at which the accumulated transmittance drops below a threshold $\sigma_z$ [S1]:
    $$
    l(r) = \min \left\{ t_i \mid \sum_{j=1}^{i-1} \sigma_j (t_{j+1} - t_j) \ge \sigma_z \right\}
    $$
    [S1].
4.  **Build Reference Dictionary:** Construct a dictionary $\mathcal{D}$ for $S_R$ by storing the mapping from pixel coordinates $(x,y,z)$ to the stylized color $\hat{C}_R(r)$ and the ray direction [S1].
5.  **Reference Ray Registration (R3):** For each training ray $r_j$ from poses $\Phi \setminus \{\phi_R\}$, find a corresponding ray $\hat{r}_j$ in the reference dictionary $\mathcal{D}$ (associated with $\phi_R$) that minimizes the Euclidean distance between their 3D intersection points. The registration constraint requires that the intersection point of $r_j$ falls into the same voxel $D(x,y,z)$ as the reference ray, and the angle between their directions $\mathbf{d}_{r_i}$ and $\mathbf{d}_{r_j}$ is less than a threshold $\theta$ [S1]:
    $$
    \hat{r}_j = \arg \min_{r \in \mathcal{D}} \| x(r_j) - x(r) \|_2 \quad \text{subject to} \quad \angle(\mathbf{d}_{r_i}, \mathbf{d}_{r_j}) < \theta
    $$
    [S1].
6.  **Construct Pseudo-Ray Supervision:** Create the supervision set $\Gamma_R$ by assigning the stylized color $\hat{C}_R(\hat{r}_j)$ from the reference dictionary to the registered training ray $r_j$ [S1].
7.  **Template-based Correspondence Module (TCM):** For regions in the reference view that are occluded in the training views (or vice versa), use TCM to propagate style implicitly. This module uses feature matching to fill occluded regions with perceptually similar styles, ensuring consistency where direct ray registration is impossible [S1].
8.  **Optimize NP Radiance Field:** Train $\omega_{NP}$ by minimizing a composite loss function $\mathcal{L}_{NP}$ that includes:
    *   **Explicit Supervision:** Photometric loss between the rendered color $\hat{C}_{NP}(r_j)$ and the stylized color from $\Gamma_R$ for registered rays.
    *   **Feature Loss:** Style loss (e.g., Gram matrix distance) between features of the rendered image and $S_R$ [S1].
    *   **Implicit Supervision:** Loss terms derived from TCM for occluded regions [S1].
    $$
    \mathcal{L}_{NP} = \mathcal{L}_{ref} + \mathcal{L}_{feat} + \mathcal{L}_{color}
    $$
    [S1].
9.  **Inference:** Render stylized images $S_T$ from arbitrary target poses $\phi_T$ using the optimized $\omega_{NP}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ref_npr_optimize(omega_P, phi_R, I_R, S_R, Phi, sigma_z, theta):
    """
    Optimizes a non-photorealistic radiance field using reference-based stylization.
    
    Args:
        omega_P: Pre-trained photorealistic radiance field.
        phi_R: Reference camera pose.
        I_R: Content image for reference view.
        S_R: Stylized reference image.
        Phi: Set of training camera poses.
        sigma_z: Depth threshold for pseudo-depth estimation.
        theta: Maximum angle threshold for ray registration.
        
    Returns:
        omega_NP: Optimized non-photorealistic radiance field.
    """
    # Step 1 & 2: Reference view is already stylized to S_R
    
    # Step 3 & 4: Build Reference Dictionary
    # Sample rays from phi_R
    ref_rays = sample_rays(phi_R)
    reference_dict = {}
    
    for ray in ref_rays:
        # Estimate pseudo-depth l(r)
        # This requires querying omega_P for density along the ray
        l_r = estimate_pseudo_depth(ray, omega_P, sigma_z)
        
        # Calculate intersection point x(r) = o + l(r)*d
        x_r = ray.origin + l_r * ray.direction
        
        # Store in dictionary mapped by voxel coordinates (x,y,z)
        voxel_coords = quantize_to_voxel(x_r)
        reference_dict[voxel_coords] = {
            'ray': ray,
            'color': get_pixel_color(S_R, ray.pixel_coords),
            'direction': ray.direction
        }
        
    # Step 5 & 6: Reference Ray Registration (R3)
    pseudo_ray_supervision = []
    
    for phi in Phi:
        if phi == phi_R:
            continue
            
        train_rays = sample_rays(phi)
        for ray_j in train_rays:
            # Estimate intersection for training ray
            l_j = estimate_pseudo_depth(ray_j, omega_P, sigma_z)
            x_j = ray_j.origin + l_j * ray_j.direction
            voxel_coords_j = quantize_to_voxel(x_j)
            
            # Find best match in reference dictionary
            best_match = None
            min_dist = infinity
            
            if voxel_coords_j in reference_dict:
                ref_entry = reference_dict[voxel_coords_j]
                ref_ray = ref_entry['ray']
                
                # Check angle constraint
                angle = angle_between(ray_j.direction, ref_ray.direction)
                if angle < theta:
                    dist = euclidean_distance(x_j, ref_ray.origin + l_ref * ref_ray.direction)
                    # Note: l_ref is implicitly handled by the voxel match
                    if dist < min_dist:
                        min_dist = dist
                        best_match = ref_entry
            
            if best_match is not None:
                # Assign stylized color from reference to training ray
                pseudo_ray_supervision.append({
                    'ray': ray_j,
                    'target_color': best_match['color']
                })
                
    # Step 7: TCM for occluded regions (Implicit Supervision)
    # This module handles regions where direct registration fails
    # Implementation details depend on specific feature matching library
    tcm_loss = compute_implicit_style_loss(S_R, omega_P, phi_R, Phi)
    
    # Step 8: Optimize NP Radiance Field
    omega_NP = initialize_radiance_field()
    
    for iteration in range(num_iterations):
        # Sample rays from training poses
        batch_rays = sample_batch_rays(Phi)
        
        # Render colors using omega_NP
        rendered_colors = render_volume(batch_rays, omega_NP)
        
        # Compute explicit loss for registered rays
        explicit_loss = 0
        for sup in pseudo_ray_supervision:
            # Find corresponding rendered color for sup['ray']
            # In practice, this is done via batch rendering
            rendered_c = get_rendered_color_for_ray(rendered_colors, sup['ray'])
            target_c = sup['target_color']
            explicit_loss += mse_loss(rendered_c, target_c)
            
        # Compute feature loss (style loss)
        feature_loss = compute_style_loss(rendered_colors, S_R)
        
        # Total loss
        total_loss = explicit_loss + feature_loss + tcm_loss
        
        # Backpropagate and update omega_NP
        update_weights(omega_NP, total_loss)
        
    return omega_NP

def estimate_pseudo_depth(ray, omega_P, sigma_z):
    """
    Estimates depth l(r) where accumulated transmittance drops below sigma_z.
    """
    # Sample points along ray
    t_values = linspace(ray.origin, ray.origin + ray.direction * max_dist, N_samples)
    accumulated_sigma = 0
    l_r = max_dist
    
    for i, t in enumerate(t_values):
        sigma = omega_P.query_density(t, ray.direction)
        delta_t = t_values[i+1] - t if i < len(t_values)-1 else 0
        accumulated_sigma += sigma * delta_t
        
        if accumulated_sigma >= sigma_z:
            l_r = t
            break
            
    return l_r
```

**Parameters:**
*   $\sigma_z$: Depth threshold for pseudo-depth estimation. Not given in the sources.
*   $\theta$: Angle threshold for ray registration. Not given in the sources.
*   $N$: Number of samples per ray. Not given in the sources.

**Edge Cases:**
*   **Occlusions:** Rays from training views may not intersect the same 3D points as reference rays due to occlusion. R3 handles explicit supervision for visible regions, while TCM handles implicit supervision for occluded regions [S1].
*   **Over-matching:** Rays with similar intersection points but vastly different directions may be incorrectly matched. The angle constraint $\theta$ prevents this [S1].
*   **Precision:** Depth estimation relies on the accuracy of $\omega_P$. Errors in $\omega_P$ propagate to the pseudo-depth and thus the ray registration [S1].

## Subtopics

### [Reference Ray Registration (R3)](15646-reference-ray-registration-r3.md)
A sub-method for establishing semantic consistency by mapping stylized reference pixels to 3D space via pseudo-depth and voxel-based matching. It provides explicit supervision for visible regions. Choose it when semantic correspondence is critical and a pre-trained geometric model is available.

## Operating characteristics
*   **Time Complexity:** Depends on the number of training views, samples per ray, and the size of the reference dictionary. Ray registration requires searching the dictionary for each training ray, which can be optimized with spatial hashing (voxels).
*   **Memory Complexity:** Requires storing the reference dictionary and the optimized NP radiance field. The dictionary size is proportional to the resolution of the reference view.
*   **Strengths:** Preserves semantic correspondence better than global style transfer methods; handles occlusions via TCM; controllable via single reference image.
*   **Limitations:** Relies on the accuracy of the pre-trained photorealistic radiance field $\omega_P$ for geometry; may struggle with large viewpoint changes where ray registration fails; requires 2D stylization quality.
*   **When to use:** When stylizing a known 3D scene with a specific artistic style from a single reference image while maintaining object identity and structure.
*   **When not to use:** When no pre-trained NeRF is available; when the reference view is significantly different from the training views (large baseline); when real-time performance is required without optimization.

## Background
Ref-NPR was introduced in 2023 by the authors of "Ref-NPR: Reference-Based Non-Photorealistic Radiance Fields for Controllable Scene Stylization" [S1]. It addresses the limitation of existing 3D scene stylization methods that lack meaningful semantic correspondences between the reference and novel views [S1]. The method builds on the differentiable rendering of NeRF and incorporates techniques from 2D style transfer (Gatys, AdaIN) and 3D geometry estimation [S1].

## Sources
[S1] Ref-NPR: Reference-Based Non-Photorealistic Radiance Fields for Controllable Scene Stylization, https://doi.org/10.1109/cvpr52729.2023.00413, fetched 2026-10-26
[S2] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
