# Two-Stage Blending

Also called: Image Blending, Per-Point Blending

## Overview
Two-Stage Blending is a rendering technique that synthesizes a final image by compositing a volume-rendered raw image with a pre-existing or auxiliary "anchor" image using a per-pixel blending map. It solves the problem of visual artifacts, such as stretching or geometric inconsistency, that occur when rendering from side views or when integrating 2D priors into 3D volume representations. It differs from standard Direct Volume Ray Casting by decoupling the volumetric integration from the final pixel composition, allowing external 2D information to correct or enhance the volumetric output [S1].

## How it works
The method operates in two distinct phases: volumetric rendering of raw data and post-process blending with an anchor source.

**Inputs and Outputs:**
*   **Inputs:**
    *   Volume dataset (scalar/vector fields).
    *   Transfer function (mapping scalar values to color and opacity).
    *   Anchor image ($I_{anchor}$ or $\hat{I}_{anchor}$), which may be a warped 2D projection or a texture-mesh projection.
    *   Camera parameters (view and projection matrices).
*   **Outputs:**
    *   Final blended image ($I_v$).
    *   Intermediate raw volume image ($I_{raw}$).
    *   Blending map ($I_{map}$ or $b(X)$).

**Data Structures:**
*   **Volume Grid:** Standard 3D texture or voxel grid storing scalar/vector properties.
*   **Blending Map:** A 2D image (typically RGBA or single-channel float) where each pixel $X$ stores a weight $b(X) \in [0, 1]$.
*   **Anchor Image:** A 2D image storing color values $c_{uv}(X)$ or $I_{anchor}$.

**Steps:**

1.  **Volume Rendering (Raw Stage):**
    Perform standard direct volume ray casting for each pixel $X$ in the target resolution.
    *   Cast ray $r(t) = o + td$.
    *   Sample the volume along the ray.
    *   Compute the accumulated color $c(X)$ and an optional blending weight $b(X)$ based on the volume data.
    *   Store the result as $I_{raw}$ and the weight map as $I_{map}$.
    *   *Note:* In some implementations (e.g., HumanGen), $b(X)$ is derived from the accumulated opacity or a specific consistency metric [S1].

2.  **Anchor Image Preparation:**
    *   If the anchor image is from a different viewpoint (e.g., frontal view), warp it to the current view. Let $\hat{I}_{anchor}$ be the warped anchor image.
    *   Retrieve the color value $c_{uv}(X)$ from the anchor image at pixel $X$.

3.  **Per-Pixel Blending:**
    For each pixel $X$, combine the raw volume color and the anchor color using the blending weight.
    *   Formula:
        $$c_b(X) = c(X) \cdot (1 - b(X)) + c_{uv}(X) \cdot b(X)$$
    *   Where:
        *   $c(X)$ is the color from $I_{raw}$.
        *   $c_{uv}(X)$ is the color from the anchor image.
        *   $b(X)$ is the blending weight from $I_{map}$.

4.  **Final Composition:**
    Construct the final image $I_v$ using the blended colors.
    *   Formula:
        $$I_v = I_{raw} \cdot (1 - I_{map}) + \hat{I}_{anchor} \cdot I_{map}$$
    *   This operation is performed component-wise (RGB) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_stage_blending(volume_data, transfer_func, anchor_image, camera_params):
    """
    Renders a volume and blends it with an anchor image.
    
    Args:
        volume_data: 3D array of scalar values.
        transfer_func: Function mapping scalar -> (color, opacity).
        anchor_image: 2D array of RGB values (warped to current view).
        camera_params: Camera view/projection matrices.
        
    Returns:
        final_image: 2D array of RGB values.
    """
    # Step 1: Volume Rendering
    # Initialize raw image and blending map
    height, width = anchor_image.shape[:2]
    I_raw = np.zeros((height, width, 3))
    I_map = np.zeros((height, width, 1)) # Weight map
    
    for y in range(height):
        for x in range(width):
            # Cast ray for pixel (x, y)
            ray_origin, ray_direction = get_ray(x, y, camera_params)
            
            # Ray marching through volume
            accumulated_color = np.array([0.0, 0.0, 0.0])
            accumulated_opacity = 0.0
            current_alpha = 1.0
            
            # Sample points along ray
            for t in sample_points(ray_origin, ray_direction):
                value = sample_volume(volume_data, t)
                color, opacity = transfer_func(value)
                
                # Front-to-back compositing
                contribution = color * opacity * current_alpha
                accumulated_color += contribution
                accumulated_opacity += opacity * current_alpha
                current_alpha *= (1.0 - opacity)
                
                # Early ray termination if fully opaque
                if accumulated_opacity > 0.99:
                    break
            
            I_raw[y, x] = accumulated_color
            
            # Calculate blending weight b(X)
            # In HumanGen, this might be based on consistency or opacity
            # Here we assume a generic weight derived from opacity or a specific metric
            b = calculate_blending_weight(value, accumulated_opacity) 
            I_map[y, x] = b

    # Step 2: Blending
    # Ensure anchor image is warped to current view
    # I_anchor_warped is assumed to be pre-processed or passed in
    
    # Per-pixel blending
    # c_b(X) = c(X) * (1 - b(X)) + c_uv(X) * b(X)
    weight = I_map  # Shape (H, W, 1)
    weight_rgb = np.repeat(weight, 3, axis=2) # Broadcast to RGB
    
    blended_color = I_raw * (1.0 - weight_rgb) + anchor_image * weight_rgb
    
    final_image = blended_color
    
    return final_image

