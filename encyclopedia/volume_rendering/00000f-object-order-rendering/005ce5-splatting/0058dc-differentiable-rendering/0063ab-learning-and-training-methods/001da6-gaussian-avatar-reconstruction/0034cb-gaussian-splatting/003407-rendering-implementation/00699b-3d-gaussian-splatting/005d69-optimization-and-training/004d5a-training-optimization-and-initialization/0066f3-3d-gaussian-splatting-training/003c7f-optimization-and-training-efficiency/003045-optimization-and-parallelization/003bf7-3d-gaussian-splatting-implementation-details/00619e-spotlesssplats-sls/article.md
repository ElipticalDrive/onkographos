# SpotLessSplats (SLS)


## Overview
SpotLessSplats (SLS) is a robust 3D Gaussian Splatting (3DGS) framework that enables high-quality reconstruction from casually captured video data containing transient distractors (e.g., moving people, wind-blown elements) [S1]. Unlike standard 3DGS which assumes static scenes, SLS uses unsupervised detection of outliers in training images by leveraging pre-trained text-to-image features and robust optimization to ignore these distractors [S1]. It sets itself apart from sibling methods like [DeSplat](12925-desplat.md) or [360 Gaussian Splatting](12929-360-gaussian-splatting.md) by specifically addressing the problem of semantic outlier modeling in casual captures, using feature-space similarities rather than just photometric residuals to identify inliers and outliers [S1].

## How it works
**Inputs:** A set of posed images $\{I_n\}_{n=1}^N$ from a casually captured scene, and a pre-trained 2D foundation model (e.g., DINOv2) to extract semantic feature maps $F$.
**Outputs:** A set of 3D Gaussians optimized for novel view synthesis, with transient distractors effectively ignored.

**Data Structures:**
*   **3D Gaussians:** Standard 3DGS representation (position, covariance, opacity, spherical harmonics coefficients).
*   **Feature Map $F$:** Semantic feature embeddings extracted from input images using the foundation model.
*   **Inlier Mask $M$:** A binary or probabilistic mask indicating inlier pixels.
*   **Residual Histogram:** A histogram with $B$ buckets of width $10^{-3}$ tracking residual magnitudes over time.
*   **MLP Classifier $H$:** A network with parameters $\theta^{(t)}$ for spatio-temporal outlier detection.

**Parameters:**
*   $\tau$: Hyperparameter for generalized median cut-off percentile. Typical values: $0.5$ (median) or $0.9$ [S1].
*   $\kappa$: Utilization-based pruning threshold. Value: $10^{-8}$ [S1].
*   $T$: Steps between utilization calculations. Value: $100$ [S1].
*   $|N_T(t)|$: Number of images accumulated for utilization calculation. Value: $100$ [S1].
*   $\lambda$: Weight for regularization loss. Value: $0.5$ [S1].

**Steps:**

1.  **Feature Extraction:** For each training image $I_n$, extract semantic feature map $F_n$ using the pre-trained foundation model [S1].
2.  **Robust Masking (Photometric Baseline):** Compute the 1-norm residual image $R_n = \|I_n - \hat{I}_n\|_1$ pixel-wise [S1]. Compute a generalized median $\rho$ of residuals over multiple batches to handle skewed distributions (see Sub-step 2a). Compute the initial inlier mask $M_n^{(t)}$ using a threshold $\rho$ and a morphological dilation [S1]:
    $$ M_n^{(t)} = \mathbb{1}\{R_n > \rho\} \circledast B > 0.5, \quad P(R_n > \rho) = \tau $$
    where $\mathbb{1}$ is the indicator function, $\circledast$ denotes convolution with a normalized $3 \times 3$ box filter $B$, and the condition $> 0.5$ applies to the result of the convolution [S1].
