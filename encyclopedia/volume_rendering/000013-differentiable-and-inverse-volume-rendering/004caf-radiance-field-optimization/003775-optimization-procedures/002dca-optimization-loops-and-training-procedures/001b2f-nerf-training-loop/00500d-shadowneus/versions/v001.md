# ShadowNeuS

Also called: Neural SDF Reconstruction by Shadow Ray Supervision

## Overview
ShadowNeuS reconstructs a neural Signed Distance Field (SDF) of a scene from single-view images under multiple lighting conditions by supervising shadow rays (rays between the light source and the scene) rather than just camera rays [S1]. The core problem solved is that NeRF does not model rays from the scene to the light source, missing shadow information which contains geometry cues via light visibility (shadows) [S1]. Unlike NeRF which uses multi-view cameras as ray sensors, this method uses a single-view camera and shadow rays to provide supervision, making it applicable to real capture setups with limited views [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Single-view images (either binary shadow images or RGB images) captured under multiple known light directions. For RGB inputs, the method requires modeling material properties. The ground plane position and orientation are assumed known to resolve scale ambiguity and bound the scene [S1].
*   **Outputs:** A neural SDF network $f$ representing the scene geometry (zero level set) and, for RGB inputs, a material network $g$ regressing diffuse and specular coefficients [S1].

**Data Structures**
*   **SDF Network $f$:** A Multi-Layer Perceptron (MLP) that takes a 3D position $\mathbf{x}$ as input and outputs a signed distance value $f(\mathbf{x})$ [S1]. For RGB inputs, it also outputs an extra 256-dimensional feature vector [S1].
*   **Material Network $g$:** An MLP used only for RGB inputs. It takes the 3D position $\mathbf{x}$ and surface normal $\mathbf{n}$ as input and outputs diffuse albedo $\rho_d$ and specular coefficients $\mathbf{y}$ [S1].
*   **Light Source Parameters:**
    *   Directional Light: Uniform direction $\mathbf{l}$ and intensity $L$ [S1].
    *   Point Light: Location $\mathbf{q}$ and scalar intensity $L_p$ [S1].

**Numbered Steps**

1.  **Scene Illumination:** Illuminate the scene multiple times using different light directions (e.g., 100 directional and 100 point lights randomly sampled on the upper hemisphere) to obtain sufficient observations of light visibility (shadows) at the scene surface [S1].
2.  **Camera Ray Marching:** For each camera ray $\mathbf{o}, \mathbf{v}$, use ray marching to compute the intersection point $\mathbf{x}$ at the current SDF. If the camera ray does not intersect the SDF, it is set to intersect with the known ground plane to bound the scene [S1].
3.  **Differentiable Intersection:** To make the intersection point $\mathbf{x}$ differentiable for backpropagation, apply implicit differentiation to the initial ray marching result $\tilde{\mathbf{x}}$:
    $$ \mathbf{x} = \tilde{\mathbf{x}} - \frac{\mathbf{v}}{\mathbf{n} \cdot \mathbf{v}} f(\tilde{\mathbf{x}}) $$
    where $\mathbf{n} = \nabla_{\mathbf{x}} f(\mathbf{x})$ is the surface normal derived from the SDF network [S1].
4.  **Boundary Detection:** Identify boundary pixels using a surface walk procedure [S1] to obtain a pixel subset $\Omega$ (pixels spanning disconnected regions at different depths) and a differentiable area ratio $w$ for each boundary pixel [S1].
5.  **Shadow Ray Sampling:**
    *   For non-boundary pixels ($\mathbf{p} \notin \Omega$): Sample a single shadow ray starting at the camera ray intersection point $\mathbf{x}$ in the light direction $\mathbf{l}$.
    *   For boundary pixels ($\mathbf{p} \in \Omega$): Locate two intersections $\mathbf{x}_n$ (near) and $\mathbf{x}_f$ (far) at different depths within the pixel. Compute shadow rays starting at both $\mathbf{x}_n$ and $\mathbf{x}_f$ [S1].
6.  **Shadow Ray Integration (Volume Rendering):** Compute the incoming irradiance along the shadow ray. The opacity $\alpha_i$ at sample $t_i$ is derived from the SDF using the CDF of the SDF values:
    $$ \alpha_i = \max \left( 1 - \frac{\Phi_s(f(t_{i+1}))}{\Phi_s(f(t_i))}, 0 \right) $$
    where $\Phi_s(x) = (1 + e^{-sx})^{-1}$ is the sigmoid function and $s$ is a learnable scalar controlling the transition between volume and surface rendering [S1].
    The incoming irradiance at the surface point $\mathbf{x}$ is:
    $$ \text{irradiance}(\mathbf{x}, \mathbf{l}) = L \prod_{i=1}^{N} (1 - \alpha_i) $$
    where $L$ is the light intensity. For point lights, $L = \frac{L_p}{\|\mathbf{q} - \mathbf{x}\|_2^2}$ and $\mathbf{l} = \frac{\mathbf{q} - \mathbf{x}}{\|\mathbf{q} - \mathbf{x}\|_2}$ [S1].
7.  **Pixel Irradiance Aggregation:** Compute the final incoming irradiance for the pixel:
    $$ \text{irradiance}_{\text{pix}} = \begin{cases} \text{irradiance}(\mathbf{x}, \mathbf{l}) & \mathbf{p} \notin \Omega \\ w \cdot \text{irradiance}(\mathbf{x}_n, \mathbf{l}) + (1 - w) \cdot \text{irradiance}(\mathbf{x}_f, \mathbf{l}) & \mathbf{p} \in \Omega \end{cases} $$
    [S1]
8.  **Outgoing Radiance Computation (RGB Only):** If inputs are RGB, decompose the outgoing radiance using inverse rendering. Model the non-Lambertian BRDF as a diffuse component $\rho_d$ and a specular component $\rho_s$. Regress material properties $(\rho_d, \mathbf{y})$ using network $g$ at $\mathbf{x}$. The specular component uses spherical Gaussian basis: $\rho_s = \mathbf{y}^T D(\mathbf{h}, \mathbf{n})$, where $\mathbf{h}$ is the half-vector between light direction $\mathbf{l}$ and view direction $-\mathbf{v}$. The outgoing radiance is:
    $$ C(\mathbf{x}, -\mathbf{v}) = (\rho_d + \rho_s) \text{irradiance}(\mathbf{x}, \mathbf{l}) (\mathbf{l} \cdot \mathbf{n}) $$
    [S1]
9.  **Loss Calculation:**
    *   For binary shadow images: $\mathcal{L}_{\text{shadow}} = \| \text{irradiance}_{\text{pix}} - I_{\text{shadow}} \|_1$ [S1].
    *   For RGB images: $\mathcal{L}_{\text{rgb}} = \| \widehat{C} - I_{\text{rgb}} \|_1$ [S1].
    *   Eikonal Loss (Regularization): $\mathcal{L}_{\text{eik}} = \frac{1}{M} \sum_{i}^{M} (\|\nabla f(\mathbf{x}_i)\|_2 - 1)^2$ [S1].
10. **Optimization:** Minimize the total loss ($\mathcal{L}_{\text{shadow}}$ or $\mathcal{L}_{\text{rgb}} + \lambda \mathcal{L}_{\text{eik}}$) using gradient descent. The iterative updating strategy optimizes both the SDF network parameters and the starting positions of the shadow rays [S1].

**Parameters**
*   **Training Iterations:** 150,000 [S1].
*   **Samples per Image:** 256 pixel positions sampled as supervision signals [S1].
*   **Batch Size:** 4 images randomly selected in each batch [S1].
*   **Light Sampling:** 100 directional lights and 100 point lights selected randomly on the upper hemisphere [S1].
*   **Input Resolution:** 800x800 for synthetic datasets [S1].
*   **Learnable Scalar $s$:** Controls the opacity transition in the SDF-based alpha calculation [S1].

**Edge Cases and Pitfalls**
*   **Surface Boundaries:** At surface boundaries, a pixel spans disconnected regions at different depths. Sampling only one shadow ray leads to incorrect gradients because the intersection point $\mathbf{x}$ moves perpendicular to the camera direction $\mathbf{v}$, significantly changing the computed radiance proportional to the area of each region. This is handled by averaging incoming radiance from two intersections (near and far) weighted by the differentiable area ratio $w$ [S1].
*   **Unbounded Rays:** Camera rays that do not intersect with the SDF are set to intersect with the ground plane to bound the scene [S1].
*   **Scale Ambiguity:** Single-view inputs suffer from scale ambiguity. This is resolved by assuming the ground plane's position and orientation are known [S1].

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. Training takes about 24 hours for 150k iterations on a single RTX 2080Ti [S1].
*   **Memory Complexity:** Not explicitly given in the sources.
*   **Measured Performance:** Evaluated on synthetic datasets (800x800 resolution) and real capture setups. The method shows significant improvements in shape reconstruction compared to previous single-view methods [S1].
*   **Strengths:**
    *   Reconstructs geometry from scratch using shadows, unlike methods that use multi-view stereo first [S1].
    *   Fully differentiable shadow ray supervision, unlike concurrent works that use non-differentiable root finding [S1].
    *   Applicable to single-view setups, unlike NeRF which requires multi-view cameras [S1].
    *   Handles both binary shadow images and RGB inputs [S1].
*   **Limitations:**
    *   Requires multiple lighting conditions (multiple images with different light directions) [S1].
    *   Assumes opaque scenes; transparency and inter-reflections are disabled [S1].
    *   Requires known ground plane for bounding and scale resolution [S1].
*   **When to use:** When reconstructing 3D shape from single-view images where multiple lighting conditions can be controlled or captured, especially when shadow information is available.
*   **When not to use:** When only a single static image with unknown lighting is available, or when the scene contains transparent/reflective objects causing inter-reflections.

## Background
ShadowNeuS was proposed to address the limitation of NeRF, which models camera rays but ignores shadow rays (rays from the scene to the light source) [S1]. The method was introduced in the paper "ShadowNeuS: Neural SDF Reconstruction by Shadow Ray Supervision" [S1]. It analogizes shadow ray supervision to NeRF's camera ray supervision, using multiple light directions as "ray sensors" to record incoming radiance [S1]. The work builds on NeuS for SDF representation and differentiable rendering techniques [S1].

## Sources
[S1] ShadowNeuS: Neural SDF Reconstruction by Shadow Ray Supervision, https://doi.org/10.1109/cvpr52729.2023.00025, fetched 2026-10-26
