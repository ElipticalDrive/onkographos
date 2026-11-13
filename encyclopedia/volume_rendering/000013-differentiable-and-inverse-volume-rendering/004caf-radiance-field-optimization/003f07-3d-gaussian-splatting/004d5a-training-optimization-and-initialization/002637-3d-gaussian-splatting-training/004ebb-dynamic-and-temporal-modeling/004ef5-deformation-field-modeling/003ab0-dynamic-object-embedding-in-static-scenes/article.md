# Dynamic Object Embedding in Static Scenes

Also called: Lightweight Dynamic NeRF Modeling via Local Deformation Field, Local deformation field model

## Overview
This method integrates dynamic advertising objects into static Neural Radiance Field (NeRF) scenes by ensuring perspective consistency through spatial alignment and joint rendering with alpha blending [S1]. It addresses the high computational cost of traditional dynamic NeRFs (e.g., DyNeRF), which require over 100M parameters and independent models per time step, by using a static NeRF for static areas and a lightweight deformation field for dynamic regions [S1].

## How it works
The method decomposes the scene into static and dynamic components. The static background is modeled by a fixed NeRF, while dynamic objects are represented by a local deformation field that warps coordinates into the static scene's space [S1].

**Inputs and Outputs:**
*   **Inputs:** Sparse video frames of the static scene (exhibition stand + background) and sparse video frames of the dynamic object (e.g., a rotating watch) [S1].
*   **Outputs:** A rendered frame at time $t$ combining the static background and the dynamically embedded object [S1].

**Data Structures:**
1.  **Static NeRF Model:** A standard NeRF representation with fixed parameters. It maps 3D coordinates $(x, y, z)$ and viewing direction $(d_x, d_y, d_z)$ to density $\sigma$ and color $c$. It contains no time dimension [S1].
2.  **Local Deformation Field:** A 3-layer Multi-Layer Perceptron (MLP) [S1].
    *   **Input:** Spatial coordinates $\mathbf{x} \in \mathbb{R}^3$ and time $t \in \mathbb{R}$.
    *   **Hidden Layers:** 2 hidden layers, each with dimension 128 [S1].
    *   **Output:** Displacement vector $\Delta \mathbf{x} \in \mathbb{R}^3$.
    *   **Parameters:** Approximately 5 million parameters [S1].

**Numbered Steps:**

1.  **Static Scene Reconstruction:** Generate a static NeRF model for the advertising scene (exhibition stand + background) using sparse input reconstruction. The model parameters are fixed and do not include a time dimension [S1].
2.  **Dynamic Object Modeling:** Collect 25–30 frames of video for the dynamic advertising object. Train a local deformation field model to learn the motion displacement of the object. This model is lightweight, using only 5M parameters [S1].
3.  **Spatial Alignment:** Align the coordinate system of the dynamic object with the static scene to ensure perspective consistency.
    *   Select four marking points (e.g., exhibition stand corners) in both the static scene and the dynamic object [S1].
    *   Use the RANSAC algorithm to compute the transformation that aligns the dynamic object's coordinates with the static scene's coordinates.
    *   Target alignment error: $< 1$ pixel [S1].
4.  **Joint Rendering:** For a given time $t$ and camera ray:
    *   Calculate the deformation position of the dynamic object using the local deformation field $\Delta(\mathbf{x}, t)$ [S1].
    *   The deformed coordinate is $\mathbf{x}' = \mathbf{x} + \Delta(\mathbf{x}, t)$ [S1].
    *   Input $\mathbf{x}'$ into the static NeRF model to obtain color and density [S1].
    *   Fuse the dynamic object with the static scene using alpha blending with a blending factor of 0.95 [S1].

**Formulas:**
The dynamic radiation field $F(\mathbf{x}, t)$ is expressed as [S1]:
$$ F(\mathbf{x}, t) = F_{static}(\mathbf{x} + \Delta(\mathbf{x}, t)) $$
Where:
*   $F_{static}$ is the static NeRF function.
*   $\mathbf{x}$ is the spatial coordinate.
*   $\Delta(\mathbf{x}, t)$ is the displacement predicted by the deformation field.
*   $t$ is the time index.

**Parameters:**
*   **Deformation Field Architecture:** 3-layer MLP, hidden layer dimension 128 [S1].
*   **Deformation Field Parameters:** 5M [S1].
*   **Alpha Blending Factor:** 0.95 [S1].
*   **Sparse Input Frames for Static Scene:** Not given in the sources.
*   **Sparse Input Frames for Dynamic Object:** 25–30 frames [S1].

