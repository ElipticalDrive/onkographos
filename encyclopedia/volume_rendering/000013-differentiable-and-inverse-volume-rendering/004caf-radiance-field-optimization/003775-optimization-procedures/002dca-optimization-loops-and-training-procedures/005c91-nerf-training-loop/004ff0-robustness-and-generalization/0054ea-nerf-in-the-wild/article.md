# NeRF in the Wild

Also called: Martin-Brualla et al, NeRF-W (NeRF in the Wild), Neural Radiance Fields for Unconstrained Photo Collections

## Overview
NeRF in the Wild (NeRF-W) is an extension of Neural Radiance Fields designed to handle unconstrained, "in-the-wild" photo collections where strict consistency assumptions fail due to variable illumination, auto-exposure, and transient occluders [S1, S4]. It disentangles the scene into a shared static geometry and image-dependent components: a learned low-dimensional appearance embedding for photometric variations and a secondary volumetric radiance field with uncertainty modeling for transient objects [S1, S2]. Unlike its siblings, NeRF-W explicitly models both appearance and transient geometry simultaneously, whereas [Aug-NeRF](10294-aug-nerf.md) focuses on robustness via perturbations, [NeRF Ensemble with Conjoint Pruning](12608-nerf-ensemble-with-conjoint-pruning.md) on compression, and [ContraNeRF](13659-contranerf.md) on cross-view consistency without per-scene optimization.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ images $\{I_i\}$ of the same scene with known camera poses (estimated via COLMAP [S1]); 3D position $\mathbf{x}$, viewing direction $\mathbf{d}$, and image index $i$.
*   **Outputs:** Rendered color $\hat{C}_i(\mathbf{r})$ for a ray $\mathbf{r}$; static density $\sigma(\mathbf{x})$, static color $\mathbf{c}(\mathbf{x}, \mathbf{d})$, transient density $\sigma_i^{(\tau)}(\mathbf{x})$, transient color $\mathbf{c}_i^{(\tau)}(\mathbf{x}, \mathbf{d})$, and uncertainty variance $\beta_i(\mathbf{x})^2$.

**Data Structures:**
1.  **Static MLP ($MLP_{\theta_1}$):** Takes $\mathbf{x}$ and positional encoding $\gamma(\mathbf{x})$. Outputs static density $\sigma(\mathbf{x})$ and intermediate features $\mathbf{z}(\mathbf{x})$.
2.  **Appearance Embedding ($\mathbf{l}_i^{(a)}$):** A learnable vector of length $n^{(a)}$ for each image $i$.
3.  **Color MLP ($MLP_{\theta_2}$):** Takes $\mathbf{z}(\mathbf{x})$, viewing direction $\mathbf{d}$ (encoded), and $\mathbf{l}_i^{(a)}$. Outputs static color $\mathbf{c}_i(\mathbf{x}, \mathbf{d})$.
4.  **Transient MLP ($MLP_{\theta_3}$):** Takes $\mathbf{x}$, $\mathbf{d}$, and a transient embedding $\mathbf{l}_i^{(\tau)}$ (length $n^{(\tau)}$). Outputs transient density $\sigma_i^{(\tau)}(\mathbf{x})$ and transient color $\mathbf{c}_i^{(\tau)}(\mathbf{x}, \mathbf{d})$.
5.  **Uncertainty MLP ($MLP_{\theta_4}$):** Takes $\mathbf{x}$ and $\mathbf{d}$. Outputs variance $\beta_i(\mathbf{x})^2$.

**Numbered Steps:**

