# Style Transfer and Illustrative Rendering

Also called: Style Transfer Functions (STF), Illustrative Volume Rendering

## Overview
This family extends direct volume ray casting by replacing or augmenting standard color and opacity mapping with artistic styles and illustrative effects, such as contour lines and non-photorealistic transparency models. It differs from sibling techniques like [Piecewise Linear Mapping and Integration](9000-piecewise-linear-mapping-and-integration.md) by decoupling the visual style from the scalar data's physical properties, allowing the same data to be rendered with multiple distinct artistic appearances (e.g., lit spheres, matcaps) via style transfer textures.

## Shared principles
All methods in this family map a scalar input $d$ (the interpolated volume value at a sample point) to a color vector $\mathbf{c}$ and an opacity scalar $\alpha$ [S2, S5, S13, unofficial]. The mapping is defined by a transfer function $T(d) \rightarrow (\mathbf{c}, \alpha)$ [S2, S2789]. The opacity $\alpha$ is typically bounded to $[0, 1]$ [G]. The resulting opacity is often corrected for the discretization step size $\Delta t$ before compositing to ensure physical consistency, using the relation $\alpha' = 1 - (1 - \alpha)^{\Delta t}$ or similar approximations [S11, S32, unofficial]. The color $\mathbf{c}$ is usually independent of the step size [G].

In this family, the color $\mathbf{c}$ is not derived from lighting models applied to the volume's gradient, but is instead sampled from a 2D style texture (such as a lit sphere or matcap) using the surface normal as a coordinate. The opacity $\alpha$ may be modulated by illustrative constraints, such as curvature-based contour thresholds or gradient magnitudes, to simulate effects like the "100-percent-rule" where transparency falls off near edges [S1].

## Subtopics

### [Raycasting with Style Transfer and Contour Lines](11409-raycasting-with-style-transfer-and-contour-lines.md)
This method performs direct volume ray casting to render 3D scalar fields with artistic style transfer and illustrative contour lines. It solves the problem of making volumetric data visually expressive by mapping surface normals and material indices to 2D style textures (such as lit spheres or matcaps) and overlaying curvature-based contour lines. It differs from standard ray casting by replacing the final color accumulation with a style lookup, and differs from pure illustrative rendering by integrating the style transfer directly into the ray marching loop rather than as a post-process.

### [Illustrative Volume Rendering with Style Transfer](11445-illustrative-volume-rendering-with-style-transfer.md)
Illustrative Volume Rendering with Style Transfer combines standard direct volume ray casting with a Style Transfer Function (STF) to apply artistic styles to volumetric data. It allows users to load volumetric datasets, define transfer functions with assigned artistic styles, and interactively rotate or zoom the resulting volume. This method distinguishes itself from sibling techniques like [Raycasting with Style Transfer and Contour Lines](11409-raycasting-with-style-transfer-and-contour-lines.md) by focusing on the integration of style transfer textures directly into the ray casting shader pipeline, rather than post-processing or contour-based stylization.

### Illustrative Transparency
#### definition
- [S1] A transparency model combining uniform opacity ($\alpha_u$) from the transfer function and directional opacity ($\alpha_d$) from the style map, modulated by curvature and gradient magnitude to simulate the 100-percent-rule.
#### step
- [S1] If $\alpha_u = 1$, the sample opacity is solely determined by $\alpha_d$.
- [S1] If $\alpha_u = 0$, the sample is completely transparent.
- [S1] Compute the term $\max(0, |n \cdot v| - \sqrt{T \kappa_v (2 - T \kappa_v)})$, which is zero on the contour and increases away from it.
- [S1] Weight the result by $\alpha_d$ to apply the final opacity.
#### formula
- [S1] Overall opacity $\alpha = \alpha_d \cdot \alpha_u \cdot \frac{0.5 + \max(0, |n \cdot v| - \sqrt{T \kappa_v (2 - T \kappa_v)}) \cdot (1 - |g|)}{1}$, where $g$ is the gradient magnitude, $n$ is normal, $v$ is view vector, and $T \kappa_v (2 - T \kappa_v)$ is the contour threshold term.
#### parameter
- [S1] $\alpha_u$: Uniform opacity value from the transfer function, controlling overall opacity independent of normal direction.
- [S1] $\alpha_d$: Directional opacity tuple (r,g,b,$\alpha$) from the lit sphere map, varying opacity based on normal direction.
- [S1] 0.5: Constant restricting maximum opacity enhancement, empirically determined.

## Background
The concept of Style Transfer Functions (STF) for illustrative volume rendering was introduced by Stefan Bruckner and Eduard Gröller in 2007 [S1]. The paper "Style Transfer Functions for Illustrative Volume Rendering" proposes using 2D style textures (like lit spheres) mapped via surface normals to replace traditional lighting models, allowing for non-photorealistic visualization. The method also introduces illustrative transparency to enhance the perception of 3D structure by modulating opacity based on curvature and gradient magnitude, adhering to the "100-percent-rule" of illustration [S1].

## Sources
[S1] Style Transfer Functions for Illustrative Volume Rendering, https://doi.org/10.1111/j.1467-8659.2007.01095.x, fetched 2026-10-26
