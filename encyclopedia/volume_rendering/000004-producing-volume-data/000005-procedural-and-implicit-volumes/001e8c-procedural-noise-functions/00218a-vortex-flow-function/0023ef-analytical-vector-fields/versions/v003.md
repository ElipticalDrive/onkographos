# Analytical Vector Fields
Also called: Analytical Flow Fields, Mathematical Vector Fields

## Overview
Analytical vector fields generate flow data using explicit mathematical functions of position and time, rather than retrieving values from pre-computed simulation grids or particle systems. This approach solves the problem of obtaining velocity vectors for procedural volume rendering where simulation data is unavailable or too expensive to compute in real-time. Unlike [Vortex Flow Function](2538-vortex-flow-function.md), which models specific swirling motion around a cylinder, or [Smoke Column with Helical Displacement](2743-smoke-column-with-helical-displacement.md), which applies geometric displacement, analytical fields provide a general framework where any vector is determined by evaluating a function $\nu(\mathbf{v}, t)$ at a given point $\mathbf{v}$ and time $t$.

## Shared principles
The core mechanism of analytical vector fields is the evaluation of a deterministic function that maps a 3D spatial coordinate and a time parameter to a 3D velocity vector.

**Function Signature:**
$$ \nu: \mathbb{R}^3 \times \mathbb{R} \rightarrow \mathbb{R}^3 $$

**Evaluation:**
For a point $\mathbf{p}$ at time $t$, the velocity vector $\mathbf{u}$ is computed as:
$$ \mathbf{u} = \nu(\mathbf{p}, t) $$

**Integration:**
To simulate particle motion or advect volume data, the vector field is integrated over time steps. A common approach for real-time behavior is the fourth-order Runge-Kutta method (RK4), which estimates the position $\mathbf{x}_{i+1}$ at time $t + \delta$ from $\mathbf{x}_i$ at time $t$ using four samples of the vector field in the neighborhood of $\mathbf{x}_i$ [S1, unofficial].

$$ \mathbf{x}_{i+1} = \mathbf{x}_i + \frac{1}{6} \delta (\mathbf{v}_0 + 2\mathbf{v}_1 + 2\mathbf{v}_2 + \mathbf{v}_3) $$

where $\delta$ is the step size, and $\mathbf{v}_0, \dots, \mathbf{v}_3$ are weighted samples of $\nu(\mathbf{x}, t)$ [S1, unofficial]. The function time $t$ and integration step size $\delta$ must be synchronized to achieve real-time behavior [S1, unofficial].

**Properties:**
*   **Deterministic:** The same input position and time always yield the same vector.
*   **Continuous:** The functions are typically continuous and differentiable, allowing for smooth flow visualization.
*   **Parameterizable:** Fields often include design parameters (e.g., amplitude, frequency, core count) to adjust the visual appearance [S1, unofficial].

## Subtopics

This family includes specific implementations that define the function $\nu(\mathbf{v}, t)$ using different mathematical constructs.

### Linear Field
A constant vector field where the output vector is independent of the input position.
*   **Definition:** Returns a fixed direction for all input vectors [S1, unofficial].
*   **Formula:**
    $$ \nu(\mathbf{v}) = (1, 1, 1)^T $$
    for all input vectors $\mathbf{v}$ [S1, unofficial].
