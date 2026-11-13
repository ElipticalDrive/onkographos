# Dynamic Object Embedding in Static Scenes

Also called: Lightweight Dynamic NeRF Modeling via Local Deformation Field, Local deformation field model

## Overview
This method integrates dynamic advertising objects into static NeRF scenes by ensuring perspective consistency through spatial alignment and joint rendering with alpha blending [S1]. It addresses the high computational cost of traditional dynamic NeRFs (e.g., DyNeRF), which require over 100M parameters and independent models per time step, by using a static NeRF for static areas and a lightweight deformation field for dynamic regions [S1]. Unlike siblings like [Dynamic 3D Gaussian Splatting (Katsumata et al.)](9168-dynamic-3d-gaussian-splatting-katsumata-et-al.md) which model dynamics via Gaussian attributes, this approach warps coordinates into a static radiance field [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse video frames of the static scene (exhibition stand + background), sparse video frames of the dynamic object (e.g., rotating watch), and four corresponding marking points (e.g., stand corners) in both domains for alignment [S1].
*   **Outputs:** A rendered frame at time $t$ combining the static background and the dynamically deformed object [S1].

**Data Structures:**
1.  **Static NeRF Model:** A standard NeRF representation for the static scene. It has fixed parameters and no time dimension [S1].
2.  **Deformation Field ($\Delta$):** A 3-layer Multi-Layer Perceptron (MLP) with a hidden layer dimension of 128. It takes spatial coordinates $x$ and time $t$ as input and outputs a displacement vector $\Delta(x, t)$ [S1].
    *   **Size:** 5 million parameters [S1].
    *   **Note:** This is 95% fewer parameters than DyNeRF [S1].

**Numbered Steps:**

1.  **Static Scene Reconstruction:** Generate a static NeRF model for the advertising scene using sparse input reconstruction. The model parameters are fixed and do not include a time dimension [S1].
2.  **Dynamic Object Modeling:** Collect 25–30 frames of video for the dynamic object. Train the local deformation field model on this data to model motion displacement [S1].
3.  **Spatial Alignment:**
    *   Select four marking points (e.g., exhibition stand corners) in both the static scene and the dynamic object [S1].
    *   Use the RANSAC algorithm to align the coordinate system of the dynamic object with the static scene [S1].
    *   **Constraint:** Ensure perspective consistency with an alignment error of less than 1 pixel [S1].
4.  **Joint Rendering (at time $t$):**
    *   Calculate the deformation position of the dynamic object using the local deformation field: $\Delta(x, t)$ [S1].
    *   Compute the deformed coordinates: $x' = x + \Delta(x, t)$ [S1].
    *   Query the static NeRF model at $x'$ to obtain color and density: $[\sigma, c] = \text{StaticNeRF}(x')$ [S1].
    *   Fuse the dynamic object with the static scene using alpha blending with a blending factor of 0.95 [S1].
    *   Apply temporal consistency loss to ensure smooth dynamic transitions (target SSIM $\ge$ 0.92) [S1].

**Formulas:**
The dynamic radiation field $F(x, t)$ is expressed as [S1]:
$$ F(x, t) = F_{static}(x + \Delta(x, t)) $$
Where:
*   $F_{static}$ is the static NeRF function.
*   $x$ is the spatial coordinate.
*   $\Delta(x, t)$ is the displacement predicted by the deformation field MLP.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_dynamic_object(static_nerf, deformation_mlp, time_t, object_points):
    """
    Renders a dynamic object embedded in a static scene.
    
    Args:
        static_nerf: The pre-trained static NeRF model.
        deformation_mlp: The 3-layer MLP (hidden dim 128) predicting displacement.
        time_t: The current time step.
        object_points: List of 3D coordinates (x, y, z) of the dynamic object.
        
    Returns:
        Rendered image or color/density values for the object.
    """
    deformed_points = []
    
    for point in object_points:
        # Step 1: Calculate displacement using the deformation field
        # Input: spatial coordinates x and time t
        displacement = deformation_mlp.forward(point, time_t)
        
        # Step 2: Compute deformed coordinates
        # x' = x + Delta(x, t)
        deformed_point = point + displacement
        deformed_points.append(deformed_point)
        
    # Step 3: Query static NeRF at deformed coordinates
    # Obtain color and density from the static scene representation
    # Note: In a full renderer, this involves ray marching or splatting
    # Here we assume direct query for the object's contribution
    colors, densities = static_nerf.query(deformed_points)
    
    # Step 4: Fuse with static scene using alpha blending
    # Blending factor is 0.95
    # Note: The exact blending logic depends on the compositing order.
    # Assuming the dynamic object is composited over the static background.
    # If 'static_color' is the background color at the same ray intersection:
    # final_color = alpha * dynamic_color + (1 - alpha) * static_color
    # The source specifies a blending factor of 0.95, likely applied to the 
    # dynamic object's contribution or as a weight in the fusion.
    # [G] Assuming standard alpha compositing where 0.95 is the opacity/weight 
    # of the dynamic layer relative to the static layer for visual consistency.
    
    return colors, densities
```

**Edge Cases and Pitfalls:**
*   **Alignment Error:** If the RANSAC alignment error exceeds 1 pixel, perspective consistency is lost [S1].
*   **Sparse Input:** The method relies on sparse input reconstruction; insufficient frames may degrade the static NeRF or deformation field quality [S1].
*   **Dynamic Region Size:** The method is designed for dynamic regions comprising $\le$ 20% of the scene [S1].

## Operating characteristics
*   **Parameter Count:** 5 million parameters for the deformation field, which is 95% less than DyNeRF's 100M+ parameters [S1].
*   **Performance:** PSNR increased by $2.3 \pm 0.2$ dB under sparse input conditions (mean $\pm$ std over 10 trials) [S1].
*   **Temporal Consistency:** Achieves SSIM $\ge$ 0.92 with temporal consistency loss [S1].
*   **Suitability:** Designed for VR devices where traditional dynamic NeRFs are too heavy [S1].
*   **Limitations:** Only models the dynamic region (up to 20% of the scene); static areas are handled by a separate static NeRF [S1].

## Background
This method was proposed to address the limitations of traditional dynamic NeRFs like DyNeRF, which require independent models for each time step and over 100M parameters, making them unsuitable for VR devices [S1]. It introduces a local deformation field model that only models the dynamic region, allowing for efficient integration of dynamic advertising objects into static scenes [S1].

## Sources
[S1] Efficient Sparse Input Scene Reconstruction and Real-Time Rendering for VR Advertising Using Optimized NeRF Framework, https://www.informatica.si/index.php/informatica/article/download/12804/6562, fetched 2026-10-29
