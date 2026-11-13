# Piecewise Linear Mapping and Integration

Also called: Piecewise Linear Transfer Functions, Linear Opacity Mapping

## Overview
This family defines opacity $\alpha$ as a continuous, piecewise linear function of the scalar volume value $V$, enabling smooth transitions between tissue types and supporting semi-transparent blending [S1][S2, unofficial]. It differs from binary thresholding by allowing gradual opacity changes and from non-linear mappings by offering computational simplicity and analytical tractability [S1][S2, unofficial].

## Shared principles
All methods in this family share the following core mechanism:

1.  **Piecewise Linear Definition**: The transfer function is defined by a set of control points $(v_k, \alpha_k)$, where $v_k$ are scalar volume values (e.g., CT numbers) and $\alpha_k$ are target opacities. The opacity for a value $v$ is determined by linear interpolation between the two nearest control points $v_n$ and $v_{n+1}$ such that $v_n \le v \le v_{n+1}$ [S1][S2, unofficial].
2.  **Linear Interpolation Formula**: For $v \in [v_n, v_{n+1}]$, the base opacity $\alpha_{base}$ is calculated as:
    $$
    \alpha_{base}(v) = \alpha_n + (\alpha_{n+1} - \alpha_n) \frac{v - v_n}{v_{n+1} - v_n}
    $$
    [S1][S2, unofficial]. Outside the defined range $[v_1, v_N]$, opacity is typically zero [S2, unofficial].
3.  **Step Size Correction**: The interpolated opacity $\alpha_{base}$ represents the probability of interaction over a unit distance. For a discrete step size $\Delta t$, the effective opacity $\alpha_i$ used in compositing is corrected to approximate continuous attenuation, often using the Beer-Lambert law: $\alpha_i = 1 - e^{-\sigma \Delta t}$, where $\sigma$ is derived from $\alpha_{base}$ or directly mapped [S26, S33, S39, unofficial]. Note: Some implementations (e.g., [7016-linear-opacity-interpolation.md]) apply a discrete correction $\alpha' = 1 - (1 - \alpha)^{\Delta t}$ [S11, S32, unofficial].
4.  **Gradient Independence (Base)**: The base piecewise linear mapping operates solely on scalar intensity $V$, ignoring spatial gradients [S1][S2, unofficial]. Variants may incorporate gradients as a post-processing modulation step.

## Subtopics

### [Piecewise Density Approximation for Reparameterized Sampling](1139-piecewise-density-approximation-for-reparameterized-sampling.md)
This method approximates the density field $\sigma_r(t)$ along a ray as piecewise constant or piecewise linear on a predefined grid to enable analytical inversion of the cumulative opacity integral. It is chosen when stochastic sampling requires sample locations $t$ that correspond to uniform random opacity samples without numerical root-finding. It differs from other siblings by focusing on the density integral's inversion for reparameterized sampling rather than direct opacity mapping.

### [Piecewise Linear Opacity Mapping with Gradient Scaling](1335-piecewise-linear-opacity-mapping-with-gradient-scaling.md)
This method scales the piecewise linear opacity by the magnitude of the local gradient $|\nabla f(x_i)|$ to suppress tissue interiors and enhance bounding surfaces. It is chosen for visualizing thin, wispy structures and creating semi-transparent surface overlays (e.g., skin over bone). It differs from base linear interpolation by explicitly using gradient magnitude to modulate visibility, making it suitable for rendering multiple semi-transparent surfaces in superposition.

### [Linear Opacity Interpolation](7016-linear-opacity-interpolation.md)
This method maps scalar volume values to opacity using linear interpolation between two threshold values. It serves as the foundational opacity mapping primitive, providing a simple, computationally inexpensive way to define transparency ranges. It differs from other siblings by operating directly on opacity without gradient scaling or density approximation, making it suitable only for scalar-intensity-based segmentation.

### Power Function Opacity Transfer
- **Definition**: A simple power function used to map voxel values to opacity before enhancement, allowing illustrative enhancements to supplement existing volume rendering mechanisms [S1].
- **Step**: Calculate the gradient used for enhancement from the dataset *after* the simple power function transfer function has been applied, as this gradient is more effective in capturing subtle features [S1].
- **Formula**: $o_v = (k_{os}v_i)^{k_{oe}}$, where $v_i$ is the volume sample value, $k_{os}$ is the scalar controlling maximum opacity, and $k_{oe}$ is the exponent [S1]. Values of $k_{oe} < 1$ soften volume differences; values $> 1$ increase contrast [S1].
- **Parameter**: $k_{os}$: Scalar controlling maximum opacity. $k_{oe}$: Exponent controlling contrast [S1].

### Piecewise Linear Transfer Function Integration
- **Definition**: A rendering technique that integrates piecewise linear transfer functions using a 2D texture map to compute correct exponential per-pixel opacity [S2, unofficial].
- **Parameter**: Texture Map: A 2D texture map parameterized by opacity and thickness [S2, unofficial].

## Background
Piecewise linear transfer functions are a standard technique in direct volume rendering, originating from the need to map scalar data (like CT numbers) to optical properties with smooth transitions [S1][S2, unofficial]. The use of piecewise linear approximations for density integration and opacity mapping has been widely adopted in both structured and unstructured grid rendering, often leveraging hardware acceleration via texture lookups or analytical inversion for stochastic sampling [S1][S2, unofficial]. The integration of gradient scaling was introduced to enhance surface visibility while maintaining transparency for interior structures [S1][S2, unofficial].

## Sources
[S1] Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
