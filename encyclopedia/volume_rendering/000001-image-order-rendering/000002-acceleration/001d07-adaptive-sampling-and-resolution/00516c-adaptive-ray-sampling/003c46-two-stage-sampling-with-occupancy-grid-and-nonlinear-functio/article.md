# Two-Stage Sampling with Occupancy Grid and Nonlinear Function

Also called: Hybrid Sampling Strategy, Two-Phase Sampling

## Overview
This method addresses the inefficiency of uniform sampling in neural implicit surface reconstruction by combining spatial skipping with biased sampling [S1]. It solves the problem of wasted computation in empty regions and the random scatter of samples near surfaces in traditional importance sampling [S1]. Unlike [Adaptive Sparse Ray Sampling](8313-adaptive-sparse-ray-sampling.md) which focuses on sparsity, or [Coarse-to-Fine Hierarchical Sampling](14488-coarse-to-fine-hierarchical-sampling.md) which uses uniform coarse stages, this method uses an Occupancy Grid to skip empty space and a nonlinear piecewise exponential function to bias samples toward the surface within occupied cells [S1].

## How it works

### Inputs and Outputs
*   **Inputs:** Camera parameters (position, direction), Occupancy Grid (binary array), Neural Networks (Geometry $F_\theta$, Appearance $F_\beta$), Hash Encoding $\gamma$.
*   **Outputs:** Rendered color $\hat{C}(x)$, accumulated opacity $\hat{O}_k$, SDF values, gradients.

### Data Structures

**Occupancy Grid**
*   **Layout:** Binary array.
*   **Fields:** Each cell indicates whether an object surface intersects that spatial region (1 if occupied, 0 if empty) [S1].
*   **Usage:** Used in the coarse stage to skip sampling in empty cells [S1].

**Hash Encoding**
*   **Levels:** 16 levels, ranging from $2^5$ to $2^{16}$ [S1].
*   **Feature Size:** 2 per level [S1].
*   **Progressive Training:** Levels are updated every 1000 iterations to activate finer features [S1].

**Network Architectures**
*   **Geometry Network ($F_\theta$):**
    *   Input: Position $x$ encoded by $\gamma$.
    *   Structure: 1-layer MLP, hidden size 64 [S1].
    *   Output: SDF value $f_{sdf}(x)$ and geometry feature $enc$ [S1].
*   **Appearance Network ($F_\beta$):**
    *   Input: Position $x$, normal $n$, geometry feature $enc$, direction $d$ (encoded via Spherical Harmonics) [S1].
    *   Structure: 2-layer MLP, hidden size 64 [S1].
    *   Output: Color $c$ [S1].

### Numbered Steps

1.  **Coarse Stage (Occupancy Grid Skip):**
    *   Cast a ray.
    *   Query the Occupancy Grid along the ray.
    *   Skip sampling in intervals where the grid indicates empty space (0) [S1].
    *   Identify occupied intervals where the grid indicates presence (1) [S1].

2.  **Fine Stage (Nonlinear Function Guidance):**
    *   For each occupied interval $[t_i, t_{i+1}]$, calculate weights $w(t)$ from the coarse stage (or initial uniform samples) [S1].
    *   Define boundary weights: $a = w(t_i)$ and $b = w(t_{i+1})$, where $a \ge 0$ and $b \le 1$ [S1].
    *   Compute the nonlinear weight function $\hat{w}(s)$ for a sample position $s$ within the interval:
        $$ \hat{w}(s) = a \left( \frac{b}{a} \right)^{\frac{s - t_i}{t_{i+1} - t_i}} $$
        *(Note: The source formula (7) is written as $a (b/a)^s$ but context implies normalization over the interval $[t_i, t_{i+1}]$. The integral constraint (8) confirms the exponential interpolation form)* [S1].
    *   The integral of the weight over the interval must satisfy:
        $$ \int_{t_i}^{t_{i+1}} w(t) dt = \int_{0}^{1} \hat{w}(s) ds = \ln b - \ln a $$
        *(Note: This equation in the source [S1] appears to describe the normalization or relationship for the exponential distribution parameters, specifically relating to the cumulative distribution function derivation for inverse transform sampling)* [S1].
    *   Use $\hat{w}(s)$ to guide sampling points closer to the surface. If $a \ll b$, weights increase sharply toward $t_{i+1}$; if $a \gg b$, weights decrease sharply [S1].
    *   This biases samples toward the surface where weight is high, avoiding the random scatter of inverse transform sampling on uniform coarse data [S1].