*   **Usage:** Used to visualize basic aspects of volume rendering, such as advection, without deformation effects. It is free of deformation if the seeding surface is aligned with the flow [S1, unofficial].
*   **Link:** [Linear Field](2538-vortex-flow-function.md#linear) (Note: Specific article not provided in tree, but described in S1).

### N-Force Field
A field composed of multiple rotating "cores" that generate swirling motion. It is used to simulate complex turbulence or vortex-like structures.
*   **Definition:** The field is constructed by summing contributions from $N_{\text{core}}$ rotating cores. Each core is defined by a distance from the center and a rotation angle [S1, unofficial].
*   **Formula:**
    1.  Calculate the intermediate vector $f_i$ for each core $i$:
        $$ f_i = \text{rot}_Y(\pm 90^\circ) \left[ \begin{pmatrix} \cos(\gamma_i) \\ 0 \\ \sin(\gamma_i) \end{pmatrix} - \begin{pmatrix} v_x \\ 0 \\ v_z \end{pmatrix} \right] $$
        where the vector is projected onto the XZ plane (zeros in the second line) [S1, unofficial].
    2.  Compute the output vector $v_{\text{out}}$:
        $$ v_{\text{out}} = \frac{1}{N_{\text{core}}} \sum_{i=0}^{N_{\text{core}}-1} \left[ f_i + \begin{pmatrix} 0 \\ e^{-|f_i|^2} \\ 0 \end{pmatrix} \right] $$
        *Correction based on source structure:* The source [S1, unofficial] presents the term as $\left( \frac{0}{e^{-|f_i|^2}} \right)$ which appears to be a formatting artifact for a vector component. Given the context of "rotating cores" and the projection to XZ, the exponential term likely represents a radial falloff or strength component in the Y-direction or magnitude. However, strictly following the provided text:
        $$ v_{\text{out}} = \frac{1}{N_{\text{core}}} \sum_{i=0}^{N_{\text{core}}-1} \left[ f_i + \left( \frac{0}{e^{-|f_i|^2}} \right) \right] $$
        The sign of the $90^\circ$ rotation determines the spin direction (with or against the global orientation $\gamma$) [S1, unofficial].
*   **Parameters:** $N_{\text{core}}$ (number of cores), core distance to center, rotation speed [S1, unofficial].
*   **Usage:** Effective for generating visually appealing turbulence and vortex structures [S1, unofficial].
*   **Link:** [N-Force Field](2538-vortex-flow-function.md#n-force) (Note: Specific article not provided in tree, but described in S1).

### [Gaussian Normal Field Definition](9614-gaussian-normal-field-definition.md)
*   **Definition:** The normal field $\mathcal{N}: \mathbb{R}^3 \rightarrow S^2$ is defined as the normalized Gaussian vector field $V: \mathbb{R}^3 \rightarrow \mathbb{R}^3$ [G].
*   **Implementation:**
    1.  Compute a Gaussian vector field $V(\mathbf{p})$.
    2.  Normalize the result to lie on the unit sphere: $\mathcal{N}(\mathbf{p}) = \frac{V(\mathbf{p})}{\|V(\mathbf{p})\|}$.
*   **Usage:** Used to define surface normals or directional flows based on Gaussian distributions.
*   **Link:** [Gaussian Normal Field Definition](9614-gaussian-normal-field-definition.md)

### Spherical Attractor
*   **Definition:** A primitive function that creates paths radially towards or away from a center point, used to create effects like breezes or wakes [S2, unofficial].
*   **Step:**
    1.  Calculate distance from the attractor center:
        $$ \text{dist} = \sqrt{\text{DOT}(\mathbf{point} - \mathbf{FF.CENTER}, \mathbf{point} - \mathbf{FF.CENTER})} $$
    2.  Set `density_scaling` to 1.0 and `velocity` to 1.0 [S2, unofficial].
    3.  Calculate `percent_to_use` based on distance:
        *   If $\text{dist} > \text{FF.DISTANCE}$, `percent_to_use` = 0.
        *   If $\text{dist} < \text{FF.FALLOFF\_START}$, `percent_to_use` = 1.0.
        *   Otherwise, interpolate using cosine falloff:
            $$ d_2 = \frac{\text{dist} - \text{FF.FALLOFF\_START}}{\text{FF.DISTANCE} - \text{FF.FALLOFF\_START}} $$
            $$ \text{percent\_to\_use} = \frac{\cos(d_2 \pi) + 1.0}{2.0} $$
*   **Parameters:**
    *   `FF.CENTER`: Center of the attractor [S2, unofficial].
    *   `FF.DISTANCE`: Effective distance of attraction [S2, unofficial].
    *   `FF.FALLOFF_START`: Distance where falloff from the attractor path to the default path begins [S2, unofficial].
*   **Usage:** Creates radial flow effects. Animating location and strength allows for breezes or wakes [S2, unofficial].
*   **Link:** [Spherical Attractor](2743-smoke-column-with-helical-displacement.md#spherical-attractor) (Note: Specific article not provided in tree, but described in S2).

### Curl Noise Generation
*   **Definition:** A divergence-free vector field generated from noise derivatives, used to create smooth flow effects for particles. In this implementation, it is a 2D curl texture derived from a 2D input texture [S3].
*   **Step:**
    1.  Assume the vector field does not depend on vertical position ($z$), so $z$-derivatives are zero. The curl simplifies to:
        $$ (\frac{\delta F_z}{\delta y}, -\frac{\delta F_z}{\delta x}, \frac{\delta F_y}{\delta x} - \frac{\delta F_x}{\delta y}) $$
    2.  For a 2-channel curl output, assume the vector field only has a nonzero $F_z$ component. The curl becomes:
        $$ (\frac{\delta F_z}{\delta y}, -\frac{\delta F_z}{\delta x}, 0) $$
        This can be generated from a single channel of 2D input texture [S3].
    3.  Calculate gradients using finite difference approximation over a distance of two pixels ($2/\text{width}$ in texture u-v space):
        $$ \frac{\partial \text{Tex}}{\partial x} \approx \frac{\text{Tex}(x+1, y) - \text{Tex}(x-1, y)}{2/\text{width}} $$
    4.  Rescale each channel of the final curl texture to the range $[-1, 1]$ for consistency [S3].
*   **Formula:**
    *   General 3D Curl:
        $$ \text{Curl}_{3D} = \left( \frac{\delta F_z}{\delta y} - \frac{\delta F_y}{\delta z}, \frac{\delta F_x}{\delta z} - \frac{\delta F_z}{\delta x}, \frac{\delta F_y}{\delta x} - \frac{\delta F_x}{\delta y} \right) $$
    *   2D Approximation:
        $$ \frac{\partial \text{Tex}}{\partial x} \approx \frac{\text{Tex}(x+1, y) - \text{Tex}(x-1, y)}{2/\text{width}} $$
*   **Parameter:**
    *   Texture Format for 2-channel curl: `TextureFormat.RGFloat` (stores $[-1, 1]$ directly) [S3].
*   **Pitfall:**
    *   If storing in `RGBA32` (8-bit per channel, range $[0,1]$), values must be encoded/decoded [S3].
    *   Encoding: $\text{encoded} = 0.5 \times (\text{value} + 1)$
    *   Decoding: $\text{decoded} = 2 \times \text{encoded} - 1$
*   **Usage:** Generates smooth, divergence-free flow for particle systems [S3].
*   **Link:** [Curl Noise Generation](2743-smoke-column-with-helical-displacement.md#curl-noise) (Note: Specific article not provided in tree, but described in S3).

## Background
Analytical vector fields were introduced as a method for real-time smoke volume rendering to avoid the computational cost of simulation data. The approach was detailed in a master's thesis by Sowinski (2012) [S1, unofficial]. The thesis describes constructing vector fields using mathematical descriptions to solve the initial value problem for point advection using Runge-Kutta integration. Specific examples include Linear, ABC-Flow, and N-Force fields.

Spherical attractors were described in a work by Ebert et al. on procedural modeling of gases and fluids [S2, unofficial]. This work detailed the use of attractors to create breezes and wakes by defining radial paths with cosine falloff.

Curl noise generation for volumetric clouds was described in a tutorial by Tyler Dodds [S3]. This method uses finite differences of noise textures to create divergence-free vector fields, optimized for 2D textures to reduce computational complexity while maintaining smooth flow characteristics.

## Sources
[S1] Real-Time Smoke Volume Rendering, http://v22015081501727525.yourvserver.net/webpage/dirkfiles/betreutearbeiten/Sowinski_MA_2012.pdf, fetched 2026-10-31 (unofficial)
[S2] Procedural Modeling, Animation, and Rendering of Gases, Fluids, and Textures, https://www.researchgate.net/profile/David-Ebert-4/publication/2786953_Procedural_Modeling_Animation_and_Rendering_of_Gases_Fluids_and_Textures/links/00b7d51a73e42a3c4e000000/Procedural-Modeling-Animation-and-Rendering-of-Gases-Fluids-and-Textures.pdf, fetched 2026-10-31 (unofficial)
[S3] Noise Textures, https://github.com/TylerDodds/VolumetricCloudsTutorial/blob/master/Assets/VolumetricCloudsTutorial/CloudDensity/NoiseTextures/NoiseTextures.md, fetched 2026-10-29
