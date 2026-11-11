# Zero Order Integration (Rectangle Rule)

Also called: Riemann Sum, Riemann sum integration, Riemann sum with rectangle rule, Riemann Sum Approximation for Volume Rendering, Quadrature Approximation of Volume Rendering Integral

## Overview
Zero Order Integration approximates the continuous volume rendering integral by discretizing the ray into $n$ equal segments and assuming volume density and color are constant within each segment, effectively evaluating a Riemann sum using the rectangle rule [S1, S2, unofficial][S7]. It sets itself apart from higher-order methods like [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md) or [Gaussian Integration for Volume Rendering](5546-gaussian-integration-for-volume-rendering.md) by requiring only one sample per segment and avoiding pre-integration tables, making it computationally simple but prone to visible discontinuities due to its piecewise-constant assumption [S7, S8].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray parameterization $r(t) = o + td$ for $t \in [0, D]$, where $D$ is the exit distance. Number of segments $n$. Extinction coefficient $\tau(t)$ and color/emission $c(t)$ or $g(t)$ [S2, unofficial][S7].
*   **Output:** Composite color $C$ and optionally accumulated opacity $A$ or expected depth [S2, S3, unofficial].

**Data Structures:**
*   **Sample Points:** $n$ positions $x_i = i\Delta x$ for $i=1 \dots n$, where $\Delta x = D/n$ [S2, unofficial][S6, book].
*   **Segment Lengths:** $\Delta x$ (uniform) or $\delta_i = t_{i+1} - t_i$ (if non-uniform) [S4, unofficial][S6, book].
*   **Optical Properties:** For each segment $i$, opacity $A_i$, transparency $T_i$, and pre-multiplied color $C_i$ (or $g_i$) [S2, S4, unofficial].

**Numbered Steps:**

1.  **Discretize the Ray:** Divide the interval $[0, D]$ into $n$ equal segments of length $\Delta x = D/n$ [S2, unofficial][S6, book]. Define sample points $x_i = i\Delta x$ for $i=1 \dots n$ [S7]. Alternatively, define endpoints $t_1, \dots, t_{n+1}$ and lengths $\delta_i = t_{i+1} - t_i$ [S4, unofficial].
2.  **Sample the Volume:** Evaluate the extinction coefficient $\tau(x_i)$ and color/emission $c(x_i)$ (or $g(x_i)$) at each sample point $x_i$ [S2, unofficial][S7].
3.  **Compute Segment Opacity and Color:**
    *   Calculate the transparency of the $i$-th segment: $T_i \approx e^{-\tau(x_i) \Delta x}$ [S1, S2, unofficial][S7].
    *   Calculate the opacity of the $i$-th segment: $A_i = 1 - T_i$ [S2, unofficial].
    *   *Optimization:* For efficiency, if $\tau(x_i) \Delta x \ll 1.0$, approximate $A_i \approx \tau(x_i) \Delta x$ to avoid exponential evaluation [S2, unofficial]. Note that this linear approximation can cause visible artifacts if the product is not small [S6, book].
    *   The color contribution is $C_i = c(x_i) \Delta x$ or simply $g(x_i)$ depending on the transfer function definition [S1, S2, unofficial].
4.  **Compute Transmittance:**
    *   The accumulated transmittance $T_i$ up to sample $i$ is the product of segment transparencies:
        $$ T_i = \prod_{j=1}^{i-1} (1 - A_j) = \prod_{j=1}^{i-1} T_j $$
        where $T_1 = 1$ [S3, unofficial][S7].
    *   Numerically, this is computed via cumulative product [G]. To prevent numerical instability when $A_j \approx 1$, add a small epsilon (e.g., $10^{-10}$) to $(1 - A_j)$ before multiplication [S3, unofficial].
