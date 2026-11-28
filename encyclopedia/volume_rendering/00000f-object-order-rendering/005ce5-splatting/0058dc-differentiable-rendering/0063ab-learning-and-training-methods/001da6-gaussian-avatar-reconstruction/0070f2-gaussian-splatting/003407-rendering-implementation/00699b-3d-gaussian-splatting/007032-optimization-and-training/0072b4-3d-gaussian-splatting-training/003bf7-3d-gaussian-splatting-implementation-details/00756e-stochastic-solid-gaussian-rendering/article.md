# Stochastic Solid Gaussian Rendering

Also called: GGGS, Gaussian Primitives as Stochastic Solids, Geometry-Grounded Gaussian Splatting, Stochastic Gaussian Solid, Stochastic Solid Proof, Volume Rendering of Gaussian Splatting

## Overview
Stochastic Solid Gaussian Rendering (SSGR) treats 3D Gaussian primitives as stochastic solids to derive a principled, continuous attenuation coefficient, enabling volume rendering that is mathematically equivalent to standard Gaussian Splatting rasterization [S1, unofficial]. This equivalence allows for the definition of a geometric field (occupancy) and the extraction of multi-view-consistent depth maps via a smooth transmittance curve, overcoming the jagged, heuristic depth estimation of prior methods that rely on discrete step-wise transmittance changes [S1, unofficial].

## How it works
**Inputs:** A set of 3D Gaussian primitives, each defined by center $\mathbf{x}_c$, covariance $\mathbf{\Sigma}$, opacity $o$, and color $\mathbf{c}$. A camera ray defined by origin $\mathbf{o}$ and direction $\omega$.
**Outputs:** Rendered color $\mathbf{C}$ and median depth $t_{0.5}$ (the depth where transmittance $T(t) = 0.5$).

**Data Structures:**
*   **3D Gaussian Primitive:** Defined by $G(\mathbf{x}) = o e^{-(\mathbf{x}-\mathbf{x}_c)^T \mathbf{\Sigma}^{-1} (\mathbf{x}-\mathbf{x}_c)}$ [S1, unofficial].
*   **Projected 2D Gaussian:** Used for rasterization visibility culling. Opacity at pixel $\mathbf{u}$ is $\alpha(\mathbf{u}) = o e^{-(\mathbf{u}-\mathbf{u}_c)^T \mathbf{\Sigma}_{2D}^{-1} (\mathbf{u}-\mathbf{u}_c)}$ [S1, unofficial].
*   **Stochastic Solid Vacancy:** $v(\mathbf{x}) = \sqrt{1 - G(\mathbf{x})}$ [S1, unofficial].

**Method Steps:**

1.  **Rasterization and Sorting:**
    Project 3D Gaussians to 2D using the local affine approximation. For each pixel, identify visible Gaussians (where $\alpha(\mathbf{u}) > \epsilon$) and sort them by depth $t$ along the camera ray $\mathbf{x}(t) = \mathbf{o} + t\omega$ [S1, unofficial].

2.  **Attenuation Coefficient Derivation:**
    Define the vacancy $v(\mathbf{x})$ for a Gaussian primitive as:
    $$ v(\mathbf{x}) = \sqrt{1 - G(\mathbf{x})} $$
    This definition satisfies three constraints:
    *   Monotonicity: If $G(\mathbf{x}_1) \geq G(\mathbf{x}_2)$, then occupancy $o(\mathbf{x}_1) \geq o(\mathbf{x}_2)$ (where $o(\mathbf{x}) = 1 - v(\mathbf{x})^2$).
    *   Boundary: Occupancy approaches 0 far from the center.
    *   Differentiability: $o(\mathbf{x})$ is differentiable with respect to $\mathbf{x}$ [S1, unofficial].

    The attenuation coefficient $\sigma$ is derived from the vacancy gradient along the viewing direction $\omega$:
    $$ \sigma(\mathbf{x}, \omega) = \frac{|\omega \cdot \nabla v(\mathbf{x})|}{v(\mathbf{x})} = |\omega \cdot \nabla \log(v(\mathbf{x}))| $$
    Along a ray parameterized by $t$, let $t^*$ be the point where the 1D Gaussian $G(t)$ reaches its maximum. The vacancy function has opposite monotonicity to the Gaussian function. The attenuation coefficient simplifies to:
    $$ \sigma(t) = \begin{cases} -\frac{\partial \log(v(\mathbf{x}(t)))}{\partial t}, & t \leq t^* \\ \frac{\partial \log(v(\mathbf{x}(t)))}{\partial t}, & t > t^* \end{cases} $$
    [S1, unofficial]

