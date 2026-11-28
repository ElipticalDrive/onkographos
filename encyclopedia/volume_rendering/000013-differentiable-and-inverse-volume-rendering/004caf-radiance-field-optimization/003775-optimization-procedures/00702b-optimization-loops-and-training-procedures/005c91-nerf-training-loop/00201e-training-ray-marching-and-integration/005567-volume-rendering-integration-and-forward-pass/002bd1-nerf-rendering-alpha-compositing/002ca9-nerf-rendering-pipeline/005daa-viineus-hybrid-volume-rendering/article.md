# ViiNeuS Hybrid Volume Rendering

Also called: Hybrid alpha composition, ViiNeuS Volume Rendering

## Overview
ViiNeuS Hybrid Volume Rendering is a progressive training strategy for implicit neural surface reconstruction that transitions from pure volumetric density representation to signed distance field (SDF) representation during the optimization loop [S1]. It solves the initialization instability and slow convergence of pure SDF methods in complex scenes by initially leveraging a robust volumetric estimator, then gradually introducing SDF-based alpha compositing [S1]. Unlike standard NeRF rendering which uses only density-based alpha, or pure NeuS which uses only SDF-based alpha, this method dynamically assigns each sample's alpha value to either $\alpha^{iv}$ or $\alpha^{if}$ based on training stage and sample density, ensuring stable surface extraction [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Ray $r$ defined by origin and direction [S1].
*   Sample points $x_i$ along the ray [S1].
*   Network outputs for each sample $x_i$:
    *   Density $\sigma_i$ (from MLP $F^h_\Theta$) [S1].
    *   Signed Distance Field value $f(x_i)$ (from MLP $F^h_\Theta$) [S1].
    *   Color $c_i$ (from MLP $F^c_\Theta$, using direction, latent vector, and normal) [S1].
*   Current training step $t$ and total training steps $T_{total}$ [S1].

**Outputs:**
*   Rendered pixel color $\hat{C}(r)$ [S1].

### Data Structures and Layout
The method relies on two Multi-Layer Perceptrons (MLPs) [S1]:
1.  **Geometry MLP ($F^h_\Theta$):**
    *   Input: Positional embedding $emb_x(x)$ [S1].
    *   Output: Tuple $(\sigma, f(x), h)$, where $\sigma$ is density, $f(x)$ is SDF value, and $h$ is a latent vector [S1].
    *   Architecture: 2 layers, 64 hidden units [S1].
2.  **Color MLP ($F^c_\Theta$):**
    *   Input: Directional embedding $emb_d(d)$, latent vector $h$, and normal vector $-\vec{n}$ [S1].
    *   Output: Color $c$ [S1].
    *   Architecture: 2 layers, 64 hidden units [S1].
    *   Normal $\vec{n}$ is computed as the gradient of the SDF: $\vec{n} = \nabla f(x)$ [G].

### Parameters
*   **Volumetric Stage Duration:** First 100 training steps [S1].
*   **Hybrid Stage End:** 35% of total training time ($0.35 \times T_{total}$) [S1].
*   **Sharpness Parameter ($s$):** Controls the standard deviation of the SDF density distribution $\phi_s$ (std = $1/s$). Regularized during training with weight decay from $10^{-3}$ to $10^{-5}$ [S1].
*   **Sample Spacing ($\delta_i$):** Distance between adjacent samples along the ray [S1].

### Formulas

**1. Volumetric Alpha ($\alpha^{iv}$):**
Computed from predicted density $\sigma_i$ and sample spacing $\delta_i$ [S1]:
$$ \alpha^{iv} = 1 - \exp(-\sigma_i \delta_i) $$

**2. SDF-based Alpha ($\alpha^{if}$):**
Computed using the sigmoid function $\Phi_s$ and its derivative $\phi_s$. Let $\Phi_s(z) = (1 + e^{-sz})^{-1}$ and $\phi_s(z) = \frac{se^{-sz}}{(1 + e^{-sz})^2}$ [S1].
For sample $i$ at position $p_i$ and next sample $i+1$ at $p_{i+1}$, the alpha is [S1]:
$$ \alpha^{if} = \max\left( \frac{\Phi_s(f(p_i)) - \Phi_s(f(p_{i+1}))}{\Phi_s(f(p_i))}, 0 \right) $$
*Note: In implementation, $f(p_i)$ is the SDF value at sample $i$ [G].*

**3. Final Rendered Color:**
Alpha compositing along the ray [S1]:
$$ \hat{C}(r) = \sum_{i=1}^{N} w_i c_i $$
where:
*   $w_i = T_i \alpha_i$ is the weight of sample $i$ [S1].
*   $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ is the accumulated transmittance [S1].
*   $\alpha_i$ is the blending factor, determined by the training stage (see Steps) [S1].

### Numbered Steps

1.  **Determine Training Stage:**
    *   If current step $t < 100$: **Volumetric Stage** [S1].
    *   If $100 \le t < 0.35 \times T_{total}$: **Hybrid Stage** [S1].
    *   If $t \ge 0.35 \times T_{total}$: **Surface Stage** [S1].

