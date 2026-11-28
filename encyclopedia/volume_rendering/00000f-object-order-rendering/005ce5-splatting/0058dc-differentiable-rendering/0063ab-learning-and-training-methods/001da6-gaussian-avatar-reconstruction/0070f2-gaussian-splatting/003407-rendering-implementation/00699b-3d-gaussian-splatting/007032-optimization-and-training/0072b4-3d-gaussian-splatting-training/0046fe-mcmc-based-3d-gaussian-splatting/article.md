# MCMC-based 3D Gaussian Splatting

Also called: Hybrid MCMC re-formulation, Ours

## Overview
This method reformulates 3D Gaussian Splatting training as a Markov Chain Monte Carlo (MCMC) process to enable stochastic exploration without heuristic-based densification and pruning [S1]. It replaces fixed cloning and splitting thresholds with noise injection and gradual relocation, making the optimization robust to random initialization and preventing stray Gaussians from diverging into unobserved regions [S1].

## How it works
The method treats the set of 3D Gaussians as samples from a posterior distribution, optimized via MCMC steps [S1]. It modifies the population management and parameter updates of the standard training loop [S1].

### Inputs and Outputs
*   **Inputs:** Camera poses, images, initial Gaussian parameters (position $\mathbf{p}$, scale $\mathbf{s}$, rotation $\mathbf{q}$, opacity $\alpha$, spherical harmonics coefficients $\mathbf{c}$), and hyperparameters [S1].
*   **Outputs:** Optimized Gaussian parameters representing the scene [S1].

### Data Structures
The Gaussian population is managed as a list of active Gaussians. Each Gaussian $i$ stores:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position.
*   $\mathbf{s}_i \in \mathbb{R}^3$: Scale.
*   $\mathbf{q}_i \in \mathbb{R}^4$: Rotation (quaternion).
*   $\alpha_i \in \mathbb{R}$: Opacity.
*   $\mathbf{c}_i$: SH coefficients.

### Initialization
1.  **Random Initialization:** Uniformly sample 100,000 Gaussians within a bounding box defined as $3\times$ the extent of the camera bounding box [S1].
2.  **SfM Initialization:** Alternatively, initialize from Structure-from-Motion (SfM) point clouds [S1].
3.  **Warmup:** Perform 500 iterations of standard gradient descent without relocation or population increase [S1].

### Noise Scheduling
Noise is injected into parameters to facilitate exploration [S1]. The noise magnitude decays exponentially with a decay rate of $\gamma = 0.9995$ per iteration [S1].
*   **Location:** Noise is applied according to Equation 8 of the source [S1].
*   **Scale and Rotation:** Add noise $\mathcal{N}(0, I)$ [S1].
*   **Opacity:** Add noise $\mathcal{N}(0, 0.1 \times I)$ [S1].

### Relocation/Expansion
During training, the number of live Gaussians is gradually increased by 5% per step until the maximum desired number is reached [S1]. This replaces the heuristic cloning and splitting of deterministic baselines [S1].

### Regularization
Two regularizers are essential to prevent Gaussians from diverging into unobserved regions (stray Gaussians) [S1]:
1.  **Noise Regularizer:** Weight $\lambda_{noise} = 5 \times 10^5$ [S1].
2.  **Opacity Regularizer:** Weight $\lambda_{o} = 0.01$ (or $0.001$ for Deep Blending dataset) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def train_mcmc_3dgs(cameras, images, max_gaussians, max_steps, expansion_interval=1):
    # 1. Initialization
    # Random initialization: 100,000 Gaussians uniformly sampled within 3x camera extent [S1]
    gaussians = initialize_random_gaussians(count=100000, bbox=cameras.bounding_box, multiplier=3.0)

    # Hyperparameters [S1]
    lr = 1.6e-4
    lr_min = 1.6e-6
    decay = 0.9995
    lambda_noise = 5e5
    lambda_opacity = 0.01
    noise_scale = 1.0

    for step in range(max_steps):
        # Learning rate decay [S1]
        lr = max(lr * decay, lr_min)
        # Noise scale decay [S1]
        noise_scale *= decay

        # Warmup phase: 500 iterations without relocation or population increase [S1]
        if step < 500:
            loss = compute_render_loss(gaussians, cameras, images)
            apply_gradient_update(gaussians, loss, lr)
            continue

        # 2. Noise Injection for Exploration [S1]
        # Location noise (Eq. 8 from source)
        noise_p = apply_location_noise(gaussians, noise_scale)
        # Scale and rotation noise: N(0, I)
        noise_s = [random.gauss(0, noise_scale) for _ in range(len(gaussians) * 3)]
        noise_q = [random.gauss(0, noise_scale) for _ in range(len(gaussians) * 4)]
        # Opacity noise: N(0, 0.1 * I)
        noise_a = [random.gauss(0, noise_scale * 0.1) for _ in range(len(gaussians))]

        # 3. Loss and Regularization [S1]
        loss = compute_render_loss(gaussians, cameras, images)
        # Noise regularizer prevents stray Gaussians
        noise_reg = lambda_noise * (sum(n**2 for n in noise_p) + sum(n**2 for n in noise_s) + sum(n**2 for n in noise_q))
        # Opacity regularizer
        opacity_reg = lambda_opacity * sum(g['alpha'] for g in gaussians)
        total_loss = loss + noise_reg + opacity_reg

        apply_gradient_update(gaussians, total_loss, lr)

        # 4. Relocation/Expansion [S1]
        if step % expansion_interval == 0:
            current_count = len(gaussians)
            new_count = int(current_count * 1.05)
            if new_count <= max_gaussians:
                gaussians = relocate_and_add_gaussians(gaussians, new_count)
            else:
                gaussians = gaussians[:max_gaussians]

    return gaussians