3.  **Outlier Detection (SLS-mlp):** Train an MLP classifier $H$ with parameters $\theta^{(t)}$ to predict pixel-wise inlier probabilities $M_{mlp}^{(t)} = H(F; \theta^{(t)})$ [S1].
    *   **Architecture:** $H$ uses $1 \times 1$ convolutions, acting i.i.d. across pixels [S1].
    *   **Loss Function:** Minimize $L(\theta^{(t)}) = L_{sup}(\theta^{(t)}) + \lambda L_{reg}(\theta^{(t)})$ [S1].
    *   **Supervision Loss:** $L_{sup}(\theta^{(t)}) = \max(U^{(t)} - H(F; \theta^{(t)}), 0) + \max(H(F; \theta^{(t)}) - L^{(t)}, 0)$ [S1].
    *   **Labels:** $U^{(t)}$ is $M_n^{(t)}$ computed with $\tau=0.5$; $L^{(t)}$ is $M_n^{(t)}$ computed with $\tau=0.9$ [S1].
    *   **Regularization:** Minimize the Lipschitz constant of $H$ via $L_{reg}$ as detailed in [Liu et al. 2022, Eq. (13)] to ensure similar features map to similar probabilities [S1].
    *   **Optimization Schedule:** Parameters $\theta^{(t)}$ are updated concurrently with 3DGS optimization, interleaved such that one is fixed while the other is optimized [S1].
    *   **Mask Selection:** Use $M_{mlp}^{(t)}$ as the final inlier/outlier mask for training [S1].
4.  **Residual Tracking (Sub-step 2a):** To handle skewed outlier distribution across images, track residual magnitudes over multiple training batches [S1].
    *   Discretize residuals into $B$ histogram buckets of width $10^{-3}$ [S1].
    *   Update bucket population via a discounted update similar to fast median filtering [Perreault and Hebert 2007] [S1].
    *   Extract generalized median $\rho$ as the $\tau$ quantile in the histogram population [S1].
5.  **Utilization-Based Pruning (UBP):** Replace standard opacity reset with UBP to compress the scene representation [S1].
    *   Track the gradient of rendered colors $\hat{I}_{h,w}$ with respect to the projected splat positions $x_g$ of each Gaussian $g$ [S1].
    *   Compute utilization $u_g$:
        $$ u_g = \sum_{t \in N_T(t)} \mathbb{E}_{w,h} \left[ M_{h,w}^{(t)} \cdot \left( \frac{\partial \hat{I}_{h,w}^{(t)}}{\partial x_g} \right)^2 \right] $$
        where the expectation is over image pixels $(w,h)$ [S1].
    *   Average across the image $(W \times H)$, computed every $T=100$ steps accumulated over $|N_T(t)|=100$ images [S1].
    *   Prune Gaussians if $u_g < \kappa$ (where $\kappa = 10^{-8}$) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spot_less_splats_training(images, poses, foundation_model, num_steps):
    # Initialize 3D Gaussians
    gaussians = initialize_3d_gaussians()
    
    # Initialize MLP for outlier detection
    mlp = MLP_1x1_conv()
    theta = mlp.parameters()
    
    # Initialize residual histogram
    B = 1000  # Number of buckets
    bucket_width = 1e-3
    histogram = np.zeros(B)
    tau_median = 0.5
    tau_high = 0.9
    lambda_reg = 0.5
    
    # Pruning parameters
    T = 100
    N_T = 100
    kappa = 1e-8
    utilization_buffer = []
    
    for step in range(num_steps):
        # 1. Sample a batch of images
        batch_images, batch_poses = sample_batch(images, poses)
        
        # 2. Extract semantic features
        features = foundation_model(batch_images)
        
        # 3. Render current Gaussians
        rendered_images = render_3dgs(gaussians, batch_poses)
        
        # 4. Compute residuals and update histogram
        residuals = np.abs(batch_images - rendered_images).mean(axis=-1) # 1-norm approx
        update_histogram(histogram, residuals.flatten(), bucket_width)
        
        # 5. Compute generalized median rho
        rho_median = get_quantile(histogram, tau_median)
        rho_high = get_quantile(histogram, tau_high)
        
        # 6. Compute photometric masks
        mask_median = (residuals > rho_median)
        mask_high = (residuals > rho_high)
        # Apply morphological dilation (3x3 box filter)
        mask_median = dilate(mask_median)
        mask_high = dilate(mask_high)
        
        # 7. Update MLP (Interleaved with GS optimization)
        # If step is even, update MLP; if odd, update Gaussians (example schedule)
        if step % 2 == 0:
            # Compute MLP loss
            pred_inlier = mlp(features)
            U = mask_median
            L = mask_high
            l_sup = np.maximum(U - pred_inlier, 0) + np.maximum(pred_inlier - L, 0)
            l_sup = l_sup.mean()
            
            # Compute Lipschitz regularization (simplified placeholder for Eq 13)
            l_reg = compute_lipschitz_reg(mlp, features)
            
            loss_mlp = l_sup + lambda_reg * l_reg
            update_parameters(theta, loss_mlp)
            
            # Use MLP mask for this step's GS update if MLP is active
            # In practice, SLS-mlp uses M_mlp as the mask
            final_mask = sigmoid(pred_inlier) # Or thresholded
        else:
            # Update 3D Gaussians
            # Use the mask from the previous step or current photometric mask
            # SLS-mlp uses M_mlp, so we assume final_mask is available
            loss_gs = compute_3dgs_loss(rendered_images, batch_images, final_mask)
            update_gaussians(gaussians, loss_gs)
            
            # Track utilization for pruning
            if step % T == 0:
                # Compute gradient of rendered color w.r.t projected splat positions
                # This requires backprop through rendering to x_g
                grads = compute_grad_rendered_vs_projected_pos(gaussians, rendered_images, final_mask)
                utilization_buffer.append(grads)
                
                if len(utilization_buffer) >= N_T:
                    # Compute utilization u_g
                    utilizations = np.zeros(num_gaussians)
                    for g_idx in range(num_gaussians):
                        # Average over accumulated images and pixels
                        # E_{w,h} [ M_{h,w} * (dI/dx_g)^2 ]
                        # Simplified: sum over buffer and average
                        utilizations[g_idx] = np.mean([
                            np.mean(buf[g_idx] ** 2) for buf in utilization_buffer
                        ])
                    
                    # Prune
                    gaussians = prune_gaussians(gaussians, utilizations, kappa)
                    
                    # Reset buffer
                    utilization_buffer = []
                    
                    # Reset histogram or continue tracking? 
                    # Sources say "tracking residual magnitudes over multiple training batches"
                    # implying continuous tracking.
