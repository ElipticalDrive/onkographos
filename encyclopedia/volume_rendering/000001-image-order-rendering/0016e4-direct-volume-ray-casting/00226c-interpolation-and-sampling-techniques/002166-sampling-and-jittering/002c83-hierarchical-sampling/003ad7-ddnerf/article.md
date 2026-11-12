# DDNeRF

Also called: Depth Distribution Neural Radiance Fields

## Overview
DDNeRF is a neural volume rendering method that improves sampling efficiency in hierarchical sampling by learning a continuous density distribution within each ray interval, rather than relying on a discrete piecewise-constant probability density function (PDF) [S1]. It sets itself apart from [MipNeRF](4866-mipnerf.md) by using two separate networks for coarse and fine passes (like original NeRF) while modeling the coarse PDF as a mixture of truncated Gaussians, and from [Hierarchical Sampling](4865-hierarchical-sampling.md) by replacing the discrete weight normalization with a parametric Gaussian distribution estimation [S1].

## How it works
**Inputs and Outputs:**
- **Input:** Ray $\mathbf{r}$ defined by origin and direction; number of coarse samples $n$; number of fine samples $n_f$ (typically equal to $n$); scene geometry bounds (for unbounded scenes) [S1].
- **Output:** Predicted pixel color $\hat{C}(\mathbf{r})$ and depth [S1].

**Data Structures:**
- **Coarse Partitions:** $T^c = \{t^c_i\}_{i=0}^n$, depths bounding $n$ intervals [S1].
- **Fine Partitions:** $T^f = \{t^f_i\}_{i=0}^{n_f}$, depths bounding $n_f$ intervals [S1].
- **Coarse Network Output per Interval $i$:** RGB color $c_i$, opacity $\alpha_i$, relative mean $\mu^r_i$, relative standard deviation $\sigma^r_i$ [S1].
- **Fine Network Output per Sample:** RGB color $c_j$, opacity $\alpha_j$ [S1].

**Numbered Steps:**

1.  **Coarse Sampling:**
    - For bounded scenes, sample $n$ points uniformly with respect to depth along the ray, creating partitions $T^c$ [S1].
    - For unbounded scenes, use a hybrid strategy: half samples uniformly below radius 1 from the origin, half using log-sampling (DONeRF) outside the sphere [S1].
    - Divide the ray into $n$ intervals $[t^c_i, t^c_{i+1}]$ [S1].

2.  **Coarse Network Forward Pass:**
    - Encode each interval $[t^c_i, t^c_{i+1}]$ using Integrated Positional Encoding (IPE) [S1].
    - Pass encoded intervals through the coarse network [S1].
    - The network outputs $c_i$, $\alpha_i$, $\mu^r_i$, and $\sigma^r_i$ [S1].
    - Apply sigmoid to $\mu^r_i$ and $\sigma^r_i$ to constrain them to $[0, 1]$ [S1].

3.  **Opacity and Influence Calculation:**
    - Calculate opacity for each coarse sample $i$:
      $$ \alpha_i = 1 - \exp(-\sigma_{density,i} \delta_i) $$
      where $\sigma_{density,i}$ is the density output from the network and $\delta_i$ is the distance along the ray [S1].
    - Calculate influence weight $w_i$:
      $$ w_i = \alpha_i \cdot \prod_{j=0}^{i-1} (1 - \alpha_j) $$
    - Normalize weights to form the discrete coarse PDF $h^c$:
      $$ h^c[i] = \frac{w_i}{\sum_{j=0}^n w_j} $$

4.  **Smoothing Coarse PDF:**
    - Apply a 2-tap max filter followed by a 2-tap blur filter to $h^c$ [S1].
    - For small sample counts ($n \le 16$), use a 1D blur filter with weights $[0.1, 0.8, 0.1]$ [S1].

5.  **Distribution Estimation (Gaussian Parameters):**
    - Convert relative parameters to absolute location and scale along the ray:
      $$ \mu_i = t^c_i + \mu^r_i \cdot (t^c_{i+1} - t^c_i) $$
      $$ \sigma_i = \sigma^r_i \cdot (t^c_{i+1} - t^c_i) $$
    - Define the untruncated Gaussian PDF inside interval $i$:
      $$ f_i(t) = \mathcal{N}(t | \mu_i, \sigma_i) $$
    - Truncate the Gaussian to the interval boundaries $[t^c_i, t^c_{i+1}]$ and normalize:
      $$ f'_i(t) = \frac{1}{k_i} f_i(t), \quad k_i = F_i(t^c_{i+1}) - F_i(t^c_i) $$
      where $F_i$ is the CDF of $f_i$ [S1].
    - Construct the total density distribution $f_{dd}(t)$ as a mixture of truncated Gaussians weighted by $h^c$:
      $$ f_{dd}(t) = h^c[i] \cdot f'_i(t) \quad \text{for } t \in [t^c_i, t^c_{i+1}] $$
    - Apply internal interval smoothing by increasing the standard deviation with an uncertainty factor $u \ge 1$:
      $$ \hat{\sigma}_i = u \cdot \sigma_i $$
      Decrease $u$ toward 1 during training [S1].

