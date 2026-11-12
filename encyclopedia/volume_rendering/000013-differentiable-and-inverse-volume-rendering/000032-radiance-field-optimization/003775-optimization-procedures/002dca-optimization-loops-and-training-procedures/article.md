# Optimization Loops and Training Procedures

Also called: Inverse Rendering Loop

## Overview
Optimization Loops and Training Procedures define the iterative algorithmic structure used to solve inverse rendering problems by minimizing the discrepancy between rendered images and ground-truth observations [S1, unofficial]. This family encompasses the complete cycle of data loading, forward differentiable rendering, loss computation, backpropagation, and parameter updates, serving as the engine for refining scene representations (geometry, appearance, lighting) and camera parameters [S1, S4, S5]. It sets itself apart from sibling methods like [Photometric Optimization](5790-photometric-optimization.md) by defining the procedural loop and scheduling rather than just the objective function, and from [Optimization Configuration](2133-optimization-configuration-adam-exponential-decay-selective-.md) by encompassing the full execution context including ray sampling and checkpointing [S11, S14, unofficial].

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

### [NeRF Training Loop](970-nerf-training-loop.md)
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth training views by updating the parameters of a continuous volumetric radiance field. It solves the inverse rendering problem by differentiating the volume rendering equation through automatic differentiation, enabling gradient descent on the neural network weights [44-radiance-field-optimization.md]. It sets itself apart from sibling methods like [Photometric Optimization](5790-photometric-optimization.md) by specifically structuring the loop around stochastic ray sampling and the coarse-to-fine dual-network architecture, and from [Optimization Configuration](2133-optimization-configuration-adam-exponential-decay-selective-.md) by encompassing the full cycle of data loading, forward rendering, loss computation, and parameter updates.

### [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md)
This configuration defines the standard hyperparameter setup for training neural radiance fields and implicit volume representations, combining the Adam optimizer with a staircase exponential learning rate decay and selective L2 weight decay. It addresses the problem of stabilizing gradient descent and preventing divergence in high-dimensional, ill-conditioned optimization landscapes typical of differentiable volume rendering. It sets itself apart from generic optimizers by explicitly disabling regularization on positional encodings and appearance embeddings while applying it to network weights, and by using a delayed decay schedule to allow initial convergence.

### [Neural Volume Rendering Training (MSE)](3429-neural-volume-rendering-training-mse.md)
This method defines the core optimization loop for training Neural Radiance Fields (NeRF) and similar implicit neural representations by minimizing the Mean Squared Error (MSE) between rendered images and ground truth observations. It addresses the problem of learning accurate density and color distributions from sparse or dense 2D image inputs without explicit 3D supervision. It sets itself apart from sibling methods like [NeRF Training Loop](970-nerf-training-loop.md) by explicitly incorporating Hierarchical Geometric Guidance (HGG) to initialize sampling regions based on sparse depth priors, thereby avoiding the geometric misalignment bias introduced by direct depth supervision.

### [Photometric Optimization](5790-photometric-optimization.md)
Photometric optimization is a training procedure for differentiable volume rendering that minimizes the pixel-wise difference between rendered images and ground-truth observations. It serves as the primary loss-driven mechanism for refining scene representations (geometry, appearance, lighting) and camera parameters by backpropagating gradients through the volume rendering equation. It sets itself apart from [Optimization Configuration](2133-optimization-configuration-adam-exponential-decay-selective-.md) by defining the objective function rather than the solver dynamics, and from [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md) by focusing on the specific loss formulation (photometric vs. multi-stage constraints) rather than the temporal scheduling of training phases.

### [Optimization Baselines](6675-optimization-baselines.md)
This family provides black-box optimization strategies for inverse rendering problems where the forward rendering operator $f$ is non-differentiable or gradient computation is prohibitively expensive. Unlike gradient-based siblings that rely on automatic differentiation or adjoint methods, these methods treat the renderer as an opaque function, minimizing a photometric loss $\mathcal{L}$ by iteratively sampling and evaluating parameter candidates. The family is distinguished by its reliance on population-based search distributions (typically Gaussian) that adapt to the local loss landscape without explicit gradient information, offering robustness against local minima and Monte Carlo noise.

### [DressiAD Optimization Loop](10728-dressiad-optimization-loop.md)
The DressiAD Optimization Loop is a define-and-run differentiable rendering optimization procedure that constructs a computational graph, marks variables for gradient tracking, and executes forward/backward passes using Stochastic Gradient Descent (SGD). It distinguishes itself from standard AD libraries by packing the backward pass into Vulkan command buffers, allowing the optimization loop to execute directly on the GPU without CPU-side backpropagation overhead.

