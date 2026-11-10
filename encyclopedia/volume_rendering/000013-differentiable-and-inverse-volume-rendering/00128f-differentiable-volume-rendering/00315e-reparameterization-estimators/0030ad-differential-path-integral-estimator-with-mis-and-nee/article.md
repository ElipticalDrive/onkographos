# Differential path integral estimator with MIS and NEE

Also called: Differential NEE estimator, MIS-weighted differential path estimator

## Overview
This method computes the gradient of the rendered image with respect to scene parameters $\theta$ by integrating differential measurement contributions over path space, using Multiple Importance Sampling (MIS) to combine BRDF and emitter sampling, and Next Event Estimation (NEE) to connect paths to emitters [S1]. Unlike standard NEE which only connects paths to emitters at the final vertex, this method allows the differential vertex to be located at any intermediate vertex $i$ or at the emitter connection, combining BRDF and emitter sampling probabilities into a single mixture weight [S1]. It solves the problem of high variance in gradient estimation for scenes with varying roughnesses by explicitly accounting for the boundary component of the gradient integral through differential sampling [S1].

## How it works
**Inputs:**
- Scene parameters $\theta$ (e.g., material properties, light intensities).
- Number of independent samples per pixel $M$.
- Maximum path depth $N$.
- Probability of switching to differential sampling at each vertex $q$ [S1].

**Outputs:**
- Gradient estimate $\nabla_\theta I$ for the pixel radiance [S1].

**Data Structures:**
- Path $x = (x_0, x_1, \dots, x_N)$, where $x_0$ is the camera position and $x_k$ are vertices in space.
- NEE path $x_j x_{e,j}$, where $x_j$ is a vertex on the main path and $x_{e,j}$ is a point on an emitter [S1].
- Differential contribution $g(x, i; \theta)$ for vertex $i$ [S1].

**Steps:**
1. Initialize gradient accumulator $G = 0$.
2. For each sample $m = 1 \dots M$:
    1. Generate a standard path $x^m$ of depth $N$ using BRDF sampling.
    2. Compute the mixture probability $p_{\text{mix}}(x^m)$ using Equation (41) [S1].
    3. Compute differential contributions $g(x^m, i; \theta)$ for all $i = 0 \dots N$.
    4. Add the term $\sum_{i=0}^{N} \frac{g(x^m, i; \theta)}{p_{\text{mix}}(x^m)}$ to $G$ [S1].
    5. For each $j = 1 \dots N-1$:
        1. Generate an NEE path $x^m_j x^m_{e,j}$ connected to an emitter from vertex $x^m_j$ [S1].
        2. Compute the mixture probability $p_{\text{mix}}(x^m, x^m_j x^m_{e,j})$ using Equation (42) [S1].
        3. Compute differential contributions $g(x^m, x^m_j x^m_{e,j}, i; \theta)$ for $i = 0 \dots j$.
        4. Add the term $\sum_{i=0}^{j} \frac{g(x^m, x^m_j x^m_{e,j}, i; \theta)}{p_{\text{mix}}(x^m, x^m_j x^m_{e,j})}$ to $G$ [S1].
3. Return $G / M$ [S1].

