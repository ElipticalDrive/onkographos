# Differential path integral estimator with MIS and NEE

Also called: Differential NEE estimator, MIS-weighted differential path estimator

## Overview
This method computes the gradient of a rendered image with respect to scene parameters $\theta$ by integrating differential measurement contributions over path space, using Multiple Importance Sampling (MIS) to combine BRDF and emitter sampling, and Next Event Estimation (NEE) to connect paths to emitters [S1]. It solves the problem of high variance in gradient estimation for scenes with varying material properties (e.g., roughness) by allowing the "differential vertex" (the vertex where the parameter derivative is applied) to be located at any intermediate vertex or at the emitter connection [S1]. Unlike standard NEE, which only connects paths to emitters at the final vertex, this method combines BRDF and emitter sampling probabilities into a single mixture weight, significantly reducing variance for microfacet and mixture BRDFs [S1]. It builds on the reparameterization principles of [Reparameterization Estimators](6499-reparameterization-estimators.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene parameters $\theta$ (e.g., material roughness, light intensity), number of samples $M$, maximum path depth $N$, switching probability $q$.
*   **Outputs:** Gradient estimate $\nabla_\theta I$.

**Data Structures:**
*   **Path $x$:** Sequence of vertices $x_0, x_1, \dots, x_N$. $x_0$ is the camera position. $x_i$ is the $i$-th bounce.
*   **NEE Path $x_j x_{e,j}$:** A path segment where vertex $x_j$ is from the main path, and $x_{e,j}$ is a point sampled on an emitter connected to $x_j$.
*   **Differential Contribution $g(x, i; \theta)$:** The contribution of the path $x$ where the derivative is taken at vertex $i$.

**Formulas:**
The rendering integral is defined as [S1]:
$$
I = \int_{S^2} W_e(x_0, \omega_{x_1,x_0}) G(x_0, x_1) L_i(x_0, \omega_{x_1,x_0}) d\omega_{x_1,x_0}
$$
The derivative of the integral with respect to $\theta$ is expressed as a sum of differential contributions over all vertices $i=0 \dots N$ [S1]:
$$
\partial_\theta I = \int_{S^2} \left[ \sum_{i=0}^N g(x, i; \theta) \right] dx
$$
where $g(x, i; \theta)$ represents the term in the expanded derivative of the path contribution where $\partial_\theta f_i(x)$ is applied [S1].

For NEE paths $x_0 \dots x_j x_{e,j}$, the probability of generating the path with differential vertex $x_i$ ($1 \le i < j$) is [S1]:
$$
p_{\text{nee}}(x, x_j x_{e,j}, i) = \frac{\prod_{k=0}^{j-1} p_f(x_k)}{p_f(x_i)} p_{\partial f}(x_i) (1-q)^{i-1} q
$$
The probability of generating the NEE path itself is [S1]:
$$
p_{\text{nee}}(x, x_j x_{e,j}) = \left[ \prod_{k=0}^{j-1} p_f(x_k) \right] p_e(x_j x_{e,j}) (1-q)^{j-1}
$$
The combined probability $p(x, x_j x_{e,j}, i)$ for the differential vertex $i$ in an NEE path is [S1]:
$$
p(x, x_j x_{e,j}, i) = \begin{cases} p_{\text{nee}}(\dots) \frac{p_f(x_{j-1}x_j x_{e,j})}{p_e(x_j x_{e,j})} & \text{if } 1 \le i < j \\ p_{\text{nee}}(\dots) \frac{p_{\partial f}(x_{j-1}x_j x_{e,j})}{p_e(x_j x_{e,j})} & \text{if } i = j \end{cases}
$$
The mixture probability for a standard path $x$ is [S1]:
$$
p_{\text{mix}}(x) = p(x) + \sum_{i=1}^{N-1} p(x, i) + p_{\text{nee}}(x, x_{N-1}x_N)
$$
The mixture probability for an NEE path $x, x_j x_{e,j}$ is [S1]:
$$
p_{\text{mix}}(x, x_j x_{e,j}) = p(x, x_j x_{e,j}) + \sum_{i=1}^j p(x, x_j x_{e,j}, i) + p_{\text{nee}}(x, x_j x_{e,j}) + \sum_{i=1}^{j-1} p_{\text{nee}}(x, x_j x_{e,j}, i)
$$
The differential measurement contribution for an NEE path with differential vertex $x_i$ is [S1]:
$$
g(x, x_j x_{e,j}, i; \theta) = \left[ \prod_{k=1}^{j-1} f_k(x) \right] G(x_j, x_{e,j}) \rho_s(x_{j-1}, x_j, x_{e,j}) \cdot \frac{\partial_\theta f_i(x)}{f_i(x)} \cdot L_e(x_{e,j})
$$

**Parameters:**
*   $M$: Number of independent samples drawn per pixel.
*   $q$: Probability of switching to differential sampling at each vertex, used in geometric series terms $(1-q)^{i-1}q$ and $(1-q)^{j-1}$ [S1].

**Steps:**
1.  Initialize gradient accumulator $G_{acc} = 0$.
2.  For each sample $m = 1 \dots M$:
    a. Generate a standard path $x$ of depth $N$ using BRDF sampling.
    b. Compute mixture probability $p_{\text{mix}}(x)$ using Eq (41).
    c. Compute differential contributions $g(x, i; \theta)$ for all $i=0 \dots N$.
    d. Add $\sum_{i=0}^N \frac{g(x, i; \theta)}{p_{\text{mix}}(x)}$ to $G_{acc}$.
    e. For each $j = 1 \dots N-1$:
        i. Generate an NEE path $x_j x_{e,j}$ connected to an emitter from vertex $x_j$.
        ii. Compute mixture probability $p_{\text{mix}}(x, x_j x_{e,j})$ using Eq (42).
        iii. Compute differential contributions $g(x, x_j x_{e,j}, i; \theta)$ for $i=0 \dots j$.
        iv. Add $\sum_{i=0}^j \frac{g(x, x_j x_{e,j}, i; \theta)}{p_{\text{mix}}(x, x_j x_{e,j})}$ to $G_{acc}$.
3.  Return $\frac{1}{M} G_{acc}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient(theta, M, N, q):
    grad_acc = 0.0
    for m in range(M):
        # 1. Standard Path Contribution
        path = generate_standard_path(N) # Returns list of vertices x_0...x_N
        p_mix_path = compute_p_mix_standard(path, N, q)
        
        # Sum over all differential vertices i=0..N
        sum_g_standard = 0.0
        for i in range(N + 1):
            g_val = compute_differential_contribution_standard(path, i, theta)
            sum_g_standard += g_val / p_mix_path
        grad_acc += sum_g_standard
        
        # 2. NEE Path Contributions
        for j in range(1, N): # j from 1 to N-1
            # Generate NEE connection from vertex j
            emitter_point = sample_emitter_from_vertex(path[j])
            nee_path_vertices = path[:j+1] + [emitter_point] # x_0...x_j, x_e,j
            
            p_mix_nee = compute_p_mix_nee(path, nee_path_vertices, j, q)
            
            # Sum over differential vertices i=0..j
            sum_g_nee = 0.0
            for i in range(j + 1):
                g_val = compute_differential_contribution_nee(path, nee_path_vertices, i, theta)
                sum_g_nee += g_val / p_mix_nee
            grad_acc += sum_g_nee
            
    return grad_acc / M

def compute_p_mix_standard(path, N, q):
    # Eq (41): p(x) + sum(p(x,i)) + p_nee(x, x_{N-1}x_N)
    # Note: Exact implementation of p(x,i) and p_nee terms requires 
    # tracking the specific sampling probabilities p_f, p_partial_f, p_e.
    # This is a placeholder for the complex probability calculation described in [S1].
    return calculate_mixture_probability_standard(path, N, q)

def compute_p_mix_nee(path, nee_vertices, j, q):
    # Eq (42): p(x, x_j x_e,j) + sum(p(x, x_j x_e,j, i)) + p_nee(...) + sum(p_nee(..., i))
    return calculate_mixture_probability_nee(path, nee_vertices, j, q)

def compute_differential_contribution_standard(path, i, theta):
    # Computes g(x, i; theta)
    # Product of BSDFs f_k for k != i, times derivative of f_i
    # See Eq (36) expansion in [S1]
    return calculate_g_standard(path, i, theta)

def compute_differential_contribution_nee(path, nee_vertices, i, theta):
    # Computes g(x, x_j x_e,j, i; theta)
    # Eq (43): Product of f_k (k=1..j-1), G(x_j, x_e,j), rho_s, (df_i/f_i), L_e
    return calculate_g_nee(path, nee_vertices, i, theta)
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $f_i(x) = 0$, the term $\frac{\partial_\theta f_i(x)}{f_i(x)}$ is undefined. This occurs when the BRDF is zero (e.g., perfect mirror at wrong angle). Guard against this by checking if $f_i(x) \approx 0$ and handling the derivative term appropriately (often zero or requiring special handling) [S1].
*   **Precision:** The geometric series terms $(1-q)^{i-1}q$ can underflow for large $i$ if $q$ is small. Use log-probabilities or careful exponentiation [S1].
*   **Boundary Conditions:** The indices for NEE paths ($j=1 \dots N-1$) and differential vertices ($i=0 \dots j$) must be strictly enforced to avoid out-of-bounds access [S1].

## Subtopics

### [Differential Vertex Sampling with MIS](6338-differential-vertex-sampling-with-mis.md)
Differential Vertex Sampling with MIS is a method to estimate the gradient of a loss function with respect to scene parameters by importance sampling an intermediate path vertex as the "differential vertex". It solves the problem of high variance in gradient estimation, particularly for specular materials, by combining differential vertex sampling with standard BRDF sampling using Multiple Importance Sampling (MIS). Unlike standard BRDF sampling, which treats all vertices identically, this method explicitly biases the sampling distribution to capture the boundary component of the gradient integral, significantly reducing variance for microfacet and mixture BRDFs. See [Differential Vertex Sampling with MIS](6338-differential-vertex-sampling-with-mis.md).

### [Differential Edge Sampling](6708-differential-edge-sampling.md)
Differential Edge Sampling is a method for computing the gradient of physically-based rendered images with respect to arbitrary scene parameters (camera, lights, materials, geometry) by explicitly sampling visibility discontinuities. It solves the problem that standard Monte Carlo estimators fail to capture derivatives at boundaries where visibility changes abruptly, such as object silhouettes and occlusion edges. Unlike its sibling [Differential Vertex Sampling with MIS](6338-differential-vertex-sampling-with-mis.md), which focuses on vertex-based contributions, this method decomposes the rendering derivative into a smooth interior integral (estimated via automatic differentiation) and a boundary integral (estimated by sampling triangle edges). See [Differential Edge Sampling](6708-differential-edge-sampling.md).

### Vectorized Total Derivative Estimation
This approach effectively shares all paths traced by Algorithm 1 to calculate derivatives with respect to all scene parameters simultaneously [S2, unofficial]. It executes Algorithm 1 once in a vectorized fashion to estimate the total scene derivative $d_\pi L$. It replaces all partial derivatives with their vector-valued total derivative counterparts and performs component-wise calculations between all total-derivative-related quantities [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method requires tracing additional NEE paths for each main path vertex, leading to a time complexity of $O(K^2)$ per sample path of length $K$, compared to $O(K)$ for conventional VPT [S2, unofficial].
*   **Memory Complexity:** Requires storing the path vertices and intermediate contributions for MIS weighting. Memory usage scales linearly with path depth $N$ and number of samples $M$.
*   **Measured Performance:** The method shows significantly lower variance compared to standard BRDF sampling for scenes with varying roughnesses, with variance ratios (method/BRDF) often less than 1 [S1].
*   **Strengths:** Unbiased gradient estimation; handles complex material models (microfacets); reduces variance for specular materials; combines BRDF and emitter sampling effectively.
*   **Limitations:** High computational cost due to NEE path generation and MIS weight calculation; complex implementation; potential numerical instability with division by small BRDF values.
*   **When to Use:** Differentiable rendering, inverse rendering, and joint material/lighting estimation where high-quality gradients are critical and computational resources allow [S1].
*   **When Not to Use:** Real-time applications or scenarios with very limited computational budget; scenes with simple, diffuse materials where standard methods may suffice.

## Background
The method originates from the work of T. Su and I. Gkioulekas, "Path Sampling Methods for Differentiable Rendering," published in the Computer Graphics Forum (Eurographics) in 2024 [S1]. It builds upon earlier work on differentiable Monte Carlo ray tracing [LADL18] and inverse path tracing [ALKN19]. The use of NEE in differentiable rendering was explored in [NDMKJ22], and the theoretical foundation for differential radiative transfer is detailed in [ZWZ*19]. The vectorized total derivative estimation concept is discussed in [S2, unofficial].

## Sources
[S1] Path Sampling Methods for Differentiable Rendering, https://doi.org/10.2312/sr.20241148, fetched 2026-10-29
[S2] PDF A Differential Theory of Radiative Transfer, https://cseweb.ucsd.edu/~ravir/diff_rendering.pdf, fetched 2026-10-29 (unofficial)
