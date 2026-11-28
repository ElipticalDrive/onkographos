# Optimization Schedule and Densification

Also called: Densification Scheduling, Primitive Growth Control

## Overview
This family of methods controls the dynamic growth, pruning, and optimization trajectory of 3D Gaussian primitives during training. It solves the problem of inefficient spatial coverage and unstable convergence by synchronizing primitive density with rendering resolution and learning progress. It sets itself apart from static initialization or uniform densification by employing adaptive schedules that accelerate growth over time, ensuring high-frequency details are fitted only after low-frequency structures are established [Parent].

## Shared principles
All methods in this family operate on the standard 3DGS primitive representation:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$
*   Scale $\mathbf{s}_i \in \mathbb{R}^3$
*   Rotation $\mathbf{q}_i \in \mathbb{R}^4$ (quaternion)
*   Opacity $\alpha_i \in [0, 1]$
*   Color coefficients $\mathbf{c}_i$ (Spherical Harmonics)

The core mechanism involves calculating an importance score for each Gaussian and removing those below a threshold or ranking. Methods typically employ a "concave-up" growth curve for primitives, where growth accelerates over time, synchronized with increasing rendering resolution to fit high-frequency details only after low-frequency structures are established [Parent].

## Subtopics

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
[Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md) is a densification strategy that samples and optimizes 2D Gaussian kernels in image space using a differentiable rasterizer, then back-projects them to 3D to initialize new 3D primitives. It solves the problem of initializing view-dependent Gaussian kernels for complex specular radiance reconstruction, where proper initialization is critical. It sets itself apart from standard 3D densification by optimizing in 2D image space with the pre-rendered image as a fixed background, allowing for precise error-driven placement before 3D projection.

### [Training Configuration and Hyperparameters](16385-training-configuration-and-hyperparameters.md)
[Training Configuration and Hyperparameters](16385-training-configuration-and-hyperparameters.md) defines the specific hyperparameters, optimization schedules, and training protocols required to stabilize and accelerate the convergence of 3D Gaussian Splatting (3DGS) decoders. While the parent method [Optimization Schedule and Densification](15107-optimization-schedule-and-densification.md) handles the dynamic growth of primitives, this configuration ensures that the learning rate, loss weights, and initialization strategies are tuned to prevent gradient instability, particularly when integrating semantic features or handling sparse view inputs. It addresses the problem of chaotic densification and poor spatial convergence by decoupling geometric and semantic optimization phases and standardizing downstream evaluation protocols.

### [Optimization Scheduling and Complexity Control](19900-optimization-scheduling-and-complexity-control.md)
[Optimization Scheduling and Complexity Control](19900-optimization-scheduling-and-complexity-control.md) focuses on accelerating the training process by managing the computational complexity of densification and resolution scaling. It addresses the inefficiency of standard split/clone operations and the unbalanced workloads inherent in distributed rendering. It sets itself apart by decoupling global spread from local refinement and employing momentum-based primitive budgeting to suppress early growth and encourage mid-stage expansion, thereby reducing optimization time without compromising quality.

### QuickSplat Optimization Process
An optimization framework for 3D Gaussian Splatting that uses learned networks for initialization, densification, and parameter updates to accelerate surface reconstruction [S1]. It iterates for $t = 0$ to $T - 1$, resetting the gradient accumulator $\nabla G_t \leftarrow 0$. For all images, it computes the rendering loss $L$, computes the gradient $\delta G_t$, and accumulates $\nabla G_t \leftarrow \nabla G_t + \delta G_t$. It generates new Gaussians $\hat{G}_t$ using a densifier network $f_D$ on current Gaussians $G_t$, accumulated gradients $\nabla G_t$, and time step $t$: $\hat{G}_t \leftarrow f_D(G_t, \nabla G_t, t)$. The new Gaussians are concatenated with existing ones, and an optimizer network $f_O$ computes the update $\Delta \bar{G}_t \leftarrow f_O(\bar{G}_t, \nabla \bar{G}_t, t)$, where $\bar{G}_t = G_t \cup \hat{G}_t$ and $\nabla \bar{G}_t = \nabla G_t \cup 0$. The final state is updated as $G_{t+1} \leftarrow \bar{G}_t + \Delta \bar{G}_t$ [S1].

### Normal Gradient Based Densification
A densification criterion to improve normal recovery in thin regions by introducing an additional densification criterion on the gradient of normals, beyond the vanilla 3DGS gradient of view space points [S2]. It densifies Gaussians whose normal gradient exceeds a threshold $T_n$. The typical value for $T_n$ is $2 \times 10^{-9}$ [S2].

### Global-to-local densification
A densification strategy that decouples split and clone operations across densification phases to accelerate optimization. It uses split for global spread at low resolution and suppresses clone, then reintroduces clone for local refinement at full resolution [S3, unofficial]. The norm of the average position gradient of each Gaussian primitive is calculated every 100 iterations. If the gradient norm exceeds a predefined threshold, the method checks the maximum scale of the Gaussian primitive. If the maximum scale exceeds a given scale threshold, it splits the Gaussian into smaller components; otherwise, it clones the Gaussian with the same parameters. During low-resolution training, split operations are promoted for global spread while clone operations are suppressed to prevent premature local clustering. When transitioning to full-resolution training, clone operations are reintroduced to refine high-frequency details [S3, unofficial]. Unlike standard 3DGS which applies split and clone simultaneously, this method decouples them across phases (global vs local) to address inefficient spatial spread and redundant primitives [S3, unofficial].

