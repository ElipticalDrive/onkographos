# Gradient-driven Sampling

Also called: Gradient-based Step Size Adjustment

## Overview
Gradient-driven sampling is a dynamic step size adjustment method for distance-field ray marching that varies the integration step length based on the local magnitude of the scalar field gradient. It solves the problem of balancing rendering fidelity and performance in volumes with sharp boundaries (e.g., CT angiography) by taking smaller steps near high-gradient regions to capture transitions accurately and larger steps in smooth interiors. It sets itself apart from uniform sampling by adapting to local feature sharpness and from hierarchical sampling by using a continuous gradient magnitude threshold rather than discrete variance-based subdivision [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar field $s(\mathbf{x})$ representing the volume data; a ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a minimum step size $h_{min}$; a maximum step size $h_{max}$; and a gradient threshold $\tau$.
*   **Output:** A sequence of sample points along the ray and the accumulated transmittance/opacity.

**Data Structures:**
*   The scalar field is typically stored as a 3D grid. Access requires trilinear interpolation to compute $s(\mathbf{x})$ and its gradient $\nabla s(\mathbf{x})$ at arbitrary positions.
*   Gradient $\nabla s$ is a 3D vector $(\frac{\partial s}{\partial x}, \frac{\partial s}{\partial y}, \frac{\partial s}{\partial z})$.

**Steps:**
1.  Initialize current position $\mathbf{p} = \mathbf{o}$ and step size $h = h_{max}$.
2.  While $\mathbf{p}$ is within the volume bounds:
    a. Evaluate the scalar field value $s(\mathbf{p})$ and the gradient vector $\nabla s(\mathbf{p})$ at the current position.
    b. Compute the gradient magnitude $g = \|\nabla s(\mathbf{p})\|$.
    c. Determine the desired step size $h_{desired}$ based on $g$ and $\tau$. If $g > \tau$, reduce $h_{desired}$ to resolve the sharp transition; if $g \le \tau$, increase $h_{desired}$ to accelerate traversal through smooth regions. The exact functional form mapping $g$ to $h_{desired}$ is not specified in the sources, but it must be monotonic decreasing with respect to $g$ [S1].
    d. Clamp the step size: $h = \min(h_{max}, \max(h_{min}, h_{desired}))$.
    e. Advance the ray: $\mathbf{p}_{next} = \mathbf{p} + h \cdot \mathbf{d}$.
    f. Sample the volume at $\mathbf{p}$ (or between $\mathbf{p}$ and $\mathbf{p}_{next}$ depending on the compositing scheme) and accumulate color/transmittance.
    g. Update $\mathbf{p} = \mathbf{p}_{next}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_driven_sampling(ray_origin, ray_direction, volume, h_min, h_max, threshold):
    p = ray_origin
    t = 0.0
    accumulated_color = 0.0
    accumulated_transmittance = 1.0
    
    # Assume volume has bounds and supports trilinear interpolation
    while volume.contains(p):
        # 1. Evaluate gradient magnitude
        grad = volume.compute_gradient(p)
        grad_magnitude = length(grad)
        
        # 2. Determine step size based on gradient
        # If gradient is high (sharp feature), take smaller steps.
        # If gradient is low (smooth), take larger steps.
        # The specific mapping function is not detailed in sources [S1].
        # A common heuristic is inverse proportionality or step-wise reduction.
        if grad_magnitude > threshold:
            # High gradient: reduce step size to capture boundary
            # Scale factor depends on implementation; must be > 0
            scale_factor = threshold / grad_magnitude 
            h_desired = h_max * scale_factor
        else:
            # Low gradient: allow larger steps, up to h_max
            h_desired = h_max
            
        # 3. Bound the step size
        h = max(h_min, min(h_max, h_desired))
        
        # 4. Advance ray
        p_next = p + h * ray_direction
        
        # 5. Sample and composite (simplified front-to-back)
        # Note: Exact compositing formula depends on the parent method
        # This step is a placeholder for the actual volume integration logic [G]
        sample_val = volume.sample(p)
        color, alpha = volume.convert_to_color_alpha(sample_val)
        accumulated_color += accumulated_transmittance * color * alpha
        accumulated_transmittance *= (1.0 - alpha)
        
        p = p_next
        
    return accumulated_color
```

**Edge Cases and Pitfalls:**
*   **Precision:** Computing the gradient requires accurate derivatives. Finite differences may introduce noise if the volume data is sparse or noisy, leading to erratic step sizes.
*   **Boundaries:** The step size clamping prevents infinite steps in empty space (if $h_{max}$ is finite) and infinite small steps in high-frequency noise (if $h_{min}$ is non-zero).
*   **Degenerate Input:** If the gradient is zero everywhere (constant volume), the step size remains at $h_{max}$, which is optimal. If the gradient is extremely high everywhere, the step size remains at $h_{min}$, potentially causing performance degradation but ensuring accuracy.

## Subtopics

### [Dynamic Sampling Strategy](9081-dynamic-sampling-strategy.md)
A procedure to dynamically determine the number and locations of sample points in a participating medium to accurately reproduce shading variations and minimize rendering errors. It adapts sample locations based on shading error, unlike prior methods that sample based on density distribution. It builds on [Gradient-driven Sampling](5281-gradient-driven-sampling.md) by using a specific gradient-based error metric to drive a recursive splitting process.

## Operating characteristics
*   **Time Complexity:** Variable. In smooth regions, complexity approaches $O(1)$ per unit distance (large steps). In high-gradient regions, it approaches $O(N)$ where $N$ is the number of samples required to resolve the feature.
*   **Memory Complexity:** $O(1)$ additional memory beyond the volume data structure, as it only stores current position and accumulated values.
*   **Measured Performance:** Achieves efficiency gains of 2-3x over uniform sampling without aliasing artifacts in boundary-heavy volumes like CT angiography data [S1].
*   **Strengths:** Excellent detail preservation in boundary-heavy volumes; efficient traversal in smooth interiors; prevents aliasing artifacts at sharp edges.
*   **Limitations:** Requires computation of the gradient at each step, which adds overhead compared to simple uniform sampling. Performance depends heavily on the choice of threshold $\tau$ and bounds $h_{min}, h_{max}$.
*   **When to use:** Volumes with sharp boundaries and sparse opaque features (e.g., medical scans, CT angiography) where uniform sampling would either alias boundaries or waste samples in empty space.
*   **When not to use:** Volumes with uniform density or no sharp features, where uniform sampling is simpler and faster.

## Background
Gradient-driven sampling is a technique used in volume ray casting to optimize the sampling rate along rays. It is particularly relevant in medical imaging applications such as CT angiography, where accurate representation of vessel boundaries is critical. The method leverages the scalar field gradient to identify regions of high feature sharpness, allowing for adaptive refinement. It is an evolution of dynamic sampling step adjustment, focusing specifically on gradient magnitude as the primary driver for step size variation.

## Sources
[S1] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
