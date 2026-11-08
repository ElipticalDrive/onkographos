# Inverse Rendering with Inter-reflections (IRGS)

Also called: GIR, GS-IR, Inverse Gaussian Splatting

## Overview
IRGS is an inverse rendering framework that jointly optimizes 3D Gaussian geometry and an environment light probe to recover physically accurate material and lighting properties [S1, S2, unofficial]. It distinguishes itself from sibling methods like GS-IR by applying the full rendering equation without simplification, explicitly modeling inter-reflections via differentiable 2D Gaussian ray tracing to compute incident radiance on the fly [S2, unofficial]. This approach corrects the inaccurate material and lighting estimates produced by earlier methods that relied on simplified equations or learned light approximations [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input images (multi-view or video), initial 3D Gaussian parameters (position, covariance, opacity, color), and an initial environment light probe.
*   **Outputs:** Optimized Gaussian geometry (position, covariance, opacity), material properties (diffuse/specular components), and the environment light probe.

**Data Structures:**
*   **Gaussian Primitives:** Standard 3D Gaussian Splatting representation. Each Gaussian is defined by a mean position $\mu \in \mathbb{R}^3$, a covariance matrix $\Sigma \in \mathbb{R}^{3 \times 3}$ (often parameterized by a rotation matrix $R$ and scale vector $s$), opacity $\alpha$, and color coefficients.
*   **Environment Light Probe:** A spherical harmonic or texture-based representation of the incident radiance field.

**Method Steps:**
1.  **Differentiable 2D Gaussian Ray Tracing:** Unlike standard rasterization which treats Gaussians as independent emitters, IRGS computes the incident radiance on each Gaussian surface element by tracing rays through the Gaussian distribution. This involves evaluating the Gaussian density along the ray path to determine the contribution of other Gaussians to the local illumination [S2, unofficial].
2.  **Full Rendering Equation:** The color of each pixel is computed by integrating the rendering equation over the Gaussian primitives. This includes direct illumination from the environment light probe and indirect illumination (inter-reflections) calculated from the incident radiance found in Step 1 [S2, unofficial]. The equation is applied without the split-sum or other simplifications used in GS-IR [S2, unofficial].
3.  **Gradient Backpropagation:** Gradients are backpropagated through the rendering equation using the differentiable rasterizer/ray-tracer [S1, unofficial]. This allows joint optimization of the Gaussian parameters (geometry, opacity) and the light probe parameters [S1, unofficial].
4.  **Optimization:** The parameters are updated via gradient descent to minimize the difference between rendered images and input images [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# IRGS Optimization Loop
# Inputs: images, initial_gaussians, initial_light_probe
# Outputs: optimized_gaussians, optimized_light_probe

def irgs_optimize(images, initial_gaussians, initial_light_probe, num_iterations):
    gaussians = initial_gaussians
    light_probe = initial_light_probe
    
    for i in range(num_iterations):
        # 1. Compute incident radiance via differentiable 2D Gaussian ray tracing
        # This accounts for inter-reflections by tracing rays through the Gaussian distribution
        incident_radiance = differentiable_gaussian_ray_trace(gaussians, light_probe)
        
        # 2. Render image using full rendering equation (no simplification)
        # Integrates direct and indirect illumination
        rendered_image = render_full_rendering_equation(gaussians, light_probe, incident_radiance)
        
        # 3. Compute loss
        loss = compute_loss(rendered_image, images[i % len(images)])
        
        # 4. Backpropagate gradients through the rendering equation
        # Optimizes both geometry and light probe
        gradients = backpropagate(loss, gaussians, light_probe)
        
        # 5. Update parameters
        gaussians = update_parameters(gaussians, gradients['gaussians'])
        light_probe = update_parameters(light_probe, gradients['light_probe'])
        
    return gaussians, light_probe
```

**Edge Cases and Pitfalls:**
*   **Precision:** Computing incident radiance on the fly via ray tracing can be computationally expensive and sensitive to numerical precision in density evaluation [S2, unofficial].
*   **Boundaries:** Handling the boundaries of the Gaussian distribution during ray tracing is critical for accurate inter-reflection modeling [S2, unofficial].
*   **Degenerate Input:** Poor initial geometry or lighting can lead to local minima in the optimization process [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Higher than standard 3DGS due to the on-the-fly computation of incident radiance via differentiable ray tracing [S2, unofficial].
*   **Memory Complexity:** Requires storing Gaussian parameters and the light probe, similar to GS-IR, but with additional overhead for ray tracing computations [S2, unofficial].
*   **Performance:** Fixes inaccurate material/lighting estimates of earlier methods, providing more physically plausible results [S2, unofficial].
*   **Strengths:** Physically accurate modeling of inter-reflections; joint optimization of geometry and lighting [S2, unofficial].
*   **Limitations:** Computationally intensive; potential for optimization instability [S2, unofficial].
*   **When to use:** When physically accurate material and lighting recovery is required, and computational resources allow for the increased complexity [S2, unofficial].
*   **When not to use:** For real-time applications where speed is critical, or when simplified lighting models are sufficient [S2, unofficial].

## Background
IRGS was marked as a 2026 SOTA update in the context of Gaussian Splatting variants [S2, unofficial]. It addresses the limitations of earlier inverse rendering methods like GS-IR and Relightable 3DG, which used simplified equations or learned light approximations [S2, unofficial]. By applying the full rendering equation without simplification, IRGS provides a more accurate solution to the inverse rendering problem in the context of 3D Gaussian Splatting [S2, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
