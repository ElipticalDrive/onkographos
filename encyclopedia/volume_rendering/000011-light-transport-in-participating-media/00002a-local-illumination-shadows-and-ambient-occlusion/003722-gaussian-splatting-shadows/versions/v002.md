# Gaussian Splatting Shadows
Also called: Nearly-Soft Shadows for Gaussian Splashing, Shadow Rendering

## Overview
This method renders nearly-soft dynamic shadows for scenes represented by Gaussian splats by projecting the splats into a light-view space and performing depth-based occlusion testing [S1]. It differs from traditional volumetric shadow ray casting by leveraging the explicit Gaussian kernel representation to generate a depth image directly, avoiding the need for ray-marching through a density field [S1]. Unlike hard shadow mapping, it computes shadowing probabilities using Chebyshev’s inequality to approximate soft shadow edges, balancing performance and visual quality for dynamic particle systems [S1].

## How it works
**Inputs:**
- A set of $N$ Gaussian kernels $\{G_i\}$, where each kernel $i$ is defined by mean position $\mathbf{\mu}_i$, covariance matrix $\mathbf{\Sigma}_i$, opacity $\alpha_i$, and color $\mathbf{c}_i$ [G].
- A point light source position $\mathbf{l}_{pos}$ and direction $\mathbf{l}_{dir}$, where the direction is aligned with the significantly bright light in the environment map [S1].
- Original image resolution $(W, H)$ [S1].

**Outputs:**
- A shadow map containing shadowing probabilities for each pixel [S1].
- A final composited image with soft shadows applied [S1].

**Data Structures:**
- **Light-View Depth Buffer:** A 2D array of size $(3W, 3H)$ storing depth values from splatted Gaussian kernels from the light's perspective [S1].
- **Shadow Map:** A 2D array storing computed shadowing probabilities for each pixel [S1].

**Steps:**
1. **Light-View Setup:** Position a virtual camera at the point light's location $\mathbf{l}_{pos}$ [S1]. Align the camera's view direction with $\mathbf{l}_{dir}$, determined as the direction of the significantly bright light in the environment map [S1]. Set the light-view image resolution to $(3W, 3H)$ to mitigate under-sampling artifacts [S1].
2. **Splatting to Light-View:** Splat all Gaussian kernels $\{G_i\}$ to the light-view camera to generate a depth image $D_{light}$ [S1].
3. **Reprojection:** Reproject points seen in the original camera view to the light-view space to obtain their coordinates and depth values [S1].
4. **Occlusion Comparison:** Compare the reprojected points' depths to the previously splatted light-view depth image $D_{light}$ [S1]. If a point's depth is larger than the light-view depth, it is occluded by a nearer point and casts a shadow [S1].
5. **Shadow Probability Computation:** Compute the shadowing probability using Chebyshev’s inequality to create a robust variance method [S1]. Softer shadows can be achieved by blurring and averaging the light-view depth image [S1]. Store the computed shadowing probabilities in a shadow map [S1].
6. **Compositing:** Composite the rendered image with the shadow map to achieve nearly-soft dynamic shadows [S1].

**Parameters:**
- **Light-View Resolution:** $3 \times$ original image resolution [S1]. This factor is chosen to address visual discrepancies caused by under-sampling [S1].

**Pitfalls:**
- **Under-sampling:** Using a light-view resolution equal to or lower than the original image can cause visual discrepancies in shadows [S1]. This is mitigated by using a light-view resolution three times that of the original image [S1].

## Subtopics

### Splatting-based Shadows
A two-pass rendering technique that uses splatting primitives during the passes to compute and project shadow information, producing both hard and soft shadows [S2, unofficial]. It is chosen when a simple two-pass pipeline is preferred over depth-comparison methods.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.

## Background
This method is derived from the "nearly-soft shadows" technique [S1], re-engineered for use with Gaussian splatting in the context of the "Gaussian Splashing" framework for unified particle motion synthesis and rendering [S1]. It was introduced to enhance visual realism in dynamic fluid and particle simulations by providing efficient, soft shadowing [S1].

## Sources
[S1] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
[S2] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