### Resolution Guided Primitive Scheduler
A scheduler that controls the number of primitives using a momentum-based budgeting scheme. The final primitive count $P_{fin}^{(i)}$ is calculated as:
$$ P_{fin}^{(i)} = \max(P_{fin}^{(i-1)}, \gamma \cdot P_{fin}^{(i-1)} + \eta \cdot P_{add}) $$
where $P_{add}$ is the number of primitives densified in step $i$. $P_{fin}^{(0)}$ is initialized as $5 \cdot P_{init}$. The power factor in primitive growth starts at 2 and gradually decreases to 1 along the optimization process to suppress early growth and encourage mid-stage growth. The hyperparameter $\gamma$ (momentum decay factor) defaults to 0.98, and $\eta$ (force scaling factor) defaults to 1. The continuous scalar resolution modulation $r(i)$ is modulated to $\lfloor r(i) \rfloor$ in practice to encourage faster primitive growth. Too much primitive growth at the end of optimization results in poor convergence, which is mitigated by the concave-up scheduling function and momentum-based budgeting [S4].

### Loss Function Ablation
An ablation study on additional losses added in the first stage of the optimization pipeline. Removing planarity loss ($L_{planarity}$) causes Gaussians to form radial spikes pointing toward the scene's center, resulting in noisier reconstruction and degraded perceptual quality. Turning off shell loss ($L_{shell}$) relaxes geometric constraints on background Gaussians, resulting in a slightly increased SSIM compared to planarity ablation, but introduces blurry backgrounds and decreased visual fidelity [S5].

### Custom Pruning for Background
A pruning strategy that disables the original size-based pruning heuristic (which deletes large Gaussians in screen/world space) to preserve large distant background Gaussians. Instead, it counts the times each Gaussian is rendered using indices returned from the rendering step. Any Gaussian with a visibility count equal to zero is removed. This logic is executed in the same pruning stage as in the original code [S5].

### Handling Dynamic Unbalanced Workloads
A phenomenon in distributed 3DGS where render intensity varies due to natural scene sparsity, varying Gaussian densities (e.g., sky vs. tree), and dynamic changes during training. Workload changes are caused by 3D position and co-variance parameter updates affecting Gaussian coverage of pixels, and opacity parameter updates affecting the number of Gaussians contributing to each pixel. Densification targets areas under construction, causing Gaussians from different regions to densify at varying rates, making workloads more unbalanced at the beginning of training [S6].

### Keyframe Selection and Offloading for Online Optimization
A strategy for online optimization that samples keyframes using a probability proportional to the number of leaf Gaussians currently in the active set that were generated by that keyframe. It removes keyframes that contribute less than 5% of the maximum contribution by offloading them to the CPU, which automatically optimizes intermediate hierarchy nodes without explicit post-optimization. The minimum keyframe count is 50 to ensure sufficient context and allow optimization of intermediate hierarchy nodes. Frames that mostly see offloaded primitives or keyframes that only see leaf/small Gaussians are avoided by the probability-based selection and offloading scheme [S7].

## Background
The concept of adaptive densification in 3D Gaussian Splatting originated with the foundational work by Kerbl et al. [Parent], which introduced split and clone operations based on position gradients. Subsequent research has focused on optimizing this schedule to reduce training time, improve geometric fidelity, and handle specific domains like background reconstruction, semantic segmentation, and online rendering. Key developments include the decoupling of split/clone operations for efficiency [S3, unofficial], momentum-based primitive budgeting for controlled growth [S4], and learned initialization/densification networks [S1].

## Sources
[S1] QuickSplat: Fast 3D Surface Reconstruction via Learned Gaussian Initialization, https://doi.org/10.1109/ICCV51701.2025.02585, fetched 2026-10-29
[S2] Relightable 3D Gaussians: Realistic Point Cloud Relighting with BRDF Decomposition and Ray Tracing, https://doi.org/10.1007/978-3-031-72995-9_5, fetched 2026-10-26
[S3] Decomposing Densification in Gaussian Splatting for Faster 3D Scene Reconstruction, https://doi.org/10.48550/arXiv.2507.20239, fetched 2026-10-30 (unofficial)
[S4] DashGaussian: Optimizing 3D Gaussian Splatting in 200 Seconds, https://doi.org/10.1109/cvpr52734.2025.01041, fetched 2026-10-31
[S5] EnvMap-GS: two-stage outdoor Gaussian reconstruction with background-to-environment map baking, https://doi.org/10.1007/s00371-026-04593-w, fetched 2026-10-29
[S6] On Scaling Up 3D Gaussian Splatting Training, https://doi.org/10.1007/978-3-031-91989-3_2, fetched 2026-10-26
[S7] Immediate 3D Gaussian Splat Reconstruction of Unordered Input with Global Consistency, https://doi.org/10.1145/3799902.3811167, fetched 2026-10-26
