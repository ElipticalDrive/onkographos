# Training Pipeline

Also called: 3DGS Learning, Full Training Procedure, G Splats, Improved 3D Gaussian Splatting Optimization, Inner 3D Gaussian Splatting Training, Iterative optimization loop, Main Training Loop

## Overview
The Training Pipeline is the iterative optimization loop for 3D Gaussian Splatting that initializes a set of 3D Gaussians and optimizes their parameters (position, scale, rotation, opacity, and color) via gradient descent [S1][S2, unofficial]. It solves the problem of reconstructing a 3D scene from 2D observations by dynamically adapting the Gaussian representation through pruning, cloning, and splitting [S1]. It is distinguished from its siblings by its core structure: a nested loop over spatial axes and slices that accumulates gradients across multiple views before a single optimizer step, coupled with periodic refinement phases that modify the Gaussian count [S1].

## How it works
The pipeline operates as a standard differentiable rendering training loop, but with specific data organization (axis-aligned slices) and dynamic topology management [S1].

**Inputs and Outputs:**
*   **Input:** A set of training slices $D = \{(I_i^{\text{axis}}, \pi_i^{\text{axis}})\}$, where $\text{axis} \in \{x, y, z\}$, $I$ is a 2D slice image, and $\pi$ is the slice position [S1].
*   **Input:** Initial Gaussian parameters $\theta$, typically initialized on a regular 3D grid of resolution $g_{\text{res}}$ or from SfM points [S2, unofficial].
*   **Input:** Hyperparameters: opacity threshold $\tau_\alpha$, gradient threshold $\tau_p$, scale threshold $\tau_s$, and maximum iterations $T_{\text{max}}$ [S1].
*   **Output:** Optimized set of Gaussians $G = \{(\boldsymbol{\mu}, \boldsymbol{\Sigma}, \mathbf{c}, \alpha)\}$ [S1].

**Data Structures:**
*   **Gaussian State ($G$):** Each Gaussian $n$ is defined by mean $\boldsymbol{\mu}_n$, covariance $\boldsymbol{\Sigma}_n$, color $\mathbf{c}_n$, and opacity $\alpha_n$. The covariance is decomposed into rotation $R_n$ and scale $S_n$ such that $\boldsymbol{\Sigma}_n = R_n S_n S_n^\top R_n^\top$ [S1].
*   **Optimizer State:** Adam optimizer state for parameters $\theta$ [S1].
*   **Loss Accumulators:** Scalar $L_{\text{total}}$ and gradient tensor $\nabla_\theta$ [S1].

**Algorithm Steps:**

1.  **Initialization:** Build a regular grid $P$ of size $g_{\text{res}}$ (or use SfM points). Initialize each Gaussian with mean $\boldsymbol{\mu}$ around grid points and default values for scale $s$, color $c$, opacity $\alpha$, and quaternion $q$ [S2, unofficial].
2.  **Optimizer Setup:** Initialize the Adam optimizer with parameters $\theta$ and learning rate $\eta$ [S2, unofficial].
3.  **Outer Loop:** Iterate from step $t = 1$ to $T_{\text{max}}$.
4.  **Reset Accumulators:** Set total loss $L_{\text{total}} \leftarrow 0$ and zero gradients $\nabla_\theta \leftarrow 0$ [S2, unofficial].
5.  **Axis Loop:** Iterate over axes $\text{axis} \in \{x, y, z\}$ [S2, unofficial].
6.  **Slice Loop:** For each slice $(I_i^{\text{axis}}, \pi_i^{\text{axis}})$ in the dataset $D_{\text{axis}}$:
    a.  **Covariance Computation:** Compute 3D covariance $\boldsymbol{\Sigma}_n \leftarrow R_n S_n S_n^\top R_n^\top$ [S2, unofficial].
    b.  **Rendering:** Rasterize the Gaussian parameters to predict the image $I_{\text{pred}} \leftarrow \text{Rasterize}(\boldsymbol{\mu}_n, \boldsymbol{\Sigma}_n, \alpha_n, \mathbf{c}_n)$ [S2, unofficial].
    c.  **Loss Computation:** Compute loss $L$ between predicted image $I_{\text{pred}}$ and ground truth slice $I_i^{\text{axis}}$ [S2, unofficial].
    d.  **Accumulate Loss:** $L_{\text{total}} \leftarrow L_{\text{total}} + L$ [S2, unofficial].
