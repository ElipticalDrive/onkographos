# Inverse Volume Exploration

Also called: Automatic Approximation, Reference Image Optimization

## Overview
Inverse Volume Exploration is a technique that approximates a desired rendering setting (viewpoint, color, opacity, light direction/magnitude) from a single reference image by optimizing transformation parameters of a pre-trained composed iVR-GS model [S1]. It solves the problem of tedious trial-and-error transfer function adjustments by allowing the model to automatically learn the parameters required to replicate a user-provided visual effect [S1]. Unlike base 3D Gaussian Splatting which bakes the scene appearance, this method enables editing of lighting and appearance via optimization, achieving perceptual fidelity even from poor viewpoint reference images [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A composed iVR-GS model (formed by composing multiple basic iVR-GS models), a target reference image, and its corresponding camera pose [S1].
*   **Output:** A transformed iVR-GS model with optimized transformation parameters for color, opacity, and lighting, capable of replicating the reference image and generating novel view synthesis (NVS) results under those learned settings [S1].

**Data Structures:**
The method operates on a composed iVR-GS model consisting of multiple basic iVR-GS models. Each basic model contains Gaussian primitives with editable attributes. The optimization targets specific transformation parameters rather than the primitive attributes themselves [S1].

**Parameters:**
*   Learning rate: 0.01 [S1].
*   Optimization iterations: 1,000 iterations on the reference image until convergence [S1].

**Steps:**
1.  **Composition:** Compose all related basic iVR-GS models into a single composed model [S1].
2.  **Reference Generation:** Render a target reference image using settings (viewpoint, color, opacity, light direction/magnitude) that differ from the training images used to create the basic models [S1]. Ensure that the visible voxel value ranges in the reference image are covered by the composed model [S1].
3.  **Freezing:** Freeze all Gaussian primitive attribute values (including position, covariance, etc.) of all editable Gaussian attributes [S1].
4.  **Optimization:** Optimize multiple transformation parameters to modify the scene's color, opacity, and light settings using the reference image as the target [S1]. The optimization uses a combination of L1 and SSIM loss [S1].
    *   **Color:** Make the global parameter $c_p$ learnable for each basic iVR-GS model [S1].
    *   **Opacity:** Multiply and optimize a scaling factor for each basic iVR-GS model to control their contribution to the VolVis scene [S1].
    *   **Lighting (Magnitude):** Optimize lighting globally for the composed model using two learnable parameter vectors $\lambda \in \mathbb{R}^{4 \times 1}$ and $b \in \mathbb{R}^{4 \times 1}$ applied to the basic iVR-GS attributes $\{k_a, k_d, k_s, \beta\}$ (ambient, diffuse, specular coefficients, and exponent) via the linear transformation:
        $$ \{k_a', k_d', k_s', \beta'\} = \lambda \{k_a, k_d, k_s, \beta\} + b $$
        [S1].
    *   **Lighting (Direction):** If the reference image employs an orbital light, optimize the light direction using two learnable parameters denoting polar and azimuthal light angles in the spherical coordinate system [S1].
5.  **Evaluation:** Render 181 images from the volume under novel viewpoints using the learned settings to evaluate reconstruction quality [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_volume_exploration(composed_model, reference_image, camera_pose):
    """
    Optimizes transformation parameters of a composed iVR-GS model to match a reference image.
    
    Args:
        composed_model: A composed iVR-GS model containing multiple basic iVR-GS models.
        reference_image: The target 2D image to approximate.
        camera_pose: The camera pose corresponding to the reference image.
        
    Returns:
        transformed_model: The composed model with optimized transformation parameters.
    """
    # Step 1: Ensure composed model covers visible voxel ranges of reference
    # (Implicit in setup; no specific code provided in source)
    
    # Step 2: Freeze all Gaussian primitive attributes
    for basic_model in composed_model.basic_models:
        basic_model.freeze_primitive_attributes()
        
    # Step 3: Initialize learnable transformation parameters
    # Color: Global parameter cp for each basic model
    for basic_model in composed_model.basic_models:
        basic_model.cp = Parameter(torch.zeros_like(basic_model.cp), requires_grad=True)
        
    # Opacity: Scaling factor for each basic model
    for basic_model in composed_model.basic_models:
        basic_model.opacity_scale = Parameter(torch.ones(1), requires_grad=True)
        
    # Lighting Magnitude: Global vectors lambda and b for the composed model
    # Attributes: {ka, kd, ks, beta}
    composed_model.lambda_vec = Parameter(torch.zeros(4, 1), requires_grad=True)
    composed_model.bias_vec = Parameter(torch.zeros(4, 1), requires_grad=True)
    
    # Lighting Direction: If orbital light is used
    if reference_image.requires_orbital_light_optimization:
        composed_model.light_polar_angle = Parameter(torch.tensor(0.0), requires_grad=True)
        composed_model.light_azimuthal_angle = Parameter(torch.tensor(0.0), requires_grad=True)
        
    # Step 4: Optimization Loop
    optimizer = torch.optim.Adam([
        *composed_model.get_color_params(),
        *composed_model.get_opacity_params(),
        composed_model.lambda_vec,
        composed_model.bias_vec,
        composed_model.light_polar_angle,
        composed_model.light_azimuthal_angle
    ], lr=0.01)
    
    for iteration in range(1000):
        optimizer.zero_grad()
        
        # Render current view
        rendered_image = composed_model.render(camera_pose)
        
        # Compute Loss: L1 + SSIM
        l1_loss = L1Loss(rendered_image, reference_image)
        ssim_loss = SSIMLoss(rendered_image, reference_image)
        loss = l1_loss + ssim_loss
        
        loss.backward()
        optimizer.step()
        
    # Step 5: Apply optimized parameters to the model
    # The parameters are already integrated into the model's forward pass
    # via the learnable attributes defined above.
    
    return composed_model
```

**Edge Cases and Pitfalls:**
*   **Voxel Range Coverage:** The visible voxel value ranges of the reference image must be covered by the composed model; otherwise, the approximation may fail [S1].
*   **Poor Viewpoints:** The method can achieve perceptual fidelity even when the reference image is associated with a poor viewpoint, though reconstruction quality may vary [S1].
*   **Orbital Light:** Optimization of light direction is conditional; it is only performed if the reference image employs an orbital light [S1].

## Subtopics

### Basic iVR-GS Model
The fundamental unit of the composed model, containing Gaussian primitives with editable transfer function attributes. It serves as the base for composing complex scenes. [Link](6685-basic-ivrgs-model.md)

### Composed iVR-GS Model
A structure formed by combining multiple basic iVR-GS models to represent complex volumetric data with multiple transfer functions. It allows for global lighting optimization. [Link](6684-composed-ivrgs-model.md)

## Operating characteristics
*   **Time Complexity:** Optimization time scales with the total number of primitives. For the tested datasets, optimization took between 48 and 122 seconds [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   **Combustion Dataset:** 5,432,519 primitives, PSNR 27.62 dB, LPIPS 0.033, Optimization time 88 s [S1].
    *   **Five-Jet Dataset:** 1,484,642 primitives, PSNR 25.86 dB, LPIPS 0.046, Optimization time 48 s [S1].
    *   **Vortex Dataset:** 7,104,441 primitives, PSNR 30.49 dB, LPIPS 0.031, Optimization time 122 s [S1].
*   **Strengths:** Eliminates tedious trial-and-error transfer function adjustments; allows automatic approximation of intended rendering settings; achieves perceptual fidelity even from poor viewpoint reference images; enables novel view synthesis under learned settings without access to original volume data [S1].
*   **Limitations:** Requires a composed iVR-GS model; optimization time increases with the number of primitives; relies on the composed model covering the visible voxel ranges of the reference image [S1].
*   **When to Use:** When a user wants to approximate a desired rendering effect from a single reference image without manually tuning transfer functions.
*   **When Not to Use:** When original volume data is available and direct editing is preferred; when real-time interactive editing is required without pre-computation.

## Background
Inverse Volume Exploration was introduced in the iVR-GS (Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting) paper [S1]. It is part of the iVR-GS framework, which extends 3D Gaussian Splatting for volume visualization by allowing editable transfer functions. The method was developed to address the inefficiency of manual scene editing in volumetric visualization, providing a way to automatically learn rendering parameters from reference images [S1].

## Sources
[S1] iVR-GS: Inverse Volume Rendering for Explorable Visualization via Editable 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2025.3567121, fetched 2026-10-26
