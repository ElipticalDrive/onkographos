# NeILF Rendering and Loss Calculation

Also called: inverse rendering with Blinn-Phong

## Overview
NeILF (Neural Inverse Lighting Field) is a differentiable rendering pipeline that extends 3D Gaussian Splatting by decomposing scene appearance into physically-based diffuse, specular, and ambient components using a Blinn-Phong model [S1, unofficial]. It solves the problem of inverse rendering in Gaussian-based representations by enabling the optimization of material properties (shininess, diffuse color) and lighting conditions from multi-view images, depth, and normal constraints [S1, unofficial]. Unlike standard 3DGS which renders view-dependent RGB colors directly, NeILF renders view-independent scalar attributes and combines them with geometric normals to compute lighting, allowing for relighting and material editing [S1, unofficial].

## How it works
The method operates in two phases: rendering the Blinn-Phong components and calculating a composite loss function for training.

### Inputs and Outputs
**Inputs:**
*   $\mathcal{G}$: Set of $M$ 3D Gaussians with attributes including diffuse factor, offset color, shininess, ambient factor, and specular factor [S1, unofficial].
*   Camera parameters: Intrinsics, extrinsics (view matrix), and position [S1, unofficial].
*   Light parameters: Incident direction $\mathbf{l}$ and view direction $\mathbf{v}$ [S1, unofficial].
*   Ground truth: Image, depth, and optionally normal maps [S1, unofficial].
*   Optimization weights: $\lambda_{render}, \lambda_{phong}, \lambda_{depth}, \lambda_{opacity}, \lambda_{normal\_render\_depth}, \lambda_{normal\_mvs\_depth}, \lambda_{offset\_color\_sparsity}, \lambda_{diffuse\_factor\_smooth}, \lambda_{ambient\_factor\_smooth}, \lambda_{normal\_smooth}, \lambda_{dssim}$ [S1, unofficial].
*   Sample count: $N_{sample}$ (default 24) for incident ray sampling [S1, unofficial].

**Outputs:**
*   Rendered image components: Diffuse, specular, ambient, and combined Phong [S1, unofficial].
*   Geometric outputs: Normal, depth, opacity [S1, unofficial].
*   Loss value: Scalar sum of weighted loss terms [S1, unofficial].

### Data Structures
The rendering result is stored in a dictionary `results` with the following keys [S1, unofficial]:
*   `render`: Final combined color image.
*   `phong`: Blinn-Phong rendered image.
*   `normal`: Rendered surface normals.
*   `pseudo_normal`: Normals derived from depth.
*   `surface_xyz`: 3D positions of rendered pixels.
*   `opacity`: Opacity map.
*   `depth`: Depth map.
*   `viewspace_points`, `visibility_filter`, `radii`, `num_rendered`, `num_contrib`: Standard 3DGS auxiliary data [S1, unofficial].
*   `diffuse_factor`, `offset_color`, `shininess`, `ambient_factor`, `specular_factor`: Material attributes [S1, unofficial].
*   `diffuse_term`, `specular_term`, `ambient_term`: Individual lighting components [S1, unofficial].
*   `hdr`, `val_gamma`: HDR handling parameters [S1, unofficial].
*   `tb_dict`, `loss`: TensorBoard dictionary and final loss scalar [S1, unofficial].

### Rendering Equation (Blinn-Phong)
The rendering process computes the lighting contribution for each pixel by aggregating Gaussian splats. For each Gaussian $i$, the contribution is modulated by its material attributes.

1.  **Lighting Geometry**:
    *   Let $\mathbf{n}_i$ be the surface normal of Gaussian $i$.
    *   Let $\mathbf{l}$ be the normalized light direction (incident ray).
    *   Let $\mathbf{v}$ be the normalized view direction.
    *   Compute the half-vector: $\mathbf{h} = \frac{\mathbf{l} + \mathbf{v}}{\|\mathbf{l} + \mathbf{v}\|}$ [S1, unofficial].
    *   Compute $\cos_l = \mathbf{n}_i \cdot \mathbf{l}$ and $\cos_h = \mathbf{n}_i \cdot \mathbf{h}$ [S1, unofficial].

