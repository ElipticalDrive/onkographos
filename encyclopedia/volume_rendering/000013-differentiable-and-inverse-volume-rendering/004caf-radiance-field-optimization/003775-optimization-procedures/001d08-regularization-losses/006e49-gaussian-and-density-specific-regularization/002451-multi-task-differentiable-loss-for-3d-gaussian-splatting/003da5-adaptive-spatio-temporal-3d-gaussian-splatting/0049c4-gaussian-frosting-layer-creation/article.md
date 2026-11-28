# Gaussian Frosting Layer Creation

Also called: Frosting Layer, SuGaR Frosting

## Overview
Gaussian Frosting Layer Creation defines a volumetric layer around a base surface mesh extracted from 3D Gaussian Splatting to represent fuzzy or translucent materials (e.g., hair, grass) that standard surface extraction misses [S1].

## How it works
**Inputs:** Unconstrained Gaussians $\mathcal{G}_u$, regularized Gaussians $\mathcal{G}_r$, and a base mesh $M$ with vertices $v_i$ and normals $n_i$ [S1].
**Outputs:** Inner and outer shift values $\delta^{in}_i$ and $\delta^{out}_i$ for each vertex $v_i$ [S1].
**Data Structures:**
- Gaussian density function $d(p)$ for any point $p \in \mathbb{R}^3$: $d(p) = \sum_{g} \alpha_g \exp \left( -\frac{1}{2}(p - \mu_g)^T \Sigma_g^{-1} (p - \mu_g) \right)$, where $\alpha_g$ is opacity, $\mu_g$ is mean, and $\Sigma_g$ is covariance [S1].
- Initial interval $I_i = [-3\sigma_i, 3\sigma_i]$, where $\sigma_i$ is the standard deviation in direction $n_i$ of the regularized Gaussian closest to $v_i$ [S1].
- Refined interval $J_i = [\epsilon^{mid}_i - k\epsilon^{half}_i, \epsilon^{mid}_i + k\epsilon^{half}_i]$, with $\epsilon^{mid}_i = (\epsilon^{in}_i + \epsilon^{out}_i)/2$ and $\epsilon^{half}_i = (\epsilon^{out}_i - \epsilon^{in}_i)/2$ [S1].
**Parameters:** Isosurface level $\lambda = 0.01$ [S1]. Expansion factor $k = 3$ [S1].
**Steps:**
1. Optimize unconstrained Gaussian Splatting for a short period [S1].
2. Save the result as $\mathcal{G}_u$ [S1].
3. Apply SuGaR regularization to enforce surface alignment, producing $\mathcal{G}_r$ [S1].
4. Extract base mesh $M$ from $\mathcal{G}_r$ via Poisson reconstruction [S1].
5. For each vertex $v_i$ with normal $n_i$:
   a. Find the regularized Gaussian $g_r \in \mathcal{G}_r$ closest to $v_i$ and compute $\sigma_i$ along $n_i$ [S1].
   b. Define $I_i = [-3\sigma_i, 3\sigma_i]$ [S1].
   c. Find proposal shifts by evaluating $d_r$ along $n_i$ within $I_i$: $\epsilon^{in}_i = \inf(T)$, $\epsilon^{out}_i = \sup(T)$, where $T = \{ t \in I_i \mid d_r(v_i + t n_i) \geq \lambda \}$ [S1].
   d. Define $J_i$ using $\epsilon^{mid}_i$ and $\epsilon^{half}_i$ [S1].
   e. Find final shifts by evaluating $d_u$ along $n_i$ within $J_i$: $\delta^{in}_i = \inf(V)$, $\delta^{out}_i = \sup(V)$, where $V = \{ t \in J_i \mid d_u(v_i + t n_i) \geq \lambda \}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def inv_3x3(M):
    """Computes the inverse of a 3x3 matrix M."""
    a, b, c = M[0][0], M[0][1], M[0][2]
    d, e, f = M[1][0], M[1][1], M[1][2]
    g, h, i = M[2][0], M[2][1], M[2][2]
    det = a*(e*i - f*h) - b*(d*i - f*g) + c*(d*h - e*g)
    if abs(det) < 1e-9:
        return [[1.0, 0.0, 0.0], [0.0, 1.0, 0.0], [0.0, 0.0, 1.0]]
    inv_det = 1.0 / det
    return [
        [(e*i - f*h)*inv_det, (c*h - b*i)*inv_det, (b*f - c*e)*inv_det],
        [(f*g - d*i)*inv_det, (a*i - c*g)*inv_det, (c*d - a*f)*inv_det],
        [(d*h - e*g)*inv_det, (b*g - a*h)*inv_det, (a*e - b*d)*inv_det]
    ]

def compute_density(point, gaussians):
    """Evaluates the Gaussian Splatting density at a point."""
    density = 0.0
    for g in gaussians:
        diff = [point[0]-g.mu[0], point[1]-g.mu[1], point[2]-g.mu[2]]
        inv_S = inv_3x3(g.sigma)
        quad = (diff[0]*(inv_S[0][0]*diff[0] + inv_S[0][1]*diff[1] + inv_S[0][2]*diff[2]) +
                diff[1]*(inv_S[1][0]*diff[0] + inv_S[1][1]*diff[1] + inv_S[1][2]*diff[2]) +
                diff[2]*(inv_S[2][0]*diff[0] + inv_S[2][1]*diff[1] + inv_S[2][2]*diff[2]))
        density += g.alpha * math.exp(-0.5 * quad)
    return density