1.  **Initialization:** Initialize MLPs with random weights. Initialize appearance embeddings $\{\mathbf{l}_i^{(a)}\}_{i=1}^N$ and transient embeddings $\{\mathbf{l}_i^{(\tau)}\}_{i=1}^N$ to zero or small random values [S1].
2.  **Sampling:** For each training image $I_i$, sample rays $\mathbf{r}$. Sample $K$ points $\{t_k\}_{k=1}^K$ along each ray using stratified sampling or importance sampling based on a coarse model [S1].
3.  **Static Query:** For each point $t_k$, query $MLP_{\theta_1}$ to get $\sigma(t_k)$ and $\mathbf{z}(t_k)$. Note: $\sigma(t_k)$ is shared across all images [S1, S2].
4.  **Static Color Query:** Query $MLP_{\theta_2}$ with $\mathbf{z}(t_k)$, $\mathbf{d}$, and $\mathbf{l}_i^{(a)}$ to get static color $\mathbf{c}_i(t_k)$ [S1].
5.  **Transient Query:** Query $MLP_{\theta_3}$ with $t_k$, $\mathbf{d}$, and $\mathbf{l}_i^{(\tau)}$ to get transient density $\sigma_i^{(\tau)}(t_k)$ and transient color $\mathbf{c}_i^{(\tau)}(t_k)$ [S1].
6.  **Uncertainty Query:** Query $MLP_{\theta_4}$ with $t_k$ and $\mathbf{d}$ to get variance $\beta_i(t_k)^2$ [S1].
7.  **Rendering (Composite):** Compute the expected color $\hat{C}_i(\mathbf{r})$ by alpha-compositing both static and transient components [S1]:
    $$
    \hat{C}_i(\mathbf{r}) = \sum_{k=1}^K \alpha_k \mathbf{c}_i(t_k) + \sum_{k=1}^K \alpha_k^{(\tau)} \mathbf{c}_i^{(\tau)}(t_k)
    $$
    where the transmittance $T_i(t_k)$ accounts for both static and transient densities:
    $$
    T_i(t_k) = \exp\left( -\sum_{k'=1}^{k-1} (\sigma(t_{k'}) + \sigma_i^{(\tau)}(t_{k'})) \delta_{k'} \right)
    $$
    and weights are $\alpha_k = T_i(t_k)(1 - \exp(-\sigma(t_k)\delta_k))$ and $\alpha_k^{(\tau)} = T_i(t_k)(1 - \exp(-\sigma_i^{(\tau)}(t_k)\delta_k))$ [S1].
8.  **Loss Calculation:** Compute the loss $L_i(\mathbf{r})$ for the ray [S1]:
    $$
    L_i(\mathbf{r}) = \frac{(C_i(\mathbf{r}) - \hat{C}_i(\mathbf{r}))^2}{2\beta_i(\mathbf{r})^2} + \frac{\log \beta_i(\mathbf{r})^2}{2} + \frac{\lambda_u}{2K} \sum_{k=1}^K \sigma_i^{(\tau)}(t_k)
    $$
    The first two terms are the negative log-likelihood of the color difference assuming Gaussian noise with variance $\beta_i(\mathbf{r})^2$. The third term is an L1 regularization on transient density to prevent it from explaining static features [S1].
9.  **Coarse Loss:** Additionally, compute a coarse rendering $\hat{C}_c^i(\mathbf{r})$ using only the static head and appearance embeddings, and add $\frac{1}{2} \| C_i(\mathbf{r}) - \hat{C}_c^i(\mathbf{r}) \|^2$ to the loss [S1].
10. **Optimization:** Minimize the total loss over all rays and images using Adam optimizer [S1]. Optimize MLP parameters $\theta$ and embeddings $\{\mathbf{l}_i^{(a)}\}, \{\mathbf{l}_i^{(\tau)}\}$ [S1].
11. **Test-Time Evaluation:** For a test image, optimize a single appearance embedding $\mathbf{l}^{(a)}$ using only the left half of the image to avoid leakage, then evaluate metrics on the right half [S1]. Render only static $\sigma(\mathbf{x})$ and $\mathbf{c}(\mathbf{x}, \mathbf{d})$ by omitting transient and uncertainty fields [S1].