2.  **Diffuse Component**:
    *   Compute diffuse intensity: $I_{diffuse} = \text{diffuse\_factor}_i \cdot \max(0, \cos_l)$ [S1, unofficial].
    *   Compute diffuse color: $\mathbf{C}_{diffuse} = (\text{ambient\_factor}_i + I_{diffuse}) \cdot \mathbf{C}_{base}$, where $\mathbf{C}_{base}$ is derived from `offset_color` and `palette_color` [S1, unofficial].
    *   Specifically, `offset_color_norm` is computed as the sum of squared `offset_color` along the last two dimensions [S1, unofficial]. The final diffuse color uses `offset_color + palette_color.clamp(0, 1).detach()` [S1, unofficial].

3.  **Specular Component**:
    *   Compute specular intensity: $I_{specular} = \text{specular\_factor}_i \cdot \begin{cases} |\cos_h|^{\text{shininess}_i} & \text{if } \cos_l \neq 0 \\ 0 & \text{otherwise} \end{cases}$ [S1, unofficial].
    *   The implementation uses `torch.where(cos_l != 0, (torch.abs(cos_h)).pow(shininess), 0.0)` [S1, unofficial].

4.  **Combination**:
    *   $\mathbf{C}_{phong} = \mathbf{C}_{diffuse} + \mathbf{C}_{specular}$ [S1, unofficial].
    *   Final color is clamped to $[0, 1]$ [S1, unofficial].

5.  **Multi-Gaussian Grouping**:
    *   If multiple Gaussian groups exist (`len(num_GSs_TFs) > 1`), iterate through groups. Set `diffuse_color[start_GS:end_GS]` to the computed value and multiply opacity by `opacity_factors[i]` if present [S1, unofficial].

6.  **Light Transforms**:
    *   If `light_transform` is not None, apply transformations to `diffuse_factor`, `shininess`, `ambient_factor`, and `specular_factor` using multiplication and addition offsets [S1, unofficial].

### Loss Calculation
The total loss $\mathcal{L}$ is a weighted sum of multiple terms, computed only if the corresponding lambda is $> 0$ [S1, unofficial].

1.  **Render Loss ($\mathcal{L}_{render}$)**:
    *   If $\lambda_{render} > 0$:
        $$ \mathcal{L}_{render} = (1 - \lambda_{dssim}) \mathcal{L}_{1}(\mathbf{I}_{render}, \mathbf{I}_{gt}) + \lambda_{dssim} (1 - \text{SSIM}(\mathbf{I}_{render}, \mathbf{I}_{gt})) $$
        where $\mathbf{I}_{render}$ is the standard 3DGS render and $\mathbf{I}_{gt}$ is the ground truth image [S1, unofficial].

2.  **Phong Loss ($\mathcal{L}_{phong}$)**:
    *   If $\lambda_{phong} > 0$:
        $$ \mathcal{L}_{phong} = \lambda_{phong} \left[ (1 - \lambda_{dssim}) \mathcal{L}_{1}(\mathbf{I}_{phong}, \mathbf{I}_{gt}) + \lambda_{dssim} (1 - \text{SSIM}(\mathbf{I}_{phong}, \mathbf{I}_{gt})) \right] $$
        where $\mathbf{I}_{phong}$ is the Blinn-Phong rendered image [S1, unofficial].

3.  **Depth Loss ($\mathcal{L}_{depth}$)**:
    *   If $\lambda_{depth} > 0$:
        $$ \mathcal{L}_{depth} = \lambda_{depth} \mathcal{L}_{1}(\mathbf{D}_{render}, \mathbf{D}_{gt}) $$
        Computed excluding pixels where `image_mask XOR depth_mask` is true [S1, unofficial].