2.  **Compute Network Outputs:**
    *   For each sample $x_i$ on the ray:
        *   Compute $emb_x(x_i)$ using hash encoding [S1].
        *   Query $F^h_\Theta(emb_x(x_i))$ to get $\sigma_i, f(x_i), h_i$ [S1].
        *   Compute normal $\vec{n}_i = \nabla f(x_i)$ [G].
        *   **SDF-Gradient Normalization:** During the Hybrid Stage, normalize $\vec{n}_i$ to prevent large gradient values from compensating for alpha distribution differences [S1]. Let $\vec{n}'_i = \frac{\vec{n}_i}{\|\vec{n}_i\|}$ [G].
        *   Compute $emb_d(d)$ using spherical harmonics [S1].
        *   Query $F^c_\Theta(emb_d(d), h_i, -\vec{n}'_i)$ to get color $c_i$ [S1].

3.  **Assign Alpha Value ($\alpha_i$):**
    *   **Volumetric Stage:** Set $\alpha_i = \alpha^{iv}_i$ for all samples [S1].
    *   **Hybrid Stage:**
        *   Identify samples with the highest predicted density values $\sigma_i$ [S1].
        *   Assign $\alpha_i = \alpha^{if}_i$ to these high-density samples to facilitate SDF initialization near the surface [S1].
        *   Assign $\alpha_i = \alpha^{iv}_i$ to the remaining samples [S1].
        *   *Note: The exact probability or threshold for "highest density" is not explicitly defined in the provided text, but the strategy is to prioritize high-density samples for SDF training* [S1].
    *   **Surface Stage:** Set $\alpha_i = \alpha^{if}_i$ for all samples [S1].

4.  **Compute Transmittance and Weights:**
    *   Initialize $T_1 = 1$.
    *   For $i = 1$ to $N$:
        *   $w_i = T_i \alpha_i$ [S1].
        *   $T_{i+1} = T_i (1 - \alpha_i)$ [S1].

5.  **Accumulate Color:**
    *   $\hat{C}(r) = \sum_{i=1}^{N} w_i c_i$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def viineus_volume_rendering(ray_origin, ray_direction, num_samples, spacing, 
                             training_step, total_steps, geometry_mlp, color_mlp, s_param):
    """
    Renders a single ray using ViiNeuS hybrid volume rendering.
    
    Args:
        ray_origin: 3D vector ray origin.
        ray_direction: 3D vector ray direction.
        num_samples: Number of samples N along the ray.
        spacing: Distance delta_i between adjacent samples.
        training_step: Current integer training step.
        total_steps: Total number of training steps.
        geometry_mlp: Function F^h_Theta taking positional embedding.
        color_mlp: Function F^c_Theta taking direction embedding, latent, normal.
        s_param: Sharpness parameter s for sigmoid.
        
    Returns:
        rendered_color: 3D vector color for the ray.
    """
    N = num_samples
    rendered_color = [0.0, 0.0, 0.0]
    accumulated_transmittance = 1.0
    
    # Determine stages
    is_volumetric_stage = training_step < 100
    is_hybrid_stage = 100 <= training_step < 0.35 * total_steps
    is_surface_stage = training_step >= 0.35 * total_steps
    
    # Pre-compute geometry outputs for all samples
    sample_data = []
    for i in range(N):
        # Compute sample position
        x_i = ray_origin + (i * spacing) * ray_direction
        
        # Get positional embedding (hash encoding) [G]
        emb_x = hash_encoding(x_i)
        
        # Query Geometry MLP
        sigma, f_x, h = geometry_mlp(emb_x)
        
        # Compute normal from SDF gradient [G]
        grad_f = numerical_gradient(lambda p: geometry_mlp(hash_encoding(p))[1], x_i)
        normal = normalize_vector(grad_f)
        
        sample_data.append({
            'sigma': sigma,
            'f_x': f_x,
            'h': h,
            'normal': normal,
            'x': x_i
        })
        
    # Determine which samples get SDF alpha in hybrid stage
    alpha_assignments = []
    
    if is_hybrid_stage:
        # Assign alpha_if to samples with highest predicted density [S1]
        sorted_indices = sorted(range(N), key=lambda k: sample_data[k]['sigma'], reverse=True)
        # Heuristic: top 50% by density get alpha_if if not specified [G]
        threshold_index = N // 2
        high_density_indices = set(sorted_indices[:threshold_index])
        
    for i in range(N):
        data = sample_data[i]
        sigma_i = data['sigma']
        f_x_i = data['f_x']
        normal_i = data['normal']
        
        # Compute delta_i
        delta_i = spacing if i < N - 1 else 0.0
            
        # Compute Volumetric Alpha
        alpha_iv = 1.0 - exp(-sigma_i * delta_i)
        
        # Compute SDF Alpha
        if i < N - 1:
            f_x_next = sample_data[i+1]['f_x']
        else:
            f_x_next = f_x_i 
            delta_i = 0.0 # Effectively alpha_if will be 0 for last sample
            
        # Sigmoid function Phi_s
        phi_s_f_x_i = sigmoid(f_x_i, s_param)
        phi_s_f_x_next = sigmoid(f_x_next, s_param)
        
        # Alpha_if formula [S1]
        if phi_s_f_x_i > 1e-6:
            alpha_if = max((phi_s_f_x_i - phi_s_f_x_next) / phi_s_f_x_i, 0.0)
        else:
            alpha_if = 0.0
            
        # Assign Alpha based on Stage
        if is_volumetric_stage:
            alpha_i = alpha_iv
        elif is_surface_stage:
            alpha_i = alpha_if
        else: # Hybrid Stage
            if i in high_density_indices:
                alpha_i = alpha_if
            else:
                alpha_i = alpha_iv
                
        # Compute Weight
        weight_i = accumulated_transmittance * alpha_i
        rendered_color += weight_i * get_color_from_data(data, ray_direction)
        
        # Update Transmittance
        accumulated_transmittance *= (1.0 - alpha_i)
        
        # Early exit if transmittance is negligible
        if accumulated_transmittance < 1e-4:
            break
            
    return rendered_color

def sigmoid(z, s):
    return 1.0 / (1.0 + exp(-s * z))

def normalize_vector(v):
    norm = sqrt(sum(x*x for x in v))
    if norm == 0: return v
    return [x/norm for x in v]

def get_color_from_data(data, direction):
    # Query Color MLP [S1]
    emb_d = spherical_harmonics(direction)
    h = data['h']
    normal = data['normal']
    # Note: Source says input is -n [S1]
    neg_normal = [-n for n in normal]
    return color_mlp(emb_d, h, neg_normal)

def hash_encoding(x):
    # Placeholder for hash encoding implementation [G]
    return x

def spherical_harmonics(d):
    # Placeholder for spherical harmonics implementation [G]
    return d

def numerical_gradient(func, x, eps=1e-4):
    # Numerical gradient computation [G]
    grad = [0.0, 0.0, 0.0]
    for i in range(3):
        x_plus = x.copy()
        x_plus[i] += eps
        x_minus = x.copy()
        x_minus[i] -= eps
        grad[i] = (func(x_plus) - func(x_minus)) / (2 * eps)
    return grad
```

### Edge Cases and Pitfalls
*   **Large SDF Gradients:** During the hybrid stage, the SDF gradient can become large, causing $\alpha^{if}$ to align incorrectly with $\alpha^{iv}$ [S1]. This is addressed by normalizing the SDF gradient before computing $\alpha^{if}$ [S1].
*   **SDF Initialization:** Pure SDF initialization assumes full scene observability and a centered region of interest, which may not hold for outdoor/autonomous driving scenes [S1]. The hybrid strategy mitigates this by starting with volumetric density [S1].
*   **Parameter $s$:** The sharpness parameter $s$ controls the standard deviation of the density distribution. It is regularized during training to stabilize convergence [S1].

## Operating characteristics

*   **Time Complexity:** Linear with respect to the number of samples $N$ per ray, $O(N)$, as each sample requires a forward pass through two MLPs and alpha computation.
*   **Memory Complexity:** Linear with respect to $N$ to store intermediate values (sigma, f(x), h, normal) for all samples along the ray.
*   **Performance:** The method is evaluated on autonomous driving datasets (KITTI-360, nuScenes, Waymo, Pandaset) [S1]. It uses a single high-tier GPU with Adam optimizer [S1].
*   **Strengths:**
    *   Faster convergence of the density estimator by starting with pure volumetric representation [S1].
    *   Robust SDF initialization in complex scenes with wide distance ranges and complex geometries by sampling near the surface [S1].
    *   Stable transition from volumetric to surface representation [S1].
*   **Limitations:**
    *   Requires careful tuning of the hybrid stage transition (35% of training time) and the probability-based sample association strategy [S1].
    *   Relies on the accuracy of the SDF gradient for normal computation and alpha calculation [S1].
*   **When to Use:** Suitable for implicit neural surface reconstruction in urban/outdoor scenes with limited image overlap where pure SDF initialization is unstable [S1].
*   **When Not to Use:** Not necessary for scenes with full observability and simple geometry where pure SDF (NeuS) or pure Volumetric (NeRF) methods converge reliably.

## Background

*   **Origin:** ViiNeuS was introduced in the paper "ViiNeuS: Volumetric Initialization for Implicit Neural Surface reconstruction of urban scenes with limited image overlap" [S1].
*   **History:** The method builds upon NeuS, which introduced SDF-based volume rendering for neural rendering [S1]. ViiNeuS extends this by introducing a hybrid training strategy to address initialization issues in challenging environments like autonomous driving scenarios [S1].
*   **Usage:** It is used for 3D scene reconstruction from images, specifically targeting outdoor/urban datasets such as KITTI-360, nuScenes, Waymo Open Dataset, and Pandaset [S1]. The final mesh is generated using Marching Cubes [S1].

## Sources

[S1] ViiNeuS: Volumetric Initialization for Implicit Neural Surface reconstruction of urban scenes with limited image overlap, https://hal.science/hal-05104965v1, fetched 2026-10-26
