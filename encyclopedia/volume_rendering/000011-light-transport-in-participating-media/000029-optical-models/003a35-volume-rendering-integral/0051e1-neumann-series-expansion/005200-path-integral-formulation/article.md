# Path Integral Formulation

Also called: Generalized Path Space, Path Integral Representation, Path Space Integral

## Overview
The Path Integral Formulation expresses the volume rendering integral as a sum over paths of scattering events, allowing for an arbitrary sequence of 2D surface locations and 3D positions in a participating medium [S1, book]. It transforms the measurement equation into a sum over path lengths $k$, where each term is an integral over a specific path space $\mathcal{P}_k^{\vec{c}}$ [S2, unofficial]. This formulation is a prerequisite for constructing participating medium-aware bidirectional integrators [S1, book].

## How it works
**Inputs and Outputs**
*   **Input:** A scene defined by a surface manifold $\mathcal{M}$ and a participating medium volume $\mathcal{V}$.
*   **Output:** The measured radiance $I_j$ for pixel $j$.

**Data Structures**
*   **Path Type Vector $\vec{c}$:** A binary vector $\vec{c} \in \{0, 1\}^{k+1}$ of length $k+1$, where $k$ is the path length [S2, unofficial]. $\vec{c}_i = 0$ indicates vertex $\vx_i$ is on the surface $\mathcal{M}$, and $\vec{c}_i = 1$ indicates it is in the volume $\mathcal{V}$ [S2, unofficial].
*   **Path Space $\mathcal{P}_k^{\vec{c}}$:** The Cartesian product of domains for each vertex based on $\vec{c}$, defined as $\mathcal{P}_k^{\vec{c}} = \prod_{i=0}^{k} \{ \mathcal{M} \text{ if } \vec{c}_i = 0, \mathcal{V} \text{ if } \vec{c}_i = 1 \}$ [S2, unofficial]. The total path space $\mathcal{P}$ is the union of all path spaces for all lengths and types: $\mathcal{P} = \bigcup_{k=1}^\infty \bigcup_{\vec{c} \in \{0,1\}^{k+1}} \mathcal{P}_k^{\vec{c}}$ [S2, unofficial]. The vertex $\vx_0$ is the camera position, and $\vx_k$ is the light emission point [S2, unofficial]. The formula integrates over the full path space including endpoints as per the source definition [S2, unofficial].

