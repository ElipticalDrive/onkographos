# Optimization Configuration (Adam + Exponential Decay + Selective L2)

Also called: Adam + Exponential Decay, Optimization Procedure, Optimizer Settings, Parameter update rule, Training optimizer setup

## Overview
This configuration defines the standard training loop for neural radiance fields and implicit surface representations, combining the Adam optimizer with a staircase exponential learning rate schedule and selective L2 weight decay [S2][S4, unofficial]. It addresses the instability of training coordinate-based neural fields by providing robust convergence parameters, particularly for noisy real-world scan data or complex geometric priors [S2]. It sets itself apart from generic optimization loops by explicitly integrating domain-specific augmentations and specific regularization techniques into the core training loop [S2][S4, unofficial].

## How it works
The method initializes a scene representation and iteratively updates parameters using a composite optimizer setup [S2][S4, unofficial]. The core mechanism involves sampling rays, computing gradients via automatic differentiation, and applying parameter updates with specific learning rate schedules and regularization masks [S2][S4, unofficial].

### Inputs and Outputs
**Inputs:**
*   Scene representation parameters (e.g., MLP weights for density and color, appearance embeddings) [S2][S4, unofficial].
*   Camera poses and intrinsics [S2, S3].
*   Target images (color and/or depth) [S2, S3].
*   Hyperparameters: Initial learning rate ($lr$), decay rate, transition steps, and L2 regularization coefficient [S2][S4, unofficial].

**Outputs:**
*   Optimized scene representation parameters [S2][S4, unofficial].

### Data Structures
**Optimizer Configuration:**
*   `param_labels`: A dictionary mapping parameter names to optimizer targets.
    *   Layout: `{"nerf": "network", "bg": "network", "appearance_embeddings": "ae"}` [S4, unofficial].
*   `mask`: A boolean mask dictating where L2 decay is applied.
    *   Layout: `{"nerf": {"density_mlp": True, "rgb_mlp": True, "position_encoder": False}, "bg": True, "appearance_embeddings": False}` [S4, unofficial].
*   `optimizer_network`: Adam optimizer instance for network weights [S4, unofficial].
    *   Parameters: `learning_rate=lr_sch`, `b1=0.9`, `b2=0.99`, `eps=1e-15`, `eps_root=1e-15` [S4, unofficial].
*   `optimizer_ae`: Adam optimizer instance for appearance embeddings [S4, unofficial].
    *   Parameters: `learning_rate=1e-4`, `b1=0.9`, `b2=0.99`, `eps=1e-8`, `eps_root=0` [S4, unofficial].
*   `combined_optimizer`: Multi-transform optimizer combining the above [S4, unofficial].
    *   Mapping: `transforms={"network": optimizer_network, "ae": optimizer_ae}` [S4, unofficial].

**Training State:**
*   `lr_sch`: Learning rate schedule instance [S4, unofficial].
    *   Parameters: `init_value=lr`, `transition_steps=10_000`, `decay_rate=1/3`, `staircase=True`, `transition_begin=10_000`, `end_value=lr/100` [S4, unofficial].
*   `step`: Current optimization step counter [G].

### Numbered Steps
1.  **Initialization:**
    *   Translate and scale the scene to lie within a $[-1, 1]^3$ bounding cube [S2].
    *   Initialize the learning rate schedule with `staircase=True` [S4, unofficial].
    *   Initialize the combined optimizer with the specified parameters [S4, unofficial].
    *   Set the initial step counter to 0 [G].

2.  **Ray Sampling:**
    *   Randomly select $|P_b| = 1024$ rays per iteration [S2].
    *   For each ray, compute sample points:
        *   Fixed samples: $S'_f = 16$ [S2].
        *   Variable samples: $S'_c$ is chosen such that there is on average one sample for every 1.5 cm of ray length [S2].
    *   Apply positional encoding: 8 bands for point coordinates, 4 bands for view direction vector [S2].

3.  **Forward Pass:**
    *   Compute the radiance field values (density $\sigma$, color $\mathbf{c}$) for the sampled points using the current network parameters [S2].
    *   Render the image using the standard emission-absorption model (see [Parent](5622-optimization-loops-and-training-procedures.md)) [S2].
    *   If applicable, apply distortion correction using a 6-layer MLP (width 128) on 2D camera pixel space [S2].