5.  **Composite the Image:**
    *   **Back-to-Front (Iteration A):** Iterate from $i=n$ down to $1$. Initialize $C'_{n+1} = 0$.
        $$ C'_i = C_i + (1 - A_i) C'_{i+1} $$
        This is equivalent to $C'_i = C_i + T_i C'_{i+1}$ [S2, unofficial][S7].
    *   **Front-to-Back:** Iterate from $i=1$ to $n$. Initialize $C'_0 = 0$ and $A'_0 = 0$.
        $$ C'_i = C'_{i-1} + (1 - A'_{i-1}) C_i $$
        $$ A'_i = A'_{i-1} + (1 - A'_{i-1}) A_i $$
        This method supports early ray termination when transmittance becomes negligible [S2, unofficial][S6, book][S7].

**Pseudocode (Front-to-Back with Early Termination):**

```python
import math

def zero_order_integration(ray_origin, ray_direction, volume_sampler, n_samples, near, far, threshold=1e-4):
    """
    Performs zero-order integration (rectangle rule) for a single ray.
    
    Args:
        ray_origin: (3,) array, camera origin.
        ray_direction: (3,) array, normalized view direction.
        volume_sampler: callable, returns (sigma, color) given a 3D position.
        n_samples: int, number of segments.
        near: float, near clipping plane distance.
        far: float, far clipping plane distance.
        threshold: float, early termination threshold for transmittance.
        
    Returns:
        color: (3,) array, composite color.
        depth: float, expected depth.
        opacity: float, accumulated opacity.
    """
    D = far - near
    delta_x = D / n_samples
    
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_opacity = 0.0
    transmittance = 1.0
    expected_depth = 0.0
    
    for i in range(n_samples):
        # Early termination check [S6, book][S7]
        if transmittance < threshold:
            break
            
        # Left endpoint sampling per segment [S2, unofficial][S6, book][S7]
        t_sample = near + i * delta_x
        pos = ray_origin + ray_direction * t_sample
        
        # Sample volume [S2, unofficial][S7]
        sigma, color = volume_sampler(pos)
        
        # Compute segment opacity [S3, unofficial]
        alpha = 1.0 - math.exp(-sigma * delta_x)
        
        # Compute weight and accumulate [S2, S3, unofficial]
        weight = transmittance * alpha
        accumulated_color[0] += weight * color[0]
        accumulated_color[1] += weight * color[1]
        accumulated_color[2] += weight * color[2]
        
        expected_depth += weight * t_sample
        accumulated_opacity += weight
        
        # Update transmittance for next segment [S3, unofficial]
        transmittance *= (1.0 - alpha + 1e-10)
        
    # Clamp depth to avoid division by zero if computing disparity later [S3, unofficial]
    if accumulated_opacity < 1e-10:
        expected_depth = 1e-10
        
    return accumulated_color, expected_depth, accumulated_opacity
```

**Edge Cases and Pitfalls:**
*   **Discontinuities:** The reconstructed signal is not $C^0$ continuous, leading to visible blocky artifacts, especially with low $n$ [S8].
*   **Linear Approximation Artifacts:** Using $A_i \approx \tau(x_i)\Delta x$ causes artifacts if $\tau(x_i)\Delta x$ is not small [S6, book].
*   **Numerical Stability:** When computing transmittance, adding $10^{-10}$ to $(1-\alpha)$ prevents NaNs if $\alpha=1$ [S3, unofficial].
*   **Homogeneity Assumption:** Uniform sampling assumes constant density/color within segments, causing artifacts in homogeneous regions if the step size is too large relative to feature size [S1, unofficial].

## Subtopics

### [Runge-Kutta 4 (RK4) Integration](3158-runge-kutta-4-rk4-integration.md)
A higher-order method that queries the vector field at four positions per step, reducing truncation error to $O(dt^5)$ and allowing larger step sizes, at the cost of four samples per step.

### [Gaussian Integration for Volume Rendering](5546-gaussian-integration-for-volume-rendering.md)
Uses unequally weighted sums at unequally spaced points to achieve higher accuracy for smooth transfer functions and density variations compared to equally spaced rectangle rules.

## Operating characteristics
*   **Time Complexity:** $O(n)$ per ray, where $n$ is the number of samples [S2, unofficial][S6, book].
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray if computed iteratively (front-to-back), or $O(n)$ if storing all samples for back-to-front [S2, unofficial][S6, book].
*   **Performance:** Very fast per sample due to simplicity and lack of pre-integration table lookups or complex quadrature weights [S7].
*   **Strengths:** Simple to implement; supports early ray termination in front-to-back mode; no pre-computation required [S2, unofficial][S6, book][S7].
*   **Limitations:** Low accuracy; visible artifacts (blockiness) due to piecewise-constant approximation; sensitive to step size [S1, unofficial][S7, S8].
*   **When to Use:** Real-time applications with strict performance constraints where visual fidelity is secondary, or as a baseline for comparison [S2, unofficial][S6, book].
*   **When Not to Use:** Not given in the sources.

## Background
The method originates from the fundamental discretization of the volume rendering integral using Riemann sums [S1, S2, S5, unofficial]. It is often referred to as "Iteration A" in early literature [S6, book]. The approach assumes a piecewise-constant function (zero-order hold) for the scalar signal along the ray [S7, S8]. It is widely used in basic volume raycasting implementations and serves as the foundation for more advanced techniques like [Partial Pre-Integration](1148-partial-pre-integration.md) which attempt to mitigate its inaccuracies [S7]. The video source [S4, unofficial] provides a visual derivation of the quadrature estimate.

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S3] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] PDF Volume Rendering - faculty.cc.gatech.edu, https://faculty.cc.gatech.edu/~hays/compvision2025spring/lectures/26.pdf, fetched 2026-10-26 (unofficial)
[S5] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
[S6] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
[S7] Optical models for direct volume rendering (PDF) - Academia.edu, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
[S8] Second Order Pre-Integrated Volume Rendering, https://people.freedesktop.org/~marcheu/pacificvis08.pdf, fetched 2026-10-29
