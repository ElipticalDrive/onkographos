# Gradient Estimation and Smoothing

Also called: Gradient-based rendering, Non-local Approximated Gradients, Smooth Gradient Estimation via Convolution

## Overview
Gradient estimation and smoothing techniques address discontinuities in differentiable rendering that cause vanishing gradients. This family modifies gradient computation by smoothing the integrand to permit differentiation under the integral sign, approximating gradients via non-local spatial statistics, or applying axis-aligned filters [6449-smooth-gradient-estimation-via-convolution.md, 6784-non-local-approximated-gradients.md, 7600-axis-aligned-neighbor-smoothing.md].

## Shared principles
Methods in this family either smooth the rendering integrand via convolution to allow Leibniz's rule to exchange differentiation and integration, or approximate gradients using differential filters on rendered images to propagate gradients beyond immediate primitive boundaries [6449-smooth-gradient-estimation-via-convolution.md, 6784-non-local-approximated-gradients.md]. Both aim to provide non-zero, directed gradients that guide optimization away from local minima [6449-smooth-gradient-estimation-via-convolution.md, 6784-non-local-approximated-gradients.md].

## Subtopics

### [Smooth Gradient Estimation via Convolution](6449-smooth-gradient-estimation-via-convolution.md)
This method enables the computation of correct gradients for differentiable volume rendering by smoothing the rendering integrand via convolution, allowing the differential operator to move inside the integral via Leibniz's rule. It solves the problem where naive automatic differentiation yields incorrect gradients because the integrand is discontinuous with respect to scene parameters, causing classic gradients to be zero almost everywhere except at overlaps. It sets itself apart from [Non-local Approximated Gradients](6784-non-local-approximated-gradients.md) by explicitly enforcing smoothness through convolution to permit differentiation under the integral sign, rather than approximating gradients via non-local statistics. See [Smooth Gradient Estimation via Convolution](6449-smooth-gradient-estimation-via-convolution.md).

### [Non-local Approximated Gradients](6784-non-local-approximated-gradients.md)
Non-local approximated gradients are a gradient estimation technique for differentiable volume and mesh rendering that computes pixel derivatives with respect to vertex positions using differential filters, allowing gradient flow from pixels outside the immediate primitive boundary. Unlike standard differentiable rendering which only propagates gradients through covered pixels, this method enables optimization objectives to influence vertices via boundary pixels, addressing the issue of poor local minima caused by the "localness" of standard rasterizer derivatives. It sets itself apart from sibling methods like [Smooth Gradient Estimation via Convolution](6449-smooth-gradient-estimation-via-convolution.md) by explicitly leveraging backpropagated loss gradients to make the gradient flow objective-aware, rather than relying solely on geometric or intensity-based smoothing. See [Non-local Approximated Gradients](6784-non-local-approximated-gradients.md).

### Gradient-Based Rendering
#### definition
- [S1] Introduced for homogeneous participating media by Gruson et al., taking advantage of the smoothness characteristics of rendered participating media to obtain smoother results than the original method.

#### step
- [S1] Four different gradient-based estimation methods were introduced: point to point, point to beam, beam to beam, and beam to light plane.

#### number
- [S1] With gradient domain rendering, smoother results are obtained than the original method.

#### comparison
- [S1] Deep learning based reconstruction yields a better reconstruction solution than Poisson reconstruction.

### [Axis-Aligned Neighbor Smoothing](7600-axis-aligned-neighbor-smoothing.md)
#### definition
- [G] A weighted smoothing filter that averages color values along a single axis using depth, normal, and color similarity to preserve edges and reduce noise.

#### step
- [G] For each neighbor offset `i` in `[-R, R]` along `axis`: compute `pos = frag_pos` with `pos[axis] += i`. Load `normal_depth2` and `color2`. Compute weights `wz`, `wn`, `wd`, `wc`. Compute `w = wn * wz * wd * wc`. Accumulate `result += color2 * w` and `sum_w += w`. Finally, `result *= 1.0 / sum_w`.

#### formula
- [G] Total weight $w = w_n \cdot w_z \cdot w_d \cdot w_c$, where $w_z = \exp(-|d - d'| \cdot 0.1 / (\text{fwidth}(d) + 0.01))$, $w_n = |\mathbf{n} \cdot \mathbf{n}'|$, $w_d = \exp(-i^2 \cdot 0.005)$, $w_c = \exp(-\|\mathbf{c} - \mathbf{c}'\|^2 \cdot 2.0)$.

#### parameter
- [G] `R: i32 = 7` (radius), `WINDOW: i32 = R * 2 + 1` (window size). Constants: `0.1` (depth weight scale), `0.01` (depth fwidth epsilon), `0.005` (distance weight scale), `2.0` (color weight scale).

#### layout
- [G] Inputs: `color_texture: texture_2d<f32>` (binding 0), `normal_depth_texture: texture_2d<f32>` (binding 1). Output: `vec4f` at location 0.

#### pitfall
- [G] Division by zero in depth fwidth term is avoided by adding `0.01`. If `sum_w` is zero, result remains zero.

## Background
Gradient estimation techniques in rendering have evolved to address the limitations of standard differentiable rendering. The concept of gradient-based rendering was introduced for homogeneous participating media by Gruson et al., leveraging the smoothness characteristics of multiple scattering [S1]. Frequency analysis of light transport was introduced for prefiltering and adaptive sampling in surface rendering, and later extended to participating media by Belcour et al., based on the observation that multiple scattering leads to a smooth appearance [S1]. More recently, deep learning-based reconstruction approaches have been proposed for participating media, yielding better reconstruction solutions than traditional Poisson reconstruction methods [S1].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
