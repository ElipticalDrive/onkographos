# Gaussian Splatting Shadows
Also called: Nearly-Soft Shadows for Gaussian Splashing, Shadow Rendering

## Overview
This method renders nearly-soft dynamic shadows for scenes represented by Gaussian splats by projecting the splats into a light-view space and performing depth-based occlusion testing [S1]. It differs from traditional volumetric shadow ray casting by leveraging the explicit Gaussian kernel representation to generate a depth image directly, avoiding the need for ray-marching through a density field [S1]. Unlike hard shadow mapping, it computes shadowing probabilities using Chebyshev’s inequality to approximate soft shadow edges, balancing performance and visual quality for dynamic particle systems [S1].

## How it works
**Inputs:**
- A set of $N$ Gaussian kernels $\{G_i\}$, where each kernel $i$ is defined by mean position $\mathbf{\mu}_i$, covariance matrix $\mathbf{\Sigma}_i$, opacity $\alpha_i$, and color $\mathbf{c}_i$.
- A point light source position $\mathbf{l}_{pos}$ and direction $\mathbf{l}_{dir}$ (aligned with the brightest direction in the environment map) [S1].
- Original image resolution $(W, H)$ [S1].

**Outputs:**
- A shadow map containing shadowing probabilities for each pixel.
- A final composited image with soft shadows applied [S1].

**Data Structures:**
- **Light-View Depth Buffer:** A 2D array of size $(3W, 3H)$ storing the minimum depth of splatted Gaussian kernels from the light's perspective [S1].
- **Shadow Map:** A 2D array of size $(3W, 3H)$ storing the computed shadowing probability for each pixel [S1].

**Steps:**

1.  **Light-View Setup:**
    Position a virtual camera at the point light's location $\mathbf{l}_{pos}$. Align the camera's view direction with $\mathbf{l}_{dir}$, which is determined as the direction of the significantly bright light in the environment map [S1]. Set the light-view image resolution to $(3W, 3H)$ to mitigate under-sampling artifacts [S1].

2.  **Splatting to Light-View:**
    Project all Gaussian kernels $\{G_i\}$ onto the light-view camera plane. For each kernel, compute its 2D projection and depth. Render these projections to generate a depth image $D_{light}$, where each pixel stores the minimum depth value encountered by any splat [S1].

3.  **Reprojection:**
    For each point $\mathbf{p}$ visible in the original camera view, reproject its 3D position into the light-view space to obtain its coordinates $(u_l, v_l)$ and depth $d_p$ in the light-view coordinate system [S1].

4.  **Occlusion Comparison:**
    Retrieve the depth value $d_{light}$ from the light-view depth image $D_{light}$ at pixel $(u_l, v_l)$ [S1].
    - If $d_p > d_{light}$, the point $\mathbf{p}$ is occluded by a nearer Gaussian kernel and is in shadow [S1].
    - If $d_p \le d_{light}$, the point is illuminated [S1].

5.  **Shadow Probability Computation:**
    To achieve soft shadows, compute the shadowing probability $P_{shadow}$ using Chebyshev’s inequality [S1]. This involves estimating the variance of the depth distribution around the splat's projection in the light-view space. The specific implementation details of the variance estimation are referenced from the "nearly-soft shadows" method [S1]. The probability is stored in the shadow map [S1].

6.  **Compositing:**
    Composite the rendered image with the shadow map. The final color $\mathbf{c}_{final}$ for a pixel is modulated by the shadowing probability:
    $$ \mathbf{c}_{final} = \mathbf{c}_{rendered} \cdot (1 - P_{shadow}) + \mathbf{c}_{ambient} \cdot P_{shadow} $$
    (Note: The exact blending formula with ambient light is not explicitly detailed in the source, but the shadow map is composited to achieve nearly-soft dynamic shadows [S1]).

**Parameters:**
- **Light-View Resolution:** $3 \times$ original image resolution. This factor is chosen to address visual discrepancies caused by under-sampling [S1].

**Pitfalls:**
- **Under-sampling:** Using a light-view resolution equal to or lower than the original image can cause visual discrepancies in shadows. Mitigation requires the $3\times$ resolution factor [S1].

## Operating characteristics
- **Time Complexity:** Dominated by the splatting step, which is $O(N)$ for $N$ Gaussian kernels, and the reprojection/comparison step, which is $O(P)$ for $P$ visible pixels. The use of a fixed $3\times$ resolution increases the pixel processing cost by a factor of 9 compared to the original resolution.
- **Memory Complexity:** Requires storage for the light-view depth buffer and shadow map, both of size $9WH$ (where $W, H$ are original dimensions).
- **Strengths:** Efficient for dynamic scenes as it avoids ray-marching. Provides soft shadows without expensive sampling.
- **Limitations:** Resolution-dependent; lower resolutions may lead to aliasing or incorrect occlusion if not upscaled sufficiently. Relies on the accuracy of the Gaussian projection.

## Background
This method is derived from the "nearly-soft shadows" technique [S1], re-engineered for use with Gaussian splatting in the context of the "Gaussian Splashing" framework for unified particle motion synthesis and rendering [S1]. It was introduced to enhance visual realism in dynamic fluid and particle simulations by providing efficient, soft shadowing [S1].

## Sources
[S1] Gaussian Splashing: Unified Particles for Versatile Motion Synthesis and Rendering, https://doi.org/10.1109/CVPR52734.2025.00057, fetched 2026-10-29