6.  **Fine Sampling:**
    - Sample $n_f$ points $T^f$ from the continuous distribution $f_{dd}(t)$ using its CDF $F_{dd}$, focusing samples in areas of high density influence [S1].

7.  **Fine Network Forward Pass:**
    - Encode fine samples $T^f$ with IPE [S1].
    - Pass through the fine network to get RGB $c_j$ and opacity $\alpha_j$ [S1].
    - Calculate fine influence weights $w_j$ and render fine color $\hat{C}_f(\mathbf{r}) = \sum w_j c_j$ [S1].

8.  **Loss Calculation:**
    - **NeRF Loss:**
      $$ L_{nerf} = \sum_{r \in R} [\|C(\mathbf{r}) - \hat{C}_f(\mathbf{r})\|^2 + \|C(\mathbf{r}) - \hat{C}_c(\mathbf{r})\|^2] $$
      where $\hat{C}_c$ is the coarse color prediction [S1].
    - **Distribution Estimation (DE) Loss:**
      - Estimate the fine PDF $\hat{h}^f$ using the coarse distribution $f_{dd}$:
        $$ \hat{h}^f[i] = F_{dd}(t^f_{i+1}) - F_{dd}(t^f_i) $$
      - Calculate actual fine PDF $h^f$ from fine samples' influence weights (normalized) [G].
      - Compute KL divergence and regularization:
        $$ DE_{Loss} = KL(\hat{h}^f, h^f) + \frac{1}{n} \left( \lambda_\mu \sum_i (\mu^r_{raw,i})^2 + \lambda_\sigma \sum_i (\sigma^r_{raw,i})^2 \right) $$
        where $\mu^r_{raw}, \sigma^r_{raw}$ are pre-sigmoid outputs [S1].
    - **Total Loss:**
      $$ L = L_{nerf} + \lambda_{DE} \cdot DE_{Loss} $$
      with $\lambda_{DE} = 0.1$, $\lambda_\mu, \lambda_\sigma \approx 0.8/n$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def sigmoid(x):
    return 1.0 / (1.0 + math.exp(-x))

def gaussian_pdf(t, mu, sigma):
    return (1.0 / (sigma * math.sqrt(2 * math.pi))) * math.exp(-0.5 * ((t - mu) / sigma) ** 2)

def gaussian_cdf(t, mu, sigma):
    return 0.5 * (1.0 + math.erf((t - mu) / (sigma * math.sqrt(2))))

def integrated_positional_encoding(t_start, t_end):
    # Standard MipNeRF IPE implementation [G]
    # Encodes the cone volume between t_start and t_end.
    return [0.0] * 16  # Fixed-size feature vector [G]

def render_coarse(n_samples, min_depth, max_depth, is_unbounded):
    # 1. Coarse Sampling
    if is_unbounded:
        t_coarse = []
        half = n_samples // 2
        for i in range(half):
            t_coarse.append(1.0 * (i + 0.5) / half)
        for i in range(half):
            t_coarse.append(math.exp(math.log(1.0) + (i + 0.5) / half * (math.log(max_depth) - math.log(1.0))))
    else:
        t_coarse = [min_depth + i * (max_depth - min_depth) / n_samples for i in range(n_samples + 1)]

    # 2. Coarse Network Forward Pass
    coarse_outputs = []
    for i in range(n_samples):
        t_start = t_coarse[i]
        t_end = t_coarse[i+1]
        encoded = integrated_positional_encoding(t_start, t_end)
        # Simulate network output: [R, G, B, alpha, mu_raw, sigma_raw]
        # In practice, these come from the coarse network forward pass [S1]
        rgb = [0.5, 0.5, 0.5]
        alpha = 0.5
        mu_raw = 0.0
        sigma_raw = 0.0
        coarse_outputs.append({
            'rgb': rgb, 'alpha': alpha, 'mu_raw': mu_raw, 'sigma_raw': sigma_raw,
            't_start': t_start, 't_end': t_end
        })
    return coarse_outputs, t_coarse