4.  **Opacity Loss ($\mathcal{L}_{opacity}$)**:
    *   If $\lambda_{opacity} > 0$:
        Clamp rendered opacity to $[10^{-6}, 1 - 10^{-6}]$.
        $$ \mathcal{L}_{opacity} = \lambda_{opacity} \left[ (1 - \lambda_{dssim}) \mathcal{L}_{1}(\mathbf{O}_{render}, \mathbf{M}_{img}) + \lambda_{dssim} (1 - \text{SSIM}(\mathbf{O}_{render}, \mathbf{M}_{img})) \right] $$
        where $\mathbf{M}_{img}$ is the image mask [S1, unofficial].

5.  **Normal Loss from Rendered Depth ($\mathcal{L}_{normal\_render\_depth}$)**:
    *   If $\lambda_{normal\_render\_depth} > 0$:
        $$ \mathcal{L}_{normal\_render\_depth} = \lambda_{normal\_render\_depth} \mathcal{MSE}(\mathbf{N}_{render}, \mathbf{N}_{pseudo}) $$
        Masked by `image_mask`. $\mathbf{N}_{pseudo}$ is derived from `results` [S1, unofficial].

6.  **Normal Loss from MVS Depth ($\mathcal{L}_{normal\_mvs\_depth}$)**:
    *   If $\lambda_{normal\_mvs\_depth} > 0$ and MVS normal map is all zeros:
        *   Compute pseudo normal $\mathbf{N}_{pseudo}$ from $\mathbf{D}_{gt}$ using `kornia.depth_to_normals` [S1, unofficial].
        *   Transform $\mathbf{N}_{pseudo}$ to world space via camera intrinsics and `world_view_transform` [S1, unofficial].
        *   $$ \mathcal{L}_{normal\_mvs\_depth} = \lambda_{normal\_mvs\_depth} \mathcal{MSE}(\mathbf{N}_{render}, \mathbf{N}_{pseudo}) $$
        Masked by `depth_mask` [S1, unofficial].

7.  **Sparsity Loss ($\mathcal{L}_{sparsity}$)**:
    *   If $\lambda_{offset\_color\_sparsity} > 0$:
        $$ \mathcal{L}_{sparsity} = \lambda_{offset\_color\_sparsity} \text{Sparsity}(\mathbf{C}_{offset\_norm}) $$
        Masked by `image_mask` [S1, unofficial].

