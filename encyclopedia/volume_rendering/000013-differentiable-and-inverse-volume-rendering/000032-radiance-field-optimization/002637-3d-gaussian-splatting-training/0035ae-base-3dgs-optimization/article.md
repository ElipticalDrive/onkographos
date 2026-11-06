# Base 3DGS Optimization

Also called: Base 3DGS Optimization with Normal Attribute

## Overview
Base 3DGS Optimization initializes the geometric attributes of 3D Gaussian primitives to provide a robust baseline for subsequent shading attribute optimization in inverse volume rendering [S1]. It differs from standard 3DGS by incorporating an explicit normal attribute $n$ and optimizing it via a normal consistency loss, which accelerates the convergence of editable Gaussian optimization without altering the final rendering output [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Multi-view images, camera poses, initial Gaussian parameters $\{\mu_i, q_i, s_i, o_i, c_i\}$, and a depth map $D$ (or a method to derive pseudo-depth) for the target viewpoint.
*   **Outputs:** Optimized Gaussian parameters $\{\mu_i, q_i, s_i, o_i, n_i, c_i\}$, where $n_i$ is the normal attribute and $c_i$ are SH coefficients.

**Data Structures:**
The $i$-th Gaussian is parameterized as:
$$ \{\mu_i, q_i, s_i, o_i, n_i, c_i\} $$
*   $\mu_i \in \mathbb{R}^3$: Mean position.
*   $q_i \in \mathbb{R}^4$: Quaternion rotation.
*   $s_i \in \mathbb{R}^3$: Scale.
*   $o_i \in \mathbb{R}$: Opacity.
*   $n_i \in \mathbb{R}^3$: Normal attribute (unit vector).
*   $c_i$: Spherical Harmonic (SH) coefficients for view-dependent color.

**Numbered Steps:**
1.  **Render Depth and Normal Maps:** For a specified viewpoint, render the depth map $D$ and normal map $N$ using the current Gaussian set. The rendering follows the volume rendering equation [S1]:
    $$ \{D, N\} = \sum_{i \in \mathcal{N}} T_i \alpha_i \{d_i, n_i\} $$
    where $\mathcal{N}$ is the set of sampled Gaussians, $T_i$ is the accumulated transmittance, $\alpha_i$ is the opacity, $d_i$ is the depth of the $i$-th Gaussian, and $n_i$ is the normal of the $i$-th Gaussian.
2.  **Compute Pseudo Normal:** Calculate a pseudo normal $\tilde{N}$ from the rendered depth map $D$ under the local planarity assumption [S1].
3.  **Compute Losses:** Calculate the total loss as a combination of:
    *   **Color and Alpha Loss:** L1 and SSIM losses for the base 3DGS output color and alpha channels [S1].
    *   **Normal Consistency Loss ($L_n$):**
        $$ L_n = ||N - \tilde{N}||_2 $$
        where $N$ is the rendered normal map and $\tilde{N}$ is the pseudo normal [S1].
4.  **Update Parameters:** Perform gradient descent to minimize the total loss. Note that optimizing $n_i$ does not affect the rendering results of the base 3DGS model itself; it is optimized solely to accelerate the convergence of subsequent editable Gaussian optimization [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def base_3dgs_optimization_step(gaussians, camera, target_image, target_depth_map):
    """
    One step of Base 3DGS optimization.
    
    Args:
        gaussians: List of Gaussian objects with attributes {mu, q, s, o, n, c}
        camera: Camera object with projection and view matrices
        target_image: Ground truth RGB image
        target_depth_map: Ground truth depth map (or derived pseudo-normal source)
        
    Returns:
        loss: Scalar loss value
    """
    # 1. Render Depth and Normal Maps
    # Rendered depth D and normal N are computed via volume rendering
    # D = sum(T_i * alpha_i * d_i)
    # N = sum(T_i * alpha_i * n_i)
    rendered_depth, rendered_normal = render_volume(gaussians, camera)
    
    # 2. Compute Pseudo Normal from Depth
    # Under local planarity assumption
    pseudo_normal = compute_pseudo_normal_from_depth(rendered_depth)
    
    # 3. Compute Losses
    # Color and Alpha Loss (L1 + SSIM)
    rendered_color, rendered_alpha = render_color_alpha(gaussians, camera)
    color_loss = l1_loss(rendered_color, target_image) + ssim_loss(rendered_color, target_image)
    alpha_loss = l1_loss(rendered_alpha, target_alpha) # Assuming target_alpha is available or derived
    
    # Normal Consistency Loss
    # L_n = ||N - N_tilde||_2
    normal_loss = l2_loss(rendered_normal, pseudo_normal)
    
    # Total Loss
    total_loss = color_loss + alpha_loss + normal_loss
    
    # 4. Backpropagate and Update
    # Note: Optimizing 'n' does not affect rendering output, only convergence speed
    total_loss.backward()
    update_gaussians(gaussians)
    
    return total_loss
```

**Parameters:**
*   **Loss Weights:** Not given in the sources.
*   **SH Degree:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Normal Attribute Impact:** Adjusting the normal attribute $n$ does not affect the rendering results of base 3DGS. It is optimized only to accelerate convergence of editable Gaussian optimization [S1].
*   **Alpha Channel Optimization:** Unlike IBR methods, 3DGS reconstructs per-pixel alpha channel information. Optimizing the alpha channel enhances reconstruction quality [S1].

## Subtopics

### Editable Gaussian Optimization
Editable Gaussian Optimization decomposes color and lighting from the scene using the Blinn-Phong model instead of SH coefficients, enabling scene editing [S1]. It differs from Base 3DGS by introducing additional shading attributes ($\Delta c, k_a, k_d, k_s, \beta$) and a shared palette color $c_p$ [S1]. Choose this variant when scene editing capabilities are required. It builds on [Base 3DGS Optimization](7549-base-3dgs-optimization.md) by using its initialized geometry [S1].
*   **Definition:** Optimization that decomposes color and lighting from the scene using the Blinn-Phong model instead of SH coefficients, enabling scene editing [S1].
*   **Steps:** Remove low-opacity Gaussians periodically to reduce model size [S1]. Encourage the opacity of Gaussians to be as low as possible to neutralize the tendency to use more Gaussians for compact surfaces due to normal-dependent color [S1].
*   **Parameters:** Voxel color $c_v$ is expressed as $c_p + \Delta c$, where $c_p \in \mathbb{R}^3$ is a palette color parameter shared by all Gaussians within a basic scene [S1]. $c_p$ is fixed as the mean color value of input multi-view images during training and can be adjusted during inference [S1].
*   **Layout:** Editable Gaussians have additional shading attributes: offset color $\Delta c \in \mathbb{R}^3$, ambient/diffuse/specular coefficients $\{k_a, k_d, k_s\}$, and shininess term $\beta$ [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Accelerates convergence of editable Gaussian optimization by providing a robust geometric baseline [S1].
*   **Limitations:** The normal attribute optimization does not directly improve the rendering quality of the base model itself [S1].
*   **When to Use:** When subsequent editable Gaussian optimization is planned, as it provides a necessary initialization for shading attributes [S1].
*   **When Not to Use:** When only static rendering is required, as the normal attribute optimization adds computational overhead without improving visual output [S1].

## Background
Base 3DGS Optimization was introduced in the iVR-GS paper as a preliminary step for inverse volume rendering with editable 3D Gaussian Splatting [S1]. It adapts the original 3DGS framework by adding a normal attribute and a normal consistency loss to ensure geometric accuracy before decomposing lighting and color [S1].

## Sources
[S1] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
