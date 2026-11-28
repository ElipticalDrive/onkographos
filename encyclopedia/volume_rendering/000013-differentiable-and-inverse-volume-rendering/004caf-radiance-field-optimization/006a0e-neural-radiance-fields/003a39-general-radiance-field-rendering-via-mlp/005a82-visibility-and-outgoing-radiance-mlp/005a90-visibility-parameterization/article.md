# Visibility Parameterization

Also called: Mixture of Logistics Visibility, NeuRay Visibility Parameterization, Visibility from occlusion probability

## Overview
Visibility Parameterization is a method to represent the visibility function $v(z)$ along a ray as a continuous, differentiable function using a neural network [S1]. It solves the problem of expensive ray tracing or sphere tracing for occlusion by replacing discrete sampling with a learned probability distribution [S2]. It sets itself apart from recursive path tracing by avoiding iterative ray-surface intersection tests, instead decoding a feature vector into a mixture of logistic distributions that directly yields the occlusion probability [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A feature vector $\mathbf{g}$ (decoded from an input view's feature map) and a depth $z$ along the ray originating from the location of $\mathbf{g}$ [S1].
*   **Output:** The visibility value $v(z)$, the occlusion CDF $t(z)$, and the hitting probability density $h_i$ for sample points [S1].

**Data Structures:**
*   **Visibility Feature Map $\mathbf{G}$:** A 2D map where each element is a feature vector $\mathbf{g}$ [S1].
*   **MLP Decoder:** A neural network that takes $\mathbf{g}$ and outputs parameters for the mixture of logistics: means $\{\mu_i\}$, standard deviations $\{\sigma_i\}$, and mixing weights $\{w_i\}$ [S1].

**Parameters:**
*   $N_l$: Number of logistic distributions in the mixture. Typical value: 2 [S1].
*   $K_t$: Number of sample points on the test ray [S1].

**Formulas:**
1.  **Visibility Definition:**
    $$v(z) = 1 - t(z)$$
    where $t(z)$ is the occlusion probability CDF [S1].

2.  **Occlusion CDF (Mixture of Logistics):**
    $$t(z;\{\mu_i,\sigma_i,w_i\})=\sum _{i=1}^{N_{l}} w_i S\left(\frac{z-\mu_i}{\sigma_i}\right)$$
    where $S(\cdot)$ is the sigmoid function, $\mu_i$ is the mean, $\sigma_i$ is the standard deviation, and $w_i$ is the mixing weight with $\sum w_i = 1$ [S1].

3.  **Alpha Value (from CDF):**
    The alpha value $\alpha_k$ for a sample point at depth $z_k$ is derived from the CDF. While the source defines hitting probability using $\alpha_k$, the alpha value itself is typically the probability of hitting the surface within the interval ending at $z_k$. Based on the hitting probability formula, $\alpha_k$ represents the probability mass at step $k$. In the context of the coarse-to-fine variant, alpha is explicitly defined as:
    $$\alpha(z_0, z_1) = \frac{t(z_1) - t(z_0)}{1 - t(z_0)}$$
    [S1]. For the standard hitting probability, $\alpha_k$ is the probability of the $k$-th interval.

4.  **Hitting Probability Density:**
    $$h_{i}=\prod _{k=1}^{i-1} (1-\alpha _k)\alpha _i$$
    where $\alpha_k$ is the alpha value derived from the distribution for the $k$-th interval [S1].

5.  **Final Color Aggregation:**
    $$\bm {c}=\sum _{i=1}^{K_t} \bm {c}_{i} h_{i}$$
    where $\bm {c}_{i}$ is the local feature/color at sample point $i$ [S1].

**Steps:**
1.  Decode the visibility feature vector $\mathbf{g}$ using an MLP to obtain parameters $\{\mu_i, \sigma_i, w_i\}$ for $i=1 \dots N_l$ [S1].
2.  For a given depth $z$, compute the occlusion CDF $t(z)$ using the mixture of logistics formula [S1].
3.  Compute visibility $v(z) = 1 - t(z)$ [S1].
4.  For rendering, compute alpha values $\alpha_k$ for sample intervals along the ray [S1].
5.  Compute hitting probabilities $h_i$ using the product formula [S1].
6.  Aggregate colors $\bm{c}_i$ weighted by $h_i$ to produce the final pixel color [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x):
    """Standard sigmoid function."""
    return 1.0 / (1.0 + math.exp(-x))

def compute_occlusion_cdf(z, params):
    """
    Compute the occlusion CDF t(z) for a given depth z.
    
    Args:
        z: float, depth along the ray.
        params: dict containing 'mus', 'sigmas', 'weights' lists of length N_l.
        
    Returns:
        float: t(z), the cumulative occlusion probability.
    """
    mus = params['mus']
    sigmas = params['sigmas']
    weights = params['weights']
    N_l = len(mus)
    
    t_z = 0.0
    for i in range(N_l):
        # S((z - mu_i) / sigma_i)
        term = sigmoid((z - mus[i]) / sigmas[i])
        t_z += weights[i] * term
    return t_z

def compute_visibility(z, params):
    """
    Compute visibility v(z) = 1 - t(z).
    
    Args:
        z: float, depth along the ray.
        params: dict containing mixture parameters.
        
    Returns:
        float: v(z), the visibility value.
    """
    t_z = compute_occlusion_cdf(z, params)
    return 1.0 - t_z

def compute_alpha_interval(t_z_0, t_z_1):
    """
    Compute alpha value for a depth interval (z_0, z_1).
    Formula: (t(z_1) - t(z_0)) / (1 - t(z_0))
    
    Args:
        t_z_0: float, CDF at start of interval.
        t_z_1: float, CDF at end of interval.
        
    Returns:
        float: alpha value.
    """
    if 1.0 - t_z_0 == 0:
        return 0.0
    return (t_z_1 - t_z_0) / (1.0 - t_z_0)

def render_ray(sample_points_depths, sample_colors, params):
    """
    Render a ray by aggregating colors weighted by hitting probabilities.
    
    Args:
        sample_points_depths: list of floats [z_1, z_2, ..., z_Kt].
        sample_colors: list of color vectors [c_1, c_2, ..., c_Kt].
        params: dict containing mixture parameters.
        
    Returns:
        float: final color c (scalar for simplicity, or vector sum).
    """
    K_t = len(sample_points_depths)
    final_color = 0.0
    
    # Precompute CDFs for all sample points to get alphas
    # Note: The formula h_i = prod(1-alpha_k) * alpha_i implies alpha_i 
    # is the probability of hitting at step i.
    # We need alpha for each interval. Let's assume intervals are defined 
    # by consecutive sample points.
    
    # Compute t(z) for all points
    t_values = [compute_occlusion_cdf(z, params) for z in sample_points_depths]
    
    # Compute alpha for each interval (z_i, z_{i+1})? 
    # The source formula h_i = prod_{k=1}^{i-1} (1-alpha_k) * alpha_i 
    # suggests alpha_i is associated with the i-th sample/interval.
    # Standard volume rendering alpha is often defined per interval.
    # Let's compute alpha_i as the probability of the i-th segment.
    # Using the interval formula: alpha(z_i, z_{i+1})
    
    alphas = []
    for i in range(K_t - 1):
        z_0 = sample_points_depths[i]
        z_1 = sample_points_depths[i+1]
        t_0 = t_values[i]
        t_1 = t_values[i+1]
        alpha_i = compute_alpha_interval(t_0, t_1)
        alphas.append(alpha_i)
    
    # The last point's alpha is not defined by an interval in this loop.
    # However, the hitting probability formula uses alpha_i.
    # In standard NeuRay, alpha_i is often derived from the CDF difference 
    # normalized by remaining transparency.
    # Let's assume the hitting probability h_i corresponds to the sample point i.
    # If K_t samples, there are K_t-1 intervals. 
    # The source says h_i for depth interval (z_i, z_{i+1}).
    # So h_1 corresponds to interval (z_1, z_2)? Or (z_0, z_1)?
    # Source: "h_i for depth interval (z_i, z_{i+1})".
    # And c = sum c_i h_i.
    # This implies h_i is the weight for color c_i.
    
    # Let's re-read carefully: "h_i for depth interval (z_i, z_{i+1})".
    # And "c = sum_{i=1}^{K_t} c_i h_i".
    # This indexing is slightly ambiguous. Usually, h_i is the probability 
    # of the ray terminating at or within the i-th interval.
    
    # Let's stick to the explicit formula: h_i = prod_{k=1}^{i-1} (1-alpha_k) * alpha_i.
    # This requires alpha_1, alpha_2, ..., alpha_{K_t}.
    # We computed alphas for intervals (z_1, z_2) ... (z_{K_t-1}, z_{K_t}).
    # We need alpha for the first interval (z_1, z_2) to be alpha_1?
    # Or is alpha_i the alpha associated with sample point i?
    
    # In NeuRay, alpha is often computed as:
    # alpha(z_0, z_1) = (t(z_1) - t(z_0)) / (1 - t(z_0))
    # This is the probability of hitting in (z_0, z_1) given survival up to z_0.
    
    # Let's assume sample points are z_1 ... z_Kt.
    # Intervals are (z_1, z_2), (z_2, z_3), ..., (z_{K_t-1}, z_{K_t}).
    # And perhaps a final interval or the last point is handled differently.
    # Given the source formula h_i = prod_{k=1}^{i-1} (1-alpha_k) * alpha_i,
    # h_1 = alpha_1.
    # h_2 = (1-alpha_1) * alpha_2.
    
    # We need alpha_1 ... alpha_{K_t}.
    # Let's define alpha_i as the alpha for the interval ending at z_i?
    # Or starting at z_i?
    # Source: "alpha value derived from the distribution".
    # And "h_i for depth interval (z_i, z_{i+1})".
    # This suggests h_i is the weight for the interval (z_i, z_{i+1}).
    # But the sum is over c_i. So c_i is the color at z_i? Or in the interval?
    # Usually c_i is the color at sample point z_i.
    
    # Let's assume the standard interpretation:
    # alpha_i is the probability of the i-th sample point being the hit.
    # This is often computed as:
    # alpha_i = (t(z_i) - t(z_{i-1})) / (1 - t(z_{i-1}))
    # with z_0 = 0 (or near field).
    
    # Let's implement this standard interpretation which matches the formula structure.
    
    # Define z_0 = 0 (or minimum depth)
    z_prev = 0.0
    t_prev = compute_occlusion_cdf(z_prev, params)
    
    hitting_probs = []
    
    for i in range(K_t):
        z_curr = sample_points_depths[i]
        t_curr = t_values[i]
        
        # Alpha for the interval (z_prev, z_curr)
        # This alpha represents the probability of hitting in this interval
        # given survival up to z_prev.
        alpha_i = compute_alpha_interval(t_prev, t_curr)
        
        # Hitting probability h_i
        # h_i = alpha_i * prod_{k=1}^{i-1} (1-alpha_k)
        # We can compute this iteratively.
        
        if i == 0:
            h_i = alpha_i
        else:
            # h_i = h_{i-1} / (1 - alpha_{i-1}) * alpha_i ? No.
            # h_i = (1-alpha_{i-1}) * ... * (1-alpha_1) * alpha_i
            # h_{i-1} = (1-alpha_{i-2}) * ... * (1-alpha_1) * alpha_{i-1}
            # So h_i = h_{i-1} * (1-alpha_{i-1}) / alpha_{i-1} * alpha_i
            # This is numerically unstable. Better to compute product.
            
            # Let's compute product explicitly for clarity
            prod_survival = 1.0
            for k in range(i):
                # We need alpha_k for k < i
                # We didn't store them. Let's store them.
                pass
        
        # Let's rewrite loop to store alphas
        pass

    # Rewritten loop
    alphas_list = []
    t_prev = compute_occlusion_cdf(0.0, params) # Assuming start at 0
    
    for i in range(K_t):
        z_curr = sample_points_depths[i]
        t_curr = t_values[i]
        alpha_i = compute_alpha_interval(t_prev, t_curr)
        alphas_list.append(alpha_i)
        t_prev = t_curr
        
    hitting_probs = []
    for i in range(K_t):
        alpha_i = alphas_list[i]
        prod_survival = 1.0
        for k in range(i):
            prod_survival *= (1.0 - alphas_list[k])
        h_i = prod_survival * alpha_i
        hitting_probs.append(h_i)
        
    # Aggregate
    final_color = 0.0
    for i in range(K_t):
        final_color += sample_colors[i] * hitting_probs[i]
        
    return final_color
```

**Edge Cases and Pitfalls:**
*   **Precision:** The product formula for hitting probabilities can suffer from underflow if many intervals have high alpha values. [G] Use log-space arithmetic or iterative updates if numerical stability is an issue.
*   **Boundaries:** The CDF $t(z)$ must be non-increasing and $0 \le v(z) \le 1$. The mixture of logistics ensures $t(z)$ is monotonic if weights are positive and sum to 1. [S1]
*   **Degenerate Input:** If $\sigma_i$ is very small, the logistic distribution becomes a step function, potentially causing discontinuities in gradients. [G] Add a small epsilon to $\sigma_i$ during training.

## Subtopics

### NeuRay Coarse-to-Fine Sampling
A method to accelerate rendering by computing a coarse hitting probability $\hat{h}_i$ from NeuRay to identify relevant sample points, avoiding feature aggregation on empty points [S1]. It sets itself apart by using input view visibility to estimate test ray occupancy without full fine-grained sampling [S1]. Choose it when rendering speed is critical and the scene has large empty regions [S1]. Link: [NeuRay](16669-neuray.md) (hypothetical link to main NeuRay article).

**Definition:**
Uses a coarse hitting probability $\hat{h}_i$ to identify points with large probability, sampling fine points only around these locations for feature aggregation [S1].

**Steps:**
1.  Compute alpha value $\tilde{\alpha}$ in depth range $(z_0, z_1)$ on an input ray [S1].
2.  Compute coarse alpha $\hat{\alpha}_i$ for point $p_i$ on test ray [S1].
3.  Compute coarse hitting probability $\hat{h}_i$ [S1].
4.  Use $\hat{h}_i$ to identify points with large probability [S1].
5.  Sample fine points around these locations for feature aggregation [S1].

**Formulas:**
1.  **Alpha Value on Input Ray:**
    $$\tilde {\alpha }(z_0,z_1)=\frac {t(z_1)-t(z_0)}{1-t(z_0)}$$
    [S1].

2.  **Coarse Alpha:**
    $$\hat {\alpha }_{i}=\frac {\sum _{j}\tilde {\alpha }_{i,j}(z_{i,j},z_{i,j}+l_i) v_{i,j}}{\sum _{j} v_{i,j}}$$
    where $j$ indexes input views, $z_{i,j}$ is depth on view $j$, $l_i$ is distance to next point, and $v_{i,j}$ is visibility [S1].

3.  **Coarse Hitting Probability:**
    $$\hat {h}_{i}=\prod _{k=1}^{i-1} (1-\hat {\alpha }_k)\hat {\alpha }_i$$
    [S1].

**Parameters:**
*   $l_i$: Distance between point $p_i$ and its subsequent point $p_{i+1}$ on the test ray [S1].

**Pitfalls:**
*   The consistency loss $\ell_{consist} = \frac{1}{K_t}\sum_{i=1}^{K_t} CE(\tilde{h}_i, h_i)$ is used to enforce consistency between coarse and fine hitting probabilities [S1].

## Operating characteristics
*   **Time Complexity:** Rendering is faster than sphere tracing because it avoids iterative ray-surface intersection tests. The MLP decoding is $O(N_l)$ per sample point [S1, S2].
*   **Memory Complexity:** Requires storing the visibility feature map $\mathbf{G}$ and the MLP parameters [S1].
*   **Measured Performance:** On synthetic objects, NeuRay achieves higher PSNR and SSIM than baselines like PixelNeRF and MVSNeRF [S1]. On real objects (DTU), it also shows improvements [S1].
*   **Strengths:** Compact representation, differentiable, avoids expensive tracing [S2].
*   **Limitations:** Requires training on input views; generalization depends on the number of input views and the quality of the feature map [S1].
*   **When to use:** When indirect illumination needs to be modeled efficiently and differentiability is required [S2].
*   **When not to use:** If real-time rendering with very limited memory is required and the scene is simple enough for traditional ray tracing [S2].

## Background
Originated in the paper "Neural Rays for Occlusion-aware Image-based Rendering" (CVPR 2022) by [S1] and "Modeling Indirect Illumination for Inverse Rendering" (CVPR 2022) by [S2]. It was developed to address the computational cost of sphere tracing in inverse rendering and image-based rendering [S2]. It is used in general radiance field rendering via MLPs, specifically in the NeuRay framework [S1].

## Sources
[S1] Neural Rays for Occlusion-aware Image-based Rendering, https://doi.org/10.1109/cvpr52688.2022.00767, fetched 2026-10-26
[S2] Modeling Indirect Illumination for Inverse Rendering, https://doi.org/10.1109/cvpr52688.2022.01809, fetched 2026-10-26