8.  **Smoothness Losses**:
    *   **Diffuse Factor Smoothness**: If $\lambda_{diffuse\_factor\_smooth} > 0$, compute bilateral smooth loss on `rendered_diffuse_factor` using `gt_image` and `image_mask` [S1, unofficial].
    *   **Ambient Factor Smoothness**: If $\lambda_{ambient\_factor\_smooth} > 0$, compute bilateral smooth loss on `rendered_specular_intensity` and `rendered_ambient_intensity` separately [S1, unofficial].
    *   **Normal Smoothness**: If $\lambda_{normal\_smooth} > 0$, compute bilateral smooth loss on `rendered_normal` using `gt_image` and `image_mask` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_neilf(camera, gaussians, light_transform=None, sample_num=24):
    """
    Renders the scene using Blinn-Phong lighting model.
    Returns results dictionary.
    """
    # 1. Base rendering to get geometric attributes
    # This step assumes a standard 3DGS rasterization that outputs 
    # normals, depth, opacity, and material attributes (diffuse_factor, 
    # offset_color, shininess, ambient_factor, specular_factor)
    base_results = render_view(camera, gaussians)
    
    normals = base_results['normal']
    diffuse_factor = base_results['diffuse_factor']
    offset_color = base_results['offset_color']
    shininess = base_results['shininess']
    ambient_factor = base_results['ambient_factor']
    specular_factor = base_results['specular_factor']
    palette_colors = base_results.get('palette_colors', None)
    
    # 2. Apply light transforms if provided
    if light_transform is not None:
        diffuse_factor = diffuse_factor * light_transform.mul + light_transform.add
        shininess = shininess * light_transform.mul + light_transform.add
        ambient_factor = ambient_factor * light_transform.mul + light_transform.add
        specular_factor = specular_factor * light_transform.mul + light_transform.add

    # 3. Compute Base Color from Offset and Palette
    # offset_color_norm is sum of squared offset_color along last two dims
    offset_color_norm = torch.sum(offset_color ** 2, dim=-1, keepdim=True)
    
    # Handle multi-Gaussian groups if necessary
    # Assuming base_results contains grouping info or we iterate
    # For simplicity, we assume a single group or that the attributes 
    # are already aggregated per pixel. If not, iterate groups:
    # for i in range(len(num_GSs_TFs)):
    #     start, end = ...
    #     diffuse_color[start:end] = (offset_color[start:end] + palette_colors[i].clamp(0,1).detach())
    #     opacity[start:end] *= opacity_factors[i]
    
    # Compute base color
    if palette_colors is not None:
        # Assuming palette_colors is broadcastable or indexed correctly
        base_color = offset_color + palette_colors.clamp(0, 1).detach()
    else:
        base_color = offset_color
        
    # 4. Compute Lighting Components
    # Get view and light directions for each pixel
    # Note: In a rasterized setting, view direction is constant per pixel 
    # (from camera center to pixel), light direction is sampled or fixed.
    view_dirs = camera.get_view_directions() # Shape: (H, W, 3)
    
    # Sample incident directions if needed, or use fixed light
    # For standard inverse rendering, light direction is often a parameter
    # Here we assume a fixed light direction or sampled from sphere
    incident_dirs = sample_light_directions(sample_num) # Shape: (sample_num, 3) or (H, W, 3)
    
    # Compute Half Vector
    # h = normalize(l + v)
    # We need to handle the case where l + v is zero
    h = torch.nn.functional.normalize(incident_dirs + view_dirs, dim=-1)
    
    # Cosines
    cos_l = torch.sum(normals * incident_dirs, dim=-1) # (H, W) or (H, W, sample_num)
    cos_h = torch.sum(normals * h, dim=-1)
    
    # Diffuse Term
    # I_diffuse = diffuse_factor * max(0, cos_l)
    diffuse_intensity = diffuse_factor * torch.clamp(cos_l, min=0.0)
    diffuse_term = (ambient_factor + diffuse_intensity) * base_color
    
    # Specular Term
    # I_specular = specular_factor * |cos_h|^shininess if cos_l != 0 else 0
    # Using torch.where to handle division by zero or invalid cases
    specular_intensity = specular_factor * torch.where(
        cos_l != 0, 
        torch.abs(cos_h).pow(shininess), 
        torch.zeros_like(cos_h)
    )
    specular_term = specular_intensity.unsqueeze(-1) # Assuming scalar intensity, expand to RGB if needed
    
    # Combine
    phong_color = diffuse_term + specular_term
    phong_color = torch.clamp(phong_color, 0.0, 1.0)
    
    # 5. Store Results
    results = {
        'render': base_results['render'], # Standard 3DGS render
        'phong': phong_color,
        'normal': normals,
        'pseudo_normal': base_results.get('pseudo_normal'),
        'surface_xyz': base_results.get('surface_xyz'),
        'opacity': base_results['opacity'],
        'depth': base_results['depth'],
        'viewspace_points': base_results['viewspace_points'],
        'visibility_filter': base_results['visibility_filter'],
        'radii': base_results['radii'],
        'num_rendered': base_results['num_rendered'],
        'num_contrib': base_results['num_contrib'],
        'diffuse_factor': diffuse_factor,
        'offset_color': offset_color,
        'shininess': shininess,
        'ambient_factor': ambient_factor,
        'specular_factor': specular_factor,
        'diffuse_term': diffuse_term,
        'specular_term': specular_term,
        'ambient_term': ambient_factor * base_color, # Approximation
        'hdr': base_results.get('hdr'),
        'val_gamma': base_results.get('val_gamma'),
        'tb_dict': {},
        'loss': torch.tensor(0.0, device=phong_color.device)
    }
    
    return results

