# Multi-Sensor and Physical Priors

Also called: Multi-Sensor Priors, Physical Priors in Radiance Fields

## Overview
This family extends differentiable volume rendering by integrating physical sensor models and geometric priors to solve inverse problems where optical assumptions fail or data is sparse. It differs from standard radiance field optimization by replacing or augmenting the emission-absorption model with physics-based signal formation equations (e.g., rigid motion blur, radar cross-sections) [8273-dp-nerf.md][8904-radar-fields.md].

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [44-radiance-field-optimization.md]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [44-radiance-field-optimization.md].

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [44-radiance-field-optimization.md]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [44-radiance-field-optimization.md].

Differentiation strategies fall into three main categories [44-radiance-field-optimization.md]:
1. **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [44-radiance-field-optimization.md].
2. **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [44-radiance-field-optimization.md].
3. **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [44-radiance-field-optimization.md].

## Subtopics

### [DP-NeRF](8273-dp-nerf.md)
DP-NeRF reconstructs sharp 3D scenes from blurry inputs (motion/defocus blur) using a Rigid Blurring Kernel (RBK) and Adaptive Weight Proposal (AWP) [8273-dp-nerf.md]. It ensures 3D consistency by modeling blur as rigid camera motion, unlike 2D pixel-wise deblurring [8273-dp-nerf.md]. Choose when inputs are calibrated but blurry. [8273-dp-nerf.md]

### [Radar Fields](8904-radar-fields.md)
Radar Fields renders Synthetic Aperture Radar (SAR) imagery by integrating signal contributions across rays in an azimuth plane sharing the same range distance [8904-radar-fields.md]. It enables 3D geometry learning from SAR data, unlike optical fields. [8904-radar-fields.md]

### NeuRAD
NeuRAD is a neural rendering method for Autonomous Driving (AD) with sensor models (camera + LiDAR) [S1, unofficial].

## Background
Originating from the need to handle non-ideal imaging conditions and multi-modal data, this family bridges computer graphics and remote sensing. DP-NeRF addresses motion blur in NeRFs [8273-dp-nerf.md], while Radar Fields adapts radiance fields for SAR tomography [8904-radar-fields.md]. NeuRAD targets autonomous driving simulation [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