**Formulas:**
The rendering integral is defined as [S1]:
$$
I = \int_{S^2} W_e(x_0, \omega_{x_1,x_0}) G(x_0, x_1) L_i(x_0, \omega_{x_1,x_0}) d\omega_{x_1,x_0}
$$
The gradient is the integral of differential contributions [S1]:
$$
\partial_\theta I = \int_{S^2} \left[ \sum_{i=0}^N g(x, i; \theta) \right] dx
$$
The probability of generating a path with NEE connection at $j$ and differential vertex $i$ ($1 \le i < j$) is [S1]:
$$
p_{\text{nee}}(x, x_j x_{e,j}, i) = \frac{\prod_{k=0}^{j-1} p_f(x_k)}{p_f(x_i)} p_{\partial f}(x_i) (1-q)^{i-1} q
$$
The probability of generating the NEE path itself is [S1]:
$$
p_{\text{nee}}(x, x_j x_{e,j}) = \left[ \prod_{k=0}^{j-1} p_f(x_k) \right] p_e(x_j x_{e,j}) (1-q)^{j-1}
$$
The combined probability for a specific differential vertex $i$ in the NEE path is [S1]:
$$
p(x, x_j x_{e,j}, i) = 
\begin{cases} 
p_{\text{nee}}(\dots) \frac{p_f(x_{j-1}x_j x_{e,j})}{p_e(x_j x_{e,j})} & \text{if } 1 \le i < j \\
p_{\text{nee}}(\dots) \frac{p_{\partial f}(x_{j-1}x_j x_{e,j})}{p_e(x_j x_{e,j})} & \text{if } i = j 
\end{cases}
$$
The mixture probability for the standard path is [S1]:
$$
p_{\text{mix}}(x) = p(x) + \sum_{i=1}^{N-1} p(x, i) + p_{\text{nee}}(x, x_{N-1}x_N)
$$
The mixture probability for the NEE path is [S1]:
$$
p_{\text{mix}}(x, x_j x_{e,j}) = p(x, x_j x_{e,j}) + \sum_{i=1}^j p(x, x_j x_{e,j}, i) + p_{\text{nee}}(x, x_j x_{e,j}) + \sum_{i=1}^{j-1} p_{\text{nee}}(x, x_j x_{e,j}, i)
$$
The differential measurement contribution for an NEE path with differential vertex $x_i$ is [S1]:
$$
g(x, x_j x_{e,j}, i; \theta) = \left[ \prod_{k=1}^{j-1} f_k(x) \right] G(x_j, x_{e,j}) \rho_s(x_{j-1}, x_j, x_{e,j}) \cdot \frac{\partial_\theta f_i(x)}{f_i(x)} \cdot L_e(x_{e,j})
$$
The final estimator is [S1]:
$$
\partial_\theta I_{\text{NEE}} = \frac{1}{M} \sum_{m=1}^M \left[ \sum_{i=0}^{N^m} \frac{g(x^m, i; \theta)}{p_{\text{mix}}(x^m)} + \sum_{j=1}^{N^m-1} \sum_{i=0}^j \frac{g(x^m, x^m_j x^m_{e,j}, i; \theta)}{p_{\text{mix}}(x^m, x^m_j x^m_{e,j})} \right]
$$

**Parameters:**
- $q$: Probability of switching to differential sampling at each vertex. Used in geometric series terms $(1-q)^{i-1}q$ and $(1-q)^{j-1}$ [S1].
- $M$: Number of independent samples drawn per pixel [S1].

**Edge Cases and Pitfalls:**
- Precision issues may arise when computing ratios of probabilities if $p_f(x_i)$ is very small [S1].
- Degenerate input occurs if the emitter probability $p_e$ is zero or undefined [S1].
- The method assumes differentiability of the BRDF $f_i$ and the geometry $G$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differential_path_integral_estimator_with_mis_and_nee(scene_params, M, N, q):
    gradient_accumulator = 0.0
    
    for m in range(M):
        # 1. Generate standard path
        path = generate_standard_path(scene_params, N)
        
        # 2. Compute mixture probability for standard path
        p_mix_standard = compute_p_mix_standard(path, N, q)
        
        # 3. Compute and add standard path contributions
        for i in range(len(path)):
            g_val = compute_differential_contribution_standard(path, i, scene_params)
            gradient_accumulator += g_val / p_mix_standard
            
        # 4. Generate NEE paths and compute contributions
        for j in range(1, len(path) - 1):
            # Generate NEE connection from vertex j to emitter
            nee_path = generate_nee_path(path[j], scene_params)
            
            # 5. Compute mixture probability for NEE path
            p_mix_nee = compute_p_mix_nee(path, nee_path, j, N, q)
            
            # 6. Compute and add NEE path contributions
            for i in range(j + 1):
                g_val = compute_differential_contribution_nee(path, nee_path, i, j, scene_params)
                gradient_accumulator += g_val / p_mix_nee
                
    return gradient_accumulator / M

def compute_p_mix_standard(path, N, q):
    # Eq (41)
    p_standard = get_path_probability(path)
    sum_intermediate = 0.0
    for i in range(1, N):
        sum_intermediate += get_path_probability_with_diff_vertex(path, i, q)
    p_nee_final = get_nee_probability(path, N, q)
    return p_standard + sum_intermediate + p_nee_final

def compute_p_mix_nee(path, nee_path, j, N, q):
    # Eq (42)
    p_standard_with_nee = get_path_probability_with_nee(path, nee_path, j)
    sum_intermediate = 0.0
    for i in range(1, j + 1):
        sum_intermediate += get_path_probability_with_nee_and_diff_vertex(path, nee_path, i, j, q)
    p_nee = get_nee_probability(path, nee_path, j, q)
    sum_nee_intermediate = 0.0
    for i in range(1, j):
        sum_nee_intermediate += get_nee_probability_with_diff_vertex(path, nee_path, i, j, q)
    return p_standard_with_nee + sum_intermediate + p_nee + sum_nee_intermediate

