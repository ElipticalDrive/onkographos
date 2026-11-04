# 3D Gaussian Ellipsoid Rendering

Also called: Differentiable Volume Renderer using Gaussian Ellipsoids, Gaussian Ellipsoid Ray Tracing, VoGE, VoGE Ray Tracing with Ellipsoidal Gaussians, VoGE Volume Density Ray Tracing, VoGE Volume Rendering, VoGE: A Differentiable Volume Renderer using Gaussian Ellipsoids for Analysis-by-Synthesis

## Overview
3D Gaussian Ellipsoid Rendering (VoGE) is a differentiable volume rendering method that represents scenes as a sum of 3D Gaussian ellipsoids and computes image synthesis via ray tracing rather than rasterization [S1, unofficial]. It solves the occlusion ambiguity inherent in distance-based blending by accumulating volume density distributions along viewing rays, enabling realistic interaction between overlapping objects [S1, unofficial]. Unlike its sibling [Gaussian Splatting](946-gaussian-splatting.md), which approximates volume rendering through 2D rasterization of projected splats, VoGE replaces rasterization with ray tracing to capture exact volume density distributions for blending [S1, unofficial].

## How it works
**Inputs:** A set of $K$ Gaussian ellipsoids, each defined by a center location $\mathbf{M}_k \in \mathbb{R}^3$ and a symmetric spatial variance matrix $\mathbf{\Sigma}_k \in \mathbb{R}^{3 \times 3}$; a camera configuration defining viewing rays $\mathbf{r}(t)$; and per-kernel attributes (e.g., color $\mathbf{c}_k$) [S1, unofficial].
**Outputs:** A synthesized 2D image where each pixel color is a weighted sum of kernel attributes [S1, unofficial].

**Data Structures:**
Each kernel $k$ stores:
- $\mathbf{M}_k$: Center coordinates $(\mu_{k,x}, \mu_{k,y}, \mu_{k,z})$.
- $\mathbf{\Sigma}_k$: $3 \times 3$ symmetric covariance matrix.
- $\mathbf{c}_k$: Attribute vector (e.g., RGB color).

**Precomputation per Kernel:**
For each kernel $k$, the method precomputes scalars to simplify ray integration:
1.  $\mathbf{V}_k$: Vector from the ray point of closest approach to the center.
2.  $l_k$: The distance along the ray where density is maximized.
3.  $q_k$: Log-density magnitude offset.
4.  $\sigma_k^2$: Variance of the 1D Gaussian projection along the ray direction.

**Numbered Steps:**

1.  **Ray Sampling:** For each pixel, sample a viewing ray $\mathbf{r}(t) = \mathbf{D} t$, where $\mathbf{D}$ is the normalized direction vector and $t$ is the distance from the camera origin [S1, unofficial].

2.  **Kernel Parameter Calculation:** For each kernel $k$ and ray direction $\mathbf{D}$, compute the following scalars [S1, unofficial]:
    *   Calculate the length $l_k$ along the ray that maximizes the density $\rho_k(\mathbf{r}(t))$:
        $$ l_k = \frac{\mathbf{M}_k^T \cdot \mathbf{\Sigma}_k^{-1} \cdot \mathbf{D} + \mathbf{D}^T \cdot \mathbf{\Sigma}_k^{-1} \cdot \mathbf{M}_k}{2 \cdot \mathbf{D}^T \cdot \mathbf{\Sigma}_k^{-1} \cdot \mathbf{D}} $$
    *   Calculate the vector $\mathbf{V}_k$ from the point $l_k \mathbf{D}$ on the ray to the center $\mathbf{M}_k$:
        $$ \mathbf{V}_k = \mathbf{M}_k - l_k \mathbf{D} $$
    *   Compute the precomputed scalar $q_k$ representing the peak density magnitude offset:
        $$ q_k = -0.5 \mathbf{V}_k^T \cdot \mathbf{\Sigma}_k^{-1} \cdot \mathbf{V}_k $$
    *   Compute the precomputed scalar $\sigma_k^2$ representing the variance of the 1D Gaussian along the ray:
        $$ \sigma_k^2 = \mathbf{D}^T \cdot \mathbf{\Sigma}_k^{-1} \cdot \mathbf{D} $$
    *   Note: The condition $\mathbf{V}_k^T \cdot \mathbf{\Sigma}_k^{-1} \cdot \mathbf{D} + \mathbf{D}^T \cdot \mathbf{\Sigma}_k^{-1} \cdot \mathbf{V}_k = 0$ holds by definition of $l_k$, simplifying the exponent [S1, unofficial].