7.  **Backpropagation:** Perform backward pass on $L_{\text{total}}$ to compute gradients $\nabla_\theta$ [S2, unofficial].
8.  **Parameter Update:** Update parameters $\theta$ using the Adam optimizer step [S2, unofficial].
9.  **Refinement Check:** If the current step is a refinement iteration (determined by schedule $IsRefinementIteration(t)$):
    a.  **Gaussian Iteration:** Iterate over all Gaussians $(\boldsymbol{\mu}, \boldsymbol{\Sigma}, \mathbf{c}, \alpha) \in G$.
    b.  **Pruning Condition:** If opacity $\alpha < \tau_\alpha$ or the Gaussian is too large ($IsTooLarge(\boldsymbol{\mu}, \boldsymbol{\Sigma})$), remove the Gaussian [S2, unofficial].
    c.  **Densification Condition:** Else if the L2 norm of the gradient of $\boldsymbol{\mu}$ with respect to loss is greater than $\tau_p$ ($\|\nabla_{\boldsymbol{\mu}} L\|_2 > \tau_p$):
        i.  **Over-reconstruction Check:** If the Frobenius norm of $\boldsymbol{\Sigma}$ is greater than $\tau_s$ ($\|\boldsymbol{\Sigma}\|_F > \tau_s$), split the Gaussian [S2, unofficial].
        ii. **Under-reconstruction Action:** Else, clone the Gaussian [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def training_pipeline(
    training_slices: List[Tuple[Image, float]], # (I, pi) for each axis
    grid_resolution: int,
    max_iterations: int,
    alpha_threshold: float,
    gradient_threshold: float,
    scale_threshold: float,
    is_refinement_iteration: Callable[[int], bool],
    initial_lr: float
) -> List[Gaussian]:
    # 1. Initialization
    # Build regular grid P of size grid_resolution
    # Initialize Gaussians G with mean mu around grid points
    # Default values for scale s, color c, opacity alpha, quaternion q
    G = []
    for point in build_regular_grid(grid_resolution):
        G.append(Gaussian(mu=point, s=default_scale, c=default_color, alpha=default_opacity, q=default_quaternion))
    
    # 2. Optimizer Setup
    # Initialize Adam optimizer with parameters theta and learning rate eta
    optimizer = Adam(parameters=G, lr=initial_lr)
    
    # 3. Outer Loop
    for step in range(1, max_iterations + 1):
        # Resolution schedule
        if step < 5000:
            resolution = (540, 960)
        else:
            resolution = (1080, 1920)
            
        # 4. Reset Accumulators
        L_total = 0.0
        optimizer.zero_grad()
        
        # 5. Axis Loop
        for axis in ['x', 'y', 'z']:
            # 6. Slice Loop
            axis_slices = [s for s in training_slices if s.axis == axis]
            
            for slice_image, slice_pos in axis_slices:
                # 6a. Covariance Computation
                for g in G:
                    g.Sigma = g.R @ (g.S @ g.S.T) @ g.R.T # R_n S_n S_n^T R_n^T
                
                # 6b. Rendering
                # I_pred = Rasterize(mu, Sigma, alpha, c)
                # Rasterize projects Gaussians to 2D slice view
                I_pred = rasterize(G, slice_pos, axis, resolution)
                
                # 6c. Loss Computation
                # L = lambda1*L1 + lambda2*(1-SSIM) + lambda3*VGG + lambda4*LPIPS
                L = compute_loss(I_pred, slice_image)
                
                # 6d. Accumulate Loss
                L_total += L
        
        # 7. Backpropagation
        L_total.backward()
        
        # 8. Parameter Update
        optimizer.step()
        
        # 9. Refinement Check
        if is_refinement_iteration(step):
            gaussians_to_remove = []
            gaussians_to_split = []
            gaussians_to_clone = []
            
            # 9a. Gaussian Iteration
            for g in G:
                # 9b. Pruning Condition
                if g.alpha < alpha_threshold or is_too_large(g.mu, g.Sigma):
                    gaussians_to_remove.append(g)
                # 9c. Densification Condition
                elif norm(g.grad_mu) > gradient_threshold:
                    # 9c-i. Over-reconstruction Check
                    if frobenius_norm(g.Sigma) > scale_threshold:
                        gaussians_to_split.append(g)
                    # 9c-ii. Under-reconstruction Action
                    else:
                        gaussians_to_clone.append(g)
            
            # 9b. Pruning Action
            for g in gaussians_to_remove:
                G.remove(g)
            
            # 9c-i. Split Action
            for g in gaussians_to_split:
                split_gaussian(G, g)
            
            # 9c-ii. Clone Action
            for g in gaussians_to_clone:
                clone_gaussian(G, g)
                
    return G

# Helper definitions
def build_regular_grid(res): ... # Not given in the sources.
def rasterize(G, pos, axis, res): ... # Calls differentiable alpha-blending rasterizer [G]
def compute_loss(I_pred, I_gt): return L1 + SSIM # Standard loss [G]
def is_too_large(mu, Sigma): return frobenius_norm(Sigma) > some_threshold # [G]
def norm(v): return sqrt(sum(x**2 for x in v))
def frobenius_norm(M): return sqrt(sum(M[i][j]**2 for i in range(3) for j in range(3)))
def split_gaussian(G, g): ... # Not given in the sources.
def clone_gaussian(G, g): ... # Not given in the sources.
```

**Edge Cases and Pitfalls:**
*   **Precision:** The covariance decomposition $\boldsymbol{\Sigma} = R S S^\top R^\top$ requires stable rotation representations (quaternions) to avoid singularities [S1].
*   **Boundaries:** The axis-loop structure assumes the scene can be adequately covered by orthogonal slices. If the scene has strong depth-dependent structures not aligned with axes, coverage may be uneven [S1].
*   **Degenerate Input:** If the initial grid resolution $g_{\text{res}}$ is too low, the model may fail to capture fine details; if too high, training becomes computationally expensive and prone to overfitting [S1].

## Subtopics

### [Mini-Splatting Pipeline](9235-mini-splatting-pipeline.md)
The Mini-Splatting Pipeline is a training strategy for 3D Gaussian Splatting that explicitly manages the total number of Gaussians through periodic densification, simplification, and depth-based reinitialization phases. It distinguishes itself from standard 3D Gaussian Splatting by enforcing a constrained Gaussian count via a manually controlled sampling ratio.

### [Validation Metric Computation & Logging](10760-validation-metric-computation-logging.md)
This method computes quantitative validation metrics (primarily PSNR) and saves visual examples by rendering novel views from the optimized 3D Gaussian Splatting model and comparing them against ground truth images. It is distinct from the training loop as it operates in evaluation mode (`is_train=False`) with gradient tracking disabled, ensuring that the optimization state is not updated during metric calculation.

### FaCT-GS Optimization Loop
The FaCT-GS optimization loop extends the standard pipeline by adding a 3D total variation (TV) regularization term [S5]. It rasterizes Gaussians into a 2D image $\mathbf{I}_r$ and compares it to projection data $\mathbf{I}_{gt}$ using L1 and SSIM losses [S5]. Additionally, it voxelize a small, randomly sampled section of the volume $\mathbf{V}_{px3}$ and calculates 3D total variation $\mathcal{L}_{TV}$ as a homogeneity regularization prior [S5]. The total loss is $\mathcal{C}_{\text{total}} = \mathcal{L}_1 (\mathbf{I}_1, \mathbf{I}_{\text{tet}}) + \alpha_{\text{SSIM}} \mathcal{L}_{\text{SSIM}} (\mathbf{I}_1, \mathbf{I}_{\text{tet}}) + \alpha_{\text{TV}} \mathcal{L}_{\text{TV}} (\mathbf{V}_{32^3})$, where $\alpha_{\text{SSIM}}$ and $\alpha_{\text{TV}}$ are SSIM and TV loss weights respectively [S5]. In the backward pass, the loss is used to prune, clone, or split Gaussians (Adaptive Control) and adjust parameters based on propagated gradients [S5]. The sampling size for voxelization is increased from $8^3$ to $32^3$ [S5].

### Cut-Set Gradient Update
The Cut-Set Gradient Update method modifies the standard gradient accumulation by identifying the "cut set" of Gaussians intersecting the current training view and updating only their parameters [S4, unofficial]. Gradients propagate to Gaussians in the middle of the hierarchy, diffusing across the entire structure over many iterations with diverse views [S4, unofficial]. This approach is used to fulfill tight memory requirements in large-scale reconstruction [S4, unofficial].

### [3D Gaussian Splatting Initialization & Training Schedule](15507-3d-gaussian-splatting-initialization-training-schedule.md)
This variant initializes the model using LiDAR point clouds and pseudo-view guidance. It projects LiDAR frames to corresponding image frames to assign color to each LiDAR point, then re-projects colored points into 3D space. It accumulates all frames of point clouds and voxel-downsamples with a voxel size of 5. The training schedule includes a warm-up phase of 500 iterations without sampling pseudo views, followed by a guided phase where 4 pseudo views are sampled for every 10 iterations. Total training iterations are 50,000.

### fMRI Temporal Adaptation
This strategy adapts the training pipeline for functional MRI (fMRI) data by freezing geometry parameters (Gaussian positions and covariances) and allowing only color and opacity to vary over time [S2, unofficial]. It uses a two-stage strategy: Stage 1 builds a static anatomical structure over 1000 warm-up iterations using initial frames; Stage 2 runs 3000 iterations where only color and opacity evolve temporally [S2, unofficial]. It processes datasets like OpenfMRI (160 temporal frames, 33 axial brain slices at 64x64 resolution) with 140 frames for training and every 8th frame (20 total) for testing [S2, unofficial]. A custom data loader ensures all slices from a given timestamp are processed synchronously to preserve inter-slice consistency [S2, unofficial].

### Foreground Reconstruction (Stage 2)
This strategy includes background Gaussians in rendering computation but keeps them fixed during optimization [S6]. It initializes foreground Gaussians using the filtered point cloud generated during segmentation [S6]. It confines foreground Gaussians to the spherical volume defined by inner radius $R_i$ [S6]. It optimizes foreground Gaussians using standard 3D Gaussian Splatting loss on original unmasked images [S6]. It performs spatial pruning: deletes any foreground Gaussian that moves beyond the inner radius $R_i$ [S6].

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the rasterization and backpropagation steps, which are linear in the number of Gaussians $N$ and pixels $P$ per slice [S1]. The nested loop over axes and slices multiplies this by the number of slices $S$, resulting in $O(S \cdot N \cdot P)$ per iteration [S1].
*   **Memory Complexity:** Memory usage is proportional to the number of Gaussians $N$ and their parameters (position, covariance, color, opacity) [S1]. The cut-set variant reduces memory by only storing gradients for the cut set [S4, unofficial].
*   **Measured Performance:** Standard 3D Gaussian Splatting training typically runs for 30,000 iterations [S1]. fMRI adaptation uses 4,000 total iterations (1,000 warm-up + 3,000 temporal) [S2, unofficial].
*   **Strengths:** Dynamic adaptation allows the model to start with a coarse representation and refine it, balancing efficiency and fidelity [S1].
*   **Limitations:** The axis-slice structure may not be optimal for all scene geometries [S1]. The refinement steps (pruning/splitting) add overhead and require careful tuning of thresholds $\tau_\alpha, \tau_p, \tau_s$ [S1].
*   **When to use:** Use for general 3D scene reconstruction from multi-view images or slices [S1]. Use FaCT-GS for CT reconstruction where homogeneity is desired [S5]. Use fMRI adaptation for temporal medical imaging [S2, unofficial].

## Background
The Training Pipeline originates from the 3D Gaussian Splatting method by Kerbl et al. [2023], which introduced differentiable rasterization for 3D Gaussians [S1][S3, unofficial]. The specific inner loop structure with axis-aligned slices and dynamic densification/pruning was detailed in subsequent works such as InnerGS [S1] and FaCT-GS [S5]. The method has been adapted for various domains including fMRI [S2, unofficial], CT [S5], and large-scale reconstruction [S4, unofficial].

## Sources
[S1] Optimization of 3D Gaussian Splatting for Accurate Image Reconstruction, https://doi.org/10.59232/dst-v4i2p101, fetched 2026-10-26
[S2] InnerGS: Internal Scenes Reconstruction and Segmentation via Factorized 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2508.13287, fetched 2026-10-26 (unofficial)
[S3] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
[S4] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S5] FaCT-GS: Fast and Scalable CT Reconstruction with Gaussian Splatting, https://doi.org/10.1007/978-3-032-37464-6_13, fetched 2026-10-26
[S6] EnvMap-GS: two-stage outdoor Gaussian reconstruction with background-to-environment map baking, https://doi.org/10.1007/s00371-026-04593-w, fetched 2026-10-29
