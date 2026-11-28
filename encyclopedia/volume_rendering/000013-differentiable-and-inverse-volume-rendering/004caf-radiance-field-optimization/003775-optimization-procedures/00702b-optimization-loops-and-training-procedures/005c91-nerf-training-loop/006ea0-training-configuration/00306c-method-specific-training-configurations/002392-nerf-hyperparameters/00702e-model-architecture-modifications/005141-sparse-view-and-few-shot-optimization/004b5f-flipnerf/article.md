# FlipNeRF

Also called: Flipped Reflection Rays for Few-shot Novel View Synthesis

## Overview
FlipNeRF is a regularization method for few-shot novel view synthesis that generates auxiliary 'flipped reflection rays' from original input rays and estimated surface normals to improve rendering quality and surface normal accuracy with sparse inputs [S1]. It sets itself apart from sibling methods like [Simple-NeRF](7371-simple-nerf.md) and [WaH-NeRF](11548-wah-nerf.md) by explicitly deriving additional training rays from geometry estimates rather than relying on heuristic unseen viewpoint sampling or pre-training on large datasets [S1]. Unlike [Deceptive-NeRF/3DGS](11859-deceptive-nerf-3dgs.md) which may focus on depth or explicit surface reconstruction, FlipNeRF focuses on accurate surface normal estimation to drive depth accuracy, using these normals to reflect rays back into the scene for consistency regularization [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of sparse input images with known camera poses, defining original input rays $\mathbf{r}$. The underlying model is built upon MixNeRF [S1], which uses a mixture density model to represent the radiance field.
*   **Outputs:** A trained radiance field model capable of rendering novel views from sparse inputs, along with accurate surface normal estimates $\hat{\mathbf{n}}$.

**Data Structures:**
The method relies on the standard NeRF volumetric representation but augments the training loop with specific ray batches and feature vectors:
1.  **Original Rays ($\mathcal{R}$):** Standard rays cast from camera origins $\mathbf{o}$ with directions $\mathbf{d}$.
2.  **Flipped Reflection Rays ($\mathcal{R}'$):** Auxiliary rays derived from $\mathcal{R}$ and estimated normals.
3.  **Bottleneck Features ($\mathbf{b}, \mathbf{b}'$):** Intermediate feature vectors output by the spatial MLP of the NeRF for original and flipped rays, respectively.
4.  **Mixture Model Parameters:** For each sample $i$ along a ray, the model outputs blending weights $\pi_i$, mean colors $\mu_i^c$, and scale parameters $\beta_i^c$ for RGB channels.

**Numbered Steps:**

1.  **Estimate Surface Normals:**
    Compute the estimated surface normal $\hat{\mathbf{n}}$ for each ray as the weighted sum of blending weights and estimated normal vectors $\mathbf{n}_i$ along the ray [S1]:
    $$ \hat{\mathbf{n}} = \sum_{i=1}^{M} w_i \mathbf{n}_i $$
    where $\mathbf{n}_i$ is the gradient of the volume density (normalized) at sample $i$, and $w_i$ is the blending weight [S1]. Note: The model does *not* explicitly normalize $\hat{\mathbf{n}}$ during training; it learns to approximate unit vectors naturally [S1].

2.  **Derive Flipped Reflection Rays:**
    For each original ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, calculate the flipped reflection direction $\mathbf{d}'$ [S1]:
    $$ \mathbf{d}' = 2(\mathbf{d} \cdot \hat{\mathbf{n}})\hat{\mathbf{n}} - \mathbf{d} $$
    Identify the position of the peak blending weight sample $p_s$ along the original ray, where the $s$-th sample has the highest blending weight [S1]:
    $$ p_s = \mathbf{o} + t_s \mathbf{d} $$
    Calculate the new origin $\mathbf{o}'$ for the flipped ray such that the peak sample of the flipped ray coincides with $p_s$ [S1]:
    $$ \mathbf{o}' = p_s - t_s \mathbf{d}' $$
    Define the flipped reflection ray as $\mathbf{r}'(t) = \mathbf{o}' + t\mathbf{d}'$ [S1].

3.  **Apply Masking Strategy:**
    Filter out ineffective flipped rays using a masking function $M(\mathbf{r}')$ based on the angle between the original direction and the normal [S1]:
    $$ M(\mathbf{r}') = \begin{cases} 1 & \text{if } \arccos(-(\hat{\mathbf{d}} \cdot \hat{\mathbf{n}})) < \tau \\ 0 & \text{otherwise} \end{cases} $$
    where $\tau$ is the threshold (typically $90^\circ$ for DTU/Realistic Synthetic 360°, $30^\circ$ for LLFF) [S1]. Only rays with $M(\mathbf{r}')=1$ are used for regularization losses [S1].

4.  **Compute Losses:**
    Calculate the following loss terms for a batch of rays:
    *   **MSE Loss ($\mathcal{L}_{\text{MSE}}$):** Standard pixel-wise mean squared error between rendered color $\hat{\mathbf{c}}(\mathbf{r})$ and ground truth $\mathbf{c}^{\text{GT}}(\mathbf{r})$ [S1]:
        $$ \mathcal{L}_{\text{MSE}} = \sum_{\mathbf{r} \in \mathcal{R}} || \hat{\mathbf{c}}(\mathbf{r}) - \mathbf{c}^{\text{GT}}(\mathbf{r}) ||_2^2 $$
    *   **NLL Loss ($\mathcal{L}_{\text{NLL}}, \mathcal{L}_{\text{NLL}}'$):** Negative log-likelihood for original and flipped rays, modeling colors with mixture density models [S1].
    *   **Uncertainty-aware Emptiness Loss (UE Loss, $\mathcal{L}_{\text{UE}}, \mathcal{L}_{\text{UE}}'$):** Reduces floating artifacts by considering output uncertainty $\rho$ [S1]:
        $$ l_{\text{UE}}(\mathbf{r}) = \frac{1}{M} \sum_{i=1}^{M} \log(1 + \rho \cdot \eta \cdot w_i) $$
        where $\rho = \frac{1}{3} \sum_{c}^{\{r,g,b\}} \sum_{i=1}^{M} \beta_i^c$ is the uncertainty, and $\eta$ is a scaling factor [S1].
    *   **Bottleneck Feature Consistency Loss (BFC Loss, $\mathcal{L}_{\text{BFC}}$):** Encourages consistency between bottleneck features of original and flipped rays using Jensen-Shannon Divergence (JSD) [S1]:
        $$ l_{\text{BFC}}(\mathbf{r}, \mathbf{r}') = JSD(\psi(\mathbf{b}), \psi(\mathbf{b}')) $$
        where $\psi$ is the softmax function, and $\mathbf{b}, \mathbf{b}'$ are bottleneck features [S1].
    *   **Orientation Loss ($\mathcal{L}_{\text{Ori.}}$):** Penalizes backward-facing normal vectors using the underlying density gradient normal $\mathbf{n}_i$ [S1]:
        $$ l_{\text{Ori.}}(\mathbf{r}) = \sum_{i=1}^{M} w_i \max(0, \mathbf{n}_i \cdot \hat{\mathbf{d}})^2 $$

5.  **Optimize:**
    Minimize the total loss $\mathcal{L}_{\text{Total}}$ using the Adam optimizer [S1]:
    $$ \mathcal{L}_{\text{Total}} = \mathcal{L}_{\text{MSE}} + \lambda_1 \mathcal{L}_{\text{NLL}} + \lambda_2 \mathcal{L}_{\text{NLL}}' + \lambda_3 \mathcal{L}_{\text{UE}} + \lambda_4 \mathcal{L}_{\text{UE}}' + \lambda_5 \mathcal{L}_{\text{BFC}} + \lambda_6 \mathcal{L}_{\text{Ori.}} $$
    where $\lambda_i$ are balancing weights [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flipnerf_train_step(model, rays, gt_colors, hyperparams):
    """
    One training step for FlipNeRF.
    
    Args:
        model: The NeRF model (based on MixNeRF).
        rays: List of Ray objects, each with origin o, direction d, and target pixel.
        gt_colors: Ground truth RGB colors for the rays.
        hyperparams: Dictionary containing thresholds, weights, and optimizer settings.
        
    Returns:
        total_loss: Scalar loss value.
    """
    optimizer.zero_grad()
    
    # 1. Forward pass for original rays to get densities, colors, and bottleneck features
    # model returns: densities (sigma), colors (c), bottleneck features (b), 
    # blending weights (w), scale params (beta), and density gradients (grad_sigma)
    outputs_orig = model.forward(rays)
    sigma_orig = outputs_orig['sigma']
    c_orig = outputs_orig['c']
    b_orig = outputs_orig['bottleneck']
    w_orig = outputs_orig['blending_weights']
    beta_orig = outputs_orig['beta']
    grad_sigma_orig = outputs_orig['grad_sigma']
    
    # 2. Estimate surface normals for original rays
    # n_hat is weighted sum of normalized density gradients
    n_hat_list = []
    for i, ray in enumerate(rays):
        # Normalize gradients to get normal vectors n_i
        n_i = grad_sigma_orig[i] / (norm(grad_sigma_orig[i]) + 1e-8)
        # Weighted sum
        n_hat = sum(w_orig[i][j] * n_i[j] for j in range(len(w_orig[i])))
        n_hat_list.append(n_hat)
    
    # 3. Generate flipped reflection rays
    flipped_rays = []
    valid_mask = []
    for i, ray in enumerate(rays):
        d = ray.direction
        n_hat = n_hat_list[i]
        
        # Calculate flipped direction d'
        # d' = 2(d . n_hat)n_hat - d
        dot_prod = dot(d, n_hat)
        d_prime = 2 * dot_prod * n_hat - d
        d_prime = normalize(d_prime)
        
        # Find peak blending weight index s
        s = argmax(w_orig[i])
        t_s = ray.sample_distances[s]
        p_s = ray.origin + t_s * d
        
        # Calculate new origin o'
        # o' = p_s - t_s * d'
        o_prime = p_s - t_s * d_prime
        
        # Create new ray object
        flipped_ray = Ray(origin=o_prime, direction=d_prime, target_pixel=ray.target_pixel)
        
        # Apply masking
        # M(r') = 1 if arccos(-(d_hat . n_hat)) < tau else 0
        # Note: d_hat is normalized d. n_hat is not explicitly normalized but treated as direction.
        # The paper uses - (d_hat . n_hat). Since n_hat might not be unit, 
        # we use the formula as given: arccos(-(d_hat . n_hat)).
        # However, dot product with non-unit vector is not cosine. 
        # The paper says "where -(d_hat . n_hat) amounts to cos theta". 
        # This implies n_hat is treated as a direction vector. 
        # In practice, we check the angle condition.
        # Let's assume n_hat is effectively a direction.
        cos_theta = -dot(normalize(d), normalize(n_hat))
        # Clamp for numerical stability in arccos
        cos_theta = max(-1.0, min(1.0, cos_theta))
        theta = arccos(cos_theta)
        
        if theta < hyperparams['tau']:
            flipped_rays.append(flipped_ray)
            valid_mask.append(True)
        else:
            valid_mask.append(False)
            
    # 4. Forward pass for valid flipped rays
    valid_flipped_rays = [r for r, m in zip(flipped_rays, valid_mask) if m]
    if valid_flipped_rays:
        outputs_flip = model.forward(valid_flipped_rays)
        b_flip = outputs_flip['bottleneck']
        w_flip = outputs_flip['blending_weights']
        beta_flip = outputs_flip['beta']
    else:
        b_flip = []
        w_flip = []
        beta_flip = []

    # 5. Compute Losses
    
    # MSE Loss
    mse_loss = mean_squared_error(c_orig, gt_colors)
    
    # NLL Loss (assuming model provides NLL calculation or we compute from mixture params)
    # MixNeRF minimizes NLL. We assume model provides nll_orig and nll_flip
    nll_loss = model.compute_nll(rays, sigma_orig, c_orig, w_orig)
    nll_flip_loss = model.compute_nll(valid_flipped_rays, outputs_flip['sigma'], outputs_flip['c'], w_flip) if valid_flipped_rays else 0.0
    
    # UE Loss
    # rho = 1/3 * sum_c sum_i beta_i^c
    # l_UE = 1/M * sum_i log(1 + rho * eta * w_i)
    ue_loss = compute_ue_loss(rays, w_orig, beta_orig, hyperparams['eta'])
    ue_flip_loss = compute_ue_loss(valid_flipped_rays, w_flip, beta_flip, hyperparams['eta']) if valid_flipped_rays else 0.0
    
    # BFC Loss
    # JSD(softmax(b), softmax(b'))
    bfc_loss = compute_jsd_loss(b_orig, b_flip) if valid_flipped_rays else 0.0
    
    # Orientation Loss
    # l_Ori = sum_i w_i * max(0, n_i . d_hat)^2
    # n_i is grad_sigma (density gradient), d_hat is ray direction
    ori_loss = compute_orientation_loss(rays, w_orig, grad_sigma_orig)
    
    # Total Loss
    total_loss = (mse_loss + 
                  hyperparams['lambda1'] * nll_loss + 
                  hyperparams['lambda2'] * nll_flip_loss + 
                  hyperparams['lambda3'] * ue_loss + 
                  hyperparams['lambda4'] * ue_flip_loss + 
                  hyperparams['lambda5'] * bfc_loss + 
                  hyperparams['lambda6'] * ori_loss)
                  
    total_loss.backward()
    optimizer.step()
    
    return total_loss

def compute_ue_loss(rays, w, beta, eta):
    M = len(rays)
    if M == 0: return 0.0
    # rho = 1/3 * sum_c sum_i beta_i^c
    # beta shape: (M, 3, N_samples) or similar. Assuming beta is (M, N_samples, 3)
    # Sum over channels and samples
    rho = (1.0/3.0) * sum(sum(beta[i]) for i in range(M))
    # l_UE = 1/M * sum_i log(1 + rho * eta * w_i)
    # w_i is the max blending weight or sum? Paper says "w_i" in sum. 
    # Usually UE loss sums over samples. Let's assume w is blending weights per sample.
    # The formula is sum_{i=1}^M log(...). M here is number of samples in the batch? 
    # No, M in the formula is number of samples along the ray? 
    # "l_UE(r) = 1/M sum_{i=1}^M ...". M is number of samples per ray.
    # But the loss is aggregated over the batch.
    # Let's aggregate over all samples in the batch.
    total_loss = 0.0
    count = 0
    for i in range(len(rays)):
        # w[i] is array of weights for samples
        # beta[i] is array of betas for samples
        for j in range(len(w[i])):
            total_loss += log(1 + rho * eta * w[i][j])
            count += 1
    return total_loss / count if count > 0 else 0.0

def compute_orientation_loss(rays, w, grad_sigma):
    total_loss = 0.0
    for i, ray in enumerate(rays):
        d_hat = normalize(ray.direction)
        for j in range(len(w[i])):
            n_i = grad_sigma[i][j]
            dot_prod = dot(n_i, d_hat)
            total_loss += w[i][j] * max(0, dot_prod)**2
    return total_loss

def compute_jsd_loss(b_orig, b_flip):
    # JSD(P, Q) = 0.5 * KL(P || M) + 0.5 * KL(Q || M)
    # where M = 0.5 * (P + Q)
    # P, Q are softmax(b)
    p = softmax(b_orig)
    q = softmax(b_flip)
    m = 0.5 * (p + q)
    kl_pm = kl_divergence(p, m)
    kl_qm = kl_divergence(q, m)
    return 0.5 * (kl_pm + kl_qm)
```

**Parameters:**
*   **Masking Threshold ($\tau$):** $90^\circ$ for Realistic Synthetic 360° and DTU; $30^\circ$ for LLFF [S1].
*   **Training Duration:** 500 pixel epochs [S1].
*   **Batch Size:** 4,096 [S1].
*   **Optimizer:** Adam with gradient clipping by value at 0.1 and norm at 0.1 [S1].
*   **Learning Rate:** Annealed from $[10^{-3}, 10^{-5}]$ for Real.Syn. 360°, $[2 \times 10^{-3}, 2 \times 10^{-5}]$ for DTU and LLFF [S1].
*   **Warm-up Step:** 512 for Real.Syn. 360° and LLFF; 1024 for DTU [S1].
*   **Delay Multiplier:** $10^{-2}$ for Real.Syn. 360°; $10^{-3}$ for DTU and LLFF [S1].
*   **Loss Balancing Weights ($\lambda$):**
    *   $\lambda_1$ (NLL): Annealed from $[4.0, 10^{-3}]$ (Real.Syn. 360°), $[0.4, 10^{-4}]$ (DTU), $[0.04, 10^{-5}]$ (LLFF) [S1].
    *   $\lambda_2$ (NLL'): Annealed from $[0.4, 10^{-4}]$ (Real.Syn. 360°), $[0.04, 10^{-5}]$ (DTU), $[0.004, 10^{-6}]$ (LLFF) [S1].
    *   $\lambda_3$ (UE): Annealed from $[10^{-4}, 0.1]$ (Real.Syn. 360°), $[10^{-5}, 0.01]$ (DTU), $[10^{-4}, 0.1]$ (LLFF) [S1].
    *   $\lambda_4$ (UE'): Annealed from $[10^{-5}, 0.01]$ (Real.Syn. 360°), $[10^{-6}, 0.001]$ (DTU), $[10^{-6}, 0.001]$ (LLFF) [S1].
    *   $\lambda_5$ (BFC): $0.01$ (Real.Syn. 360°), $0.001$ (DTU), $0.00001$ (LLFF) [S1].
    *   $\lambda_6$ (Ori.): $0.1$ (Real.Syn. 360°), $0.01$ (DTU), $0.001$ (LLFF) [S1].
    *   *Note:* Weights decrease as the number of training views increases to prevent over-regularization [S1].

**Edge Cases and Pitfalls:**
*   **Explicit Normalization:** Using explicitly normalized surface normals ($\hat{\mathbf{n}}$) causes training instability and degenerate results (PSNR 16.72) compared to not using explicit normalization (PSNR 22.24) [S1]. The model should *not* explicitly normalize $\hat{\mathbf{n}}$; it learns to approximate unit vectors naturally through regularization [S1].
*   **Initial Normal Quality:** In the initial training stage, estimated surface normals $\hat{\mathbf{n}}$ are not unit vectors, with most $\|\hat{\mathbf{n}}\|^2$ far from 1. The masking strategy and regularization help concentrate $\|\hat{\mathbf{n}}\|^2$ close to 1 after training [S1].
*   **Ineffective Rays:** Without masking, flipped rays may be cast toward non-photo-consistent points, leading to performance degradation. The masking threshold $\tau$ is critical [S1].

## Operating characteristics
*   **Time and Memory Complexity:** The method adds overhead for generating flipped rays and computing additional losses (UE, BFC, Ori.). The complexity is dominated by the NeRF forward pass, which is repeated for both original and flipped rays. Memory usage increases slightly to store bottleneck features for both ray sets and the flipped ray batch.
*   **Measured Performance:** Achieves SOTA performance on Realistic Synthetic 360°, DTU, and LLFF benchmarks, outperforming baselines by a large margin in 3/4-view settings [S1]. On the evaluation dataset, masking with $\tau=90^\circ$ achieved PSNR 19.55, SSIM 0.767, LPIPS 0.180, and Average Error 0.101, outperforming no masking ($\tau=180^\circ$) which had PSNR 18.76 [S1].
*   **Scaling:** The method is designed for few-shot settings (3-9 views). It does not scale to dense input settings as the regularization is specifically tuned for sparse data [S1].
*   **Strengths:** Improves surface normal accuracy, reduces floating artifacts, and does not require pre-training or heuristic viewpoint sampling [S1].
*   **Limitations:** Relies on the quality of the underlying mixture density model (MixNeRF). Performance is sensitive to the masking threshold $\tau$ and loss balancing weights [S1].
*   **When to Use:** Use for few-shot novel view synthesis (3-9 views) where accurate surface normals and reduced floating artifacts are desired.
*   **When Not to Use:** Do not use for dense input settings or when pre-training on large datasets is feasible and preferred. Avoid if explicit normalization of normals is required by other pipeline components.

## Background
FlipNeRF was proposed in 2023 by researchers including those from the Korean Government-funded NRF and IITP [S1]. It is built upon MixNeRF [S1], which leverages a mixture density model framework [2]. The method addresses the limitation of NeRF requiring dense multi-view images for practical application [S1]. It differs from pre-training methods [5, 6, 12, 19, 29, 35, 39, 43] and unseen viewpoint sampling strategies [25, 15, 17] by deriving rays from estimated normals without heuristic factors [S1]. It also differs from Ref-NeRF [36] by generating additional training rays rather than reparameterizing the radiance field [S1].

## Sources
[S1] FlipNeRF: Flipped Reflection Rays for Few-shot Novel View Synthesis, https://doi.org/10.1109/iccv51070.2023.02092, fetched 2026-10-29
