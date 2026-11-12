# FlipNeRF

Also called: Flipped Reflection Rays for Few-shot Novel View Synthesis

## Overview
FlipNeRF is a regularization framework for few-shot novel view synthesis that generates auxiliary "flipped reflection rays" from original input rays and estimated surface normals to improve rendering quality and surface normal accuracy with sparse inputs [S1]. It sets itself apart from sibling methods like [Ref-NeRF](11859-deceptive-nerf-3dgs.md) by generating *additional* training rays rather than reparameterizing the input viewing direction, and from [Uncertainty-aware NeRF](12968-uncertainty-aware-nerf.md) by specifically targeting floating artifacts through uncertainty-aware emptiness regularization [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse set of input images with known camera poses; the underlying radiance field function $F_\Theta$ (typically built upon MixNeRF [S1]).
*   **Outputs:** Rendered color images for novel views; estimated surface normals $\hat{\mathbf{n}}$; bottleneck features $\mathbf{b}$.

**Data Structures:**
The method relies on the mixture density model framework of MixNeRF [S1]. For each ray $\mathbf{r}$, the model outputs a mixture of $M$ components. Each component $i$ has:
*   Blending weight $w_i$ (or $\pi_i$ in mixture notation).
*   Mean color $\mu_i^{\mathbf{c}}$.
*   Scale parameter $\beta_i^{\mathbf{c}}$ (representing uncertainty/variance) for each color channel $c \in \{r, g, b\}$ [S1].
*   Sample position $x_i$ along the ray.
*   Estimated normal vector $\mathbf{n}_i$ derived from the density gradient [S1].

**Numbered Steps:**

1.  **Estimate Surface Normals:**
    Compute the estimated surface normal $\hat{\mathbf{n}}$ for a ray $\mathbf{r}$ as the weighted sum of blending weights and estimated normal vectors $\mathbf{n}_i$ along the ray [S1]:
    $$
    \hat{\mathbf{n}} = \sum_{i=1}^{M} w_i \mathbf{n}_i
    $$
    where $\mathbf{n}_i$ is the normalized gradient of the volume density at sample $i$ [S1].

2.  **Identify Peak Weight Sample:**
    Find the sample index $s$ with the highest blending weight along the original ray $\mathbf{r}$. Calculate the position of this peak sample $\mathbf{p}_s$ [S1]:
    $$
    \mathbf{p}_s = \mathbf{o} + t_s \mathbf{d}
    $$
    where $\mathbf{o}$ is the ray origin, $\mathbf{d}$ is the ray direction, and $t_s$ is the distance to the $s$-th sample [S1].

3.  **Calculate Flipped Reflection Direction:**
    Compute the flipped direction $\mathbf{d}'$ using the reflection formula [S1]:
    $$
    \mathbf{d}' = 2(\mathbf{d} \cdot \hat{\mathbf{n}})\hat{\mathbf{n}} - \mathbf{d}
    $$

4.  **Calculate Flipped Ray Origin:**
    Determine the origin $\mathbf{o}'$ for the flipped ray such that the peak sample of the flipped ray coincides with $\mathbf{p}_s$ [S1]:
    $$
    \mathbf{o}' = \mathbf{p}_s - t_s \mathbf{d}'
    $$
    The flipped reflection ray is defined as $\mathbf{r}'(t) = \mathbf{o}' + t \mathbf{d}'$ [S1].

5.  **Apply Masking Strategy:**
    Filter out ineffective flipped rays using a mask $M(\mathbf{r}')$ based on the angle between the flipped direction and the estimated normal [S1]:
    $$
    M(\mathbf{r}') = \begin{cases} 
    1 & \text{if } \arccos(-(\hat{\mathbf{d}} \cdot \hat{\mathbf{n}})) < \tau \\
    0 & \text{otherwise}
    \end{cases}
    $$
    where $\tau$ is the masking threshold (typically $90^\circ$ for DTU/Realistic Synthetic 360°, $30^\circ$ for LLFF) [S1]. Only rays with $M(\mathbf{r}') = 1$ are used for training [S1].

6.  **Compute Regularization Losses:**
    *   **Uncertainty-aware Emptiness Loss (UE Loss):** Reduces floating artifacts by penalizing uncertain rays [S1]:
        $$
        l_{\text{UE}}(\mathbf{r}) = \frac{1}{M} \sum_{i=1}^{M} \log(1 + \rho \cdot \eta \cdot w_i)
        $$
        where $\rho = \frac{1}{3} \sum_{c}^{\{r,g,b\}} \sum_{i=1}^{M} \beta_i^c$ is the uncertainty metric derived from scale parameters, and $\eta$ is a scaling factor [S1]. This loss is applied to both original rays $\mathbf{r}$ and valid flipped rays $\mathbf{r}'$ (denoted $\mathcal{L}_{\text{UE}}$ and $\mathcal{L}_{\text{UE}}'$ respectively) [S1].
    *   **Bottleneck Feature Consistency Loss (BFC Loss):** Encourages consistency between bottleneck features $\mathbf{b}$ (outputs of the spatial MLP) of original and flipped rays using Jensen-Shannon Divergence (JSD) [S1]:
        $$
        l_{\text{BFC}}(\mathbf{r}, \mathbf{r}') = \text{JSD}(\psi(\mathbf{b}), \psi(\mathbf{b}'))
        $$
        where $\psi(\cdot)$ is the softmax function [S1].
    *   **Orientation Loss:** Penalizes backward-facing normal vectors using the underlying density gradient normal $\mathbf{n}_i$ instead of predicted normals [S1]:
        $$
        l_{\text{Ori.}}(\mathbf{r}) = \sum_{i=1}^{M} w_i \max(0, \mathbf{n}_i \cdot \hat{\mathbf{d}})^2
        $$

7.  **Minimize Total Loss:**
    Train the model by minimizing the total loss [S1]:
    $$
    \mathcal{L}_\text{Total} = \mathcal{L}_\text{MSE} + \lambda_1 \mathcal{L}_\text{NLL} + \lambda_2 \mathcal{L}_\text{NLL}' + \lambda_3 \mathcal{L}_\text{UE} + \lambda_4 \mathcal{L}_\text{UE}' + \lambda_5 \mathcal{L}_\text{BFC} + \lambda_6 \mathcal{L}_\text{Ori.}
    $$
    where $\mathcal{L}_\text{MSE}$ is the mean squared error between rendered and ground truth colors, and $\mathcal{L}_\text{NLL}$ is the negative log-likelihood loss for the mixture model [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flipnerf_step(input_ray, gt_color, radiance_field, hyperparams):
    """
    input_ray: dict with 'origin' (o), 'direction' (d), 'samples' (list of t values)
    gt_color: ground truth color for the ray
    radiance_field: function F_Theta that returns mixture parameters for a ray
    hyperparams: dict containing tau, eta, lambda weights, etc.
    """
    # 1. Forward pass on original ray
    mixture_params = radiance_field(input_ray)
    # mixture_params contains: weights (w), means (mu), scales (beta), normals (n)
    
    # 2. Estimate surface normal
    # n_hat is weighted sum of sample normals
    n_hat = sum(w * n for w, n in zip(mixture_params.weights, mixture_params.normals))
    
    # 3. Find peak weight sample index s
    s = argmax(mixture_params.weights)
    t_s = input_ray.samples[s]
    p_s = input_ray.origin + t_s * input_ray.direction
    
    # 4. Calculate flipped ray parameters
    d_prime = 2 * dot(input_ray.direction, n_hat) * n_hat - input_ray.direction
    o_prime = p_s - t_s * d_prime
    
    flipped_ray = {
        'origin': o_prime,
        'direction': d_prime,
        'samples': input_ray.samples # Reuse sample distances
    }
    
    # 5. Masking
    # Calculate angle theta between -d and n_hat
    # cos_theta = dot(-d, n_hat) / (|d| * |n_hat|)
    # Note: d is usually unit vector. n_hat might not be.
    cos_theta = dot(-input_ray.direction, n_hat) / norm(n_hat)
    theta = arccos(cos_theta)
    
    is_valid = theta < hyperparams.tau
    
    if not is_valid:
        # Return gradients for original ray only if flipped ray is invalid
        # Or simply skip flipped ray contributions
        loss_mse = mse(render_color(mixture_params), gt_color)
        loss_nll = nll_loss(mixture_params)
        loss_ue = ue_loss(mixture_params, hyperparams.eta)
        loss_ori = orientation_loss(mixture_params, input_ray.direction)
        return loss_mse + hyperparams.lambda1 * loss_nll + \
               hyperparams.lambda3 * loss_ue + hyperparams.lambda6 * loss_ori

    # 6. Forward pass on flipped ray
    flipped_mixture_params = radiance_field(flipped_ray)
    
    # 7. Compute Losses
    # Original Ray Losses
    loss_mse = mse(render_color(mixture_params), gt_color)
    loss_nll = nll_loss(mixture_params)
    loss_ue = ue_loss(mixture_params, hyperparams.eta)
    loss_ori = orientation_loss(mixture_params, input_ray.direction)
    
    # Flipped Ray Losses
    loss_nll_prime = nll_loss(flipped_mixture_params)
    loss_ue_prime = ue_loss(flipped_mixture_params, hyperparams.eta)
    
    # Bottleneck Feature Consistency
    b = get_bottleneck_features(input_ray, radiance_field)
    b_prime = get_bottleneck_features(flipped_ray, radiance_field)
    loss_bfc = js_divergence(softmax(b), softmax(b_prime))
    
    # Total Loss
    total_loss = loss_mse + \
                 hyperparams.lambda1 * loss_nll + \
                 hyperparams.lambda2 * loss_nll_prime + \
                 hyperparams.lambda3 * loss_ue + \
                 hyperparams.lambda4 * loss_ue_prime + \
                 hyperparams.lambda5 * loss_bfc + \
                 hyperparams.lambda6 * loss_ori
                 
    return total_loss

def ue_loss(mixture_params, eta):
    M = len(mixture_params.weights)
    # rho is average sum of scale parameters across RGB channels and samples
    # Assuming mixture_params.scales is shape (M, 3)
    rho = (1/3) * sum(sum(beta_c for beta_c in beta) for beta in mixture_params.scales) / M
    
    loss = 0
    for i in range(M):
        w_i = mixture_params.weights[i]
        loss += log(1 + rho * eta * w_i)
    return loss / M

def orientation_loss(mixture_params, ray_direction):
    loss = 0
    for w, n in zip(mixture_params.weights, mixture_params.normals):
        # n is the density gradient normal
        dot_prod = dot(n, ray_direction)
        loss += w * max(0, dot_prod)**2
    return loss
```

**Parameters:**
*   **Masking Threshold ($\tau$):** $90^\circ$ for Realistic Synthetic 360° and DTU; $30^\circ$ for LLFF [S1].
*   **Training Duration:** 500 pixel epochs [S1].
*   **Batch Size:** 4,096 [S1].
*   **Optimizer:** Adam with gradient clipping by value at 0.1 and norm at 0.1 [S1].
*   **Learning Rate Annealing:**
    *   Realistic Synthetic 360°: $[10^{-3}, 10^{-5}]$ [S1].
    *   DTU: $[2 \times 10^{-3}, 2 \times 10^{-5}]$ [S1].
    *   LLFF: $[2 \times 10^{-3}, 2 \times 10^{-5}]$ [S1].
*   **Warm-up Steps:** 512 (Realistic Synthetic 360°), 1024 (DTU), 512 (LLFF) [S1].
*   **Delay Multiplier:** $10^{-2}$ (Realistic Synthetic 360°), $10^{-3}$ (DTU, LLFF) [S1].
*   **Loss Balancing Weights ($\lambda$):**
    *   $\lambda_1$ (NLL): Annealed from $[4.0, 10^{-3}]$ (Realistic Synthetic 360°), $[4 \times 10^{-1}, 10^{-4}]$ (DTU), $[4 \times 10^{-2}, 10^{-5}]$ (LLFF) [S1].
    *   $\lambda_2$ (NLL'): Annealed from $[4 \times 10^{-1}, 10^{-4}]$ (Realistic Synthetic 360°), $[4 \times 10^{-2}, 10^{-5}]$ (DTU), $[4 \times 10^{-3}, 10^{-6}]$ (LLFF) [S1].
    *   $\lambda_3$ (UE): Annealed from $[10^{-4}, 10^{-1}]$ (Realistic Synthetic 360°), $[10^{-5}, 10^{-2}]$ (DTU), $[10^{-4}, 10^{-1}]$ (LLFF) [S1].
    *   $\lambda_4$ (UE'): Annealed from $[10^{-5}, 10^{-2}]$ (Realistic Synthetic 360°), $[10^{-6}, 10^{-3}]$ (DTU), $[10^{-6}, 10^{-3}]$ (LLFF) [S1].
    *   $\lambda_5$ (BFC): $10^{-2}$ (Realistic Synthetic 360°), $10^{-3}$ (DTU), $10^{-5}$ (LLFF) [S1].
    *   $\lambda_6$ (Ori): $10^{-1}$ (Realistic Synthetic 360°), $10^{-2}$ (DTU), $10^{-3}$ (LLFF) [S1].

**Edge Cases and Pitfalls:**
*   **Explicit Normalization:** Explicitly normalizing the estimated surface normals $\hat{\mathbf{n}}$ causes training instability and degenerate results (PSNR 16.72 vs 22.24 without normalization) [S1]. The method relies on regularization and masking to naturally drive $\|\hat{\mathbf{n}}\|^2$ close to 1 [S1].
*   **Initial Training Stage:** In the initial phase, $\hat{\mathbf{n}}$ are not unit vectors, with $\|\hat{\mathbf{n}}\|^2$ far from 1 [S1]. The masking strategy is critical to prevent these inaccurate normals from generating misleading flipped rays [S1].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Achieves SOTA performance on Realistic Synthetic 360°, DTU, and LLFF benchmarks, outperforming baselines by a large margin in 3/4-view settings [S1]. On the evaluation dataset, masking with $\tau=90^\circ$ achieved PSNR 19.55, SSIM 0.767, LPIPS 0.180, and Average Error 0.101, outperforming no masking ($\tau=180^\circ$) which had PSNR 18.76 [S1].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Effective for few-shot novel view synthesis; improves surface normal accuracy; reduces floating artifacts; does not require pre-training or large-scale datasets; does not rely on heuristic factors for unseen viewpoint sampling [S1].
*   **Limitations:** Performance depends on the accuracy of the initial surface normal estimation; requires careful tuning of loss balancing weights [S1].
*   **When to use:** Sparse input views (e.g., 3-9 views); scenes where floating artifacts are prominent; when accurate surface normals are desired [S1].
*   **When not:** Dense input views (where standard NeRF variants may suffice); scenes with extreme lighting variations not captured by the mixture model [S1].

## Background
FlipNeRF was proposed in 2023 by researchers supported by NRF, IITP, and Samsung Electronics [S1]. It is built upon MixNeRF [S1], which leverages mixture density models to represent input rays. The method addresses the limitation of existing regularization methods that focus on depth estimation or require pre-training, by instead focusing on surface normal reconstruction via flipped reflection rays [S1]. It is evaluated on Realistic Synthetic 360°, DTU, and LLFF benchmarks [S1].

## Sources
[S1] FlipNeRF: Flipped Reflection Rays for Few-shot Novel View Synthesis, https://doi.org/10.1109/iccv51070.2023.02092, fetched 2026-10-29
