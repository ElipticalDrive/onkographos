# Dynamic Object Embedding in Static Scenes

Also called: Lightweight Dynamic NeRF Modeling via Local Deformation Field, Local deformation field model

## Overview
This method addresses the problem of integrating dynamic advertising objects into static NeRF scenes while maintaining perspective consistency and computational efficiency. It differs from traditional dynamic NeRF approaches (e.g., DyNeRF) by decoupling the scene into a static background and a localized dynamic region, using a lightweight deformation field to warp coordinates rather than training independent models for each time step [S1]. This reduces parameter count by 95% compared to full-scene dynamic models, making it suitable for VR devices [S1].

## How it works
The method integrates dynamic elements into a static radiance field through four main stages: static scene reconstruction, dynamic object modeling, spatial alignment, and joint rendering.

### Inputs and Outputs
*   **Inputs:**
    *   Sparse video frames of the static scene (exhibition stand + background).
    *   Sparse video frames of the dynamic object (e.g., a rotating watch, 25–30 frames) [S1].
    *   Four corresponding marking points (e.g., stand corners) in both static and dynamic data for alignment [S1].
*   **Outputs:**
    *   A static NeRF model for the background.
    *   A local deformation field $\Delta(x, t)$ for the dynamic object.
    *   Final rendered frames with alpha-blended dynamic objects [S1].

### Data Structures
1.  **Static NeRF Model:** A standard NeRF representation with fixed parameters and no time dimension. It maps 3D coordinates $(x, y, z)$ and viewing direction $(d_x, d_y, d_z)$ to density $\sigma$ and color $c$ [S1].
2.  **Deformation Field $\Delta(x, t)$:** A 3-layer Multi-Layer Perceptron (MLP) that predicts displacement.
    *   **Architecture:** 3 layers.
    *   **Hidden Dimension:** 128.
    *   **Input:** Spatial coordinates $x$ and time $t$.
    *   **Output:** Displacement vector $\Delta x$.
    *   **Parameter Count:** 5 million parameters [S1].

### Steps
1.  **Static Scene Reconstruction:** Generate a static NeRF model for the advertising scene using sparse input reconstruction. The model parameters are fixed, and no time dimension calculation is required [S1].
2.  **Dynamic Object Modeling:** Train a local deformation field model on the dynamic object's sparse video frames (25–30 frames). This models the motion displacement of the object [S1].
3.  **Spatial Alignment:** Align the coordinate system of the dynamic object with the static scene to ensure perspective consistency.
    *   Select four marking points in both the static scene and the dynamic object.
    *   Use the RANSAC algorithm to compute the transformation [S1].
    *   Target alignment error: <1 pixel [S1].
4.  **Joint Rendering:** For a given time $t$ and camera ray:
    *   Calculate the deformation position of the dynamic object using $\Delta(x, t)$.
    *   Compute the deformed coordinates: $x' = x + \Delta(x, t)$.
    *   Query the static NeRF at $x'$ to obtain color and density.
    *   Fuse the dynamic object with the static scene using alpha blending with a blending factor of 0.95 [S1].

### Formulas
The dynamic radiation field $F(x, t)$ is expressed as [S1]:
$$ F(x, t) = F_{static}(x + \Delta(x, t)) $$
Where:
*   $x$ is the spatial coordinate.
*   $t$ is the time.
*   $\Delta(x, t)$ is the displacement predicted by the deformation field.
*   $F_{static}$ is the static NeRF function.

### Parameters
*   **Deformation Field MLP:** 3 layers, hidden dimension 128, 5M parameters [S1].
*   **Alpha Blending Factor:** 0.95 [S1].
*   **Temporal Consistency Loss:** Used to ensure smooth dynamic transitions, targeting SSIM $\ge$ 0.92 [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_frame(static_nerf, deformation_mlp, time_t, camera_ray):
    """
    Renders a frame by embedding a dynamic object into a static scene.
    
    Args:
        static_nerf: The pre-trained static NeRF model.
        deformation_mlp: The 3-layer MLP (5M params) predicting displacement.
        time_t: The current time step.
        camera_ray: The ray origin and direction.
        
    Returns:
        final_color: The blended color of the dynamic object and static scene.
    """
    # 1. Sample points along the ray for the dynamic object region
    # Note: Specific ray marching logic for the dynamic region is not detailed in [S1],
    # but typically involves sampling points x along the ray within the object's bounding volume.
    
    dynamic_color = 0.0
    dynamic_accumulated_alpha = 0.0
    
    for point_x in dynamic_region_samples:
        # 2. Calculate deformation using the local deformation field
        # Input: spatial coordinates x and time t
        displacement = deformation_mlp.forward(point_x, time_t)
        
        # 3. Compute deformed coordinates
        deformed_x = point_x + displacement
        
        # 4. Query static NeRF at deformed coordinates
        # Output: density sigma and color c
        sigma, color = static_nerf.forward(deformed_x, camera_ray.direction)
        
        # 5. Alpha blending accumulation
        # Standard volume rendering accumulation
        alpha = 1.0 - torch.exp(-sigma * delta_depth)
        weight = alpha * (1.0 - dynamic_accumulated_alpha)
        dynamic_color += weight * color
        dynamic_accumulated_alpha += weight
        
    # 6. Fuse with static scene using alpha blending
    # The static scene is rendered separately or queried for the background
    static_color = static_nerf.render_background(camera_ray)
    
    # Blending factor specified as 0.95 in [S1]
    blending_factor = 0.95
    final_color = blending_factor * dynamic_color + (1.0 - blending_factor) * static_color
    
    return final_color
```

### Edge Cases and Pitfalls
*   **Alignment Error:** If the RANSAC alignment error exceeds 1 pixel, perspective consistency is lost, leading to visual artifacts [S1].
*   **Sparse Input:** The method relies on sparse video frames (20–30 frames). Insufficient frames may lead to poor deformation field training [S1].
*   **Dynamic Region Size:** The method is designed for dynamic regions comprising $\le$20% of the scene. Larger dynamic areas may require more complex modeling [S1].

## Operating characteristics
*   **Parameter Efficiency:** Uses 5M parameters, which is 95% less than DyNeRF's 100M+ parameters [S1].
*   **Performance:** PSNR increased by $2.3 \pm 0.2$ dB under sparse input conditions (mean $\pm$ std over 10 trials) [S1].
*   **Temporal Consistency:** Achieves SSIM $\ge$ 0.92 with temporal consistency loss [S1].
*   **Suitability:** Designed for VR devices due to lower computational and memory requirements compared to full-scene dynamic NeRFs [S1].
*   **Limitations:** Limited to scenes where the dynamic region is small ($\le$20%) and can be effectively modeled by a local deformation field [S1].

## Background
This method was proposed in the context of VR advertising, aiming to integrate dynamic advertising objects into static NeRF scenes efficiently. It addresses the limitations of traditional dynamic NeRFs like DyNeRF, which require independent models per time step and excessive parameters [S1]. The approach was developed to support real-time rendering on VR devices using consumer-grade devices for data acquisition [S1].

## Sources
[S1] Efficient Sparse Input Scene Reconstruction and Real-Time Rendering for VR Advertising Using Optimized NeRF Framework, https://www.informatica.si/index.php/informatica/article/download/12804/6562, fetched 2026-10-29
