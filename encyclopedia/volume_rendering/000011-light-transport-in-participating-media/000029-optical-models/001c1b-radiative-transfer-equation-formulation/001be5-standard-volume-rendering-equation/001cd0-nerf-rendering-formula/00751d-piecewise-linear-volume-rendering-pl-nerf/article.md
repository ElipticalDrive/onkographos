# Piecewise Linear Volume Rendering (PL-NeRF)
Also called: Linear opacity NeRF, PL-NeRF (Piecewise Linear Opacity), Piecewise Linear Opacity, Piecewise Linear Opacity Formulation

## Overview
Piecewise Linear Volume Rendering (PL-NeRF) is a volume rendering method that reformulates the sample-based rendering equation to correspond to the exact integral under the assumption of piecewise linear volume density (opacity) and piecewise constant color [S1]. It serves as a drop-in replacement for the classical rendering equation used in NeRFs, resolving quadrature instability, ray conflicts, and non-differentiability of quantiles associated with piecewise constant opacity assumptions [S1]. Unlike [Radiance Mapping](20852-radiance-mapping.md) which alters the sampling distribution via neural networks or occupancy caching, PL-NeRF derives an invertible Cumulative Distribution Function (CDF) directly from the opacity model, enabling precise importance sampling without surrogate functions [S1]. Compared to [GIRAFFE Volume Rendering](20868-giraffe-volume-rendering.md), PL-NeRF focuses on the mathematical stability of the integration step itself rather than hierarchical feature aggregation [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- Ray origin $o$ and direction $d$.
- Near and far clipping distances $s_{near}$ and $s_{far}$.
- Number of coarse samples $N_c$ and fine samples $N_i$ [S1].
- Neural network outputs for each sample point $s_k$: volume density $\tau_k$ (opacity) and color $c_k$ [S1].

**Outputs:**
- Rendered color $C(r)$ for the ray $r$.
- Depth map (via expected depth calculation) [S1].

### Data Structures
The method operates on a 1D array of samples along the ray.
- **Samples:** An array of $N$ positions $s_0, s_1, \dots, s_N$ where $s_0 = s_{near}$ and $s_N = s_{far}$.
- **Properties:** For each sample $k$, the model predicts scalar opacity $\tau_k \ge 0$ and vector color $c_k \in \mathbb{R}^3$ [S1].
- **Transmittance Array:** An array $T$ of size $N+1$, where $T[k]$ stores the transmittance at sample $k$ [S1].
- **Probability Array:** An array $P$ of size $N$, where $P[k]$ stores the probability of termination in interval $k$ [S1].

### Formulas
The method assumes opacity $\tau(s)$ is piecewise linear within each interval $I_j = [s_j, s_{j+1}]$ and color $c(s)$ is piecewise constant [S1].

**1. Opacity Interpolation:**
For $s \in [s_j, s_{j+1}]$, opacity is interpolated linearly between $\tau_j = \tau(s_j)$ and $\tau_{j+1} = \tau(s_{j+1})$ [S1]:
$$
\tau(s) = \frac{1}{s_{j+1} - s_j} \left[ (\tau_{j+1} - \tau_j)s + (s_{j+1}\tau_j - s_j\tau_{j+1}) \right]
$$
Alternatively expressed as:
$$
\tau(s) = \left( \frac{s_{j+1} - s}{s_{j+1} - s_j} \right) \tau_j + \left( \frac{s - s_j}{s_{j+1} - s_j} \right) \tau_{j+1}
$$

**2. Transmittance Calculation:**
Transmittance $T(s_i)$ is the exponential of the negative integral of opacity from the ray origin to $s_i$ [S1]:
$$
T(s_i) = \exp\left( -\int_0^{s_i} \tau(u) \, du \right) = \prod_{k=1}^i \exp\left( -\int_{s_{k-1}}^{s_k} \tau(u) \, du \right)
$$
The integral over interval $k$ is the area of the trapezoid formed by $\tau_{k-1}$ and $\tau_k$:
$$
\int_{s_{k-1}}^{s_k} \tau(u) \, du = \frac{(\tau_k + \tau_{k-1})(s_k - s_{k-1})}{2}
$$
Thus:
$$
T(s_i) = \prod_{k=1}^i \exp\left( -\frac{(\tau_k + \tau_{k-1})(s_k - s_{k-1})}{2} \right)
$$
With $T(s_0) = 1$ [S1].

**3. Interval Probability:**
The probability $P_j$ that the ray terminates in interval $j$ is derived from the exact integral of the radiance equation, which simplifies to the difference in transmittance at the interval boundaries [S1]:
$$
P_j = T(s_j) - T(s_{j+1})
$$
Substituting the transmittance formula:
$$
P_j = T(s_j) \cdot \left( 1 - \exp\left( -\frac{(\tau_{j+1} + \tau_j)(s_{j+1} - s_j)}{2} \right) \right)
$$

**4. Rendering Equation:**
The final color $C(r)$ is the weighted sum of colors over all intervals [S1]:
$$
C(r) = \sum_{j=0}^{N-1} P_j c_j
$$
Where $c_j$ is the color at sample $j$ [S1].

**5. Importance Sampling (Inverse Transform):**
Because $\tau(s)$ is piecewise linear, the CDF $F(s) = 1 - T(s)$ is invertible in closed form. To sample a termination distance $s$ from the distribution $P$:
1. Generate a uniform random number $u \in [0, 1]$.
2. Find the interval $j$ such that $\sum_{k=0}^{j-1} P_k \le u < \sum_{k=0}^{j} P_k$.
3. Solve for $s \in [s_j, s_{j+1}]$ such that $T(s) = T(s_j) - u \cdot (\text{remaining probability mass})$.
   Specifically, if $u$ falls in interval $j$, let $u' = u - \sum_{k=0}^{j-1} P_k$. We need $T(s) = T(s_j) - u'$.
   $$
   T(s_j) \exp\left( -\frac{(\tau_{j+1} + \tau_j)(s - s_j)}{2} \right) = T(s_j) - u'
   $$
   $$
   \exp\left( -\frac{(\tau_{j+1} + \tau_j)(s - s_j)}{2} \right) = 1 - \frac{u'}{T(s_j)}
   $$
   $$
   s = s_j - \frac{2}{\tau_{j+1} + \tau_j} \ln\left( 1 - \frac{u'}{T(s_j)} \right)
   $$
   [G] Guard against division by zero if $\tau_{j+1} + \tau_j \approx 0$; in this case, $s = s_{j+1}$ or handle as transparent. Guard against $\ln$ argument $\le 0$ due to floating point errors; clamp argument to $\epsilon$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_pl_nerf(ray_origin, ray_direction, s_near, s_far, N_samples, get_density_color):
    """
    Renders a single ray using Piecewise Linear Volume Rendering.
    
    Args:
        ray_origin: 3D vector
        ray_direction: 3D vector (normalized)
        s_near: float, near clipping plane
        s_far: float, far clipping plane
        N_samples: int, number of samples along the ray
        get_density_color: function(s) -> (tau, c)
            Returns opacity tau (scalar >= 0) and color c (3D vector) for position s.
            
    Returns:
        color: 3D vector, rendered color
        depth: float, expected depth
    """
    # 1. Sample positions
    # Uniform sampling for coarse stage, or stratified/inverse CDF for fine stage
    # Here we assume uniform positions for the base derivation, 
    # but importance sampling can replace this step.
    s_positions = [s_near + i * (s_far - s_near) / N_samples for i in range(N_samples + 1)]
    
    # 2. Query neural network for opacity and color at each sample point
    # tau[i] corresponds to s_positions[i]
    taus = []
    colors = []
    for s in s_positions:
        tau, c = get_density_color(s)
        taus.append(tau)
        colors.append(c)
        
    # 3. Compute Transmittance T[i]
    # T[0] = 1.0
    # T[i] = T[i-1] * exp(- integral of tau from s[i-1] to s[i])
    # Integral over interval i-1 is trapezoidal area: (tau[i-1] + tau[i]) * (s[i] - s[i-1]) / 2
    
    T = [0.0] * (N_samples + 1)
    T[0] = 1.0
    
    for i in range(1, N_samples + 1):
        s_prev = s_positions[i-1]
        s_curr = s_positions[i]
        tau_prev = taus[i-1]
        tau_curr = taus[i]
        
        delta_s = s_curr - s_prev
        # Avoid division by zero or invalid log later if needed, though here just for integral
        if delta_s <= 0:
            integral_tau = 0.0
        else:
            integral_tau = (tau_prev + tau_curr) * delta_s / 2.0
            
        T[i] = T[i-1] * math.exp(-integral_tau)
        
    # 4. Compute Interval Probabilities P[i]
    # P[i] = T[i] - T[i+1]
    # This is the probability that the ray terminates in the interval [s[i], s[i+1]]
    P = [0.0] * N_samples
    for i in range(N_samples):
        P[i] = T[i] - T[i+1]
        
    # 5. Compute Final Color
    # C = sum(P[i] * c[i])
    final_color = [0.0, 0.0, 0.0]
    for i in range(N_samples):
        for k in range(3):
            final_color[k] += P[i] * colors[i][k]
            
    # 6. Compute Expected Depth (Optional but common for supervision)
    # Depth contribution of interval i: (s[i] + s[i+1])/2 * P[i]
    expected_depth = 0.0
    for i in range(N_samples):
        mid_depth = (s_positions[i] + s_positions[i+1]) / 2.0
        expected_depth += mid_depth * P[i]
        
    return final_color, expected_depth

def sample_from_pl_nerf_distribution(s_near, s_far, N_samples, taus):
    """
    Performs inverse transform sampling to find a termination distance s
    based on the piecewise linear opacity model.
    
    Args:
        s_near, s_far: bounds
        N_samples: number of intervals
        taus: list of opacity values tau[0]...tau[N_samples]
        
    Returns:
        s: float, sampled termination distance
    """
    s_positions = [s_near + i * (s_far - s_near) / N_samples for i in range(N_samples + 1)]
    
    # Precompute Transmittance T
    T = [0.0] * (N_samples + 1)
    T[0] = 1.0
    for i in range(1, N_samples + 1):
        delta_s = s_positions[i] - s_positions[i-1]
        integral_tau = (taus[i-1] + taus[i]) * delta_s / 2.0
        T[i] = T[i-1] * math.exp(-integral_tau)
        
    # Precompute Probabilities P
    P = [0.0] * N_samples
    for i in range(N_samples):
        P[i] = T[i] - T[i+1]
        
    # Generate random number
    u = random.uniform(0, 1)
    
    # Find interval j
    cumulative_p = 0.0
    j = 0
    for i in range(N_samples):
        cumulative_p += P[i]
        if u < cumulative_p:
            j = i
            break
            
    # Inverse transform within interval j
    # We need s in [s[j], s[j+1]] such that T(s) = T[j] - (u - sum(P[0]...P[j-1]))
    # Let u_prime = u - sum(P[0]...P[j-1])
    # T(s) = T[j] * exp( - (tau[j] + tau[j+1]) * (s - s[j]) / 2 )
    # T[j] * exp(...) = T[j] - u_prime
    # exp(...) = 1 - u_prime / T[j]
    
    sum_prev_p = sum(P[:j])
    u_prime = u - sum_prev_p
    
    # Guard against numerical issues
    if T[j] <= 0:
        return s_positions[j+1] # Should not happen if T[0]=1 and tau>=0
        
    ratio = 1.0 - u_prime / T[j]
    
    # Clamp ratio to (0, 1] to avoid log(0) or log(negative)
    if ratio <= 0.0:
        ratio = 1e-7
    if ratio > 1.0:
        ratio = 1.0
        
    tau_sum = taus[j] + taus[j+1]
    
    if tau_sum <= 1e-7:
        # If opacity is effectively zero, sample uniformly or at end
        return s_positions[j+1]
        
    exponent = math.log(ratio)
    # exponent is negative or zero.
    # - (tau_sum * (s - s[j]) / 2) = exponent
    # s - s[j] = -2 * exponent / tau_sum
    # s = s[j] - 2 * exponent / tau_sum
    
    s = s_positions[j] - 2.0 * exponent / tau_sum
    
    # Clamp to interval bounds
    if s < s_positions[j]:
        s = s_positions[j]
    if s > s_positions[j+1]:
        s = s_positions[j+1]
        
    return s
```

### Parameters
- $N_c$: Number of coarse samples. Typical value: 64 or 128 [S1].
- $N_i$: Number of fine samples. Typical value: 64 or 128 [S1].

### Edge Cases and Pitfalls
- **Ray Conflicts and Blurriness:** Piecewise constant opacity causes blurriness along object sides due to ray conflicts from perpendicular and grazing angle views during optimization [S1]. PL-NeRF alleviates this [S1].
- **Internal Structure Loss:** Piecewise constant opacity causes loss of internal structure (e.g., inside a microphone) and instability/noise in textures (e.g., gold specs on green texture) when varying camera-to-scene distance [S1].
- **Numerical Conditioning:** Choosing polynomial degree $d \ge 2$ for piecewise polynomial opacity makes the problem numerically ill-conditioned and difficult to optimize [S1]. Piecewise linear ($d=1$) is stable [S1].
- **Quadrature Instability:** Piecewise constant opacity is unstable with respect to the choice of samples along the ray (quadrature instability) [S1].

## Operating characteristics
**Time and Memory Complexity:**
- Time complexity is linear with respect to the number of samples $N$, $O(N)$, as it involves a single pass to compute transmittance and probabilities, and a final summation [S1].
- Memory complexity is $O(N)$ to store transmittance and probability arrays [S1].

**Measured Performance:**
- On the Blender dataset (8 scenes), Precise PL-NeRF with $N_c=128, N_i=64$ achieves Avg PSNR 31.10, SSIM 0.948, LPIPS 4.39 ($\times 10^{-2}$), outperforming Surrogate ($N_c=64, N_i=128$) which achieves Avg PSNR 30.25, SSIM 0.940, LPIPS 5.50 ($\times 10^{-2}$) [S1].
- On the DTU dataset (15 scenes), PL-NeRF (Linear) achieves Avg PSNR 28.43, SSIM 0.918, LPIPS 7.73 ($\times 10^{-2}$), outperforming Constant (Vanilla) which achieves Avg PSNR 27.96, SSIM 0.909, LPIPS 8.58 ($\times 10^{-2}$) [S1].
- On Blender dataset with 64 coarse and 64 fine samples, PL-NeRF achieves Avg PSNR 30.09, SSIM 0.939, LPIPS 0.056, outperforming Vanilla NeRF which achieves Avg PSNR 29.86, SSIM 0.937, LPIPS 0.059 [S1].
- Geometry extraction error (CD $\times 10^{-3}$) on Blender dataset Avg: PL-NeRF 10.10 vs Vanilla NeRF 10.43 [S1].

**Strengths and Limitations:**
- **Strengths:** Sharper textures, better geometric reconstruction, stronger depth supervision, and alleviation of blurriness from grazing angles compared to Vanilla NeRF [S1]. Provides an invertible CDF for exact importance sampling [S1].
- **Limitations:** Requires computing transmittance products which can underflow for very long rays with high opacity, though this is common to all volume rendering methods [G]. The closed-form solution is specific to linear opacity; higher degrees are ill-conditioned [S1].

**When to Use:**
- Use when high-fidelity texture and geometry reconstruction are required, especially in scenes with grazing angles or internal structures [S1].
- Use when precise importance sampling is needed without surrogate CDF approximations [S1].

**When Not to Use:**
- Not necessary if computational budget is extremely tight and lower quality is acceptable, as Vanilla NeRF is simpler to implement [S1].
- Not suitable if higher-order polynomial opacity approximations are theoretically required, as they are numerically unstable [S1].

## Background
PL-NeRF was introduced in the paper "NeRF Revisited: Fixing Quadrature Instability in Volume Rendering" [S1]. The work addresses the instability of the classical sample-based rendering equation which assumes piecewise constant volume density [S1]. It builds on the standard volume rendering equation [S1] and improves upon prior NeRF works that used surrogate invertible functions for importance sampling due to the non-invertible CDF of piecewise constant opacity [S1]. The method demonstrates that piecewise linear opacity ($d=1$) resolves quadrature instability while maintaining good numerical conditioning, unlike $d=0$ (unstable) or $d \ge 2$ (ill-conditioned) [S1]. The project page is available at [pl-nerf.github.io](https://pl-nerf.github.io) [S1].

## Sources
[S1] NeRF Revisited: Fixing Quadrature Instability in Volume Rendering, https://doi.org/10.52202/075280-1139, fetched 2026-10-26
