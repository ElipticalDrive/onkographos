# Candidate-Driven Projection and Gradient Storage

Also called: BoundedGS Projection, Candidate Indexing

## Overview
Candidate-Driven Projection and Gradient Storage is a memory optimization technique for differentiable 3D Gaussian Splatting that restricts expensive projection and gradient accumulation to a subset of Gaussians likely to affect the current view [S1, unofficial]. It reduces memory consumption proportional to the number of view candidates $n_v$ rather than the total Gaussian count, addressing the peak memory usage caused by storing full-model activation states and dense gradients [S1, unofficial]. It builds on [2D Gaussian Projection with Covariance Stabilization](1198-2d-gaussian-projection-with-covariance-stabilization.md) by using conservative screening to avoid computing full rotated covariances and spherical harmonic colors for irrelevant primitives during the initial pass [S1, unofficial].

## How it works
The method operates in three phases: candidate construction, candidate-indexed projection, and compact gradient storage [S1, unofficial]. It maintains a single global model but avoids creating a full activated tensor or parameter copy for every Gaussian [S1, unofficial].

### Inputs and Outputs
*   **Inputs:** Global Gaussian parameters (position, scale, rotation, opacity, spherical harmonic coefficients), camera parameters, rasterizer configuration [S1, unofficial].
*   **Outputs:** Projected 2D Gaussians for rendering, compact gradient buffers indexed by global Gaussian ID, Adam optimizer state updates [S1, unofficial].

### Data Structures
*   **Global Parameters:** Full layout for all Gaussians. Includes Adam moments (first and second order) for all parameters [S1, unofficial].
*   **Candidate Set ($C_v$):** A list of global Gaussian IDs for view $v$. Size $n_v = |C_v|$ [S1, unofficial].
*   **Compact Gradient Buffer:** A buffer of size $n_v \times d$, where $d$ is the parameter dimension per Gaussian. Stores gradients for position, spherical harmonics, scale, and rotation [S1, unofficial].
*   **ID Map:** A mapping from compact buffer indices to global Gaussian IDs [S1, unofficial].

### Steps

1.  **Candidate Construction (Screening):**
    *   For each Gaussian, compute a conservative screen coverage estimate using position, opacity, maximum scale, and camera parameters [S1, unofficial].
    *   Use the maximum scale to bound the covariance independently of rotation. This avoids computing full rotated covariances and evaluating spherical harmonics during screening [S1, unofficial].
    *   Include every Gaussian that may affect the current view under the rasterizer. The screening must retain uncertain cases and permit false positives to ensure no relevant Gaussian is missed [S1, unofficial].
    *   The set of candidate IDs is $C_v$ [S1, unofficial].

2.  **Candidate-Indexed Projection (Forward Pass):**
    *   For view $v$, let $C_v$ be the candidate IDs and $n_v = |C_v|$ [S1, unofficial].
    *   Read original parameter rows by index from the global model [S1, unofficial].
    *   Internally activate parameters for the candidates only [S1, unofficial].
    *   Compute geometry (2D covariance, mean) and evaluate spherical harmonics for the candidates [S1, unofficial].
    *   Rasterize the candidates.
    *   Let $\theta_i$ be Gaussian $i$'s parameters, $f_v(\theta_i)$ its projected attributes, and $L_{img}^v$ the regional image objective. The parameter gradient for candidate $i \in C_v$ is computed as [S1, unofficial]:
        $$
        g_i = \frac{\partial q_i}{\partial \theta_i} = \left( \frac{\partial f_v(\theta_i)}{\partial \theta_i} \right)^\top \boxplus_{\ell \in R_v} w_\ell \frac{\partial L_\ell^{img}}{\partial q_i}
        $$
        where $q_i = \sum_{\ell \in R_v} w_\ell L_\ell^{img}$ is the regional image objective contribution, $R_v$ is the set of rasterized pixels affected by candidate $i$, and $w_\ell$ are weights. Unaffected regions contribute zero [S1, unofficial].

3.  **Backward Pass (Gradient Accumulation):**
    *   Retain 3DGS’s direct gradient accumulation (DirectGrad) [S1, unofficial].
    *   Write gradients to the compact buffer indexed by global Gaussian ID [S1, unofficial].
    *   Compute activation derivatives on this path without a separate full-model activation graph [S1, unofficial].
    *   Final gradients for position, spherical harmonics, scale, and rotation retain candidate rows linked to global parameters by the ID map [S1, unofficial].
    *   Parameters and Adam state keep full layouts; outside-candidate image gradients are zero [S1, unofficial].
    *   Global opacity regularization requires a dense opacity gradient, which is added to the image contributions [S1, unofficial].

4.  **Optimizer Update:**
    *   The optimizer reads compact gradients and IDs directly [S1, unofficial].
    *   For rows outside the candidate set, Adam still updates the moments and parameters using a zero current image gradient [S1, unofficial].
    *   Gradients and maps are released after the update [S1, unofficial].