**Generalized Scattering Distribution Function $\bar{f}$**
Combines surface BSDF and volumetric phase function with scattering coefficient [S2, unofficial]:
$$
\bar{f}(\bx' \to \bx \to \bx'') := \begin{cases} 
f_s(\bx' \to \bx \to \bx'') & \bx \in \mathcal{M} \\
\sigma_s(\bx) f_p(\bx' \to \bx \to \bx'') & \bx \in \mathcal{V}
\end{cases}
$$
where $f_s$ is the surface BSDF, $\sigma_s$ is the scattering coefficient, and $f_p$ is the volumetric phase function [S2, unofficial].

**Measure $d\mu$**
The measure for a path $\bar{\vx}_k$ with type $\vec{c}$ mixes area and volume elements [S2, unofficial]:
$$
d\mu_k^{\vec{c}}(\bar{\vx}_k) := \prod_{i=0}^k \begin{cases} 
dA(\vx_i) & \text{if } \vec{c}_i = 0 \\
dV(\vx_i) & \text{if } \vec{c}_i = 1 
\end{cases}
$$
where $dA$ is the surface area element and $dV$ is the volume element [S2, unofficial].

**Measurement Contribution Function $f_j$**
For a path $\bar{\vx}_k = (\vx_0, \vx_1, \dots, \vx_k)$, the integrand is [S2, unofficial]:
$$
f_j(\bar{\vx}_k) = W_e^j(\vx_1 \to \vx_0) G(\vx_1 \to \vx_0) \prod_{i=1}^{k-1} \left[ \bar{f}(\vx_{i+1} \to \vx_i \to \vx_{i-1}) G(\vx_{i+1} \to \vx_i) \right] L_e(\vx_k \to \vx_{k-1})
$$
where $W_e^j$ is the pixel weight function, $G(\cdot \to \cdot)$ is the geometric term including visibility and attenuation, and $L_e$ is the emitted radiance [S2, unofficial]. The product runs over the internal vertices $i=1$ to $k-1$ [S2, unofficial].

**Algorithm**
The total radiance $I_j$ is the sum over all path lengths $k$ and all path types $\vec{c}$ [S2, unofficial]:
$$
I_j = \sum_{k=1}^\infty \sum_{\vec{c} \in \{0,1\}^{k+1}} \int_{\mathcal{P}_k^{\vec{c}}} f_j(\bar{\vx}_k) \, d\mu_k^{\vec{c}}(\bar{\vx}_k)
$$
This can be written as a single integral over the total path space $\mathcal{P}$ [S2, unofficial]:
$$
I_j = \int_{\mathcal{P}} f_j(\bar{\vx}) \, d\mu(\bar{\vx})
$$

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def path_integral_formulation(scene, pixel_index, max_depth):
    """
    Evaluates the path integral formulation for measured radiance I_j.
    Iterates over path lengths k and binary configuration vectors c.
    """
    total_radiance = 0.0
    for k in range(1, max_depth + 1):
        for c_int in range(1 << (k + 1)):
            c = [(c_int >> i) & 1 for i in range(k + 1)]
            
            def evaluate_path(v):
                # v is a tuple (v0, v1, ..., vk)
                contrib = scene.pixel_weight(pixel_index, v[1], v[0]) * scene.geometric_term(v[1], v[0])
                for i in range(1, k):
                    contrib *= scene.generalized_scattering(v[i+1], v[i], v[i-1]) * scene.geometric_term(v[i+1], v[i])
                contrib *= scene.emitted_radiance(v[k], v[k-1])
                return contrib

            def measure_product(v):
                prod = 1.0
                for i in range(k + 1):
                    if c[i] == 0:
                        prod *= scene.area_element(v[i])
                    else:
                        prod *= scene.volume_element(v[i])
                return prod

            # The integral over P_k^c is estimated via Monte Carlo sampling
            # of paths conforming to type c, dividing by the sampling PDF [G].
            # The formulation defines the exact mathematical target.
            pass
    return total_radiance
```

**Edge Cases and Pitfalls**
*   **Infinite Sum:** The sum over $k$ is infinite and must be truncated or terminated via Russian Roulette in practice [S2, unofficial].
*   **Path Type Explosion:** The number of path types $\vec{c}$ grows as $2^{k+1}$, making explicit summation infeasible [S2, unofficial]. Standard path tracing implicitly handles this by sampling the domain directly [S2, unofficial].
*   **Measure Definition:** The measure $d\mu$ mixes area and volume elements, requiring careful PDF matching during sampling [S2, unofficial].
*   **Special Cases:** The case where $\vec{c}_i = 0$ for all $i$ reduces to the standard surface path integral. The case where $\vec{c}_i = 1$ for all internal vertices represents paths entirely within the medium.

## Subtopics

### Surface Path Integral
The special case where $\vec{c}_i = 0$ for all $i$, reducing to the standard surface path integral.

### Volume Path Integral
The special case where $\vec{c}_i = 1$ for all internal vertices, representing paths entirely within the medium.

## Operating characteristics
*   **Complexity:** The formulation is an exact mathematical identity. The complexity lies in the numerical integration via Monte Carlo methods [S2, unofficial]. The number of path types grows exponentially with depth, but standard path tracing samples paths directly without enumerating types [S2, unofficial].
*   **Performance:** Not given in the sources.
*   **Strengths:** Provides a unified framework for handling both surface and volume scattering, enabling bidirectional integrators for participating media [S1, book].
*   **Limitations:** The infinite sum and high-dimensional integrals make direct evaluation impossible, relying on stochastic methods [S2, unofficial].

## Background
The derivation of the medium-aware path integral is attributed to Pauly et al. (2000) and Chapter 3 of Jakob’s Ph.D. thesis (2013) [S1, book]. It extends the surface path integral formulation to include participating media [S1, book].

## Sources
[S1] Light Transport II: Volume Rendering / The Equation of Transfer, https://www.pbr-book.org/3ed-2018/Light_Transport_II_Volume_Rendering/The_Equation_of_Transfer.html, fetched 2026-10-27 (book)
[S2] ボリュームレンダリング方程式(Volume Rendering Equation) 2, https://rayspace.xyz/notes/VLTE2/, fetched 2026-10-30 (unofficial)
