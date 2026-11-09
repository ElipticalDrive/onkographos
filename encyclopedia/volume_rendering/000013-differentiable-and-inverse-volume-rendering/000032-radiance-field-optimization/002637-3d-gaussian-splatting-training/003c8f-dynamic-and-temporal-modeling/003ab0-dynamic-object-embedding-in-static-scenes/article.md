# Dynamic Object Embedding in Static Scenes

Also called: Lightweight Dynamic NeRF Modeling via Local Deformation Field, Local deformation field model

## Overview
This method integrates dynamic advertising objects into static Neural Radiance Field (NeRF) scenes by ensuring perspective consistency through spatial alignment and joint rendering with alpha blending [S1]. It addresses the computational inefficiency of traditional dynamic NeRFs (e.g., DyNeRF), which require over 100M parameters and independent models per time step, by using a static NeRF for fixed areas and a lightweight deformation field for dynamic regions [S1]. Unlike siblings such as [Neural Transient Field (NeTF)](4692-neural-transient-field-netf.md) or [D-NeRF](8660-d-nerf.md) that model the entire scene's temporal evolution, this approach decomposes the scene, modeling only the dynamic region (typically ≤20% of the scene) with a 5M parameter model [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse video frames of the static scene (exhibition stand + background), sparse video frames of the dynamic object (e.g., rotating watch), and four corresponding marking points (e.g., stand corners) in both domains for alignment.
*   **Outputs:** A static NeRF model, a local deformation field model, and rendered frames for any time $t$.

**Data Structures:**
1.  **Static NeRF:** A standard MLP-based NeRF with fixed parameters and no time dimension. It maps 3D coordinates $\mathbf{x}$ and viewing direction $\mathbf{d}$ to color $\mathbf{c}$ and density $\sigma$.
2.  **Deformation Field:** A 3-layer Multi-Layer Perceptron (MLP).
    *   Input: Spatial coordinate $\mathbf{x} \in \mathbb{R}^3$ and time $t \in \mathbb{R}$.
    *   Hidden Layers: 2 hidden layers, each with dimension 128.
    *   Output: Displacement vector $\Delta(\mathbf{x}, t) \in \mathbb{R}^3$.
    *   Parameters: Approximately 5 million parameters [S1].

**Numbered Steps:**

1.  **Static Scene Reconstruction:** Generate a static NeRF model for the advertising scene using sparse input reconstruction. The model parameters are fixed and do not include a time dimension [S1].
2.  **Dynamic Object Modeling:** Collect 25–30 frames of video for the dynamic object using consumer-grade devices. Train a local deformation field model to learn the motion displacement of the object. This model is trained independently of the static scene [S1].
3.  **Spatial Alignment:** Select four marking points (e.g., corners of the exhibition stand) visible in both the static scene and the dynamic object's video. Use the RANSAC algorithm to align the coordinate system of the dynamic object with the static scene. The goal is to ensure perspective consistency with an alignment error of less than 1 pixel [S1].
4.  **Joint Rendering:** For a given time $t$ and ray origin/direction:
    *   If the ray intersects the static region, query the static NeRF directly.
    *   If the ray intersects the dynamic region, calculate the deformed position $\mathbf{x}' = \mathbf{x} + \Delta(\mathbf{x}, t)$ using the deformation field.
    *   Query the static NeRF at $\mathbf{x}'$ to obtain color and density.
    *   Fuse the dynamic object's contribution with the static scene using alpha blending with a blending factor of 0.95 [S1].

**Formulas:**
The dynamic radiation field $F(\mathbf{x}, t)$ is expressed as [S1]:
$$
F(\mathbf{x}, t) = F_{static}(\mathbf{x} + \Delta(\mathbf{x}, t))
$$
Where:
*   $F_{static}$ is the static NeRF function.
*   $\Delta(\mathbf{x}, t)$ is the displacement vector output by the deformation field MLP.
*   $\mathbf{x}$ is the 3D spatial coordinate.
*   $t$ is the time coordinate.

