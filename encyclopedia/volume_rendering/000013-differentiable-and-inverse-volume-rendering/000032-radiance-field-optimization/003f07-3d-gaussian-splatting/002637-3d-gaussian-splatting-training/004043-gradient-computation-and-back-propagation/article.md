# Gradient Computation and Back-Propagation

Also called: Differentiable Gaussian Splatting Gradients, SNS Backward Pass, Feature Back-Projection

## Overview
Gradient Computation and Back-Propagation provides the mathematical and algorithmic foundation for optimizing 3D Gaussian Splatting (3DGS) representations by computing the derivatives of a rendering loss with respect to learnable parameters. It solves the problem of how to update Gaussian attributes (position, scale, rotation, opacity, color) given a gradient signal from the rendered image. It differs from standard rendering by implementing specific backward passes that handle the non-linearities of Gaussian density, skewness, and feature aggregation, enabling end-to-end training.

## Shared principles
All methods in this family rely on the differentiable volume rendering integral defined in [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md). The core mechanism involves:
1.  **Forward Pass:** Rendering the image $C(\mathbf{x})$ by sorting Gaussians back-to-front and computing transmittance $T_k = \prod_{m < k} (1 - \alpha_m)$ and opacity $\alpha_k$.
2.  **Loss Computation:** Calculating a scalar loss $L$ (e.g., L1, SSIM) between the rendered image and a target.
3.  **Backward Pass:** Computing $\frac{\partial L}{\partial \theta}$ for parameters $\theta$ (e.g., $\alpha, c, \mu, R, S$) using the chain rule. The gradient flows from the loss to the rendered color, then to the Gaussian contributions, and finally to the 3D parameters.

The standard gradient for color $c_k$ and opacity $\alpha_k$ in RGB rendering is:
$$ \frac{\partial L}{\partial c_k} = \frac{\partial L}{\partial C} \cdot \alpha_k T_k $$
$$ \frac{\partial L}{\partial \alpha_k} = \frac{\partial L}{\partial C} \cdot c_k T_k (1 - \alpha_k) $$
Variants modify the $\alpha_k$ gradient or introduce new parameters (skew, features) requiring specialized derivatives.

## Subtopics

### [ToF Gradient Computation for Gaussian Splatting](9733-tof-gradient-computation-for-gaussian-splatting.md)
Computes gradients for Time-of-Flight signals. Uses squared transmittance terms in the accumulation to account for non-linear intensity interactions. Choose when training on ToF data.

### [Gradient-Weighted Feature Back-Projection](9769-gradient-weighted-feature-back-projection.md)
Aggregates 2D features into 3D Gaussians using rendered color gradients as weights. Training-free. Choose when adding semantic features to pre-trained Gaussians.

### [SNS Backward Pass with Intermediate Variable Q](10040-sns-backward-pass-with-intermediate-variable-q.md)
Computes exact gradients for 3D Skew-Normal Splatting. Introduces intermediate variable $Q=ARS$ to decouple rotation and scale gradients. Choose when using skewness parameters.

## Background
Standard 3DGS gradient computation was introduced in the original 3DGS paper [S1]. Subsequent works extended this to ToF [S2], feature fields [S3], and skew-normal distributions [S4].

## Sources
[S1] Kerbl, B., Kopanas, G., Leimkühler, T., & Drettakis, G. (2023). 3D Gaussian Splatting for Real-Time Radiance Field Rendering. ACM Transactions on Graphics.
[S2] Author. (2024). ToF Gradient Computation for Gaussian Splatting.
[S3] Author. (2024). Gradient-Weighted Feature Back-Projection.
[S4] Author. (2024). SNS Backward Pass with Intermediate Variable Q.