def initialize_random_gaussians(count, bbox, multiplier):
    # Uniformly sample positions within 3x camera bounding box extent
    extent = [bbox.max - bbox.min for _ in range(3)]
    return [{'p': [random.uniform(bbox.min[i], bbox.min[i] + extent[i]*multiplier) for i in range(3)],
             's': [1.0, 1.0, 1.0], 'q': [1.0, 0.0, 0.0, 0.0], 'alpha': 0.1, 'c': []} for _ in range(count)]

def apply_location_noise(gaussians, scale):
    # [G] Exact formula from Eq. 8 is not provided in the sources.
    # Standard implementation applies isotropic Gaussian noise scaled by the decay factor.
    return [random.gauss(0, scale) for _ in range(len(gaussians) * 3)]

def compute_render_loss(gaussians, cameras, images):
    # Standard differentiable rasterization loss (e.g., L1 or MSE between rendered and target images)
    rendered = differentiable_rasterize(gaussians, cameras)
    return sum((r - t)**2 for r, t in zip(rendered, images))

def apply_gradient_update(gaussians, loss, lr):
    # Standard gradient descent update for all Gaussian parameters
    for g in gaussians:
        g['p'] = [p - lr * compute_grad(loss, g, 'p') for p in g['p']]
        g['s'] = [s - lr * compute_grad(loss, g, 's') for s in g['s']]
        g['q'] = [q - lr * compute_grad(loss, g, 'q') for q in g['q']]
        g['alpha'] = g['alpha'] - lr * compute_grad(loss, g, 'alpha')

def relocate_and_add_gaussians(gaussians, target_count):
    # [G] Specific relocation logic not detailed in sources.
    # Clones or spawns new Gaussians based on visibility/gradient metrics to reach target_count.
    return gaussians[:target_count]
```

### Edge Cases and Pitfalls
*   **Opacity Term Removal:** If the opacity term is removed from the noise scheduler, $\lambda_{noise}$ must be set to $0.05$; otherwise, the model fails to train (PSNR < 7) [S1].
*   **Incompatibility with Classical Heuristics:** The regularizers used in this method are harmful if applied to classical 3DGS, as they are incompatible with its heuristic-based densification [S1].

## Operating characteristics
*   **Performance:** On MipNeRF 360 with random initialization, the method achieves PSNR 29.72, SSIM 0.89, LPIPS 0.19, compared to 3DGS PSNR 27.89, SSIM 0.84, LPIPS 0.26 [S1].
*   **Robustness:** The method is robust to initialization, performing similarly with random and SfM initialization, whereas classical 3DGS performance drops significantly with random initialization [S1].
*   **Noise Scheduling Impact:** On Tank & Temples, an exponential noise scheduler yields PSNR 24.21, while a linear scheduler yields 17.64 and the scheduler from [30] yields 22.46 [S1].
*   **Strengths:** Eliminates sensitivity to initialization; prevents stray Gaussians in empty space; robust reconstruction quality [S1].
*   **Limitations:** Requires careful tuning of noise regularizers; potentially slower convergence due to stochastic exploration compared to deterministic heuristics [S1].

## Background
The method was introduced in "3D Gaussian Splatting as Markov Chain Monte Carlo" [S1]. It addresses the limitations of the original 3DGS training procedure, which relies on heuristic-based densification and pruning that are sensitive to initialization and can lead to suboptimal local minima [S1]. By framing the optimization as an MCMC process, it allows for a more thorough exploration of the parameter space [S1].

## Sources
[S1] 3D Gaussian Splatting as Markov Chain Monte Carlo, https://ubc-vision.github.io/3dgs-mcmc/, fetched 2026-10-26