def compute_frosting_shifts(mesh_vertices, mesh_normals, gaussians_unconstrained, gaussians_regularized, lambda_val=0.01, k=3):
    shifts_in = []
    shifts_out = []
    step = 0.001  # Implementation-dependent sampling step [G]
    
    for v_i, n_i in zip(mesh_vertices, mesh_normals):
        # Step 5a: Find closest regularized Gaussian and compute sigma_i
        min_dist = float('inf')
        closest_g_r = None
        for g in gaussians_regularized:
            diff = [v_i[0]-g.mu[0], v_i[1]-g.mu[1], v_i[2]-g.mu[2]]
            dist = diff[0]**2 + diff[1]**2 + diff[2]**2
            if dist < min_dist:
                min_dist = dist
                closest_g_r = g
        
        S = closest_g_r.sigma
        sigma_i = (n_i[0]**2 * S[0][0] + 2 * n_i[0] * n_i[1] * S[0][1] + n_i[1]**2 * S[1][1] +
                   2 * n_i[0] * n_i[2] * S[0][2] + 2 * n_i[1] * n_i[2] * S[1][2] + n_i[2]**2 * S[2][2]) ** 0.5
        I_i_min, I_i_max = -3 * sigma_i, 3 * sigma_i

        # Step 5c: Find proposal shifts from regularized density
        T = []
        t = I_i_min
        while t <= I_i_max:
            point = [v_i[0] + t * n_i[0], v_i[1] + t * n_i[1], v_i[2] + t * n_i[2]]
            density_r = compute_density(point, gaussians_regularized)
            if density_r >= lambda_val:
                T.append(t)
            t += step

        if not T:
            # The sources do not specify behavior when T is empty [G].
            epsilon_in_i, epsilon_out_i = 0.0, 0.0
        else:
            epsilon_in_i, epsilon_out_i = min(T), max(T)

        # Step 5d: Define refined interval J_i
        epsilon_mid_i = (epsilon_in_i + epsilon_out_i) / 2
        epsilon_half_i = (epsilon_out_i - epsilon_in_i) / 2
        J_i_min = epsilon_mid_i - k * epsilon_half_i
        J_i_max = epsilon_mid_i + k * epsilon_half_i

        # Step 5e: Find final shifts from unconstrained density
        V = []
        t = J_i_min
        while t <= J_i_max:
            point = [v_i[0] + t * n_i[0], v_i[1] + t * n_i[1], v_i[2] + t * n_i[2]]
            density_u = compute_density(point, gaussians_unconstrained)
            if density_u >= lambda_val:
                V.append(t)
            t += step

        if not V:
            # The sources do not specify behavior when V is empty [G].
            delta_in_i, delta_out_i = 0.0, 0.0
        else:
            delta_in_i, delta_out_i = min(V), max(V)

        shifts_in.append(delta_in_i)
        shifts_out.append(delta_out_i)
    return shifts_in, shifts_out
```

**Edge Cases and Pitfalls:**
- The sources do not specify behavior when sets $T$ or $V$ are empty [G].
- Precision near $\lambda$ may require careful sampling or root-finding [G].
- The initial interval $I_i$ depends on the closest regularized Gaussian; if none is close, $\sigma_i$ may not represent the local surface thickness accurately [G].

## Subtopics

### [Frosting Optimization and Parameterization](12686-frosting-optimization-and-parameterization.md)
Defines how Gaussians are constrained within prismatic cells formed by the inner and outer bounds during optimization.

## Operating characteristics
- **Time Complexity:** Evaluating density is $O(N_g)$ per point [G]. Searching intervals requires sampling or root-finding [G]. Overall complexity is $O(N_v \cdot N_g \cdot N_s)$ where $N_v$ is vertices, $N_g$ Gaussians, and $N_s$ samples per interval [G].
- **Memory Complexity:** Stores $\mathcal{G}_u$, $\mathcal{G}_r$, $M$, and shifts [S1].
- **Measured Performance:** Not given in the sources [S1].
- **Scaling:** Linear in vertices and Gaussians per vertex evaluation [G].
- **Strengths:** Adapts to scene complexity; thicker in fuzzy areas, thinner in flat areas [S1].
- **Limitations:** Relies on base mesh quality and regularization alignment [S1].
- **When to use:** Fuzzy or translucent materials requiring volumetric thickness [S1].
- **When not to use:** Well-defined flat surfaces where a thin layer suffices [S1].

## Background
Introduced by A. Guédon and V. Lepetit in "Gaussian Frosting: Editable Complex Radiance Fields with Real-Time Rendering" [S1]. Builds on SuGaR for surface extraction [S1].

## Sources
[S1] Gaussian Frosting: Editable Complex Radiance Fields with Real-Time Rendering, https://doi.org/10.1007/978-3-031-73650-6_24, fetched 2026-10-26