def calculate_blending_weight(value, opacity):
    """
    Placeholder for specific weight calculation logic.
    In [S1], weights are learned or derived from consistency losses.
    In [S2] (Extended Blending), weights are space-variant and time-dependent.
    For basic Two-Stage Blending, opacity is often used as a proxy for confidence.
    """
    # [G] Default implementation: use accumulated opacity as weight.
    # This assumes the volume rendering is more reliable where opacity is high.
    return np.clip(opacity, 0.0, 1.0)
```

**Parameters:**
*   **Blending Weight ($b(X)$):** Typically in $[0, 1]$. A value of 0 uses only the volume rendering; 1 uses only the anchor image.
*   **Sampling Rate:** Number of samples per ray. Not given in the sources.
*   **Anchor View:** The viewpoint from which the anchor image is derived. In [S1], this is often a frontal view [S1].

**Edge Cases and Pitfalls:**
*   **Misalignment:** If the anchor image is not correctly warped to the current view, blending artifacts (ghosting) will occur.
*   **Weight Estimation:** Incorrect estimation of $b(X)$ can lead to over-reliance on the anchor (losing volume detail) or under-reliance (retaining volume artifacts).
*   **Boundary Artifacts:** At the edges of the volume or anchor image, weights may be undefined. [G] Handle by clamping weights or using edge-aware filtering.

## Subtopics

### Extended Blending Scheme for IBFV
A generalized blending equation used in Image-Based Flow Visualization (IBFV) that replaces the affine combination of advected and injected values with a generic linear combination. It allows independent blending of multiple materials with different exponential decay lengths and supports dye advection by adding material on top of existing material. The formula is $\rho(x,t) = W(x,t) \circ \rho(x(t - \Delta t),t - \Delta t) + V(x,t) \circ I(x,t)$, where $\rho$ is the property field, $W$ and $V$ are space-variant and time-dependent weights, $\circ$ is component-wise multiplication, and $I$ is the injection texture [S2].

## Operating characteristics
*   **Time Complexity:** Dominated by the volume rendering stage, which is $O(N \cdot S)$ where $N$ is the number of pixels and $S$ is the number of samples per ray. The blending stage is $O(N)$.
*   **Memory Complexity:** Requires storage for the volume data, the raw image, the blending map, and the anchor image. The blending map adds $O(N)$ memory overhead.
*   **Strengths:** Effectively reduces stretching artifacts in side views; integrates 2D priors into 3D rendering; allows for real-time advection in flow visualization [S1, S2].
*   **Limitations:** Relies on the quality and alignment of the anchor image; may introduce 2D artifacts if the anchor is noisy or misaligned.
*   **When to Use:** When rendering from side views causes artifacts; when integrating 2D texture or mesh information into volume rendering; in flow visualization for dye advection [S1, S2].
*   **When Not to Use:** When high-fidelity 3D geometry is required without 2D priors; when the anchor image is unavailable or unreliable.

## Background
Two-Stage Blending originated in the context of Image-Based Flow Visualization (IBFV) and more recently in neural radiance field generation. The extended blending scheme for IBFV was introduced by Weinkove et al. (2004) to allow unified noise and dye advection [S2]. The application of two-stage blending for human radiance field generation (HumanGen) was proposed by Chen et al. (2023) to alleviate stretching artifacts from side views by blending with a frontal anchor image [S1].

## Sources
[S1] HumanGen: Generating Human Radiance Fields with Explicit Priors, https://doi.org/10.1109/cvpr52729.2023.01207, fetched 2026-10-29
[S2] Real-Time Advection and Volumetric Illumination for the Visualization of 3D Unsteady Flow, https://doi.org/10.2312/vissym/eurovis05/013-020, fetched 2026-10-30