def compute_distribution(coarse_outputs, n_samples, uncertainty_factor):
    # 3. Opacity and Influence Calculation
    weights = []
    transmittance = 1.0
    for out in coarse_outputs:
        w = out['alpha'] * transmittance
        weights.append(w)
        transmittance *= (1.0 - out['alpha'])
    sum_w = sum(weights)
    h_c = [w / sum_w for w in weights]

    # 4. Smoothing Coarse PDF
    if n_samples <= 16:
        smoothed = []
        for i in range(n_samples):
            val = 0.8 * h_c[i]
            if i > 0: val += 0.1 * h_c[i-1]
            if i < n_samples - 1: val += 0.1 * h_c[i+1]
            smoothed.append(val)
        h_c = smoothed
    else:
        # 2-tap max filter followed by 2-tap blur filter [S1]
        pass

    # 5. Distribution Estimation (Gaussian Parameters)
    gaussians = []
    for i in range(n_samples):
        out = coarse_outputs[i]
        mu_r = sigmoid(out['mu_raw'])
        sigma_r = sigmoid(out['sigma_raw'])
        mu = out['t_start'] + mu_r * (out['t_end'] - out['t_start'])
        sigma = sigma_r * (out['t_end'] - out['t_start'])
        sigma_smoothed = uncertainty_factor * sigma
        k = gaussian_cdf(out['t_end'], mu, sigma_smoothed) - gaussian_cdf(out['t_start'], mu, sigma_smoothed)
        gaussians.append({
            'mu': mu, 'sigma': sigma_smoothed, 'k': k, 'weight': h_c[i],
            't_start': out['t_start'], 't_end': out['t_end']
        })
    return h_c, gaussians

def sample_fine(gaussians, n_fine_samples):
    # 6. Fine Sampling
    # Sample from mixture of truncated Gaussians using inverse CDF [G]
    fine_samples = []
    for _ in range(n_fine_samples):
        u = random.uniform(0, 1)
        # Find which interval the CDF crosses u
        cdf_val = 0.0
        for g in gaussians:
            cdf_end = cdf_val + g['weight'] * (gaussian_cdf(g['t_end'], g['mu'], g['sigma']) - gaussian_cdf(g['t_start'], g['mu'], g['sigma'])) / g['k']
            if u <= cdf_end:
                # Sample within this interval using truncated Gaussian inverse CDF
                t_start = g['t_start']
                t_end = g['t_end']
                mu = g['mu']
                sigma = g['sigma']
                k = g['k']
                u_local = (u - cdf_val) * k / g['weight']
                t_sample = mu + sigma * math.sqrt(2) * math.erfinv(2 * u_local * k + math.erf((t_start - mu) / (sigma * math.sqrt(2))) - math.erf((t_start - mu) / (sigma * math.sqrt(2))))
                # Simplified inverse CDF for truncated Gaussian [G]
                fine_samples.append(t_sample)
                break
            cdf_val = cdf_end
    return fine_samples

def render_fine(fine_samples, fine_net):
    # 7. Fine Network Forward Pass
    fine_outputs = []
    for t in fine_samples:
        encoded = integrated_positional_encoding(t, t)
        # Simulate network output: [R, G, B, alpha]
        rgb = [0.5, 0.5, 0.5]
        alpha = 0.5
        fine_outputs.append({'rgb': rgb, 'alpha': alpha, 't': t})
        
    # Render
    color = [0.0, 0.0, 0.0]
    transmittance = 1.0
    for out in fine_outputs:
        w = out['alpha'] * transmittance
        for k in range(3):
            color[k] += w * out['rgb'][k]
        transmittance *= (1.0 - out['alpha'])
    return color, transmittance