**Parameters:**
*   $\lambda_u$: Regularization weight for transient density. Not given in the sources.
*   $\beta_{min}$: Hyperparameter $>0$ added to the softplus activation for $\beta_i(\mathbf{x})^2$ to ensure minimum importance [S1].
*   $n^{(a)}, n^{(\tau)}$: Dimensions of appearance and transient embeddings. Not given in the sources.
*   Batch size: 2048 [S1].
*   Training steps: 300,000 [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def softplus(x, beta_min):
    # Softplus activation shifted by beta_min
    return np.log1p(np.exp(x)) + beta_min

def positional_encoding(x, num_freqs):
    # Standard NeRF positional encoding
    # x: (..., 3)
    # Returns: (..., 3 * 2 * num_freqs)
    scales = 2.0 ** np.arange(num_freqs)
    shaped = x.reshape(-1, 1, 3) * scales.reshape(1, -1, 1)
    shaped = shaped.reshape(-1, 3 * num_freqs)
    return np.concatenate([np.sin(shaped), np.cos(shaped)], axis=-1)

def render_ray(x, d, l_a, l_tau, beta_min, lambda_u, 
               mlp_static, mlp_color, mlp_transient, mlp_uncertainty, 
               K, delta):
    """
    Renders a single ray and computes loss.
    x: (K, 3) sampled points
    d: (3,) viewing direction
    l_a: (n_a,) appearance embedding
    l_tau: (n_tau,) transient embedding
    """
    # 1. Static Query
    # mlp_static takes x and returns sigma and z
    sigma, z = mlp_static(x) # sigma: (K,), z: (K, feature_dim)
    
    # 2. Static Color Query
    # mlp_color takes z, d, l_a
    c_static = mlp_color(z, d, l_a) # (K, 3)
    
    # 3. Transient Query
    # mlp_transient takes x, d, l_tau
    sigma_tau, c_tau = mlp_transient(x, d, l_tau) # sigma_tau: (K,), c_tau: (K, 3)
    
    # 4. Uncertainty Query
    # mlp_uncertainty takes x, d
    beta_sq = softplus(mlp_uncertainty(x, d), beta_min) # (K,)
    
    # 5. Alpha Compositing
    # Total density
    sigma_total = sigma + sigma_tau
    
    # Transmittance
    # T_k = exp(-sum_{j<k} sigma_total_j * delta_j)
    # Using cumulative sum
    cum_sum = np.cumsum(sigma_total * delta)
    T = np.exp(-np.concatenate([[0], cum_sum[:-1]]))
    
    # Alpha weights
    alpha = T * (1 - np.exp(-sigma * delta))
    alpha_tau = T * (1 - np.exp(-sigma_tau * delta))
    
    # Composite Color
    c_hat = np.sum(alpha[:, None] * c_static + alpha_tau[:, None] * c_tau, axis=0)
    
    # 6. Loss Calculation
    # Assume true_color is passed or retrieved
    # L = (C - C_hat)^2 / (2 * beta^2) + log(beta^2)/2 + lambda_u/2K * sum(sigma_tau)
    # Note: beta_sq is per point, but loss is usually per pixel/ray. 
    # The paper sums over K points for the regularization, but the likelihood term 
    # is typically aggregated. The formula in S1 sums the likelihood terms? 
    # Actually, S1 Eq 13 shows sum over k for the regularization, but the likelihood 
    # is for the final pixel color C(r) vs C_hat(r). 
    # However, the variance beta_i(r) is per ray (or per point?). 
    # S1 says "variance beta_i(r)^2". In implementation, it's often per point or averaged.
    # Let's follow the structure: The likelihood term compares the final rendered color.
    # But beta is predicted per point. 
    # Re-reading S1: "observed pixel intensities ... variance beta_i(r)^2".
    # And "beta_i(t) ... activation for beta_i(t)".
    # It seems beta is predicted per sample point, but the likelihood is on the pixel.
    # Standard practice in NeRF-W: The likelihood term uses the variance associated with the ray.
    # If beta is per point, we might average or use the last point. 
    # S1 Eq 13: L_i(r) = ... + log beta_i(r)^2 / 2. 
    # Let's assume beta_i(r) is the variance for the ray, approximated by the mean or last point.
    # For implementation clarity, we will use the mean beta_sq over K points for the likelihood term.
    
    beta_ray_sq = np.mean(beta_sq)
    
    # Likelihood term
    diff = true_color - c_hat
    likelihood = np.sum(diff**2) / (2 * beta_ray_sq) + np.log(beta_ray_sq) / 2
    
    # Regularization term
    reg = (lambda_u / (2 * K)) * np.sum(sigma_tau)
    
    total_loss = likelihood + reg
    
    return total_loss, c_hat, sigma, sigma_tau, beta_ray_sq

def train_nerf_w(images, poses, mlp_static, mlp_color, mlp_transient, mlp_uncertainty, 
                 l_a, l_tau, beta_min, lambda_u, batch_size, num_steps, K, delta):
    """
    Training loop for NeRF-W.
    """
    N = len(images)
    
    for step in range(num_steps):
        # Sample batch of rays
        batch_indices = np.random.choice(N, batch_size)
        total_loss = 0
        
        for i in batch_indices:
            # Sample rays for image i
            rays = sample_rays(images[i], poses[i]) # Returns x, d, true_color
            
            # Get embeddings
            emb_a = l_a[i]
            emb_tau = l_tau[i]
            
            # Render and compute loss
            loss, _, _, _, _ = render_ray(
                rays.x, rays.d, emb_a, emb_tau, beta_min, lambda_u,
                mlp_static, mlp_color, mlp_transient, mlp_uncertainty,
                K, rays.delta
            )
            
            # Add coarse loss if needed (omitted for brevity, see S1 Eq 14)
            # coarse_loss = 0.5 * ||C_i - C_hat_c_i||^2
            
            total_loss += loss
            
        # Update parameters (Adam)
        # gradients = compute_gradients(total_loss, [mlp_static, mlp_color, mlp_transient, mlp_uncertainty, l_a, l_tau])
        # optimizer.step(gradients)
        pass

    return mlp_static, mlp_color, mlp_transient, mlp_uncertainty, l_a, l_tau

def test_nerf_w(test_image, test_pose, mlp_static, mlp_color, beta_min):
    """
    Test-time rendering.
    """
    # Optimize appearance embedding for test image
    # Use left half of image for optimization to avoid leakage
    l_a_test = optimize_embedding(test_image, mlp_static, mlp_color, beta_min)
    
    # Render using only static components
    rays = sample_rays(test_image, test_pose)
    c_static = mlp_color(mlp_static(rays.x)[1], rays.d, l_a_test)
    sigma = mlp_static(rays.x)[0]
    
    # Alpha composite static only
    T = np.exp(-np.cumsum(sigma * rays.delta))
    alpha = T * (1 - np.exp(-sigma * rays.delta))
    c_hat = np.sum(alpha[:, None] * c_static, axis=0)
    
    return c_hat
```

## Subtopics

### Transient Filtering via Candidate Head
An architectural variant (e.g., in UP-NeRF) that uses a candidate head to predict transient properties (color, density, beta) alongside static properties, enabling robust pose optimization by filtering out transient occluders [S6, unofficial]. It predicts transient color $\mathbf{c}_i^{(\tau)}$, transient density $\sigma_i^{(\tau)}$, and variance $\beta_i$ using predicted candidate features $\mathbf{f}_i$ and candidate embeddings $\mathbf{l}_i^{(c)}$ [S6, unofficial]. Unlike NeRF-W, which does not isolate the transient network and relies on different architecture assumptions, this method isolates the transient network and uses deep features (DINO) for candidate filtering [S6, unofficial]. Merging transient and candidate heads degrades performance, causing failures in pose estimation and rendering quality [S6, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** Training requires optimizing per-image embeddings ($N$ vectors) alongside MLP weights, increasing memory usage compared to standard NeRF. Training takes 300,000 steps [S1].
*   **Measured Performance:** On 6 landmark scenes from the Phototourism dataset, NeRF-W improves over the previous state-of-the-art (NRW) by an average margin of 4.4dB in PSNR and up to 40% in MS-SSIM [S1].
*   **Strengths:** Handles variable illumination, auto-exposure, and transient occluders (pedestrians) in unconstrained photo collections [S1, S4]. Enables controllable appearance by interpolating in the latent space [S1].
*   **Limitations:** Cannot handle major appearance changes (e.g., seasonal changes) [S5, unofficial]. Requires known camera poses (COLMAP) [S1]. Test-time evaluation requires optimizing an appearance embedding for each test image, which can be slow [S1].
*   **When to use:** Unconstrained internet photo collections of static landmarks with varying lighting and transient objects [S1, S4].
*   **When not to use:** Scenes with significant structural changes over time or where camera poses are unknown/unreliable [S5, unofficial].

## Background
NeRF in the Wild (NeRF-W) was introduced by Martin-Brualla et al. in 2021 [S4]. It was developed to address the limitations of NeRF in controlled settings, specifically its inability to model real-world phenomena like variable illumination and transient occluders in unstructured image collections [S1]. It was tested on large-scale image collections of tourist landmarks such as the Brandenburg Gate and the Trevi Fountain [S4]. A subsequent development by Sun et al. (2022) used appearance embeddings but sought output in meshes rather than radiance fields [S4].

## Sources
[S1] NeRF in the Wild: Neural Radiance Fields for Unconstrained Photo Collections, https://doi.org/10.1109/cvpr46437.2021.00713, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S4] NEURAL RADIANCE FIELDS (NERF): REVIEW AND POTENTIAL APPLICATIONS TO DIGITAL CULTURAL HERITAGE, https://doi.org/10.5194/isprs-archives-xlviii-m-2-2023-453-2023, fetched 2026-10-29
[S5] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S6] UP-NeRF: Unconstrained Pose-Prior-Free Neural Radiance Fields, https://arxiv.org/abs/2311.03784v2, fetched 2026-10-26 (unofficial)