### [Phase-Aware Gradient Descent (PAGD)](12876-phase-aware-gradient-descent-pagd.md)
Phase-Aware Gradient Descent (PAGD) is an optimization algorithm designed to encode complex wave fields into phase-only holograms for 3D holographic displays with continuous depth cues. It solves the problem of ringing artifacts and unnatural blur spreading inherent in amplitude-only or RGB-D holographic synthesis methods by leveraging phase information for supervision. Unlike amplitude-based approaches that rely on discrete multi-plane supervision, PAGD uses a camera-calibrated learned forward model to propagate the phase-only hologram, enabling accurate reconstruction of tilted wave fields and view-dependent occlusion.

### [Training & Optimization Protocol](12939-training-optimization-protocol.md)
This protocol defines the iterative optimization procedure for decomposed Gaussian splatting methods, specifically targeting distractor-free rendering by jointly optimizing static scene geometry and learned distractor points. It differs from standard NeRF training [970-nerf-training-loop.md] by explicitly modeling and optimizing distractor elements (e.g., transient objects, people) alongside the static scene, using a specialized densification schedule and per-parameter learning rates. It extends the base Splatfacto configuration [12945-staged-and-alternating-optimization.md] with specific regularization and distractor initialization strategies.

### [Staged and Alternating Optimization](12945-staged-and-alternating-optimization.md)
Staged and Alternating Optimization decomposes complex inverse rendering problems into sequential sub-problems to stabilize gradient descent and manage computational resources [4241-stage-based-differentiable-volume-radiance-field-training.md, 5832-staggered-optimization-scheme.md, 7387-two-stage-inverse-rendering-optimization.md]. Unlike joint optimization methods that update all parameters simultaneously, this family alternates between optimizing distinct parameter groups (e.g., geometry vs. material, or shape vs. blur) or progressively refines resolution [4241-stage-based-differentiable-volume-radiance-field-training.md, 5832-staggered-optimization-scheme.md, 7387-two-stage-inverse-rendering-optimization.md]. It distinguishes itself from single-stage radiance field methods by explicitly decoupling ill-posed variables, such as separating geometric proxy recovery from physically-based material refinement [7387-two-stage-inverse-rendering-optimization.md].

### 4DGS Baseline Implementation
Optimize the first frame for 3,000 iterations, then optimize the full multi-view video sequence for 14,000 iterations [S2].

### Geometry Focused Model Optimization
Initial Stage: Apply the rendering equation directly to each Gaussian primitive to compute outgoing radiance using its material and geometry properties, then alpha-blend during rasterization. This stabilizes gradient transfer to geometry in the initial phase [S3, unofficial].

### Continuous Coarse-To-Fine Optimization
A progressive optimization strategy that avoids shape-radiance ambiguity by restricting the Level of Detail (LOD) to coarse levels initially, then relaxing this regularization to recover scene details. It operates continuously in LOD space rather than discretely [S4, unofficial]. During cone casting, clamp the samples' LOD up to a maximal LOD $L$. Set grids of level $L + 1$ and above to zero and do not optimize them. The maximal LOD $L$ is updated per epoch using the linear scheduler: $L = (L_0 + n / n_{ctf}) \in \mathbb{R}^+$, where $n$ is the current epoch, $n_{ctf}$ is a hyperparameter describing the speed of the process, and $L_0$ defines the number of used grids at initialization [S4, unofficial]. Unlike I-NGP-based architectures where keeping grid values to zero leaves some decoder latent space dimensions unsupervised (causing global weight refinement issues when new grids are optimized), this method keeps supervising all decoder latent space dimensions. Adding new grid levels only adds degrees of freedom for the mapping function without modifying decoder weights or previously trained grids [S4, unofficial].

## Background
The concept of using differentiable renderers within optimization loops originates from the broader field of inverse rendering. As noted by Bai Liping, while the loop itself is simple—initializing parameters, tracing rays, computing loss, and updating via gradients—the difficulty lies in choosing a faithful model, identifiable parameters, a stable estimator, and good initialization [S1, unofficial]. The specific application of these loops to Neural Radiance Fields (NeRF) and subsequent implicit representations has driven the development of specialized training procedures, such as the coarse-to-fine strategies and selective regularization techniques described in the subtopics.

## Sources
[S1] Differentiable ray tracing — from paths to gradients | Bai Liping, https://bailiping.com/differentiable-ray-tracing/, fetched 2026-10-28 (unofficial)
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
[S3] Reflective Gaussian Splatting, https://doi.org/10.48550/arxiv.2412.19282, fetched 2026-10-26 (unofficial)
[S4] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