3.  **Network Evaluation:**
    *   For each sample point $x$:
        *   Compute SDF: $f_{sdf}(x) = F_\theta(\gamma(x))$ [S1].
        *   Compute Normal $n$ using finite differences to ensure smoothness:
            $$ \nabla f_{sdf}(x) \approx \frac{f_{sdf}(x + \epsilon) - f_{sdf}(x - \epsilon)}{2\epsilon} $$
            where $\epsilon$ is the hash grid size [S1].
            $n = \nabla f_{sdf}(x)$ [S1].
        *   Compute Color: $c = F_\beta(x, n, enc, d)$ [S1].

4.  **Rendering:**
    *   Compute accumulated opacity $\hat{O}_k$ for pixel $k$:
        $$ \hat{O}_k = \sum_{i=1}^{n} T_{k,i} \alpha_{k,i} $$
        where $T_{k,i}$ is the transmittance and $\alpha_{k,i}$ is the opacity at sample $i$ [S1].

### Parameters
*   **Learning Rate:** 0.001 for both Geometry Network $F_\theta$ and Appearance Network $F_\beta$ [S1].
*   **Optimizer:** AdamW with betas $(0.9, 0.99)$ [S1].
*   **Regularization Weights:** $\lambda_1$ (Eikonal) = 0.1, $\lambda_2$ (Mask) = 0.1 [S1].
*   **Progressive Update Step:** 1000 iterations [S1].
*   **Hash Encoding:** 16 levels ($2^5$ to $2^{16}$), feature size 2 [S1].

### Edge Cases and Pitfalls
*   **Normal Noise:** Using analytic gradients for normals in SDF networks can be noisy. This method uses finite differences for smoother normals [S1].
*   **Low Resolution Grid:** If the Occupancy Grid resolution is low, uniform sampling within occupied cells still introduces invalid samples. The nonlinear function mitigates this by biasing samples within those cells [S1].
*   **Boundary Conditions:** The nonlinear function $\hat{w}(s)$ requires $a > 0$ for the logarithm in the integral derivation. If $a=0$, special handling for the exponential distribution is required (not explicitly detailed in source, but implied by the formula structure) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sample_ray(ray_origin, ray_direction, occupancy_grid, geometry_net, appearance_net, hash_encoding, epsilon=1e-4):
    """
    Performs two-stage sampling along a ray.
    """
    samples = []
    
    # 1. Coarse Stage: Identify occupied intervals using Occupancy Grid
    # Assume ray is discretized into steps or intervals [t_i, t_{i+1}]
    occupied_intervals = []
    for i in range(num_intervals):
        t_mid = (ray_origin + ray_direction * (t_i + t_{i+1}) / 2)
        if occupancy_grid.query(t_mid) == 1: # 1 indicates occupied
            occupied_intervals.append((t_i, t_{i+1}))
            
    # 2. Fine Stage: Sample within occupied intervals using nonlinear function
    for (t_start, t_end) in occupied_intervals:
        # Initial coarse weights to determine a and b
        # In practice, this might use a proposal network or previous iteration weights
        # Here we assume we have weights w(t_start) and w(t_end) from a coarse pass
        w_start = compute_coarse_weight(t_start, ...) 
        w_end = compute_coarse_weight(t_end, ...)
        
        a = w_start
        b = w_end
        
        # Generate samples within [t_start, t_end] biased by nonlinear function
        # The function pushes samples towards the end with higher weight
        num_fine_samples = 8
        for j in range(num_fine_samples):
            # Uniform random variable for inverse transform sampling
            u = np.random.uniform(0, 1)
            
            # Inverse CDF for exponential interpolation
            # CDF(s) = (a/b)^((s-t_start)/(t_end-t_start)) ... simplified derivation
            # s = t_start + (t_end - t_start) * log(1 - u * (1 - (b/a))) / log(b/a)
            # Note: Source formula (7) and (8) imply exponential interpolation.
            # If a == b, uniform sampling.
            
            if abs(a - b) < 1e-8:
                s = t_start + u * (t_end - t_start)
            else:
                # Exponential interpolation parameter
                # Based on integral constraint: integral of w(s) ds = ln(b) - ln(a)
                # The specific inverse transform formula for w(s) = a * (b/a)^((s-t_start)/(t_end-t_start))
                # is derived from CDF.
                # Let k = (b/a). w(s) = a * k^((s-t_start)/L) where L = t_end - t_start.
                # CDF(x) = integral_0^x a * k^(t/L) dt = (a*L / ln(k)) * (k^(x/L) - 1)
                # Set CDF(x) = u * Total_Integral
                # Total_Integral = (a*L / ln(k)) * (k - 1)
                # u = (k^(x/L) - 1) / (k - 1)
                # k^(x/L) = 1 + u * (k - 1)
                # x = L * log(1 + u * (k - 1)) / log(k)
                
                k = b / a
                L = t_end - t_start
                s = t_start + L * np.log(1 + u * (k - 1)) / np.log(k)
            
            # 3. Network Evaluation
            x = ray_origin + ray_direction * s
            
            # Geometry Network
            enc = hash_encoding.encode(x)
            sdf, enc_feature = geometry_net(enc)
            
            # Normal via Finite Differences
            # epsilon is hash grid size
            x_plus = ray_origin + ray_direction * (s + epsilon)
            x_minus = ray_origin + ray_direction * (s - epsilon)
            sdf_plus, _ = geometry_net(hash_encoding.encode(x_plus))
            sdf_minus, _ = geometry_net(hash_encoding.encode(x_minus))
            
            grad_sdf = (sdf_plus - sdf_minus) / (2 * epsilon)
            normal = grad_sdf / np.linalg.norm(grad_sdf)
            
            # Appearance Network
            direction_encoded = spherical_harmonics_encode(ray_direction)
            color = appearance_net(x, normal, enc_feature, direction_encoded)
            
            samples.append({
                'position': x,
                'sdf': sdf,
                'color': color,
                'normal': normal,
                'weight': b if s > t_start + L/2 else a # Approximation for opacity calculation
            })
            
    return samples