def calculate_loss(results, gt_image, gt_depth, image_mask, depth_mask, 
                   lambda_render=1.0, lambda_phong=1.0, lambda_depth=0.0, 
                   lambda_opacity=0.0, lambda_normal_render_depth=0.0, 
                   lambda_normal_mvs_depth=0.0, lambda_offset_color_sparsity=0.0,
                   lambda_diffuse_factor_smooth=0.0, lambda_ambient_factor_smooth=0.0,
                   lambda_normal_smooth=0.0, lambda_dssim=0.2):
    """
    Calculates the total loss for NeILF training.
    """
    loss = torch.tensor(0.0, device=results['phong'].device)
    
    # 1. Render Loss (Standard 3DGS)
    if lambda_render > 0:
        l1_render = torch.abs(results['render'] - gt_image).mean()
        ssim_render = ssim(results['render'], gt_image)
        loss += lambda_render * ((1 - lambda_dssim) * l1_render + lambda_dssim * (1 - ssim_render))
        
    # 2. Phong Loss
    if lambda_phong > 0:
        l1_phong = torch.abs(results['phong'] - gt_image).mean()
        ssim_phong = ssim(results['phong'], gt_image)
        loss += lambda_phong * ((1 - lambda_dssim) * l1_phong + lambda_dssim * (1 - ssim_phong))
        
    # 3. Depth Loss
    if lambda_depth > 0:
        # Exclude pixels where image_mask XOR depth_mask is true
        exclude_mask = torch.logical_xor(image_mask, depth_mask)
        valid_mask = ~exclude_mask
        l1_depth = torch.abs(results['depth'][valid_mask] - gt_depth[valid_mask]).mean()
        loss += lambda_depth * l1_depth
        
    # 4. Opacity Loss
    if lambda_opacity > 0:
        opacity_clamped = torch.clamp(results['opacity'], 1e-6, 1 - 1e-6)
        l1_opacity = torch.abs(opacity_clamped - image_mask).mean()
        ssim_opacity = ssim(opacity_clamped, image_mask)
        loss += lambda_opacity * ((1 - lambda_dssim) * l1_opacity + lambda_dssim * (1 - ssim_opacity))
        
    # 5. Normal Loss from Rendered Depth
    if lambda_normal_render_depth > 0:
        pseudo_normal = results['pseudo_normal']
        mse_normal = torch.mean((results['normal'] - pseudo_normal) ** 2 * image_mask)
        loss += lambda_normal_render_depth * mse_normal
        
    # 6. Normal Loss from MVS Depth
    if lambda_normal_mvs_depth > 0:
        # Check if MVS normal is all zeros
        mvs_normal = results.get('mvs_normal', None)
        if mvs_normal is not None and torch.all(mvs_normal == 0):
            # Compute pseudo normal from gt_depth
            pseudo_normal_mvs = depth_to_normals(gt_depth)
            # Transform to world space
            pseudo_normal_mvs = transform_to_world_space(pseudo_normal_mvs, camera_intrinsics, world_view_transform)
            mse_normal_mvs = torch.mean((results['normal'] - pseudo_normal_mvs) ** 2 * depth_mask)
            loss += lambda_normal_mvs_depth * mse_normal_mvs
            
    # 7. Sparsity Loss
    if lambda_offset_color_sparsity > 0:
        offset_color_norm = torch.sum(results['offset_color'] ** 2, dim=-1, keepdim=True)
        sparsity_loss = torch.mean(offset_color_norm * image_mask)
        loss += lambda_offset_color_sparsity * sparsity_loss
        
    # 8. Smoothness Losses
    if lambda_diffuse_factor_smooth > 0:
        smooth_diffuse = bilateral_smooth_loss(results['diffuse_factor'], gt_image, image_mask)
        loss += lambda_diffuse_factor_smooth * smooth_diffuse
        
    if lambda_ambient_factor_smooth > 0:
        smooth_specular = bilateral_smooth_loss(results['specular_intensity'], gt_image, image_mask)
        smooth_ambient = bilateral_smooth_loss(results['ambient_intensity'], gt_image, image_mask)
        loss += lambda_ambient_factor_smooth * (smooth_specular + smooth_ambient)
        
    if lambda_normal_smooth > 0:
        smooth_normal = bilateral_smooth_loss(results['normal'], gt_image, image_mask)
        loss += lambda_normal_smooth * smooth_normal
        
    results['loss'] = loss
    return loss
