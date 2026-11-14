# Path Integral Formulation

Also called: Generalized Path Space, Path Integral Representation, Path Space Integral

## Overview
The Path Integral Formulation expresses the volume rendering integral as a sum over paths of scattering events, allowing for an arbitrary sequence of 2D surface locations and 3D positions in a participating medium [S1, book]. It transforms the measurement equation into a sum over path lengths $k$, where each term is an integral over a specific path space $\mathcal{P}_k^{\vec{c}}$ [S2, unofficial]. This formulation is a prerequisite for constructing participating medium-aware bidirectional integrators [S1, book].

## How it works
**Inputs and Outputs**
*   **Input:** A scene defined by a surface manifold $\mathcal{M}$ and a participating medium volume $\mathcal{V}$.
*   **Output:** The measured radiance $I_j$ for pixel $j$.

**Data Structures**
*   **Path Type Vector $\vec{c}$:** A binary vector $\vec{c} \in \{0, 1\}^{k+1}$ of length $k+1$, where $k$ is the path length (number of segments).
    *   $\vec{c}_i = 0$: Vertex $\vx_i$ is on the surface $\mathcal{M}$.
    *   $\vec{c}_i = 1$: Vertex $\vx_i$ is in the volume $\mathcal{V}$.
*   **Path Space $\mathcal{P}_k^{\vec{c}}$:** The Cartesian product of domains for each vertex based on $\vec{c}$.
    *   $\mathcal{P}_k^{\vec{c}} = \prod_{i=0}^{k} \{ \mathcal{M} \text{ if } \vec{c}_i = 0, \mathcal{V} \text{ if } \vec{c}_i = 1 \}$ [S2, unofficial].
    *   Note: [S1, book] defines the product for $i=1$ to $n$ for paths of length $n$, but [S2, unofficial] uses $i=0$ to $k$ for paths of length $k$. The vertex $\vx_0$ is the camera position (fixed), and $\vx_{k+1}$ is the light source position (fixed or integrated). The integration variables are typically the intermediate vertices. We follow [S2, unofficial] for the explicit measure definition including endpoints in the product notation, but standard path tracing integrates over the free vertices. The formula below integrates over the full path space including endpoints as per the source definition.