5.  **Visibility and Densification:**
    *   Visibility is combined by union across candidates [S1, unofficial].
    *   Screen gradients by accumulation before computing per-Gaussian densification statistics [S1, unofficial].
    *   Perform scheduled view-level structure updates [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def candidate_driven_projection_and_gradient_storage(
    global_params, camera, rasterizer, optimizer
):
    # 1. Candidate Construction
    candidates = []
    for i in range(global_params.num_gaussians):
        pos = global_params.position[i]
        max_scale = max(global_params.scale[i])
        opacity = global_params.opacity[i]
        if is_conservatively_visible(pos, max_scale, opacity, camera):
            candidates.append(i)
    n_v = len(candidates)
    
    # 2. Candidate-Indexed Projection
    candidate_params = [global_params[i] for i in candidates]
    projected_attrs = []
    for i in range(n_v):
        projected_attrs.append(activate_and_project(candidate_params[i], camera))
    pixel_grads, pixel_weights = rasterizer.rasterize(projected_attrs, camera)
    
    # 3. Backward Pass
    candidate_gradients = []
    for i in range(n_v):
        jacobian = compute_projection_jacobian(candidate_params[i], camera)
        pixel_grad_sum = accumulate_pixel_gradients(i, pixel_grads, pixel_weights)
        candidate_gradients.append(jacobian.T @ pixel_grad_sum)
        
    # 4. Compact Gradient Storage
    id_map = candidates
    full_gradients = zeros_like(global_params)
    for k in range(n_v):
        full_gradients[id_map[k]] = candidate_gradients[k]
    full_gradients.opacity += compute_opacity_regularization_gradient(global_params.opacity)
    
    # 5. Optimizer Update
    optimizer.step(global_params, full_gradients)
    
    # 6. Cleanup
    del projected_attrs, candidate_gradients, pixel_grads
    return global_params

# Helper definitions
def is_conservatively_visible(pos, max_scale, opacity, camera):
    """Conservative screen coverage estimate using position, max scale, opacity, camera."""
    return True  # Implementation depends on rasterizer bounds

def activate_and_project(params, camera):
    """Internally activate parameters, compute 2D geometry, evaluate spherical harmonics."""
    cov_2d = compute_2d_covariance(params, camera)
    mean_2d = project_mean(params, camera)
    sh_colors = eval_spherical_harmonics(params, camera)
    return {"cov": cov_2d, "mean": mean_2d, "sh": sh_colors}

def compute_projection_jacobian(params, camera):
    """Jacobian of projection and SH evaluation w.r.t parameters."""
    return jacobian_matrix  # Standard analytical Jacobian

def accumulate_pixel_gradients(idx, pixel_grads, pixel_weights):
    """Sum weighted pixel gradients for Gaussian idx."""
    return sum(pixel_grads[i] * pixel_weights[i] for i in rasterized_pixels(idx))

def zeros_like(params):
    """Initialize buffer with zeros matching params layout."""
    return {k: np.zeros_like(v) for k, v in params.items()}

def compute_opacity_regularization_gradient(opacity):
    """Compute dense gradient for global opacity regularization."""
    return regularization_term_grad  # Standard regularization gradient
```

### Parameters and Edge Cases
*   **Conservative Screening:** Must permit false positives. Missing a Gaussian that affects the view leads to incorrect gradients [S1, unofficial].
*   **Maximum Scale:** Used for screening to bound covariance independently of rotation. This is a simplification that avoids full rotation matrix multiplication during screening [S1, unofficial].
*   **Adam State:** For rows outside the candidate set, Adam still updates the moments and parameters using a zero current image gradient [S1, unofficial].
*   **Opacity Regularization:** Requires a dense gradient, added to the image contributions. This is a small memory overhead compared to the full parameter gradients [S1, unofficial].

## Subtopics

### Compact Gradients
Stores candidate gradients compactly to reduce memory usage during parameter updates [S1, unofficial]. It stores candidate gradients compactly after candidate-indexed projection [S1, unofficial]. This variant reaches a 1.66 GiB working set and 10.46 seconds time, cutting the working set by 83.3% and time by 14.2% relative to the Original method [S1, unofficial]. All modified variants’ terminal renderings have SSIM near one relative to Original [S1, unofficial].

## Operating characteristics
*   **Memory Complexity:** Scales with $n_v$ (number of candidates) for detailed projection and gradient buffers, rather than total Gaussian count $N$. The global model and Adam state keep full layouts [S1, unofficial].
*   **Time Complexity:** Screening scales with $N$. Projection and backward pass scale with $n_v$. Overall speedup depends on the ratio $n_v/N$ and the cost of screening vs. full projection [S1, unofficial].
*   **Measured Performance:** Relative to the Original method, Compact Gradients reduces the working set by 83.3% (from ~10 GiB to 1.66 GiB) and reduces time by 14.2% (from ~12.2s to 10.46s) [S1, unofficial].
*   **Strengths:** Significantly reduces peak memory usage, enabling high-resolution training. Maintains visual quality (SSIM near 1.0) [S1, unofficial].
*   **Limitations:** Screening must be conservative, potentially including unnecessary Gaussians (false positives) [S1, unofficial].
*   **When to Use:** High-resolution training where memory is the bottleneck. When $n_v \ll N$.
*   **When Not to Use:** When $n_v \approx N$, the overhead of candidate management may not be worth it [S1, unofficial].

## Background
Candidate-Driven Projection and Gradient Storage is part of the BoundedGS method, introduced in "BoundedGS: Memory-Efficient High-Resolution Training for 3D Gaussian Splatting" [S1, unofficial]. It addresses the memory peaks caused by storing full-model attributes, projection, and gradients during training [S1, unofficial]. The method uses view candidates throughout detailed projection, backward accumulation, and gradient storage [S1, unofficial].

## Sources
[S1] BoundedGS: Memory-Efficient High-Resolution Training for 3D Gaussian Splatting, https://arxiv.org/abs/2610.34233v1, fetched 2026-10-26 (unofficial)