def compute_differential_contribution_standard(path, i, scene_params):
    # Based on Eq (36) and expansion
    product_f = 1.0
    for k in range(len(path)):
        if k != i:
            product_f *= get_brdf(path[k], scene_params)
    # Handle boundary terms and geometry terms as per derivation
    return product_f * get_geometry_term(path, i) * get_emission_term(path, i, scene_params)

def compute_differential_contribution_nee(path, nee_path, i, j, scene_params):
    # Eq (43)
    product_f = 1.0
    for k in range(1, j):
        product_f *= get_brdf(path[k], scene_params)
    
    geometry_term = get_geometry_term(path[j], nee_path[1]) # G(x_j, x_e,j)
    rho_s = get_scattering_term(path[j-1], path[j], nee_path[1]) # rho_s(x_{j-1}, x_j, x_e,j)
    
    # Differential part
    f_i = get_brdf(path[i], scene_params)
    df_i = get_derivative_brdf(path[i], scene_params)
    diff_term = df_i / f_i if f_i != 0 else 0.0
    
    emission = get_emission(nee_path[1], scene_params)
    
    return product_f * geometry_term * rho_s * diff_term * emission
```

## Subtopics

### [Differential Vertex Sampling with MIS](6338-differential-vertex-sampling-with-mis.md)
Differential Vertex Sampling with MIS is a method to estimate the gradient of a loss function with respect to scene parameters by importance sampling an intermediate path vertex as the "differential vertex". It solves the problem of high variance in gradient estimation, particularly for specular materials, by combining differential vertex sampling with standard BRDF sampling using Multiple Importance Sampling (MIS). Unlike standard BRDF sampling, which treats all vertices identically, this method explicitly biases the sampling distribution to capture the boundary component of the gradient integral, significantly reducing variance for microfacet and mixture BRDFs. See [Differential Vertex Sampling with MIS](6338-differential-vertex-sampling-with-mis.md).

### [Differential Edge Sampling](6708-differential-edge-sampling.md)
Differential Edge Sampling is a method for computing the gradient of physically-based rendered images with respect to arbitrary scene parameters (camera, lights, materials, geometry) by explicitly sampling visibility discontinuities. It solves the problem that standard Monte Carlo estimators fail to capture derivatives at boundaries where visibility changes abruptly, such as object silhouettes and occlusion edges. Unlike its sibling [Differential Vertex Sampling with MIS](6338-differential-vertex-sampling-with-mis.md), which focuses on vertex-based contributions, this method decomposes the rendering derivative into a smooth interior integral (estimated via automatic differentiation) and a boundary integral (estimated by sampling triangle edges). See [Differential Edge Sampling](6708-differential-edge-sampling.md).

### Vectorized Total Derivative Estimation
This approach effectively shares all paths traced by Algorithm 1 to calculate derivatives with respect to all scene parameters simultaneously [S2, unofficial]. It executes Algorithm 1 once in a vectorized fashion to estimate the total scene derivative $d_\pi L$. It replaces all partial derivatives with their vector-valued total derivative counterparts and performs component-wise calculations between all total-derivative-related quantities [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The method has a time complexity of $O(K^2)$ for each sample path of length $K$ due to the additional side paths needed for NEE and MIS weighting, compared to $O(K)$ for conventional VPT [S2, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** The method shows lower mean variance ratios compared to BRDF sampling in scenes with varying roughnesses [S1].
- **Strengths:** Reduces variance for specular materials and mixture BRDFs by combining BRDF and emitter sampling [S1]. Allows differential vertex at any intermediate vertex or emitter connection [S1].
- **Limitations:** Significantly more expensive than conventional VPT due to side paths [S2, unofficial]. Convergence rate is $O(1/\sqrt{N})$ [S2, unofficial].
- **When to use:** Differentiable rendering, inverse rendering, and joint material/lighting estimation [S1].
- **When not to use:** When computational efficiency is critical and high variance can be tolerated or handled by other means.

## Background
The method originates from the work of T. Su and I. Gkioulekas, "Path Sampling Methods for Differentiable Rendering," published in the Proceedings of the Eurographics Symposium on Rendering (2024) [S1]. It builds upon earlier work on differentiable Monte Carlo ray tracing through edge sampling [LADL18] and inverse path tracing [ALKN19]. The approach is used in differentiable rendering and inverse rendering applications [S1].

## Sources
[S1] Path Sampling Methods for Differentiable Rendering, https://doi.org/10.2312/sr.20241148, fetched 2026-10-29
[S2] PDF A Differential Theory of Radiative Transfer, https://cseweb.ucsd.edu/~ravir/diff_rendering.pdf, fetched 2026-10-29 (unofficial)
