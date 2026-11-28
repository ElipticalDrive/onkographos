# Once-More Scattered Next Event Estimation

Also called: OM-NEE, Once-More Next Event Estimation, once-more scattering, extended next event estimation, once-more collided flux estimator, volumetric double scattering estimator

## Overview
Once-More Scattered Next Event Estimation (OM-NEE) is a Monte Carlo path tracing technique that improves direct lighting estimation in participating media by inserting an additional scattering vertex between a light path vertex and the next event source [S1]. It specifically addresses the high variance caused by strongly forward-scattering phase functions (e.g., water, haze) where standard Next Event Estimation (NEE) fails to capture the characteristic "glow" around submerged objects [S1]. Unlike equiangular sampling or polynomial approximations, OM-NEE analytically samples the product of an arbitrary phase function and the two reciprocal squared distance geometry terms, making it efficient for thin, homogeneous media with high mean cosine values [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A light path vertex $\mathbf{x}_0$ (the last vertex before connection), a next event vertex $\mathbf{x}_2$ (on the light source or camera aperture), the scattering coefficient $\mu_s$, the transmittance functions $T(\cdot,\cdot)$, the camera responsivity $W(\mathbf{x}_2)$, and a phase function $f_s$ capable of solid angle sampling.
*   **Output:** An unbiased estimator weight $\hat{I}$ for the contribution of the path segment $\mathbf{x}_0 \leftrightarrow \mathbf{x}_1 \leftrightarrow \mathbf{x}_2$.

**Data Structures**
No complex data structures are required. The method operates on 3D vectors and scalar parameters. The geometry is normalized such that the distance $s = |\mathbf{x}_0 - \mathbf{x}_2| = 1$ for the derivation of sampling distributions, but the final estimator uses the actual distance $s$.

**Parameters**
*   $\mu_s$: Scattering coefficient.
*   $f_s$: Phase function. Typical values for the mean cosine $g$ (or $\bar{\nu}$) in ocean water are $0.85$, while highly forward scattering components can have $g \ge 0.95$ [S1].
*   $\xi$: Uniform random number in $[0, 1]$.

**Algorithm Steps**
1.  **Define Endpoints:** Let $\mathbf{x}_0$ be the current light path vertex and $\mathbf{x}_2$ be the NEE vertex (light source or camera). Calculate the distance $s = |\mathbf{x}_0 - \mathbf{x}_2|$ [S1].
2.  **Sample Phase Function:** Sample an outgoing direction at $\mathbf{x}_1$ using the standard solid angle phase function sampling routine for $f_s$. Store the polar angle $\theta$ between the segments $\mathbf{x}_0 \mathbf{x}_1$ and $\mathbf{x}_1 \mathbf{x}_2$ [S1].
    *   *Note:* If $f_s$ has a backscattering component (e.g., standard Henyey-Greenstein), rejection sampling is required to restrict sampling to the forward hemisphere ($\theta \in [0, \pi/2]$ or similar, depending on definition) [S1].
3.  **Sample Fractional Distance $t$:** Sample a fractional distance $t \in (0, 1)$ along the axis $\mathbf{x}_0 \mathbf{x}_2$ using the conditional PDF $p(t|\theta)$ derived from the geometry.
    *   Compute the intermediate variable $A$:
        $$ A := \cot\theta - \sqrt{\sin^{-2}\theta - (2t-1)^2} $$
        *Correction:* The variable $A$ depends on $t$, so it cannot be computed before sampling $t$. The inverse CDF is used directly.
    *   Use the inverse CDF $P^{-1}(\xi|\theta)$ to sample $t$ from $\xi \sim U(0,1)$:
        $$ t = P^{-1}(\xi|\theta) = \frac{\cos(\theta - \xi\theta) \sin(\xi\theta)}{\sin\theta} $$
    *   *Edge Case:* If $\theta = 0$, use the limit $\lim_{\theta \rightarrow 0} \frac{\sin(\xi\theta)}{\sin\theta} = \xi$, so $t = \xi$ [S1].
4.  **Sample Azimuthal Angle $\phi$:** Sample $\phi$ uniformly from $[0, 2\pi]$ or reuse the azimuthal angle from the phase function sampling if available [S1].
5.  **Reconstruct $\mathbf{x}_1$:** Compute the position of the scattering vertex $\mathbf{x}_1$ using $t$, $\phi$, and the geometry.
    *   First, compute the circumcircle radius $R$ of the triangle $\mathbf{x}_0 \mathbf{x}_1 \mathbf{x}_2$ in the normalized space ($s=1$):
        $$ R(\theta) = \frac{1}{2 \sin\theta} $$
    *   Compute the normalized polar radius $r$ in terms of $t$ and $\theta$:
        $$ r(t, \theta) = \sqrt{\frac{1}{4 \sin^2\theta - (1/2 - t)^2}} - \sqrt{\frac{1}{4 \sin^2\theta - 1/4}} $$
    *   Convert $(t, r, \phi)$ to 3D coordinates. Let $\mathbf{u} = \frac{\mathbf{x}_2 - \mathbf{x}_0}{s}$ be the unit vector from $\mathbf{x}_0$ to $\mathbf{x}_2$. Let $\mathbf{v}$ and $\mathbf{w}$ be orthonormal vectors perpendicular to $\mathbf{u}$ (forming a local basis). The center of the circumcircle in the normalized space is at $\mathbf{c}_{norm} = (t - 0.5)\mathbf{u}$. The vertex $\mathbf{x}_1$ is located at:
        $$ \mathbf{x}_1 = \mathbf{x}_0 + s \left( \mathbf{c}_{norm} + r \cos\phi \, \mathbf{v} + r \sin\phi \, \mathbf{w} \right) $$
        *Note:* The exact construction of $\mathbf{v}$ and $\mathbf{w}$ depends on the coordinate system, but they must form a right-handed system with $\mathbf{u}$ [S1].
6.  **Compute Distances:** Calculate $d_1 = |\mathbf{x}_1 - \mathbf{x}_0|$ and $d_2 = |\mathbf{x}_2 - \mathbf{x}_1|$ [S1].
7.  **Calculate Estimator Weight:**
    *   Compute the constant connection term $f_c$:
        $$ f_c = \cos\theta_0 \cdot \cos\theta_2 \cdot \mu_s \cdot T(\mathbf{x}_0, \mathbf{x}_1) \cdot T(\mathbf{x}_1, \mathbf{x}_2) \cdot W(\mathbf{x}_2) $$
        Where $\theta_0$ and $\theta_2$ are the angles between the surface normals (if any) and the segments $\mathbf{x}_0 \mathbf{x}_1$ and $\mathbf{x}_1 \mathbf{x}_2$ respectively. For pure volume scattering without surface normals, these terms are 1 or handled by the specific rendering equation variant [S1].
    *   The final estimator weight is:
        $$ \hat{I} = f_c \cdot \frac{\theta}{s \sin\theta} $$
    *   *Edge Case:* If $\theta = 0$, the term $\frac{\theta}{\sin\theta}$ approaches 1 [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def sample_omnee(x0, x2, phase_func_sampler, mu_s, T_func, W_func, cos_theta0=1.0, cos_theta2=1.0):
    """
    Samples the once-more scattered next event estimation contribution.
    
    Args:
        x0: numpy array, light path vertex.
        x2: numpy array, NEE vertex (light/camera).
        phase_func_sampler: function that samples direction and returns (direction, theta, phi).
        mu_s: float, scattering coefficient.
        T_func: function returning transmittance between two points.
        W_func: function returning camera responsivity at x2.
        cos_theta0: float, cosine of angle between x0 normal and x0->x1.
        cos_theta2: float, cosine of angle between x2 normal and x2->x1.
        
    Returns:
        float: The estimator weight.
    """
    # 1. Define Endpoints
    s = np.linalg.norm(x2 - x0)
    if s == 0:
        return 0.0
    
    # 2. Sample Phase Function
    # phase_func_sampler returns (direction, theta, phi)
    # theta is the angle between the two segments x0-x1 and x1-x2
    direction, theta, phi = phase_func_sampler()
    
    # Handle backscattering rejection if necessary (implementation dependent on phase_func_sampler)
    # If the sampler returns a backward direction, reject and resample or return 0.
    
    # 3. Sample Fractional Distance t
    xi = np.random.uniform(0.0, 1.0)
    
    # Inverse CDF: t = cos(theta - xi*theta) * sin(xi*theta) / sin(theta)
    # Handle singularity at theta = 0
    if theta == 0.0:
        t = xi
    else:
        sin_theta = math.sin(theta)
        # Avoid division by zero if sin_theta is very close to 0
        if sin_theta < 1e-8:
            t = xi
        else:
            t = (math.cos(theta - xi * theta) * math.sin(xi * theta)) / sin_theta
            
    # 4. Sample Azimuthal Angle phi
    # phi is already provided by the phase function sampler. 
    # If not, sample uniformly: phi = np.random.uniform(0, 2 * math.pi)
    
    # 5. Reconstruct x1
    # Normalized geometry: s=1
    # Unit vector u from x0 to x2
    u = (x2 - x0) / s
    
    # Create orthonormal basis v, w perpendicular to u
    # Pick an arbitrary vector not parallel to u
    if abs(u[0]) < 0.9:
        arbitrary = np.array([1.0, 0.0, 0.0])
    else:
        arbitrary = np.array([0.0, 1.0, 0.0])
        
    v = np.cross(u, arbitrary)
    v = v / np.linalg.norm(v)
    w = np.cross(u, v)
    
    # Compute normalized polar radius r
    # r(t, theta) = sqrt(1 / (4 sin^2 theta - (0.5 - t)^2)) - sqrt(1 / (4 sin^2 theta - 0.25))
    sin_theta_sq = math.sin(theta) ** 2
    denom1 = 4 * sin_theta_sq - (0.5 - t) ** 2
    denom2 = 4 * sin_theta_sq - 0.25
    
    # Check for valid geometry (denominators must be positive)
    if denom1 <= 0 or denom2 <= 0:
        return 0.0
        
    r = math.sqrt(1.0 / denom1) - math.sqrt(1.0 / denom2)
    
    # Center of circumcircle in normalized space
    c_norm = (t - 0.5) * u
    
    # Vertex position in normalized space
    x1_norm = c_norm + r * math.cos(phi) * v + r * math.sin(phi) * w
    
    # Scale back to actual distance s and offset by x0
    x1 = x0 + s * x1_norm
    
    # 6. Compute Distances
    d1 = np.linalg.norm(x1 - x0)
    d2 = np.linalg.norm(x2 - x1)
    
    # 7. Calculate Estimator Weight
    # Transmittance
    T01 = T_func(x0, x1)
    T12 = T_func(x1, x2)
    
    # Responsivity
    W = W_func(x2)
    
    # Constant connection term
    # f_c = cos_theta0 * cos_theta2 * mu_s * T01 * T12 * W
    f_c = cos_theta0 * cos_theta2 * mu_s * T01 * T12 * W
    
    # Weight factor
    if theta == 0.0:
        weight_factor = 1.0
    else:
        weight_factor = theta / (s * math.sin(theta))
        
    return f_c * weight_factor
```

**Subtopics**
*   **Volumetric Path Tracing with MIS and Surface Lighting:** Combines OM-NEE with surface lighting estimators using Multiple Importance Sampling (MIS) to handle both volumetric and surface interactions efficiently. [Link](6625-volumetric-path-tracing-with-mis-and-surface-lighting.md)
*   **Manifold Next Event Estimation Improvement:** OM-NEE is conceptually similar to MNEE, enforcing geometric constraints by sampling the phase function first and then filling the path, but OM-NEE is specific to volumes and does not require the complex manifold traversal of surface MNEE. [Link](8025-manifold-next-event-estimation-improvement.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample. The method involves a constant number of trigonometric operations and vector calculations.
*   **Memory Complexity:** $O(1)$. No additional data structures or tables are required.
*   **Measured Performance:** In unit tests with 4 samples per pixel (spp), OM-NEE shows improved efficiency over [VGGN21] as the mean cosine $g$ increases from 0.0 to 0.99 [S1]. Render times for one progression with one sample per pixel were 310ms for plain light tracing, 430ms for equiangular sampling, and 410ms for OM-NEE [S1].
*   **Strengths:**
    *   Efficiently captures the "glow" around objects in forward-scattering media (e.g., water, haze) where equiangular sampling produces noisy results [S1].
    *   Works with any phase function that can be sampled in solid angle, without requiring tabulation or polynomial approximation [S1].
    *   Lightweight and easy to implement compared to data-driven path guiding methods [S1].
    *   Analytically samples both geometry terms, avoiding the approximation errors of polynomial methods [S1].
*   **Limitations:**
    *   Designed specifically for forward scattering. It must be combined with other estimators (e.g., standard NEE or equiangular sampling) to cover backward scattering contributions [S1].
    *   Disregards transmittance in the sampling scheme, making it suboptimal for isotropic or strongly backward-scattering media [S1].
    *   Does not account for occlusion by other geometry, which may increase variance in complex scenes [S1].
    *   Requires rejection sampling if the phase function has a significant backscattering component (e.g., standard Henyey-Greenstein) [S1].
*   **When to Use:** Thin, homogeneous media with strongly forward-scattering phase functions ($g \ge 0.85$) where the characteristic glow is visually important.
*   **When Not to Use:** Isotropic media, strongly backward-scattering media, or scenes with heavy occlusion where the lack of occlusion handling would cause excessive variance.

## Background
OM-NEE was introduced by Johannes Hanika, Andrea Weidlich, and Marc Droske from Weta Digital, Karlsruhe Institute of Technology, NVIDIA, and Unity [S1]. It was published in the paper "Once‐more scattered next event estimation for volume rendering" [S1]. The method addresses the limitations of previous techniques like Georgiev et al. [GKH*13] (which required tabulation) and Villeneuve et al. [VGGN21] (which used polynomial approximations) by providing an analytic solution for arbitrary phase functions. It is particularly relevant for rendering atmospheric haze and underwater scenes where forward scattering dominates [S1].

## Sources
[S1] Once‐more scattered next event estimation for volume rendering, https://doi.org/10.1111/cgf.14583, fetched 2026-10-26
