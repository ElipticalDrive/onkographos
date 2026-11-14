# Spherical Harmonics Optimization

Also called: SH Optimization, SH Degree Reduction, SH Distillation, SH Vector Quantization

## Overview
Spherical Harmonics (SH) Optimization in 3D Gaussian Splatting addresses the high memory bandwidth and computational costs associated with storing and evaluating high-degree SH coefficients for view-dependent appearance. It encompasses techniques to reduce parameter count, accelerate training, and compress storage by modifying the SH degree, distilling knowledge from higher to lower orders, or quantizing coefficients into shared palettes. It differs from general rendering optimizations by focusing specifically on the representation and optimization dynamics of the view-dependent color attributes, distinct from [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md) which targets rasterization efficiency.

## Shared principles
The 3D Gaussian Splatting representation encodes view-dependent color using SH coefficients. For a maximum degree $L$, each color channel (R, G, B) stores $(L+1)^2$ coefficients. The total number of SH coefficients per Gaussian is $3(L+1)^2$.

The rendered color $\mathbf{C}$ for a Gaussian at a viewing direction $\mathbf{v}$ is computed by projecting $\mathbf{v}$ onto the SH basis functions $\mathbf{Y}(\mathbf{v})$ and dotting with the coefficient vector $\mathbf{c}$:
$$
\mathbf{C}(\mathbf{v}) = \sum_{l=0}^{L} \sum_{m=-l}^{l} c_{l,m} Y_{l,m}(\mathbf{v})
$$
where $c_{l,m}$ are the SH coefficients and $Y_{l,m}$ are the real SH basis functions.

Optimization strategies typically operate on the coefficient vector $\mathbf{c}$ or the loss landscape associated with it. Common principles include:
1.  **Degree Truncation:** Reducing $L$ to decrease the dimensionality of $\mathbf{c}$ from $3(L+1)^2$ to $3(L'+1)^2$.
2.  **Distillation:** Using a high-degree model as a teacher to guide the optimization of a low-degree student, minimizing the difference in rendered outputs rather than coefficient values directly.
3.  **Quantization:** Replacing continuous coefficient vectors with discrete indices from a learned codebook (palette).

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
First-order Spherical Harmonics Distillation reduces the SH degree from third-order ($L=3$, 16 coefficients per channel) to first-order ($L=1$, 4 coefficients per channel) to enable efficient rendering on mobile devices. Unlike simple truncation, this method uses knowledge distillation where a pretrained third-order teacher model guides the optimization of a first-order student model, preserving visual fidelity despite the significant reduction in representational capacity. It builds on [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md) by replacing fixed basis truncation with a learned approximation. The student model's coefficients are optimized to minimize a combined color and scale-invariant depth distillation loss. See [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md).

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
This method derives analytical Jacobians of the SE(3) camera pose with respect to 3D Gaussian primitives to enable efficient gradient-based camera tracking without the overhead of automatic differentiation. It solves the problem of computationally expensive view synthesis gradients in SLAM systems by providing minimal parameterization that matches the degrees of freedom of the Lie group, eliminating redundant computations inherent in standard matrix derivatives. Unlike prior ray-intersection methods that assume fewer Gaussians, this approach is tailored for EWA splatting and 3D Gaussian Splatting (3DGS) rasterization. See [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md).

### [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md)
This method enables multi-view mini-batch training for 3D Gaussian Splatting (3DGS) to reduce gradient variance and stabilize optimization, addressing the limitation that standard 3DGS rasterization forces single-view training due to high overhead. It achieves this by replacing naive multi-view rendering with a thread-efficient partial rendering scheme that eliminates idle GPU threads, and by modifying the Adaptive Density Control (ADC) to handle gradients from multiple image planes. Unlike standard 3DGS which processes one image per iteration, this approach renders multiple views simultaneously, leveraging the statistical benefit that multi-view mini-batches exhibit smaller gradient variance than single-view ones. See [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md).

### [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md)
Spherical Harmonics Vector Quantization (SH VQ) is a compression technique for 3D Gaussian Splatting that replaces per-splat floating-point SH coefficients with indices pointing to a shared palette of distinct SH vectors, significantly reducing memory footprint at the cost of minor lighting artifacts. It sets itself apart from other optimization methods by treating the high-dimensional SH data as a clustering problem rather than a per-pixel or per-splat quantization problem, allowing for shared representation across the entire scene. See [Spherical Harmonics Vector Quantization](14181-spherical-harmonics-vector-quantization.md).

#### SH Degree Zero Optimization
**Definition:** Setting the maximum degree of Spherical Harmonics (SH) coefficients to 0 to remove superfluous operations and improve training speed, as higher-degree SH coefficients for color remain unchanged during training [S1, unofficial].
**Step:** Modify the model code to delete operations relative to the max degree in the model and set the max degree parameter to 0 [S1, unofficial].
**Pitfall:** In the original 3D Gaussian Splatting model, features represented by SH coefficients with degree > 0 are unchanged during all training, making them useless [S1, unofficial].
**Number:** Directly changing the max degree to 0 yields a 16.25% improvement compared to the default. With specific changing code, it yields a 16.57% improvement but increases CPU-GPU process time [S1, unofficial].

## Background
3D Gaussian Splatting (3DGS) was introduced by Kerbl et al. in 2023, utilizing SH coefficients for view-dependent rendering [S1, unofficial]. Subsequent research identified inefficiencies in the standard SH usage, particularly regarding the static nature of higher-order coefficients during optimization and the high memory cost of third-order SH. The I3DGS paper (2024) highlighted that higher-degree SH coefficients for color often remain unchanged during training, suggesting that degree reduction or zeroing could improve efficiency without quality loss [S1, unofficial]. Further optimizations include distillation techniques for mobile deployment and vector quantization for storage compression.

## Sources
[S1] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S2] First-order Spherical Harmonics Distillation for Mobile 3D Gaussian Splatting, https://github.com/ashusmall/first-order-sh-distillation, fetched 2026-10-26 (unofficial)