4.  **Loss Computation:**
    *   Compute the loss between rendered and target images.
    *   Loss weights: $\lambda_1 = 0.1$, $\lambda_2 = 10$, $\lambda_3 = 6 \times 10^3$ [S2].

5.  **Gradient Computation:**
    *   Compute gradients with respect to the scene parameters using the specified loss weights and positional encodings [S2].
    *   Use reverse-mode automatic differentiation [44-radiance-field-optimization.md].

6.  **Parameter Update:**
    *   Update the learning rate based on the current step:
        *   If `staircase=True`, the learning rate is held constant for `transition_begin` steps, then decays by `decay_rate` every `transition_steps` steps [S4, unofficial].
        *   Note: `staircase=True` combined with `transition_begin=10_000` delays the first learning rate drop to step 20,000 [S4, unofficial].
    *   Apply the Adam update rule to the parameters, respecting the `mask` for L2 regularization:
        *   L2 decay coefficient: $1 \times 10^{-6}$ [S4, unofficial].
        *   L2 decay is applied to `density_mlp`, `rgb_mlp`, and `bg` parameters, but **not** to `position_encoder` or `appearance_embeddings` [S4, unofficial].
    *   Increment the step counter.

7.  **Iteration:**
    *   Repeat steps 2-6 until convergence or maximum iterations reached.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_learning_rate(step, init_value, transition_begin, transition_steps, decay_rate, end_value, staircase):
    if staircase:
        if step < transition_begin:
            return init_value
        decay_steps = (step - transition_begin) // transition_steps
        lr = init_value * (decay_rate ** decay_steps)
        return max(lr, end_value)
    return init_value * (decay_rate ** (step / transition_steps))

def apply_l2_decay(params, mask, l2_coeff):
    l2_loss = 0.0
    for key, value in params.items():
        if key in mask:
            if isinstance(mask[key], bool):
                if mask[key]:
                    l2_loss += np.sum(value ** 2)
            elif isinstance(mask[key], dict):
                for sub_key, sub_mask in mask[key].items():
                    if sub_mask:
                        l2_loss += np.sum(params[key][sub_key] ** 2)
    return l2_loss * l2_coeff

def adam_step(params, grads, m, v, step, lr, b1, b2, eps, eps_root):
    t = step + 1
    for key in params:
        m[key] = b1 * m[key] + (1 - b1) * grads[key]
        v[key] = b2 * v[key] + (1 - b2) * grads[key] ** 2
        m_hat = m[key] / (1 - b1 ** t)
        v_hat = v[key] / (1 - b2 ** t)
        params[key] = params[key] - lr * m_hat / (np.sqrt(v_hat) + eps)
    return params, m, v

def train_step(params, images, cameras, lr_sch, optimizer_cfg, mask, l2_coeff):
    step = optimizer_cfg.get('step', 0)
    num_rays = 1024
    indices = np.random.choice(images.shape[0], num_rays, replace=False)
    rays = [generate_ray(cameras[idx]) for idx in indices]
    rendered_images = render_rays(params, rays)
    loss = np.mean((rendered_images - images) ** 2)
    grads = compute_gradients(loss, params)
    current_lr = compute_learning_rate(step, **lr_sch)
    params, optimizer_cfg['m'], optimizer_cfg['v'] = adam_step(
        params, grads, optimizer_cfg['m'], optimizer_cfg['v'],
        step, current_lr, optimizer_cfg['b1'], optimizer_cfg['b2'],
        optimizer_cfg['eps'], optimizer_cfg['eps_root']
    )
    optimizer_cfg['step'] = step + 1
    return params, loss

def generate_ray(camera):
    return {'origin': camera['origin'], 'direction': camera['direction'], 'length': camera['far'] - camera['near']}

def render_rays(params, rays):
    points = [sample_points(ray, 16, ray['length'] / 1.5) for ray in rays]
    directions = [ray['direction'] for ray in rays]
    encoded_points = positional_encoding(points, 8)
    encoded_directions = positional_encoding(directions, 4)
    density, color = mlp_forward(params['network'], encoded_points, encoded_directions)
    return volume_rendering(density, color, rays)