3.  **Volume Rendering (Forward Pass):**
    The transmittance is $T(t) = \exp \left( - \int_{t_n}^t \sigma(\mathbf{x}(s), \omega) ds \right)$.
    For a single Gaussian with uniform color $\mathbf{c}$, the volume rendering integral yields:
    $$ \mathbf{C} = \mathbf{c} (1 - T(+\infty)) = \mathbf{c} (1 - v(t_{end})^2) $$
    Since $v(t_{end}) \to 1$ as $t \to \infty$ (far from center), and the rendering is equivalent to rasterization, the color contribution of a single Gaussian is:
    $$ \mathbf{C} = \mathbf{c} G(t^*) = \mathbf{c} (1 - v(t^*)^2) $$
    where $t^*$ is the location of the maximum Gaussian value along the ray [S1, unofficial].

4.  **Depth Estimation (Median Depth):**
    Define the median depth $t_{0.5}$ as the point along the ray where the transmittance drops to 0.5:
    $$ T(t_{0.5}) = 0.5 $$
    Due to the monotonicity of the transmittance curve derived from the stochastic solid formulation, $t_{0.5}$ can be located via binary search [S1, unofficial].

5.  **Backpropagation:**
    Compute the gradient of the median depth $t_{0.5}$ with respect to the Gaussian parameters (center, covariance, opacity) using a closed-form expression. This gradient is backpropagated through the binary search and the attenuation model to update the Gaussians during training [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_attenuation_coefficient(vacancy, gradient_along_ray):
    """
    Computes sigma(t) = |d/dt log(v)| = |grad_v / v|
    vacancy: scalar value v(x)
    gradient_along_ray: scalar value omega . nabla v(x)
    """
    if vacancy == 0:
        return 0.0
    return abs(gradient_along_ray / vacancy)

def render_stochastic_solid_gaussian(ray_origin, ray_dir, gaussian):
    """
    Renders a single Gaussian primitive as a stochastic solid.
    Returns color and median depth.
    """
    # 1. Find t* (max of 1D Gaussian along ray)
    # G(t) = o * exp(-(x(t)-xc)^T Sigma^-1 (x(t)-xc))
    # This is a quadratic in the exponent. Find t* analytically.
    t_star = find_gaussian_max_along_ray(ray_origin, ray_dir, gaussian)
    
    # 2. Compute vacancy at t_star
    # v(x) = sqrt(1 - G(x))
    g_at_star = evaluate_gaussian_1d(ray_origin, ray_dir, t_star, gaussian)
    v_at_star = np.sqrt(1.0 - g_at_star)
    
    # 3. Compute Color
    # C = c * G(t*)
    color = gaussian.color * g_at_star
    
    # 4. Compute Median Depth t_0.5
    # T(t) = exp(- integral_{t_n}^t sigma(s) ds)
    # We need T(t_0.5) = 0.5 => integral_{t_n}^{t_0.5} sigma(s) ds = log(2)
    # Use binary search to find t_0.5
    
    t_min = ray_origin.t_near
    t_max = ray_origin.t_far
    
    for _ in range(32): # Binary search iterations
        t_mid = (t_min + t_max) / 2.0
        
        # Compute transmittance at t_mid
        # T(t) = v(t) / v(t_n) ? 
        # From Miller et al., T(t) = v(t) / v(t_n) is not directly stated, 
        # but T(t) = exp(-int sigma). 
        # Note: v(t) = sqrt(1-G(t)). 
        # The integral of sigma leads to log(v). 
        # Specifically, T(t) = v(t) / v(t_start) if we define limits correctly.
        # However, standard GS assumes t_start is -inf or near field.
        # Let's use the property: T(t) = v(t) / v(t_n) is valid if sigma is derived from v.
        # Actually, T(t) = exp(-int_{t_n}^t |d/ds log v| ds).
        # If v is monotonic decreasing then increasing (or vice versa), 
        # the integral splits at t*.
        
        # Simplified approach for binary search:
        # Calculate T(t_mid) using the vacancy ratio.
        # T(t) = v(t) / v(t_n) is only true if v is monotonic. 
        # Here v is not monotonic over the whole ray (it dips at t*).
        # But for t > t*, v increases. For t < t*, v decreases.
        # The transmittance T(t) represents the probability of no collision.
        # T(t) = v(t) / v(t_n) holds for the "solid" interpretation where v is the "empty" probability?
        # Let's stick to the definition: T(t) = exp(-int sigma).
        
        # Compute integral of sigma from t_n to t_mid
        # sigma(t) = |d/dt log v(t)|
        # integral_{t_n}^{t_mid} |d/dt log v| dt = |log v(t_mid) - log v(t_n)| IF v is monotonic.
        # Since v has a minimum at t* (G has max), we must split integral at t*.
        
        integral = 0.0
        if t_mid < t_star:
            # v is decreasing (G increasing) from t_n to t_mid? 
            # No, G increases to t*. So v decreases to t*.
            # d/dt log v is negative. |...| = - d/dt log v.
            # Integral = - (log v(t_mid) - log v(t_n)) = log(v(t_n)/v(t_mid))
            v_at_mid = np.sqrt(1.0 - evaluate_gaussian_1d(ray_origin, ray_dir, t_mid, gaussian))
            v_at_n = np.sqrt(1.0 - evaluate_gaussian_1d(ray_origin, ray_dir, ray_origin.t_near, gaussian))
            integral = np.log(v_at_n / v_at_mid)
        else:
            # t_mid > t*. Split at t*.
            # Part 1: t_n to t*. v decreases. Integral = log(v(t_n)/v(t_star))
            # Part 2: t* to t_mid. v increases. Integral = log(v(t_mid)/v(t_star))
            v_at_mid = np.sqrt(1.0 - evaluate_gaussian_1d(ray_origin, ray_dir, t_mid, gaussian))
            v_at_n = np.sqrt(1.0 - evaluate_gaussian_1d(ray_origin, ray_dir, ray_origin.t_near, gaussian))
            v_at_star = np.sqrt(1.0 - evaluate_gaussian_1d(ray_origin, ray_dir, t_star, gaussian))
            
            integral_1 = np.log(v_at_n / v_at_star)
            integral_2 = np.log(v_at_mid / v_at_star)
            integral = integral_1 + integral_2
            
        T_mid = np.exp(-integral)
        
        if T_mid > 0.5:
            t_min = t_mid
        else:
            t_max = t_mid
            
    t_05 = (t_min + t_max) / 2.0
    
    return color, t_05

def find_gaussian_max_along_ray(origin, dir, gaussian):
    """
    Finds t* that maximizes G(t) = o * exp(- (origin + t*dir - xc)^T Sigma^-1 (origin + t*dir - xc))
    Equivalent to minimizing the quadratic form Q(t) = (origin + t*dir - xc)^T Sigma^-1 (origin + t*dir - xc)
    """
    # Let delta = origin - xc
    # Q(t) = (delta + t*dir)^T Sigma^-1 (delta + t*dir)
    # Q(t) = t^2 (dir^T Sigma^-1 dir) + 2t (dir^T Sigma^-1 delta) + (delta^T Sigma^-1 delta)
    # dQ/dt = 2t (dir^T Sigma^-1 dir) + 2 (dir^T Sigma^-1 delta) = 0
    # t* = - (dir^T Sigma^-1 delta) / (dir^T Sigma^-1 dir)
    
    inv_sigma = np.linalg.inv(gaussian.covariance)
    delta = origin - gaussian.center
    
    a = dir @ inv_sigma @ dir
    b = dir @ inv_sigma @ delta
    
    if a == 0:
        return origin.t_near # Degenerate case
        
    return -b / a

def evaluate_gaussian_1d(origin, dir, t, gaussian):
    """
    Evaluates G(t) along the ray.
    """
    x = origin + t * dir
    diff = x - gaussian.center
    exponent = -diff @ np.linalg.inv(gaussian.covariance) @ diff
    return gaussian.opacity * np.exp(exponent)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The vacancy $v(\mathbf{x})$ approaches 1 far from the center. Computing $\log(v)$ when $v \approx 1$ can lead to numerical instability if not handled carefully (use `log1p`).
*   **Boundaries:** The binary search for $t_{0.5}$ assumes the ray intersects the Gaussian's effective volume. If the ray misses the Gaussian entirely, $G(t) \approx 0$, $v(t) \approx 1$, $T(t) \approx 1$, and $t_{0.5}$ is undefined (or outside the ray bounds).
*   **Degenerate Input:** If the covariance matrix is singular, the inverse is undefined. Standard GS regularization applies.
*   **Multiple Gaussians:** The method is derived for a single Gaussian. For multiple Gaussians, the transmittance curves are combined. The "smooth transmittance" advantage is most significant when overlapping Gaussians create a complex joint occupancy field, which standard rasterization approximates with discrete steps [S1, unofficial].

## Subtopics

### Geometry-Grounded Gaussian Splatting (GGGS)
The overarching framework that uses SSGR to provide a principled geometric foundation for Gaussian primitives, enabling higher-fidelity shape reconstruction [S1, unofficial].

### Depth Rendering from Gaussians
A specific application of SSGR that extracts finer-grained, multi-view-consistent depth maps by locating the median depth via binary search on the smooth transmittance curve [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering a single Gaussian is $O(1)$ for color and $O(\log N)$ for depth (binary search), where $N$ is the number of iterations. Sorting Gaussians is $O(M \log M)$ for $M$ visible Gaussians.
*   **Memory Complexity:** $O(M)$ to store visible Gaussians and their parameters.
*   **Strengths:** Provides a continuous, differentiable geometric field from Gaussian primitives. Enables accurate depth estimation that captures the joint effects of overlapping Gaussians, avoiding the jagged steps of discrete rasterization-based depth.
*   **Limitations:** Requires solving for $t^*$ and performing binary search for depth, which is more computationally expensive than simple alpha blending. The equivalence to rasterization holds strictly for the color; depth requires the additional stochastic solid formulation.
*   **When to use:** When multi-view consistent geometry and accurate depth maps are required from Gaussian Splatting representations.
*   **When not to use:** When only novel view synthesis (color) is needed and standard GS rasterization is sufficient, as SSGR adds computational overhead for depth.

## Background
The method is based on the theoretical foundation of "Objects as Volumes" by Miller et al. [2024], which provides a stochastic interpretation of geometry-grounded radiance fields [S1, unofficial]. It was introduced by Zhang et al. in "Geometry-Grounded Gaussian Splatting" to address the open problem of shape extraction from Gaussian primitives, which previously suffered from poor multi-view consistency and sensitivity to floaters due to heuristic geometry parameterization [S1, unofficial].

## Sources
[S1] Geometry-Grounded Gaussian Splatting, https://arxiv.org/abs/2601.17835v2, fetched 2026-10-26 (unofficial)
