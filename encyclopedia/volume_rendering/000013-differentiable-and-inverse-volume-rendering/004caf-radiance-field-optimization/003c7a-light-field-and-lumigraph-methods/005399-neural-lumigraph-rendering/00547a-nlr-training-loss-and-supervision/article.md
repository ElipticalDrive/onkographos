# NLR++ Training Loss and Supervision

Also called: NLR++, NLR++ Loss, NLR++ Optimization

## Overview
NLR++ Training Loss and Supervision defines the end-to-end differentiable optimization objective for the Neural Lumigraph Representation (NLR++), which jointly reconstructs shape via a Signed Distance Function (SDF) and appearance via a radiance field from posed 2D images [S1, unofficial]. It distinguishes itself from standard volumetric rendering losses by introducing a soft mask loss on the SDF minimum along rays and a specific training schedule that decouples shape approximation from feature refinement to accelerate convergence [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** A batch of $k \le N$ input images $\{I_n\}_{n=1}^k$, their corresponding binary object masks $\{M_n\}_{n=1}^k$, and viewpoint parameters $\{C_n\}_{n=1}^k$ [S1, unofficial].
*   **Outputs:** A synthesized target image $\hat{I}_t$, a rendered mask $\hat{M}_t$, and updated model parameters after minimizing the total loss [S1, unofficial].

**Data Structures**
*   **SDF ($\Phi_\theta$):** An implicit function mapping 3D coordinates to signed distance values, initialized as a unit sphere of radius 0.5 using a pre-trained SIREN [S1, unofficial].
*   **Radiance Field:** Defined by the encoder $E_\xi$, decoder $D_\psi$, and aggregator $\Gamma_\zeta$, which map view-dependent features to color [S1, unofficial].
*   **Model Parameters:** $\Theta = [\xi, \psi, \zeta, \theta]$ [S1, unofficial].

**Numbered Steps**
1.  **Batch Sampling:** For each training iteration, sample a set of $k \le N$ images $\{I_n\}_{n=1}^k$ from the available $N$ input images [S1, unofficial].
2.  **Target Selection:** Designate one image as the ground-truth target $I_t$, with its mask $M_t$ and viewpoint $C_t$ [S1, unofficial].
3.  **Multi-Target Gradient Computation:** To improve gradient accuracy, treat $l < k$ images in the batch as target images, computing gradients for each from the other images in the batch, limited by GPU memory [S1, unofficial].
4.  **Synthesis:** Generate the synthesized target image $\hat{I}_t$ and rendered mask $\hat{M}_t$ from viewpoint $C_t$ using the NLR++ image formation model [S1, unofficial].
5.  **Loss Calculation:** Compute the total loss $\mathcal{L}$ as the sum of image reconstruction loss, soft mask loss, and eikonal regularization [S1, unofficial].
6.  **Scheduled Shape Supervision:** Enforce shape-related losses $\mathcal{L}_M$ and $\mathcal{L}_E$ only for the first $t_1$ iterations, and then every $t_2$ iterations thereafter, allowing initial shape approximation before refining with better feature encoding [S1, unofficial].
7.  **Optimization:** Update parameters $\Theta$ using the Adam optimizer [S1, unofficial].

**Formulas**
The total loss is defined as [S1, unofficial]:
$$
\mathcal{L}(\{\hat{I}_t, \hat{M}_t\}, \{I_t, M_t\}, \Phi_\theta) = \mathcal{L}_R(\hat{I}_t, \{I_t, M_t\}) + \lambda_1 \mathcal{L}_M(\Phi_\theta, M_t) + \lambda_2 \mathcal{L}_E(\Phi_\theta)
$$

1.  **Image Reconstruction Loss ($\mathcal{L}_R$):** A masked L1 loss on rendered images [S1, unofficial]:
    $$
    \mathcal{L}_R(\hat{I}_t, \{I_t, M_t\}) = \frac{1}{P} \sum_{p | M_t[p]=1} |I_t[p] - \hat{I}_t[p]|
    $$
    where $P$ is the number of pixels where $M_t[p]=1$.

2.  **Soft Mask Loss ($\mathcal{L}_M$):** Applied to bootstrap neural shape learning from object masks [S1, unofficial]:
    $$
    \mathcal{L}_M(\Phi_\theta, M_t) = \frac{1}{\alpha \sum_{p | M_t[p]=0 \lor \Phi_{\min}[p]<\tau} M_t[p]} \sum_{p | M_t[p]=0 \lor \Phi_{\min}[p]<\tau} \text{BCE}(\text{sigmoid}(-\alpha \Phi_{\min}[p]), M_t[p])
    $$
    where:
    *   $\Phi_{\min}[p]$ is the minimum SDF value along the ray traced from pixel $p$.
    *   $\tau$ is a threshold for whether the zero level set $L_0(\Phi_\theta)$ has been intersected.
    *   $\alpha$ is a softness hyperparameter.
    *   $\text{BCE}(y, \hat{y}) = -[y \log(\hat{y}) + (1-y) \log(1-\hat{y})]$ is the Binary Cross Entropy.

3.  **Eikonal Regularization ($\mathcal{L}_E$):** Enforces the eikonal constraint on randomly sampled points $p_i \in \mathbb{R}^3$ in a unit cube containing the scene [S1, unofficial]:
    $$
    \mathcal{L}_E(\Phi_\theta) = \frac{1}{P} \sum_{i=0}^{P} ||(||\nabla_{p_i} \Phi_\theta(p_i)||^2 - 1)||^2_2
    $$

**Parameters**
*   $\alpha$: Softness hyperparameter for mask loss. Starting value: 50 [S1, unofficial].
*   $\tau$: Threshold for zero level set intersection. Starting value: $1 \times 10^{-3}$ [S1, unofficial].
*   $\lambda_1$: Weight for mask loss. Calculated as $1 \times 10^2 / \alpha$ [S1, unofficial].
*   $\lambda_2$: Weight for eikonal regularization. Starting value: 3 [S1, unofficial].
*   $t_1$: Number of initial iterations to enforce shape losses. Starting value: 50 [S1, unofficial].
*   $t_2$: Frequency (in iterations) to enforce shape losses after $t_1$. Starting value: 7 [S1, unofficial].
*   Learning Rates: Adam optimizer with starting LR $1 \times 10^{-4}$ for $\Phi$ and $5 \times 10^{-4}$ for $E, D, \Gamma$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nlr_plus_plus(batch_images, batch_masks, batch_cameras, 
                        model_params, optimizer, hyperparams):
    """
    Performs one training step for NLR++.
    Args:
        batch_images: List of k images {I_n}
        batch_masks: List of k masks {M_n}
        batch_cameras: List of k camera parameters {C_n}
        model_params: Parameters [xi, psi, zeta, theta] for E, D, Gamma, Phi
        optimizer: Adam optimizer instance
        hyperparams: Dict containing alpha, tau, lambda1, lambda2, t1, t2, l, P_eikonal
    Returns:
        loss_value: Scalar loss value
    """
    k = len(batch_images)
    l = hyperparams['l']
    t1 = hyperparams['t1']
    t2 = hyperparams['t2']
    current_iter = get_current_iteration() # [G] Global iteration counter
    alpha = hyperparams['alpha']
    tau = hyperparams['tau']
    lambda1 = hyperparams['lambda1']
    lambda2 = hyperparams['lambda2']
    P_eikonal = hyperparams['P_eikonal']
    
    total_loss = 0.0
    
    # Multi-Target Gradient Computation: treat l < k images as targets [S1, unofficial]
    for idx in range(l):
        I_t = batch_images[idx]
        M_t = batch_masks[idx]
        C_t = batch_cameras[idx]
        
        # Synthesis: generate synthesized target image and mask [S1, unofficial]
        I_hat_t, M_hat_t = synthesize_image(C_t, batch_images, model_params)
        
        # Image Reconstruction Loss (Masked L1) [S1, unofficial]
        mask_pixels = [p for p in range(len(I_t)) if M_t[p] == 1]
        lr_loss = sum(abs(I_t[p] - I_hat_t[p]) for p in mask_pixels) / len(mask_pixels) if mask_pixels else 0.0
        
        # Scheduled Shape Supervision [S1, unofficial]
        lm_loss = 0.0
        le_loss = 0.0
        if current_iter <= t1 or current_iter % t2 == 0:
            # Soft Mask Loss [S1, unofficial]
            valid_pixels = []
            for p in range(len(M_t)):
                ray = get_ray_from_pixel(p, C_t) # [G]
                phi_min = find_min_sdf_along_ray(ray, model_params.phi) # [G]
                if M_t[p] == 0 or phi_min < tau:
                    valid_pixels.append((p, phi_min))
            
            denom = sum(M_t[p] for p, _ in valid_pixels)
            num = sum(binary_cross_entropy(sigmoid(-alpha * phi_min), M_t[p]) for p, phi_min in valid_pixels)
            lm_loss = num / (alpha * denom) if denom > 0 else 0.0
            
            # Eikonal Regularization [S1, unofficial]
            points = sample_random_points(P_eikonal, bounds=[-1, 1]) # [G]
            le_loss = sum((norm(compute_gradient(model_params.phi, p_i))**2 - 1)**2 for p_i in points) / len(points)
            
        # Combine Losses [S1, unofficial]
        loss = lr_loss + lambda1 * lm_loss + lambda2 * le_loss
        total_loss += loss
        
    total_loss /= l
    optimizer.zero_grad()
    total_loss.backward()
    optimizer.step()
    return total_loss

# Standard math/vector operations [G]
def sigmoid(x): return 1.0 / (1.0 + exp(-x))
def binary_cross_entropy(y_pred, y_true): return -(y_true * log(y_pred) + (1 - y_true) * log(1 - y_pred))
def norm(v): return sqrt(sum(x**2 for x in v))
def compute_gradient(phi, p): return [phi(p + eps * d) - phi(p - eps * d) for d in [1, 0, 0, 0, 1, 0, 0, 0, 1]] # Central difference
def sample_random_points(n, bounds): return [[uniform(bounds[0], bounds[1]) for _ in range(3)] for _ in range(n)]
def get_ray_from_pixel(p, C): return [C.origin + p * C.direction] # [G]
def find_min_sdf_along_ray(ray, phi): return min(phi(pt) for pt in ray) # [G]
def synthesize_image(C, context, params): return render_image(C, context, params) # [G]
def get_current_iteration(): return 0 # [G] Placeholder for external state
```

**Edge Cases and Pitfalls**
*   **Compute Bottleneck:** $\mathcal{L}_M$ requires finding the minimum of $\Phi_\theta$ along a ray, which necessitates dense sampling of the network and accounts for most of the compute time of each forward pass [S1, unofficial].
*   **Precision:** The threshold $\tau$ must be small enough to detect the zero level set but large enough to avoid noise. The softness parameter $\alpha$ controls the sharpness of the mask boundary; if too high, gradients may become unstable [S1, unofficial].

## Subtopics

### MetaNLR++ Meta-Learning Initialization
MetaNLR++ uses meta-learning to learn a prior over feature encoding, decoding, aggregation, and shape representation by optimizing the initial network weights $\Theta_0$ for fast convergence on unseen scenes [S1, unofficial]. It differs from standard NLR++ by using the Reptile algorithm to update $\Theta_0$ based on inner-loop optimization on sampled tasks [S1, unofficial]. The meta-learning objective is to find initial weights $\Theta_0^*$ that minimize the expected loss after $m$ optimization steps on an unseen scene $T$: $\Theta_0^* = \arg \min_{\Theta_0} \mathbb{E}_{T \sim \mathcal{T}} [\mathcal{L}(\Theta_m(\Theta_0, T))]$ [S1, unofficial]. The update rule is $\Theta_0^{j+1} = \Theta_0^j + \beta(\Theta_m(\Theta_0^j, T_j) - \Theta_0^j)$, where $\beta$ is the meta-learning rate (starting value $1 \times 10^{-1}$) and $m$ is the fixed number of inner-loop steps [S1, unofficial]. This approach significantly reduces training time; for example, on the ShapeNet chairs split, MetaNLR++ reaches 30dB PSNR in 1.5 minutes compared to NLR++'s 3.6 minutes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training is dominated by dense sampling for $\mathcal{L}_M$, accounting for most compute time per forward pass [S1, unofficial]. Multi-target batching adds forward passes to $\Gamma$ and $D$, which are fast to evaluate [S1, unofficial].
*   **Memory Complexity:** Number of target images $l$ treated as targets is limited by GPU memory [S1, unofficial].
*   **Performance:** On ShapeNet, NLR++ converges slower than MetaNLR++; on chairs split, NLR++ takes 3.6 min to reach 30dB PSNR vs MetaNLR++'s 1.5 min [S1, unofficial].
*   **Strengths:** Scheduled shape supervision decouples shape approximation from feature refinement for faster initial convergence [S1, unofficial]. CNN encoder/decoder increases image loss receptive field for more meaningful gradients [S1, unofficial].
*   **Limitations:** Dense sampling for $\mathcal{L}_M$ is computationally expensive [S1, unofficial]. NLR++ without meta-learning converges slower than MetaNLR++ [S1, unofficial].
*   **When to Use:** Use NLR++ for standard neural lumigraph representation without meta-learning overhead [S1, unofficial]. Use MetaNLR++ when rapid convergence on unseen scenes is critical [S1, unofficial].

## Background
NLR++ was introduced as an end-to-end differentiable neural lumigraph representation optimized to reconstruct target views from posed 2D images using a CNN encoder/decoder and an SDF for shape [S1, unofficial]. It serves as the baseline for MetaNLR++, which improves upon it using meta-learning [S1, unofficial]. The implementation details, including hyperparameters and training schedules, are described in the supplementary material of the source paper [S1, unofficial]. Experiments were conducted on Nvidia Quadro RTX8000 and RTX6000 GPUs [S1, unofficial].

## Sources
[S1] Fast Training of Neural Lumigraph Representations using Meta Learning, https://doi.org/10.48550/arxiv.2106.14942, fetched 2026-10-26 (unofficial)