def positional_encoding(x, num_bands):
    frequencies = 2 ** np.arange(num_bands) * np.pi
    return np.concatenate([np.sin(f * x) for f in frequencies], axis=-1)

def mlp_forward(params, points, directions):
    h = np.maximum(0, params['w1'] @ points + params['b1'])
    return params['w2'] @ h + params['b2'], params['w3'] @ h + params['b3']

def volume_rendering(density, color, rays):
    t_vals = np.linspace(rays[0]['near'], rays[0]['far'], 16 + int(rays[0]['length']/1.5))
    deltas = t_vals[1:] - t_vals[:-1]
    alpha = 1 - np.exp(-density * deltas)
    T = np.cumprod(1 - alpha + 1e-10, axis=-1)
    weights = alpha * T
    return np.sum(weights * color, axis=-2)
```

### Parameters
*   **Optimizer:** Adam [S2, S3][S4, unofficial].
*   **Initial Learning Rate ($lr$):** Passed to the optimizer factory [S4, unofficial]. Specific value: $5 \times 10^{-4}$ [S2].
*   **Learning Rate Schedule:** Exponential decay [S2][S4, unofficial].
    *   `transition_steps`: 10,000 [S4, unofficial].
    *   `decay_rate`: 1/3 [S4, unofficial].
    *   `staircase`: True [S4, unofficial].
    *   `transition_begin`: 10,000 [S4, unofficial].
    *   `end_value`: $lr / 100$ [S4, unofficial].
*   **Adam Parameters:**
    *   `b1`: 0.9 [S4, unofficial].
    *   `b2`: 0.99 [S4, unofficial].
    *   `eps`: $1 \times 10^{-15}$ [S4, unofficial].
    *   `eps_root`: $1 \times 10^{-15}$ [S4, unofficial].
*   **Appearance Embedding Optimizer:**
    *   `learning_rate`: $1 \times 10^{-4}$ [S4, unofficial].
    *   `eps`: $1 \times 10^{-8}$ [S4, unofficial].
    *   `eps_root`: 0 [S4, unofficial].
*   **L2 Regularization:**
    *   Coefficient: $1 \times 10^{-6}$ [S4, unofficial].
    *   Applied to: `density_mlp`, `rgb_mlp`, `bg` [S4, unofficial].
    *   Not applied to: `position_encoder`, `appearance_embeddings` [S4, unofficial].
*   **Ray Sampling:**
    *   Number of rays per iteration: 1024 [S2].
    *   Fixed samples: 16 [S2].
    *   Variable samples: 1 sample per 1.5 cm of ray length [S2].
*   **Positional Encoding:**
    *   Point coordinates: 8 bands [S2].
    *   View direction: 4 bands [S2].
*   **Loss Weights:**
    *   $\lambda_1 = 0.1$ [S2].
    *   $\lambda_2 = 10$ [S2].
    *   $\lambda_3 = 6 \times 10^3$ [S2].
*   **Distortion Correction:**
    *   6-layer MLP, width 128 [S2].

### Edge Cases and Pitfalls
*   **Learning Rate Schedule Delay:** `staircase=True` combined with `transition_begin=10_000` delays the first learning rate drop to step 20,000, as the initial LR is held for the first 10k steps before the staircase division triggers [S4, unofficial].
*   **L2 Decay on Encodings:** Applying L2 decay to position encoders or appearance embeddings is explicitly disabled to prevent divergence or suboptimal convergence during long training runs [S4, unofficial].
*   **Numerical Stability:** Small epsilon values ($1 \times 10^{-15}$) are used in Adam to prevent division by zero in the adaptive learning rate calculation [S4, unofficial].

## Subtopics

### [Gradient Descent Solver](3280-gradient-descent-solver.md)
The Gradient Descent Solver is a matrix-free optimization method used to update voxel grid parameters in differentiable volume rendering, such as in DiffVox. It supports both constant update rates and the Adam optimizer, providing stable behavior for ill-conditioned systems. It distinguishes itself from second-order methods like [Newton's Method](6442-newton-s-method.md) by avoiding Hessian computation, and from direct least-squares solvers like [CGLS Solver](3285-cgls-solver-conjugate-gradient-least-squares.md) by minimizing the residual via iterative gradient steps rather than solving normal equations directly.

### [BFGS Solver with Moreau Envelope](3781-bfgs-solver-with-moreau-envelope.md)
This method solves inverse problems, specifically tomographic reconstruction, by minimizing a composite objective function consisting of a data fidelity term and a regularizer. It sets itself apart by using the BFGS quasi-Newton algorithm to handle the non-smooth L1 regularization via its Moreau envelope, providing a differentiable approximation that allows second-order optimization steps. This contrasts with first-order Gradient Descent solvers and avoids the explicit projection steps required by constrained material solvers.

### [Training Protocol](4528-training-protocol.md)
The Training Protocol defines the standard optimization procedure for neural networks in differentiable volume rendering, encompassing optimizer settings, learning rate scheduling, and data augmentation strategies. It addresses the instability of training coordinate-based neural fields by providing a robust configuration for convergence, particularly when dealing with noisy real-world scan data or complex geometric priors. It sets itself apart from generic optimization loops by explicitly integrating domain-specific augmentations (such as Gaussian noise for point clouds) and specific regularization techniques (like CVT for discretization stability) into the core training loop.

### [Euclidean Projection for Material Constraints](7899-euclidean-projection-for-material-constraints.md)
This method enforces physical validity on material parameters during differentiable rendering optimization by projecting updated values back into their feasible domain after each optimizer step. It specifically addresses the problem of unconstrained gradient updates (e.g., from Adam) driving color or BSDF parameters outside physically plausible ranges, which causes numerical instability or non-physical reflectance. It differs from its siblings by being a post-update constraint enforcement mechanism rather than a solver or training schedule; it builds on [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md) by applying a hard boundary check after the parameter update.

### [Inverse Rendering Optimization Setup](7165-inverse-rendering-optimization-setup.md)
*   **Optimizer:** Adam.
*   **Loss Function:** L2 loss.
*   **Number of Views:** 8.
*   **Batch Size:** 2.

## Operating characteristics
*   **Time Complexity:** Convergence time is between 9 and 13 hours on a single NVIDIA RTX 3090 GPU, depending on scene size [S2].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Robust convergence for coordinate-based neural fields; stable behavior for ill-conditioned systems; explicit handling of domain-specific augmentations and regularization [S2][S4, unofficial].
*   **Limitations:** High memory usage for reverse-mode AD if not optimized; sensitive to learning rate schedule parameters [S4, unofficial].
*   **When to Use:** Standard training for neural radiance fields and implicit surface representations; noisy real-world scan data; complex geometric priors [S2][S4, unofficial].
*   **When Not to Use:** When second-order methods are required for faster convergence; when constrained optimization is needed [S4, unofficial].

## Background
*   **Origin:** The Adam optimizer was introduced by Kingma and Ba in 2014 [S2][S4, unofficial]. The specific configuration of Adam with exponential decay and selective L2 regularization is used in various neural radiance field and implicit surface reconstruction methods [S2][S4, unofficial].
*   **Usage:** Used in neural RGB-D surface reconstruction [S2], JAX-based neural graphics primitives [S4, unofficial], and path-space differentiable rendering of implicit surfaces [S3].

## Sources
[S1] Differentiable surface splatting for point-based geometry processing, https://doi.org/10.1145/3355089.3356513, fetched 2026-10-26
[S2] Neural RGB-D Surface Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00619, fetched 2026-10-26
[S3] Learning Neural Duplex Radiance Fields for Real-Time View Synthesis, https://doi.org/10.1109/cvpr52729.2023.00803, fetched 2026-10-26
[S4] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S5] CATRF: Codec-Adaptive TriPlane Radiance Fields for Volumetric Content Delivery, https://doi.org/10.1109/cvprf71909.2026.00048, fetched 2026-10-30
[S6] SelfNeRF: Fast Training NeRF for Human from Monocular Self-rotating Video, https://doi.org/10.48550/arxiv.2210.01651, fetched 2026-10-30 (unofficial)
[S7] carlinds/splatad, https://github.com/carlinds/splatad, fetched 2026-10-27 (unofficial)
