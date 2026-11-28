# Neural Volume Rendering Training (MSE)

Also called: Hierarchical Geometric Guidance (HGG), Joint Training of Fourier Feature Volume and MLP, Training and Encoding Pipeline

## Overview
Neural Volume Rendering Training (MSE) using Hierarchical Geometric Guidance (HGG) optimizes neural radiance fields by minimizing photometric error while using sparse depth priors solely to schedule sampling region expansion, avoiding direct depth supervision bias [S1]. It differs from standard NeRF training by replacing global initialization with a local-to-global sampling schedule that gradually extends near and far planes from the depth prior to full scene bounds [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** Sparse depth priors $t_{depth}$ from Structure-from-Motion (SfM) [S1]; ground-truth images $C_{gt}$; camera poses; total training iterations $N_{total}$; HGG iteration count $N_{hgg}$; minimum adjusting rate $\epsilon_{hgg}$ [S1].
- **Outputs:** Optimized neural radiance field parameters $\Theta$ (density $\sigma$ and color $\mathbf{c}$) [S1].

**Data Structures:**
- **Scene Representation:** A multi-layer perceptron (MLP) $f_\Theta$ mapping 3D coordinates $\mathbf{x}$ and viewing direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$ [S1].
- **Sampling Points:** A set of $N$ points $t_i$ along each ray $\mathbf{r}$, defined by near plane $t_n$ and far plane $t_f$ [S1].
- **Iteration State:** Current iteration $i$ [S1].

**Numbered Steps:**
1. **Initialization:** For each ray, initialize the near plane $t_n$ and far plane $t_f$ close to the sparse depth prior $t_{depth}$ to define a local sampling region [S1].
2. **Region Expansion Schedule:** For each iteration $i$, compute the region adjustment rate $\gamma(i)$ and update $t_n(i)$ and $t_f(i)$ using the formulas below [S1].
3. **Volume Rendering:** Sample $N$ points uniformly or stratified within the current bounds $[t_n(i), t_f(i)]$ [G]. Query $f_\Theta$ for $\sigma_i$ and $\mathbf{c}_i$ [G]. Compute the rendered color $\hat{C}(\mathbf{r})$ using the standard emission-absorption model [44-radiance-field-optimization.md]:
   $$
   \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
   $$
   where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ and $\delta_i = t_{i+1} - t_i$ [S1].
4. **Loss Computation:** Compute the photometric loss between the rendered color $\hat{C}(\mathbf{r})$ and the ground-truth color $C_{gt}(\mathbf{r})$ [S1]. (Note: The specific HGG implementation minimizes Huber loss for outlier robustness rather than strict MSE [S1]).
5. **Optimization:** Update $\Theta$ using an Adam optimizer with exponential learning rate decay [S1].

**Formulas:**
- **Region Adjustment Rate:**
  $$
  \gamma(i) = \frac{1 - \cos\left( (\min(\max(i/N_{hgg}, \epsilon_{hgg}), 1)) \pi \right)}{2}
  $$
  where $i$ is the current iteration, $N_{hgg}$ is the number of iterations until full bounds are reached, and $\epsilon_{hgg}$ is the minimum adjusting rate [S1].
- **Near/Far Plane Update:**
  $$
  t_n(i) = t_{depth} + (t_n - t_{depth}) \gamma(i)
  $$
  $$
  t_f(i) = t_{depth} + (t_f - t_{depth}) \gamma(i)
  $$
  where $t_n, t_f$ are the initial near/far planes, and $t_{depth}$ is the depth prior [S1].

**Parameters:**
- $N_{hgg}$: Number of iterations for region expansion. Typical value: 10% of total training iterations [S1].
- $\epsilon_{hgg}$: Minimum adjusting rate. Typical value: 0.2 [S1].
- Learning Rate: $1 \times 10^{-3}$, exponentially decaying to $1 \times 10^{-5}$ [S1].
- Samples: 64 volume points for coarse model, 128 for fine model (if using Hierarchical Photometric Guidance variant) [S1].

**Edge Cases and Pitfalls:**
- **Depth Bias:** Direct supervision of sparse depth prior introduces bias $\Delta P$ due to keypoint mismatching ($\Delta d$) and small translation between frames, causing geometric misalignment [S1]. HGG avoids this by using depth only for sampling region initialization [S1].
- **Precision:** The $\min(\max(i/N_{hgg}, \epsilon_{hgg}), 1)$ term in $\gamma(i)$ inherently clamps the adjustment rate to $[\epsilon_{hgg}, 1]$, preventing invalid bounds or division errors [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_gamma(i, N_hgg, epsilon_hgg):
    """Compute the region adjustment rate gamma(i)."""
    ratio = i / N_hgg
    clamped_ratio = max(epsilon_hgg, min(ratio, 1.0))
    return (1.0 - math.cos(clamped_ratio * math.pi)) / 2.0

def update_bounds(t_n_init, t_f_init, t_depth, i, N_hgg, epsilon_hgg):
    """Update near and far planes based on iteration i."""
    gamma = compute_gamma(i, N_hgg, epsilon_hgg)
    t_n = t_depth + (t_n_init - t_depth) * gamma
    t_f = t_depth + (t_f_init - t_depth) * gamma
    return t_n, t_f

def render_ray(t_n, t_f, num_samples, query_fn):
    """Render a single ray using the emission-absorption model."""
    samples = [t_n + (t_f - t_n) * (j + 0.5) / num_samples for j in range(num_samples)]
    sigmas = [query_fn(x)[0] for x in samples]
    colors = [query_fn(x)[1] for x in samples]
    deltas = [samples[j+1] - samples[j] for j in range(num_samples - 1)]
    if num_samples > 1:
        deltas.append(samples[-1] - samples[-2])
    else:
        deltas.append(1e-6)
    transmittances = [math.exp(-sum(sigmas[k] * deltas[k] for k in range(j))) for j in range(num_samples)]
    weights = [(1.0 - math.exp(-sigmas[j] * deltas[j])) * transmittances[j] for j in range(num_samples)]
    return sum(w * c for w, c in zip(weights, colors))

def compute_loss(rendered, ground_truth):
    """Compute photometric loss (Huber or MSE)."""
    return sum(abs(r - g) ** 1.5 for r, g in zip(rendered, ground_truth)) / len(rendered)

def train_hgg(depth_prior, t_n_init, t_f_init, total_iterations, epsilon_hgg=0.2, samples_per_ray=64):
    """Train a neural volume rendering model using HGG."""
    N_hgg = int(0.1 * total_iterations)
    model = initialize_model() # [G] Standard MLP parameter initialization
    optimizer = initialize_optimizer(model) # [G] Standard Adam optimizer setup
    for i in range(total_iterations):
        t_n, t_f = update_bounds(t_n_init, t_f_init, depth_prior, i, N_hgg, epsilon_hgg)
        rendered_color = render_ray(t_n, t_f, samples_per_ray, model)
        loss = compute_loss(rendered_color, ground_truth_color) # [G] Ground truth for current batch
        optimizer.step(loss) # [G] Standard gradient update
    return model
```

## Subtopics

### Hierarchical Photometric Guidance
Trains a coarse model and a fine model to improve novel view synthesis performance by combining their predicted colors in the loss function [S1]. It differs from standard HGG by using a two-stage hierarchical loss rather than just geometric guidance [S1]. Choose it when higher fidelity is needed at the cost of increased memory and computation [S1].
- **Definition:** Combines coarse and fine model predictions in the loss [S1].
- **Steps:** 1. Query coarse model $f_\Theta$ with stratified volume samples $t_c$ to get $\hat{C}_{crecon}$ [S1]. 2. Query fine model with union of coarse and fine samples $t_c \cup t_f$ (produced by inverse transform sampling) to get $\hat{C}_{frecon}$ [S1]. 3. Compute MSE loss against ground truth $C_{gt}$ for both predictions [S1].
- **Formula:** $L_{hpg} = \frac{1}{2} \sum_{r \in R} \left( C_{gt}(r) - \hat{C}_{crecon}(r; t_c) \right)^2 + \left( C_{gt}(r) - \hat{C}_{frecon}(r; t_c \cup t_f) \right)^2$, where $R$ is the set of all rays, $t_c$ are coarse stratified samples, and $t_f$ are fine samples produced by inverse transform sampling [S1].

## Operating characteristics
- **Time Complexity:** Linear in the number of samples and rays, similar to standard NeRF, with minor overhead for computing the expansion schedule and potentially querying two models if using Hierarchical Photometric Guidance [S1].
- **Memory Complexity:** $O(n)$ for storing intermediate rendering states, similar to standard NeRF [44-radiance-field-optimization.md].
- **Measured Performance:** Evaluated on the DTU dataset for 3, 6, and 9 views, showing more fine details and distinct edges in depth maps compared to SOTA baselines [S1].
- **Strengths:** Avoids geometric bias from sparse depth supervision; improves novel view synthesis in sparse view scenarios [S1].
- **Limitations:** Requires sparse depth priors from SfM; performance may degrade if depth priors are inaccurate [S1].
- **When to Use:** Sparse view reconstruction where direct depth supervision is unreliable [S1].

## Background
Originated in the HG3-NeRF paper (2024) by authors of the arXiv preprint "HG3-NeRF: Hierarchical Geometric, Semantic, and Photometric Guided Neural Radiance Fields for Sparse View Inputs" [S1]. It addresses the problem of geometric misalignment in sparse view NeRF reconstruction caused by bias in sparse depth supervision [S1].

## Sources
[S1] HG3-NeRF: Hierarchical Geometric, Semantic, and Photometric Guided Neural Radiance Fields for Sparse View Inputs, https://doi.org/10.48550/arXiv.2401.11711, fetched 2026-10-29
