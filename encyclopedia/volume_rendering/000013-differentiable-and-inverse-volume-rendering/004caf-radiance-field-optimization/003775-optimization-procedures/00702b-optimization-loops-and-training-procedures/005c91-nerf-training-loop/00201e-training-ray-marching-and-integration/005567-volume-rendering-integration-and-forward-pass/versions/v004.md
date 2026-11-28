# Volume Rendering Integration and Forward Pass

Also called: Alpha Compositing, Volume Rendering Integral Discretization, Ray Marching Integration

## Overview
This family defines the forward pass that synthesizes a 2D image pixel color from a 3D implicit scene representation by integrating radiance along camera rays [G]. It solves the problem of mapping a continuous 5D implicit function to a 2D image by discretizing the ray integral [G]. It sets itself apart from sibling methods like [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by focusing strictly on the geometric integration and compositing logic rather than loss computation or metric evaluation [G].

## Shared principles
All methods in this family compute the expected color $\hat{C}(r)$ for a ray $r$ by discretizing the medium interaction interval into steps and accumulating emission weighted by transmittance [G]. The core mathematical formulation relies on the Beer-Lambert law to convert volume density $\sigma$ into opacity $\alpha$ and transmittance $T$ [G].

The discrete volume rendering equation is:
$$ \hat{C}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$

Where:
*   $N$ is the number of samples along the ray [G].
*   $\sigma_i$ is the volume density at sample $i$ [G].
*   $c_i$ is the emitted color at sample $i$ [G].
*   $\delta_i$ is the distance between sample $i$ and $i+1$ [G].
*   $T_i$ is the accumulated transmittance up to sample $i$:
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) = \prod_{j=1}^{i-1} (1 - \alpha_j) $$
*   $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity of sample $i$ [G].

**Numerical Stability:**
To prevent NaNs during gradient computation, a small epsilon $\epsilon$ (typically $10^{-10}$) is added to $(1 - \alpha_j)$ before the cumulative product or exponential calculation [G].

**Background Handling:**
If a background color $c_{bg}$ (typically white $(1,1,1)$) is assumed, the final color includes the contribution of rays that pass through the scene without terminating:
$$ \hat{C}_{final}(r) = \hat{C}(r) + \left(1 - \sum_{i=1}^{N} w_i\right) c_{bg} $$
where $w_i = \alpha_i T_i$ is the weight of sample $i$ [G].

**Depth Estimation:**
Expected depth $\hat{D}(r)$ is computed analogously to color:
$$ \hat{D}(r) = \sum_{i=1}^{N} w_i t_i $$
where $t_i$ is the distance of sample $i$ from the camera origin [G].

## Subtopics

### [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md)
NeRF Rendering (Alpha Compositing) is the forward pass method that synthesizes a 2D image pixel color from a 3D implicit scene representation by integrating radiance along camera rays. It solves the problem of differentiable volume rendering by approximating the continuous volume rendering integral as a discrete weighted sum, where weights are determined by the probability of the ray terminating at each sample point. It sets itself apart from sibling methods like [NeRF-style Volume Rendering Integrator](5964-nerf-style-volume-rendering-integrator.md) by explicitly defining the standard multi-layer perceptron (MLP) query pipeline, stratified sampling strategy, and the specific alpha-compositing formulation ($\alpha_i = 1 - \exp(-\sigma_i \delta_i)$) that underpins the original NeRF architecture.

### [NeRF-style Volume Rendering Integrator](5964-nerf-style-volume-rendering-integrator.md)
This method computes the expected color $\hat{C}(r)$ for a ray $r$ by discretizing the medium interaction interval into fixed steps and accumulating emission weighted by transmittance. It differs from its sibling [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by using a fixed-step marching loop with explicit throughput tracking and jittering, rather than bin-based stratified sampling. It is designed for dense grid-based media where direction-dependent emission is not required.

### NeRF Forward Pass with Alpha Masking and Two-Stage Shading
This variant extends the standard forward pass by incorporating alpha masking and NDC (Normalized Device Coordinates) ray sampling. It handles rays that hit empty space in an alpha mask by marking them invalid [S1, unofficial]. It adjusts distances by ray norm and normalizes view directions if NDC simulation is enabled [S1, unofficial]. Density features are converted to sigma using `feature2density` if not component-wise, and alpha weights are computed using `raw2alpha` with distance scaling [S1, unofficial]. The depth map is computed as a weighted sum of z-values, adjusted by background weight and ray direction Z component, and clamped if NDC Euclidean depth simulation is enabled [S1, unofficial].

## Background
Not given in the sources.

## Sources
[S1] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