**Parameters:**
*   **Deformation Field Architecture:** 3-layer MLP, hidden dimension 128 [S1].
*   **Total Parameters:** 5M [S1].
*   **Blending Factor:** 0.95 for alpha blending [S1].
*   **Alignment Error Threshold:** <1 pixel [S1].
*   **Temporal Consistency Loss:** Used to ensure smooth dynamic transitions, targeting SSIM $\ge$ 0.92 [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class StaticNeRF:
    """
    Represents the pre-trained static NeRF model.
    Assumes standard NeRF query interface.
    """
    def __init__(self, model_path):
        # Load pre-trained static NeRF weights
        # Not given in the sources. [G]
        pass

    def query(self, x, d):
        """
        Query color and density for static scene.
        x: (3,) spatial coordinate
        d: (3,) viewing direction
        Returns: color (3,), density (1,)
        """
        # Not given in the sources. [G]
        pass

class DeformationField:
    """
    3-layer MLP for local deformation.
    Input: x (3,), t (scalar)
    Output: delta (3,)
    """
    def __init__(self, weights):
        self.weights = weights
        # Hidden dim 128, 3 layers total (input->hidden1->hidden2->output)
        # Not given in the sources. [G]

    def forward(self, x, t):
        # Not given in the sources. [G]
        pass

def spatial_alignment(markers_static, markers_dynamic):
    """
    Align dynamic object coordinates to static scene using RANSAC.
    markers_static: (4, 3) points in static scene
    markers_dynamic: (4, 3) points in dynamic object space
    Returns: transformation matrix T
    """
    # Use RANSAC algorithm to find transformation
    # Not given in the sources. [G]
    pass

def render_frame(static_nerf, deformation_field, T, rays, time_t):
    """
    Render a single frame by fusing static and dynamic components.
    rays: list of (origin, direction) tuples
    time_t: float, current time
    Returns: image (H, W, 3)
    """
    # Initialize image buffer
    # Not given in the sources. [G]
    
    for ray in rays:
        origin, direction = ray
        
        # Cast ray to find intersection with dynamic object bounding box
        # Not given in the sources. [G]
        
        if intersects_dynamic:
            # Get intersection point x in dynamic object's local space
            # Not given in the sources. [G]
            
            # Apply alignment transformation to get x in static scene space
            x_aligned = T @ np.append(x, 1)
            x_aligned = x_aligned[:3]
            
            # Calculate deformation
            delta = deformation_field.forward(x_aligned, time_t)
            x_deformed = x_aligned + delta
            
            # Query static NeRF at deformed position
            color, density = static_nerf.query(x_deformed, direction)
            
            # Alpha blend with static scene background
            # Blending factor 0.95
            # Not given in the sources. [G]
        else:
            # Query static NeRF directly
            color, density = static_nerf.query(origin, direction)
            
        # Accumulate color
        # Not given in the sources. [G]
        
    return image
```

## Subtopics

### Neural Transient Field (NeTF)
[S1]: Models transient phenomena by decomposing the scene into static and transient components using a canonical space. Choose when dealing with fast-moving, non-rigid transients rather than rigidly aligned dynamic objects.

### SWAG
[S1]: Uses a 4D Gaussian Splatting approach for dynamic scenes. Choose when real-time rendering of large-scale dynamic scenes is required and Gaussian primitives are preferred over volumetric NeRFs.

### Projected Color-Guided Dynamic NeRF
[S1]: Incorporates color guidance from projections to improve dynamic region reconstruction. Choose when color consistency in dynamic regions is critical and sparse input is available.

### On-the-Fly Dynamic NeRF Training
[S1]: Enables real-time training and rendering of dynamic scenes. Choose when the scene dynamics are unknown or changing rapidly during capture.

### D-NeRF
[S1]: The baseline dynamic NeRF method using a single MLP with time as input. Choose when the entire scene is dynamic and computational resources are not a constraint.

## Operating characteristics
*   **Time Complexity:** Rendering time is comparable to static NeRF for static regions, plus the cost of querying the deformation field for dynamic regions. The deformation field is a small 3-layer MLP, making it computationally lightweight [S1].
*   **Memory Complexity:** Significantly reduced compared to traditional dynamic NeRFs. The static NeRF stores only spatial information, and the deformation field uses only 5M parameters, which is 95% less than DyNeRF's 100M+ parameters [S1].
*   **Measured Performance:** In experiments with sparse input (20 frames), the method increased PSNR by $2.3 \pm 0.2$ dB over 10 trials [S1]. Temporal consistency loss ensures SSIM $\ge$ 0.92 [S1].
*   **Strengths:** Low parameter count (5M), suitable for VR devices, handles perspective consistency via alignment, and separates static and dynamic components for efficiency [S1].
*   **Limitations:** Requires manual or automatic selection of marking points for alignment, assumes the dynamic object can be modeled by a local deformation field, and may struggle with complex non-rigid deformations outside the local field's scope [S1].
*   **When to Use:** When embedding dynamic advertising objects into static exhibition stands or scenes with sparse input data, especially for VR applications where computational efficiency is critical [S1].
*   **When Not to Use:** When the entire scene is dynamic, or when the dynamic object undergoes complex non-rigid deformations that cannot be captured by a local deformation field [S1].

## Background
This method was proposed in the paper "Efficient Sparse Input Scene Reconstruction and Real-Time Rendering for VR Advertising Using Optimized NeRF Framework" [S1]. It addresses the limitations of traditional dynamic NeRFs like DyNeRF, which are too computationally expensive for VR devices due to their large parameter counts and independent model training per time step [S1]. The approach leverages the observation that in advertising scenes, most of the environment is static, allowing for a decomposition into static and dynamic components [S1].

## Sources
[S1] Efficient Sparse Input Scene Reconstruction and Real-Time Rendering for VR Advertising Using Optimized NeRF Framework, https://www.informatica.si/index.php/informatica/article/download/12804/6562, fetched 2026-10-29
