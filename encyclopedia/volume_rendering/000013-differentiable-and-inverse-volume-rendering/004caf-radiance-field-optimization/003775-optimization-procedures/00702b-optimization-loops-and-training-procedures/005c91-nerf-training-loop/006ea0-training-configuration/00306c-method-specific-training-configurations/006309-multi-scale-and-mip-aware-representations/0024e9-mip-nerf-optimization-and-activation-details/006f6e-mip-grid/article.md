# Mip-Grid

Also called: Mip-TensoRF, Mip-K-Planes

## Overview
Mip-Grid is a method for integrating anti-aliasing techniques into grid-based radiance field representations, mitigating aliasing artifacts that occur when rendering scenes at varying camera distances or resolutions [S1]. Unlike methods that store distinct grids for each scale or rely on heavy multisampling, Mip-Grid generates multi-scale grid representations from a single shared base grid using learnable depth-wise convolutions, allowing features to vary with camera distance while maintaining fast training times and minimal storage overhead [S1]. It sets itself apart from siblings like [Mip-NeRF 360 Optimization Setup](3426-mip-nerf-360-optimization-setup.md) by operating directly on the grid structure rather than just the ray integration, and from [Scale-Dependent Downweighting for Grid Interpolation](8571-scale-dependent-downweighting-for-grid-interpolation.md) by explicitly generating hierarchical feature grids rather than downweighting interpolation weights.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A shared single-scale grid representation (density $G_\sigma$ and appearance $G_c$), camera parameters (ray origin, direction, and pixel dimensions), and a scale-aware coordinate $s$.
*   **Outputs:** Density $\sigma$ and color $c$ for each sampled point along a ray, derived from the generated multi-scale grids.

**Data Structures and Layouts:**
The method modifies standard grid-based representations (such as TensoRF or K-Planes) by adding a scale dimension $S$ to the grids.
*   **Multi-scale Density Grid:** $\tilde{G}_\sigma \in \mathbb{R}^{H \times W \times L \times S \times C_\sigma}$
*   **Multi-scale Appearance Grid:** $\tilde{G}_c \in \mathbb{R}^{H \times W \times L \times S \times C_c}$
    *   $H, W, L$: Spatial resolutions along axes $X, Y, Z$.
    *   $S$: Number of generated scales (typically 4).
    *   $C_\sigma, C_c$: Number of density and appearance feature channels.
*   **Convolution Kernels:** Learnable filters applied to the base grid factors.
    *   For **Mip-TensoRF** (VM factorization):
        *   Vector filters: $k_{i,v_X}^v \in \mathbb{R}^{K \times R}$ for vector $v_r^X$.
        *   Matrix rank filters: $k_{i,r}^{M_{YZ}} \in \mathbb{R}^K$ for rank $r$ of matrix $M_r^{YZ}$.
        *   Matrix filters: $k_{i,M_{YZ}} \in \mathbb{R}^{K \times K \times R}$ for matrix $M_r^{YZ}$.
        *   Separate kernels exist for each rank, axis, and scale $i$.
    *   For **Mip-K-Planes** (Plane factorization):
        *   Plane filters: $k_i^{YZ} \in \mathbb{R}^{1 \times K \times K \times R}$, $k_i^{XZ} \in \mathbb{R}^{K \times 1 \times K \times R}$, $k_i^{XY} \in \mathbb{R}^{K \times K \times 1 \times R}$.
    *   $K$: Kernel size (typically 3).
    *   $R$: Tensor rank.

**Numbered Steps:**

1.  **Initialize Shared Grids:** Train the base single-scale grids $G_\sigma$ and $G_c$ (or their factorized components) using standard grid-based NeRF optimization.
2.  **Generate Multi-Scale Grids:** Apply learnable depth-wise convolutions to the shared grid factors to generate $S$ scales of multi-scale grids. This is done during the rendering process to minimize storage costs [S1].
    *   For **Mip-TensoRF**, the $i$-th scale of the multi-scale grid $\tilde{G}_i$ is computed as:
        $$ \tilde{G}_i = \sum_{r=1}^{R} \left( (v_r^X * k_{i,v_X}^v) \circ (M_r^{YZ} * k_{i,r}^{M_{YZ}}) + (v_r^Y * k_{i,v_Y}^v) \circ (M_r^{XZ} * k_{i,r}^{M_{XZ}}) + (v_r^Z * k_{i,v_Z}^v) \circ (M_r^{XY} * k_{i,r}^{M_{XY}}) \right) $$
        Where $*$ denotes convolution, $\circ$ denotes tensor product, and $k$ are the learnable depth-wise convolution filters [S1].
    *   For **Mip-K-Planes**, the $i$-th scale $\tilde{G}_{ik}$ is computed as:
        $$ \tilde{G}_{ik} = (M^{YZ} * k_i^{YZ}) \odot (M^{XZ} * k_i^{XZ}) \odot (M^{XY} * k_i^{XY}) $$
        Where $\odot$ is broadcast element-wise multiplication [S1].