**Generalized Scattering Distribution Function $\bar{f}$**
Combines surface BSDF and volumetric phase function:
$$
\bar{f}(\bx' \to \bx \to \bx'') := \begin{cases} 
f_s(\bx' \to \bx \to \bx'') & \bx \in \mathcal{M} \\
\sigma_s(\bx) f_p(\bx' \to \bx \to \bx'') & \bx \in \mathcal{V}
\end{cases}
$$
where $f_s$ is the surface BSDF, $\sigma_s$ is the scattering coefficient, and $f_p$ is the volumetric phase function [S2, unofficial].

**Measure $d\mu$**
The measure for a path $\bar{\vx}_k$ with type $\vec{c}$ is:
$$
d\mu_k^{\vec{c}}(\bar{\vx}_k) := \prod_{i=0}^k \begin{cases} 
dA(\vx_i) & \text{if } \vec{c}_i = 0 \\
dV(\vx_i) & \text{if } \vec{c}_i = 1 
\end{cases}
$$
where $dA$ is the surface area element and $dV$ is the volume element [S2, unofficial].

**Measurement Contribution Function $f_j$**
For a path $\bar{\vx}_k = (\vx_0, \vx_1, \dots, \vx_k)$:
$$
f_j(\bar{\vx}_k) = W_e^j(\vx_1 \to \vx_0) G(\vx_1 \to \vx_0) \prod_{i=1}^{k-1} \left[ \bar{f}(\vx_{i+1} \to \vx_i \to \vx_{i-1}) G(\vx_{i+1} \to \vx_i) \right] L_e(\vx_k \to \vx_{k-1})
$$
where:
*   $W_e^j$ is the pixel weight function.
*   $G(\cdot \to \cdot)$ is the geometric term (including visibility/attenuation).
*   $L_e$ is the emitted radiance.
*   The product runs over the internal vertices $i=1$ to $k-1$ [S2, unofficial].

**Algorithm**
The total radiance $I_j$ is the sum over all path lengths $k$ and all path types $\vec{c}$:
$$
I_j = \sum_{k=1}^\infty \sum_{\vec{c} \in \{0,1\}^{k+1}} \int_{\mathcal{P}_k^{\vec{c}}} f_j(\bar{\vx}_k) \, d\mu_k^{\vec{c}}(\bar{\vx}_k)
$$
This can be written as a single integral over the total path space $\mathcal{P}$:
$$
I_j = \int_{\mathcal{P}} f_j(\bar{\vx}) \, d\mu(\bar{\vx})
$$
where $\mathcal{P} = \bigcup_{k=1}^\infty \bigcup_{\vec{c} \in \{0,1\}^{k+1}} \mathcal{P}_k^{\vec{c}}$ [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def path_integral_formulation(scene, pixel_index):
    """
    Computes the radiance for a pixel using the path integral formulation.
    This is a theoretical decomposition; practical implementation requires
    Monte Carlo integration over the path space.
    """
    total_radiance = 0.0
    
    # Sum over path lengths k from 1 to infinity
    # In practice, this is truncated at a maximum depth
    max_depth = 100
    for k in range(1, max_depth + 1):
        # Sum over all possible path types (surface/volume configurations)
        # There are 2^(k+1) possible types for a path of length k
        # Note: The first vertex v0 is camera (fixed), last vk+1 is light (fixed/integrated)
        # The source defines c for vertices 0 to k. 
        # We iterate over all binary vectors of length k+1.
        
        # Generate all binary vectors of length k+1
        # This is computationally expensive and not done explicitly in standard path tracing,
        # but is the definition of the path space.
        for c in all_binary_vectors(length=k+1):
            
            # Define the path space P_k^c
            # P_k^c = product of M if c_i=0 else V for i in 0..k
            
            # Integrate over this specific path space
            # The integral is over f_j(v_k) d mu_k^c(v_k)
            # This integral is typically estimated via Monte Carlo sampling
            # of paths that conform to the type c.
            
            integral_value = monte_carlo_integrate_path_space(
                scene=scene,
                pixel_index=pixel_index,
                path_length=k,
                path_type=c
            )
            
            total_radiance += integral_value
            
    return total_radiance

def monte_carlo_integrate_path_space(scene, pixel_index, path_length, path_type):
    """
    Estimates the integral over a specific path space P_k^c using Monte Carlo.
    This function is a placeholder for the specific sampling strategy
    (e.g., path tracing, bidirectional path tracing) that samples vertices
    from M or V according to path_type.
    """
    # 1. Sample a path v_0, ..., v_k from the space P_k^path_type
    # v_0 is the camera position.
    # For i in 1..k:
    #   if path_type[i] == 0: sample v_i from surface M
    #   if path_type[i] == 1: sample v_i from volume V
    
    # 2. Compute the contribution f_j(v_k)
    # f_j(v_k) = W_e(v_1->v_0) * G(v_1->v_0) * 
    #            Product_{i=1}^{k-1} [ f_bar(v_{i+1}->v_i->v_{i-1}) * G(v_{i+1}->v_i) ] * 
    #            L_e(v_k->v_{k-1})
    
    # 3. Divide by the PDF of the sampled path to get the estimator
    # Note: The measure d mu includes dA or dV. The PDF must account for the 
    # sampling distribution over these domains.
    
    # Return the estimator value
    return 0.0 # Placeholder
```

**Edge Cases and Pitfalls**
*   **Infinite Sum:** The sum over $k$ is infinite. In practice, it must be truncated or terminated via Russian Roulette.
*   **Path Type Explosion:** The number of path types $\vec{c}$ grows as $2^{k+1}$. Explicitly summing over all types is infeasible. Standard path tracing implicitly handles this by sampling the domain directly.
*   **Measure Definition:** The measure $d\mu$ mixes area and volume elements. Care must be taken in sampling to ensure the PDF matches the measure (i.e., sampling from $dA$ vs $dV$) [S2, unofficial].

## Subtopics

### Surface Path Integral
The special case where $\vec{c}_i = 0$ for all $i$. This reduces to the standard surface path integral. [Link to Surface Path Integral if available, otherwise link to Parent].

### Volume Path Integral
The special case where $\vec{c}_i = 1$ for all $i$ (except endpoints). This represents paths entirely within the medium. [Link to Volume Path Integral if available, otherwise link to Parent].

## Operating characteristics
*   **Complexity:** The formulation itself is an exact mathematical identity. The complexity lies in the numerical integration (Monte Carlo). The number of path types grows exponentially with depth, but standard path tracing samples paths directly without enumerating types.
*   **Performance:** Not given in the sources.
*   **Strengths:** Provides a unified framework for handling both surface and volume scattering, enabling bidirectional integrators for participating media [S1, book].
*   **Limitations:** The infinite sum and high-dimensional integrals make direct evaluation impossible; relies on stochastic methods.

## Background
The derivation of the medium-aware path integral is attributed to Pauly et al. (2000) and Chapter 3 of Jakob’s Ph.D. thesis (2013) [S1, book]. It extends the surface path integral formulation to include participating media [S1, book].

## Sources
[S1] Light Transport II: Volume Rendering / The Equation of Transfer, https://www.pbr-book.org/3ed-2018/Light_Transport_II_Volume_Rendering/The_Equation_of_Transfer.html, fetched 2026-10-27 (book)
[S2] ボリュームレンダリング方程式(Volume Rendering Equation) 2, https://rayspace.xyz/notes/VLTE2/, fetched 2026-10-30 (unofficial)