3.  **Volume Density Definition:** The volume density $\rho(\mathbf{X})$ at any location $\mathbf{X}$ is the sum of ellipsoidal Gaussian kernels [S1, unofficial]:
    $$ \rho(\mathbf{X}) = \sum_{k=1}^K \frac{1}{\sqrt{2\pi} \|\mathbf{\Sigma}_k\|_2} \exp\left(-\frac{1}{2} (\mathbf{X} - \mathbf{M}_k)^T \cdot \mathbf{\Sigma}_k^{-1} \cdot (\mathbf{X} - \mathbf{M}_k)\right) $$
    Along the ray $\mathbf{r}(s) = s\mathbf{D}$, the density of the $k$-th kernel reduces to a 1D Gaussian [S1, unofficial]:
    $$ \rho_k(\mathbf{r}(s)) = \frac{e^{q_k}}{\sqrt{2\pi} \|\mathbf{\Sigma}_k\|_2} \exp\left(-\frac{(s - l_k)^2}{2 \sigma_k^2}\right) $$

4.  **Occupancy Accumulation:** Compute the occupancy $T(t)$ along the ray, which represents the probability of not being occluded up to distance $t$. This is derived by integrating the total volume density [S1, unofficial]:
    $$ T(t) = \exp\left(-\tau \int_{-\infty}^t \rho(\mathbf{r}(s)) ds\right) $$
    Substituting the 1D Gaussian form and using the error function $\operatorname{erf}(x) = \frac{2}{\sqrt{\pi}} \int_0^x e^{-u^2} du$, the closed-form solution is [S1, unofficial]:
    $$ T(t) = \exp\left(-\sum_{k=1}^K e^{q_k} \frac{\operatorname{erf}\left((t - l_k)/\sigma_k\right) + 1}{2}\right) $$
    Here, $\tau$ is a scattering coefficient [S1, unofficial]. The integral limits are effectively $-\infty$ to $t$ because the Gaussian support is infinite [S1, unofficial].

5.  **Kernel-to-Pixel Weight Computation:** Compute the weight $W_k$ for each kernel to determine its contribution to the final pixel color. The exact weight is the integral of the reweighted density [S1, unofficial]:
    $$ W_k(t) = \int_{-\infty}^{\infty} T(t) \rho_k(\mathbf{r}(t)) dt $$
    Since the integral of $T(t)$ involving $\operatorname{erf}$ is infeasible analytically, VoGE approximates $T(t)$ as constant $T(l_k)$ over the kernel's support (centered at $l_k$) [S1, unofficial]:
    $$ W_k \approx T(l_k) e^{q_k} \int_{-\infty}^{\infty} \frac{1}{\sqrt{2\pi} \|\mathbf{\Sigma}_k\|_2} \exp\left(-\frac{(t - l_k)^2}{\sigma_k^2}\right) dt $$
    The integral of the normalized Gaussian part equals 1. Thus, the approximate closed-form weight is [S1, unofficial]:
    $$ W_k = T(l_k) e^{q_k} $$
    The error introduced by this approximation converges to a fixed value of 0.0256, which is negligible compared to the integral result $W=1$ [S1, unofficial].