def compute_loss(samples, gt_color, gt_mask):
    """
    Computes the total loss as defined in the source.
    """
    # L_color: L2 loss between rendered color and ground truth
    l_color = 0
    for sample in samples:
        l_color += np.sum((sample['color'] - gt_color) ** 2)
        
    # L_eikonal: SDF gradient magnitude should be 1
    # Using the computed normals from finite differences
    l_eikonal = 0
    for sample in samples:
        grad_mag = np.linalg.norm(sample['normal'])
        l_eikonal += (grad_mag - 1) ** 2
        
    # L_mask: BCE between accumulated opacity and ground truth mask
    # Accumulated opacity O_k = sum(T_i * alpha_i)
    # This requires transmittance calculation which depends on previous samples
    # Simplified representation:
    accumulated_opacity = compute_accumulated_opacity(samples)
    l_mask = binary_cross_entropy(gt_mask, accumulated_opacity)
    
    lambda1 = 0.1
    lambda2 = 0.1
    
    total_loss = l_color + lambda1 * l_eikonal + lambda2 * l_mask
    return total_loss
```

## Subtopics

### NeuS Sampling
Uses uniform coarse sampling followed by inverse transform sampling based on weights. Results in random scatter of fine samples near the surface [S1].

### Instant-NGP Sampling
Uses Occupancy Grid to skip empty regions but uses uniform sampling within occupied cells. Inefficient at low grid resolutions [S1].

### DoNeRF Sampling
Uses a depth occlusion prediction network to guide sampling. Requires RGBD input [S1].

## Operating characteristics

*   **Time Complexity:** Reduced compared to uniform sampling due to skipping empty regions (Occupancy Grid) and fewer samples needed for convergence (biased sampling) [S1].
*   **Memory Complexity:** Requires storage for the Occupancy Grid and Hash Encoding tables (16 levels, feature size 2) [S1].
*   **Performance:** Achieves better surface reconstruction quality (Chamfer Distance) and new view synthesis (PSNR) on DTU and NeRF Synthetic datasets compared to NeuS and Neus-acc [S1].
*   **Strengths:** Avoids invalid sampling in empty space; biases samples toward surfaces without needing a separate depth prediction network; smoother normals via finite differences [S1].
*   **Limitations:** Requires tuning of the nonlinear function parameters (implicitly handled by weights $a$ and $b$); dependent on the quality of the Occupancy Grid [S1].
*   **When to use:** When training neural implicit surfaces with RGB data only, and computational efficiency is critical [S1].
*   **When not to use:** When RGBD data is available, other methods like DoNeRF might offer different trade-offs [S1].

## Background

*   **Origin:** Proposed by Siyuan He and Xinglin Liu from Wuyi University, published in March 2024 [S1].
*   **History:** Builds upon NeRF [1] and NeuS [2]. Addresses limitations of NeuS (uniform sampling waste) and Instant-NGP/NeuS2 (uniform sampling in occupied cells) [S1]. Inspired by DoNeRF's analysis of sampling strategies but removes the need for RGBD input [S1].
*   **Usage:** Neural implicit surface reconstruction, 3D representation, surface recovery, and novel view synthesis [S1].

## Sources

[S1] Research on Efficient Sampling of Neural Implicit Surface Reconstruction, https://doi.org/10.12677/csa.2024.143065, fetched 2026-10-29