3.  **Compute Scale-Aware Coordinate:** Calculate the scale coordinate $s$ based on the camera and ray properties. Three variants are defined:
    *   **Discrete Scale Coordinate:** Defined as the mean of the pixel width and height in world coordinates scaled by $2/\sqrt{12}$. This value is unique for each image scale and used for all sampled points within that image [S1].
    *   **Continuous Scale Coordinate:** The discrete scale coordinate scaled by the distance from the ray origin to the sampled point. This provides distinct scale values for each point, allowing smooth interpolation across broader ranges of scales [S1].
    *   **2-Dimensional Scale Coordinate:** Consists of the continuous scale coordinate and the distance from the ray origin to the sampled point [S1].
4.  **Feature Extraction:** Interpolate features from the generated multi-scale grids $\tilde{G}_\sigma$ and $\tilde{G}_c$ using the scale-aware coordinate $s$ and the spatial coordinate $x$.
    *   Features are extracted from each of the $S$ scales independently.
    *   The feature sets from all scales are averaged to produce the final features [S1].
5.  **Decode Density and Color:**
    *   Density: $\sigma = f_\sigma(x, s; \tilde{G}_\sigma)$
    *   Color: $c = f_c(x, s, d; \theta, \tilde{G}_c)$
    *   Where $d$ is the viewing direction and $\theta$ are MLP parameters [S1].