```

### Parameters
*   `sample_num`: Number of incident rays sampled for normal-based lighting. Default: 24 [S1, unofficial].
*   `lambda_render`: Weight for standard render loss. Default: 1.0 [S1, unofficial].
*   `lambda_phong`: Weight for Blinn-Phong render loss. Default: 1.0 [S1, unofficial].
*   `lambda_depth`: Weight for depth loss. Default: 0.0 [S1, unofficial].
*   `lambda_opacity`: Weight for opacity loss. Default: 0.0 [S1, unofficial].
*   `lambda_normal_render_depth`: Weight for normal loss from rendered depth. Default: 0.0 [S1, unofficial].
*   `lambda_normal_mvs_depth`: Weight for normal loss from MVS depth. Default: 0.0 [S1, unofficial].
*   `lambda_offset_color_sparsity`: Weight for offset color sparsity. Default: 0.0 [S1, unofficial].
*   `lambda_diffuse_factor_smooth`: Weight for diffuse factor smoothness. Default: 0.0 [S1, unofficial].
*   `lambda_ambient_factor_smooth`: Weight for ambient factor smoothness. Default: 0.0 [S1, unofficial].
*   `lambda_normal_smooth`: Weight for normal smoothness. Default: 0.0 [S1, unofficial].
*   `lambda_dssim`: Weight for SSIM in L1/SSIM loss combination. Default: 0.2 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Frustum Culling**: Gaussians with radius 0 or those frustum culled are excluded from value updates in splitting criteria via `visibility_filter` (where `radii > 0`) [S1, unofficial].
*   **MVS Normal Zeros**: The MVS normal map may be all zeros. In this case, the pseudo normal must be computed from the ground truth depth using `kornia.depth_to_normals` and transformed to world space using camera intrinsics and `world_view_transform` [S1, unofficial].
*   **Division by Zero**: The specular term uses `torch.where(cos_l != 0, ...)` to avoid invalid specular highlights when the light is behind the surface [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Dominated by the rasterization of $M$ Gaussians, which is $O(M)$ per pixel. The lighting calculation adds a constant factor per pixel based on the number of light samples ($N_{sample}$).
*   **Memory Complexity**: Requires storing additional attributes per Gaussian (diffuse factor, shininess, ambient/specular factors) compared to standard RGB, increasing memory usage by a factor proportional to the number of material channels.
*   **Strengths**: Enables inverse rendering, relighting, and material editing. Provides physically plausible lighting decomposition.
*   **Limitations**: Requires more complex loss functions and potentially more training time due to the additional constraints. Relies on accurate normal estimation, which can be noisy in Gaussian splatting.
*   **When to Use**: When the goal is to recover material properties or lighting conditions from images, or to enable relighting.
*   **When Not to Use**: When only view-dependent appearance is needed, or when computational resources are limited and standard 3DGS suffices.

## Background
NeILF was introduced as an extension to 3D Gaussian Splatting to support inverse rendering tasks. It decomposes the rendering process into diffuse, specular, and ambient components using a Blinn-Phong model, allowing for the optimization of material properties and lighting. The method is implemented in the VEG repository [S1, unofficial].

## Sources
[S1] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