def compute_loss(coarse_outputs, fine_color, gt_color, n_samples):
    # 8. Loss Calculation
    # NeRF Loss
    coarse_color = [0.0, 0.0, 0.0]
    transmittance = 1.0
    for out in coarse_outputs:
        w = out['alpha'] * transmittance
        for k in range(3):
            coarse_color[k] += w * out['rgb'][k]
        transmittance *= (1.0 - out['alpha'])
        
    l_nerf = sum((c - g)**2 for c, g in zip(coarse_color, gt_color)) + sum((f - g)**2 for f, g in zip(fine_color, gt_color))
    
    # DE Loss
    # 1. Estimate h^f from fine samples
    fine_alphas = [out['alpha'] for out in fine_outputs]
    fine_ws = []
    transmittance = 1.0
    for alpha in fine_alphas:
        w = alpha * transmittance
        fine_ws.append(w)
        transmittance *= (1.0 - alpha)
    sum_fine_w = sum(fine_ws)
    h_f = [w / sum_fine_w for w in fine_ws]
    
    # 2. Estimate h^f_hat from coarse distribution
    fine_partitions = [0] + [s['t'] for s in fine_outputs] + [max_depth]
    h_f_hat = []
    for i in range(len(fine_partitions)-1):
        t_start = fine_partitions[i]
        t_end = fine_partitions[i+1]
        prob = 0.0
        for g in gaussians:
            prob += g['weight'] * (gaussian_cdf(min(t_end, g['t_end']), g['mu'], g['sigma']) - gaussian_cdf(max(t_start, g['t_start']), g['mu'], g['sigma'])) / g['k']
        h_f_hat.append(prob)
        
    sum_h_f_hat = sum(h_f_hat)
    h_f_hat = [x / sum_h_f_hat for x in h_f_hat]
    
    # KL Divergence
    kl = 0.0
    for hf, hf_hat in zip(h_f, h_f_hat):
        if hf > 0 and hf_hat > 0:
            kl += hf * math.log(hf / hf_hat)
            
    # Regularization
    reg_mu = sum(out['mu_raw']**2 for out in coarse_outputs)
    reg_sigma = sum(out['sigma_raw']**2 for out in coarse_outputs)
    
    lambda_mu = 0.8 / n_samples
    lambda_sigma = 0.8 / n_samples
    
    de_loss = kl + (1.0/n_samples) * (lambda_mu * reg_mu + lambda_sigma * reg_sigma)
    
    lambda_de = 0.1
    total_loss = l_nerf + lambda_de * de_loss
    
    return total_loss
```

**Parameters:**
- $n$: Number of coarse samples (e.g., 4, 8, 16, 32) [S1].
- $\lambda_{DE}$: DE Loss coefficient, set to 0.1 [S1].
- $\lambda_\mu, \lambda_\sigma$: Regularization coefficients, set to approximately $0.8/n$ (range 0.01–0.1) [S1].
- $u$: Uncertainty factor for internal smoothing, $u \ge 1$, decreased to 1 during training [S1].

**Edge Cases and Pitfalls:**
- **Small Sample Counts:** With few samples (e.g., $n=4$), the discrete PDF resolution is limited ($1/n^2$ of scene depth). DDNeRF mitigates this by using continuous Gaussian distributions within intervals [S1].
- **Unbounded Scenes:** Standard uniform sampling fails for deep/unbounded scenes. Use hybrid sampling (uniform inside radius 1, log-sampling outside) for the coarse pass [S1].
- **Sigmoid Vanishing Gradients:** The DE loss regularization prevents $\mu^r$ and $\sigma^r$ from collapsing to 0 or 1, keeping them in the effective range of the sigmoid [S1].
- **Interval Independence:** The method assumes each Gaussian affects only its own interval. This simplifies calculation but may ignore cross-interval density correlations [S1].

## Operating characteristics
- **Time Complexity:** Similar to hierarchical NeRF, but with additional overhead for computing Gaussian CDFs and KL divergence. The fine sampling is more efficient as it concentrates samples in high-density regions, allowing fewer total samples for similar quality [S1].
- **Memory Complexity:** Requires storing Gaussian parameters ($\mu, \sigma$) for each coarse interval in addition to RGB and opacity. Minimal increase over standard NeRF [S1].
- **Performance:** Outperforms MipNeRF and standard NeRF in PSNR, SSIM, and LPIPS across various sample budgets (4, 8, 16, 32) on LLFF and Synthetic 360 datasets [S1]. For example, on LLFF fern with 4 samples, DDNeRF achieves PSNR 20.81 vs MipNeRF 20.20 [S1].
- **Strengths:** Better depth resolution with few samples; more accurate density representation; improved sampling efficiency [S1].
- **Limitations:** Requires two separate networks (coarse and fine) unlike MipNeRF's single network; increased training complexity due to DE loss [S1].

## Background
DDNeRF was introduced in the paper "DDNeRF: Depth Distribution Neural Radiance Fields" (2023) [S1]. It builds on MipNeRF's use of integrated positional encoding and cone-based ray representation, but reintroduces the two-network hierarchical sampling structure of original NeRF to allow for the prediction of distribution parameters in the coarse pass [S1]. It addresses the limitations of discrete PDFs in hierarchical sampling, which suffer from low depth resolution and wasted samples in empty regions [S1].

## Sources
[S1] DDNeRF: Depth Distribution Neural Radiance Fields, https://doi.org/10.1109/wacv56688.2023.00082, fetched 2026-10-27