6.  **Image Synthesis:** For each pixel, sum the contributions of all kernels weighted by their computed weights $W_k$ and their attributes $\mathbf{c}_k$ [S1, unofficial]:
    $$ \mathbf{C}_{pixel} = \sum_{k=1}^K W_k \mathbf{c}_k $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_kernel_params(M_k, Sigma_k, D):
    """
    Precompute scalars for a single kernel along a ray direction D.
    M_k: (3,) center vector
    Sigma_k: (3,3) symmetric covariance matrix
    D: (3,) normalized ray direction vector
    Returns: l_k, q_k, sigma_k_sq
    """
    Sigma_inv = np.linalg.inv(Sigma_k)
    
    term1 = np.dot(M_k, np.dot(Sigma_inv, D))
    term2 = np.dot(D, np.dot(Sigma_inv, M_k))
    denominator = 2.0 * np.dot(D, np.dot(Sigma_inv, D))
    
    if denominator == 0:
        # Guard against degenerate covariance projection
        return 0.0, 0.0, 1e-8 
        
    l_k = (term1 + term2) / denominator
    
    V_k = M_k - (l_k * D)
    q_k = -0.5 * np.dot(V_k, np.dot(Sigma_inv, V_k))
    sigma_k_sq = np.dot(D, np.dot(Sigma_inv, D))
    
    return l_k, q_k, sigma_k_sq

def render_voge(kernels, ray_origin, ray_direction, tau=1.0):
    """
    Render a single ray using VoGE.
    kernels: List of dicts with keys 'M', 'Sigma', 'color'
    ray_origin: (3,) camera origin
    ray_direction: (3,) normalized direction
    tau: scattering coefficient
    Returns: pixel_color (3,)
    """
    # Step 1 & 2: Precompute kernel parameters for this ray
    kernel_data = []
    for k in kernels:
        M = k['M']
        Sigma = k['Sigma']
        l_k, q_k, sigma_k_sq = compute_kernel_params(M, Sigma, ray_direction)
        sigma_k = math.sqrt(sigma_k_sq) if sigma_k_sq > 0 else 1e-8
        kernel_data.append({
            'l_k': l_k,
            'q_k': q_k,
            'sigma_k': sigma_k,
            'color': k['color']
        })
    
    total_weight = 0.0
    pixel_color = np.zeros(3)
    
    # Step 4 & 5: Compute Occupancy T(l_k) and Weight W_k for each kernel
    for kd in kernel_data:
        l_k = kd['l_k']
        q_k = kd['q_k']
        sigma_k = kd['sigma_k']
        
        # Compute T(l_k): Occupancy at the peak of kernel k
        # T(t) = exp(-tau * sum_{j} e^{q_j} * (erf((t - l_j)/sigma_j) + 1)/2 )
        density_integral_sum = 0.0
        for other_kd in kernel_data:
            l_other = other_kd['l_k']
            q_other = other_kd['q_k']
            sigma_other = other_kd['sigma_k']
            
            arg = (l_k - l_other) / sigma_other
            erf_val = math.erf(arg)
            term = math.exp(q_other) * (erf_val + 1.0) / 2.0
            density_integral_sum += term
            
        T_lk = math.exp(-tau * density_integral_sum)
        
        # Step 5: Compute Weight W_k
        W_k = T_lk * math.exp(q_k)
        
        # Step 6: Accumulate Color
        pixel_color += W_k * kd['color']
        total_weight += W_k
        
    if total_weight > 0:
        pixel_color /= total_weight
        
    return pixel_color