6.  **Volume Rendering:** Perform differentiable volume rendering using the computed $\sigma$ and $c$ to generate the final pixel color [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mip_grid_render(ray_origin, ray_direction, pixel_width, pixel_height, 
                    base_grid_factors, kernels, scale_coord_type):
    """
    Renders a single ray using Mip-Grid.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        pixel_width: float, width of the pixel in world coordinates.
        pixel_height: float, height of the pixel in world coordinates.
        base_grid_factors: Dictionary containing base grid factors (e.g., vectors and matrices for TensoRF).
        kernels: Dictionary containing learnable convolution kernels for each scale and factor.
        scale_coord_type: str, one of 'discrete', 'continuous', or '2d'.
        
    Returns:
        color: float, rendered RGB color.
    """
    # 1. Compute Scale-Aware Coordinate
    # Mean of pixel width and height scaled by 2/sqrt(12)
    base_scale = (pixel_width + pixel_height) / 2 * (2 / (12 ** 0.5))
    
    # Sample points along the ray (standard volume rendering sampling)
    # Assume samples is a list of distances t from ray_origin
    samples = get_ray_samples(ray_origin, ray_direction)
    
    # 2. Generate Multi-Scale Grids and Extract Features
    # Note: In practice, grids are generated on-the-fly or cached. 
    # Here we simulate the feature extraction for each sample.
    
    final_density_features = []
    final_color_features = []
    
    num_scales = len(kernels) # Typically 4
    
    for i in range(num_scales):
        # Generate i-th scale grid factors using convolutions
        # This step is equivalent to applying 3D convolutions to the 3D tensor
        # due to the linearity of convolution and factorization structure.
        
        # Example for Mip-TensoRF VM factorization:
        # G_i = sum_r ( (v_X * k_i_vX) o (M_YZ * k_i_r_MYZ) + ... )
        # We assume a helper function that applies the specific convolution logic
        # for the chosen grid representation (TensoRF or K-Planes)
        scale_grid_factors = generate_scale_grid(base_grid_factors, kernels[i])
        
        # 3. Determine Scale Coordinate for this sample
        if scale_coord_type == 'discrete':
            s = base_scale
        elif scale_coord_type == 'continuous':
            # Scale by distance from ray origin to sampled point
            # Note: 'samples' contains distances t
            # We need to iterate over samples here, but for pseudocode clarity,
            # we assume we process one sample at a time or vectorize.
            # Let's assume we are processing a single sample with distance t
            t = get_sample_distance() # Placeholder for current sample distance
            s = base_scale * t
        elif scale_coord_type == '2d':
            t = get_sample_distance()
            s_continuous = base_scale * t
            s = (s_continuous, t)
            
        # 4. Interpolate Features from Multi-Scale Grid
        # Interpolate density and color features from scale_grid_factors using spatial coord x and scale s
        # x = ray_origin + t * ray_direction
        x = ray_origin + get_sample_distance() * ray_direction
        
        density_feat = interpolate_features(scale_grid_factors['density'], x, s)
        color_feat = interpolate_features(scale_grid_factors['appearance'], x, s)
        
        final_density_features.append(density_feat)
        final_color_features.append(color_feat)
        
    # 5. Average Features from all scales
    avg_density_feat = average(final_density_features)
    avg_color_feat = average(final_color_features)
    
    # 6. Decode Density and Color
    # f_sigma and f_color are typically small MLPs
    sigma = decode_density(avg_density_feat)
    color = decode_color(avg_color_feat, ray_direction)
    
    # 7. Return for volume rendering accumulation
    return sigma, color

def generate_scale_grid(base_factors, kernels):
    """
    Generates multi-scale grid factors by applying depth-wise convolutions.
    Implementation depends on whether using Mip-TensoRF or Mip-K-Planes.
    """
    # Placeholder for the specific convolution logic described in the formulas
    # For Mip-TensoRF:
    # Apply 1D/2D depth-wise convolutions to vectors and matrices
    # Return the resulting factorized representation
    pass

def interpolate_features(grid_factors, x, s):
    """
    Interpolates features from the grid at spatial coordinate x and scale s.
    """
    # Standard trilinear or higher-order interpolation
    # The grid is effectively 4D (H, W, L, S) or factorized equivalent
    pass
```

**Parameters and Typical Values:**
*   **Kernel Size ($K$):** 3 [S1].
*   **Number of Scales ($S$):** 4 [S1].
*   **Training Iterations:**
    *   Mip-TensoRF: 40k iterations on multi-scale Blender dataset, 25k on multi-scale LLFF dataset [S1].
    *   Mip-K-Planes: 30k iterations [S1].
*   **Kernel Training Start:**
    *   Mip-TensoRF: Kernel training begins after grid upsampling is completed [S1].
    *   Mip-K-Planes: Kernels are optimized from the beginning [S1].
*   **Loss Scaling:** Pixel loss is scaled following mip-NeRF to account for scale differences [S1].

**Edge Cases and Pitfalls:**
*   **Overfitting to Scales:** Using only the discrete scale coordinate can lead to overfitting to the specific scales present in the training dataset, making it difficult to render images at unseen scales. The continuous scale coordinate mitigates this by providing distinct scale values for each sampled point, enabling smoother interpolation across a broader range of scales [S1].
*   **Storage Complexity:** While Mip-Grid avoids the dramatic increase in space complexity associated with storing distinct grids for each scale, it still introduces additional parameters for the convolution kernels. However, this is significantly less than storing full multi-resolution grids [S1].

## Operating characteristics
*   **Time Complexity:** Rendering involves generating multi-scale grids via convolutions and interpolating from $S$ scales. The convolutions operate on reduced dimensions (1D or 2D instead of 3D), minimizing computational costs compared to full 3D convolutions [S1].
*   **Memory Complexity:** Uses a single shared base grid plus learnable kernels. This avoids the storage overhead of maintaining distinct grids for each scale, which would dramatically increase space complexity [S1].
*   **Performance:**
    *   **Mip-TensoRF on Full Res Blender:** PSNR 34.77, SSIM 0.9786, LPIPS 0.0168 [S1].
    *   **Training Time:** Mip-TensoRF training time on RTX 4090 is not explicitly listed in the source table but is noted to be significantly faster than mip-NeRF (>30 hours) [S1]. Mip-TensoRF training took 0.27 hours for the multi-scale Blender dataset in the provided table, compared to 0.16 hours for original TensoRF [S1].
*   **Strengths:**
    *   Mitigates aliasing artifacts effectively.
    *   Fast training time compared to MLP-based methods like mip-NeRF.
    *   Easily integrated into existing grid-based methods (TensoRF, K-Planes) with minimal modifications [S1].
    *   Single-sampling approach, unlike Zip-NeRF which requires ~50 samplings [S1].
*   **Limitations:**
    *   Requires careful tuning of scale coordinates to avoid overfitting.
    *   Performance depends on the quality of the learned convolution kernels [S1].
*   **When to Use:** Use when anti-aliasing is critical for grid-based NeRFs and fast training is desired, especially when dealing with multi-scale datasets or varying camera distances.
*   **When Not to Use:** Avoid if storage for kernels is prohibitive (though it is low) or if the scene does not exhibit significant aliasing issues at the target resolutions.

## Background
Mip-Grid was introduced in the paper "Mip-Grid: Anti-aliased Grid Representations for Neural Radiance Fields" [S1]. It builds upon the concept of mipmapping in computer graphics to address aliasing in grid representations. The method was developed to combine the anti-aliasing benefits of mip-NeRF with the fast training speeds of grid-based methods like TensoRF and K-Planes. It was tested on multi-scale Blender and LLFF datasets, demonstrating superior performance over original grid-based methods and competitive performance with mip-NeRF [S1].

## Sources
[S1] Mip-Grid: Anti-aliased Grid Representations for Neural Radiance Fields, https://arxiv.org/abs/2402.14196v1, fetched 2026-10-27