**Edge Cases and Pitfalls:**
*   **Alignment Error:** If the spatial alignment error exceeds 1 pixel, perspective consistency is lost, leading to visual artifacts [S1].
*   **Dynamic Region Coverage:** The method is designed for dynamic regions comprising $\le 20\%$ of the scene. If the dynamic region is larger, the assumption of a static background may fail [S1].
*   **Temporal Consistency:** Without temporal consistency loss, visual discontinuities (color jumps, sudden light/shadow changes) may occur during dynamic processes [S1]. The method introduces temporal consistency loss to ensure smooth transitions (SSIM $\ge 0.92$) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class StaticNeRF:
    """
    Represents the static background NeRF model.
    Initialized from sparse input reconstruction.
    """
    def __init__(self):
        # Parameters are fixed, no time dimension
        pass

    def query(self, x, d):
        """
        Query density and color at 3D point x with viewing direction d.
        Returns: sigma (density), c (color)
        """
        # Implementation details of the static NeRF are not provided in [S1].
        # This function queries the pre-trained static NeRF.
        return sigma, c

class DeformationField:
    """
    Lightweight 3-layer MLP for dynamic object displacement.
    Input: x (3D), t (time)
    Output: delta_x (3D displacement)
    Parameters: ~5M
    """
    def __init__(self):
        # 3-layer MLP with hidden dimension 128
        # Weights and biases initialized from training on dynamic object video
        pass

    def forward(self, x, t):
        """
        Calculate displacement delta_x for point x at time t.
        """
        # Implementation of the 3-layer MLP
        # Input: [x, t]
        # Hidden 1: ReLU(Linear([x, t], 128))
        # Hidden 2: ReLU(Linear(Hidden 1, 128))
        # Output: Linear(Hidden 2, 3)
        return delta_x

def spatial_align_static_and_dynamic(static_points, dynamic_points):
    """
    Align dynamic object coordinates to static scene coordinates.
    Uses RANSAC with 4 marking points (e.g., corners).
    """
    # Select 4 marking points in static scene and corresponding points in dynamic object
    # Use RANSAC to find transformation T
    # Ensure alignment error < 1 pixel
    return T

def render_frame(static_nerf, deformation_field, T, camera_params, t, alpha_blend_factor=0.95):
    """
    Render a single frame at time t.
    """
    # 1. Get ray origins and directions from camera_params
    # 2. For each pixel/ray:
    #    a. Intersect ray with dynamic object bounding box or mask
    #    b. If intersection exists at point x_dynamic:
    #       i. Transform to static coordinate system: x_static = T * x_dynamic
    #       ii. Calculate deformation: delta_x = deformation_field.forward(x_static, t)
    #       iii. Deformed coordinate: x_prime = x_static + delta_x
    #       iv. Query static NeRF: sigma, c = static_nerf.query(x_prime, direction)
    #       v. Alpha blend with background (or previous layers)
    #          color = alpha_blend_factor * c + (1 - alpha_blend_factor) * background_color
    #    c. Else:
    #       i. Query static NeRF directly: sigma, c = static_nerf.query(x, direction)
    #       ii. color = c
    # 3. Return rendered image
    pass

# Example usage
# static_nerf = StaticNeRF()
# static_nerf.load('static_scene_model.pt')
# deformation_field = DeformationField()
# deformation_field.load('dynamic_object_deformation.pt')
# T = spatial_align_static_and_dynamic(static_corners, dynamic_corners)
# image = render_frame(static_nerf, deformation_field, T, camera, t=0.5)
```

## Operating characteristics
*   **Parameter Count:** 5M parameters for the deformation field, which is 95% less than DyNeRF's 100M+ parameters [S1].
*   **Performance:** PSNR increased by $2.3 \pm 0.2$ dB under sparse input conditions (mean $\pm$ std over 10 trials) [S1].
*   **Temporal Consistency:** SSIM $\ge 0.92$ with temporal consistency loss [S1].
*   **Suitability:** Suitable for VR devices due to lower parameter count and computational cost compared to traditional dynamic NeRFs [S1].
*   **Limitations:** Designed for dynamic regions comprising $\le 20\%$ of the scene [S1]. Requires sparse input reconstruction for both static and dynamic components [S1].

## Background
This method was proposed in the paper "Efficient Sparse Input Scene Reconstruction and Real-Time Rendering for VR Advertising Using Optimized NeRF Framework" [S1]. It addresses the limitations of traditional dynamic NeRFs like DyNeRF, which require independent models for each time step and have high parameter counts, making them unsuitable for VR devices [S1]. The approach is specifically tailored for advertising scenarios where dynamic objects (e.g., rotating watches) are embedded in static exhibition stands [S1].

## Sources
[S1] Efficient Sparse Input Scene Reconstruction and Real-Time Rendering for VR Advertising Using Optimized NeRF Framework, https://www.informatica.si/index.php/informatica/article/download/12804/6562, fetched 2026-10-29