```

**Parameters:**
- $\tau$: Scattering coefficient. Typical value: 1.0 [S1, unofficial].
- $K$: Total number of Gaussian kernels. Varies by scene complexity [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Distance-based blending ambiguity:** Traditional splatting blends based on viewing distance, causing unrealistic color shifts when foreground objects move relative to background objects (e.g., purple blending) [S1, unofficial]. VoGE avoids this by using density-based blending [S1, unofficial].
- **Integral infeasibility:** The exact integral of $T(t)$ involving the error function is analytically intractable. The approximation $W_k \approx T(l_k)e^{q_k}$ is used. The error is bounded and negligible (0.0256) [S1, unofficial].
- **Degenerate Covariance:** If $\mathbf{D}^T \cdot \mathbf{\Sigma}_k^{-1} \cdot \mathbf{D}$ is zero or near-zero, $l_k$ is undefined. This occurs if the ray direction is orthogonal to the principal axes in a way that makes the projection singular. Guard against division by zero [G].

## Subtopics

### [Differentiable Ellipsoid Primitive](3490-differentiable-ellipsoid-primitive.md)
The Differentiable Ellipsoid Primitive replaces long-tailed Gaussian kernels with constant-density ellipsoidal shapes to reduce spatial overlap and memory usage. It addresses the non-differentiability of hard-boundary ellipsoids by employing a sigmoid-based formulation, allowing for stable gradient-based optimization against 3D intensity fields. Unlike its sibling [Ever Rendering](3814-ever-rendering.md), which relies on ray-based accumulation to implicitly soften boundaries, this method ensures the primitive itself is fully differentiable, making it suitable for direct volumetric training.

### [Ever Rendering](3814-ever-rendering.md)
Ever Rendering is a real-time volumetric rendering method that represents a radiance field as a set of constant-density volumetric ellipsoids, avoiding the 2D billboard approximation of 3D Gaussian Splatting (3DGS). Unlike its sibling [Differentiable Ellipsoid Primitive](3490-differentiable-ellipsoid-primitive.md), it replaces rasterization with ray tracing to capture volume density distributions for blending, enabling exact transmittance calculation without approximation errors.

### Gaussian Point Representation
Each point is represented by a 3D Gaussian ellipsoid $G_V$ to facilitate general grids; it becomes a sphere for cubic grids [S2, unofficial]. The volume-space Gaussian is defined as $G_V = \frac{1}{2\pi V} e^{-0.5(x - v_j)^T V^{-1} (x - v_j)}$ [S2, unofficial]. This ellipsoid is transformed into view-space $G_{MVM}$ using the viewing matrix $M$ [S2, unofficial].

## Operating characteristics
**Time Complexity:** VoGE provides real-time level rendering speed, competitive with PyTorch3D [S1, unofficial]. The complexity is dominated by the per-ray, per-kernel evaluation of the error function and exponential terms. With $N$ pixels and $K$ kernels, the complexity is roughly $O(N \cdot K)$. The use of closed-form approximations avoids expensive numerical integration [S1, unofficial].

**Memory Complexity:** Memory usage is proportional to the number of kernels $K$, storing $\mathbf{M}_k$, $\mathbf{\Sigma}_k$, and attributes for each [S1, unofficial].

**Strengths:**
- Resolves occlusion ambiguities better than distance-based blending [S1, unofficial].
- Fully differentiable, suitable for analysis-by-synthesis tasks like pose estimation and texture fitting [S1, unofficial].
- Real-time performance achievable via CUDA implementation [S1, unofficial].

**Limitations:**
- Approximation error in weight calculation (though negligible) [S1, unofficial].

**When to use:** When accurate occlusion reasoning and differentiability are required, such as in object pose estimation, shape/texture fitting, and scenes with significant overlapping objects [S1, unofficial].

**When not to use:** When maximum frame rate is critical and occlusion accuracy is less important than speed, or when the scene can be well-approximated by 2D billboards (standard 3DGS) [S1, unofficial].

## Background
VoGE (Volume Gaussian Ellipsoid) was proposed as a differentiable volume renderer that utilizes volumetric Gaussian reconstruction kernels as geometric primitives [S1, unofficial]. It was developed to address the limitations of current state-of-the-art differentiable renderers that use rasterization and distance-based blending, which suffer from ambiguity when objects overlap [S1, unofficial]. The code is available at https://github.com/Angtian/VoGE [S1, unofficial].

## Sources
[S1] VoGE: A Differentiable Volume Renderer using Gaussian Ellipsoids for Analysis-by-Synthesis, https://doi.org/10.48550/arxiv.2205.15401, fetched 2026-10-26 (unofficial)
[S2] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