```

**Subtopics:**
*   **SLS-mlp (Spatio-temporal Clustering):** Uses an MLP classifier trained unsupervised to predict inlier probabilities based on semantic features. It differs from spatial clustering by leveraging temporal consistency across the dataset. Choose when semantic information is strong and distractors are complex. [S1]
*   **Spatial Clustering:** Uses non-parametric clustering of local feature embeddings to find image regions of structured outliers. Simpler but less robust to temporal variations than SLS-mlp. [S1]

## Operating characteristics
*   **Time Complexity:** Adds overhead for feature extraction (foundation model) and MLP training, but the MLP is lightweight ($1 \times 1$ convs). The interleaved optimization ensures convergence.
*   **Memory Complexity:** Requires storing feature maps and the MLP parameters. The histogram tracking uses constant memory. UBP reduces memory usage by pruning Gaussians.
*   **Performance:** Achieves state-of-the-art reconstruction quality on casually captured scenes. Reduces the number of Gaussians by 2-4 times while maintaining quality [S1].
*   **Strengths:** Robust to transient distractors (people, wind) without manual annotation. Avoids overfitting to distractors with colors similar to the background, a common failure mode of residual-only methods like RobustNeRF [S1].
*   **Limitations:** Requires a pre-trained foundation model. Performance depends on the quality of semantic features.
*   **When to use:** Casual captures with transient objects.
*   **When not to use:** Static, controlled environments where standard 3DGS suffices, or when computational resources for foundation model inference are limited.

## Background
SpotLessSplats was introduced by Sabour, Goli, Kopanas, Mathews, and Lagun from Google Deepmind and University of Toronto [S1]. It addresses the limitation of 3DGS which requires highly controlled environments (no moving people, consistent lighting) [S1]. The method builds on prior work like RobustNeRF [Sabour et al. 2023] and NeRF-W, but improves upon them by using semantic features instead of just photometric residuals [S1]. It also incorporates insights from [Goli et al. 2024] for utilization-based pruning [S1].

## Sources
[S1] SpotLessSplats: Ignoring Distractors in 3D Gaussian Splatting, https://doi.org/10.1145/3727143, fetched 2026-10-27
