# Differential path integral estimator with MIS and NEE

Also called: Differential NEE estimator, MIS-weighted differential path estimator

## Overview
This method computes the gradient of the rendered image with respect to scene parameters $\theta$ by integrating differential measurement contributions over path space [S1]. It solves the problem of high variance in gradient estimation for specular materials and complex lighting by combining standard BRDF sampling with Next Event Estimation (NEE) using Multiple Importance Sampling (MIS) [S1]. Unlike standard NEE which only connects paths to emitters at the final vertex, this method allows the differential vertex (the location where the parameter derivative is applied) to be located at any intermediate vertex $i$ or at the emitter connection, combining BRDF and emitter sampling probabilities into a single mixture weight [S1]. It builds on [Reparameterization Estimators](6499-reparameterization-estimators.md) by extending the reparameterization principle to full path-space integrals with explicit MIS weights for differential contributions.

## How it works
**Inputs:**
*   Scene parameters $\theta$ (e.g., material roughness, light intensity).
*   Number of independent samples per pixel $M$.
*   Maximum path depth $N$.
*   Switching probability $q$ for differential sampling.

**Outputs:**
*   Gradient estimate $\partial_\theta I_{\text{NEE}}$.

**Data Structures:**
*   Path $x = (x_0, x_1, \dots, x_N)$, where $x_0$ is the camera position and $x_k$ are surface vertices.
*   Emitter connection path $x_j x_{e,j}$, where $x_{e,j}$ is a point on an emitter sampled from $x_j$.
*   Differential contribution $g(x, i; \theta)$, a scalar value representing the derivative contribution of vertex $i$ along path $x$.

**Formulas:**
The rendering integral is defined as [S1]:
$$
I = \int_{S^2} W_e(x_0, \omega_{x_1,x_0}) G(x_0, x_1) L_i(x_0, \omega_{x_1,x_0}) d\omega_{x_1,x_0}
$$
The gradient is the integral of differential contributions [S1]:
$$
\partial_\theta I = \int_{S^2} \left[ \sum_{i=0}^N g(x, i; \theta) \right] dx
$$
where $g(x, i; \theta)$ is the measurement contribution of path $x$ with the derivative applied at vertex $i$.

For NEE paths $x, x_j x_{e,j}$, the differential contribution is [S1]:
$$
g(x, x_j x_{e,j}, i; \theta) = \left[ \prod_{k=1}^{j-1} f_k(x) \right] G(x_j, x_{e,j}) \rho_s(x_{j-1}, x_j, x_{e,j}) \cdot \frac{\partial_\theta f_i(x)}{f_i(x)} \cdot L_e(x_{e,j})
$$
where $f_k(x)$ is the BSDF at vertex $x_k$, $G$ is the geometry term, $\rho_s$ is the shadow term, and $L_e$ is the emitter radiance.

**Mixture Probabilities:**
The estimator uses MIS to combine probabilities from BRDF sampling and NEE. The probability of generating a standard path $x$ with differential vertex $i$ is denoted $p(x, i)$. The probability of generating an NEE path $x, x_j x_{e,j}$ with differential vertex $i$ is $p_{\text{nee}}(x, x_j x_{e,j}, i)$ [S1]:
$$
p_{\text{nee}}(x, x_j x_{e,j}, i) = \frac{\prod_{k=0}^{j-1} p_f(x_k)}{p_f(x_i)} p_{\partial f}(x_i) (1-q)^{i-1} q \quad \text{for } 1 \le i < j
$$
$$
p_{\text{nee}}(x, x_j x_{e,j}) = \left[ \prod_{k=0}^{j-1} p_f(x_k) \right] p_e(x_j x_{e,j}) (1-q)^{j-1}
$$
The total mixture probability for a standard path $x$ is [S1]:
$$
p_{\text{mix}}(x) = p(x) + \sum_{i=1}^{N-1} p(x, i) + p_{\text{nee}}(x, x_{N-1}x_N)
$$
The total mixture probability for an NEE path $x, x_j x_{e,j}$ is [S1]:
$$
p_{\text{mix}}(x, x_j x_{e,j}) = p(x, x_j x_{e,j}) + \sum_{i=1}^j p(x, x_j x_{e,j}, i) + p_{\text{nee}}(x, x_j x_{e,j}) + \sum_{i=1}^{j-1} p_{\text{nee}}(x, x_j x_{e,j}, i)
$$

**Algorithm Steps:**
1.  Initialize gradient accumulator $G_{\text{total}} = 0$.
2.  For each sample $m = 1 \dots M$:
    a. Generate a standard path $x$ of depth $N$ using BRDF sampling.
    b. Compute $p_{\text{mix}}(x)$ using Eq (41).
    c. For each vertex $i = 0 \dots N$:
        i. Compute differential contribution $g(x, i; \theta)$.
        ii. Add $\frac{g(x, i; \theta)}{p_{\text{mix}}(x)}$ to $G_{\text{total}}$.
    d. For each vertex $j = 1 \dots N-1$:
        i. Generate an NEE path $x_j x_{e,j}$ by sampling an emitter from $x_j$.
        ii. Compute $p_{\text{mix}}(x, x_j x_{e,j})$ using Eq (42).
        iii. For each vertex $i = 0 \dots j$:
            1. Compute differential contribution $g(x, x_j x_{e,j}, i; \theta)$.
            2. Add $\frac{g(x, x_j x_{e,j}, i; \theta)}{p_{\text{mix}}(x, x_j x_{e,j})}$ to $G_{\text{total}}$.
3.  Return $\frac{1}{M} G_{\text{total}}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient(theta, M, N, q, scene):
    """
    Computes the gradient of the rendered image with respect to theta
    using the Differential Path Integral Estimator with MIS and NEE.
    
    Args:
        theta: Scene parameters.
        M: Number of samples per pixel.
        N: Maximum path depth.
        q: Probability of switching to differential sampling.
        scene: Scene object providing path generation and evaluation functions.
        
    Returns:
        Gradient estimate.
    """
    gradient_sum = 0.0
    
    for m in range(M):
        # 1. Generate standard path
        x = scene.generate_path(theta, N)
        
        # 2. Compute mixture probability for standard path
        # p_mix(x) = p(x) + sum(p(x, i)) + p_nee(x, x_{N-1}x_N)
        # Note: p(x) is the probability of generating x via BRDF sampling.
        # p(x, i) is the probability of generating x with differential vertex i.
        # p_nee(x, x_{N-1}x_N) is the probability of generating x connected to emitter at N-1.
        p_mix_x = compute_p_mix_standard(x, N, q, scene)
        
        # 3. Accumulate contributions from standard path
        for i in range(N + 1):
            g = compute_differential_contribution_standard(x, i, theta, scene)
            gradient_sum += g / p_mix_x
            
        # 4. Generate and accumulate NEE paths
        for j in range(1, N):
            # Generate NEE path: x connected to emitter at x_j
            x_ee = scene.generate_nee_path(x, j, theta)
            # x_ee consists of vertices x_0...x_j and emitter point x_{e,j}
            
            # Compute mixture probability for NEE path
            # p_mix(x, x_j x_{e,j}) includes terms for differential vertices 0..j
            p_mix_x_ee = compute_p_mix_nee(x, x_ee, j, N, q, scene)
            
            # Accumulate contributions from NEE path
            for i in range(j + 1):
                g_ee = compute_differential_contribution_nee(x, x_ee, j, i, theta, scene)
                gradient_sum += g_ee / p_mix_x_ee
                
    return gradient_sum / M

def compute_p_mix_standard(x, N, q, scene):
    """
    Computes p_mix(x) for a standard path x.
    """
    # p(x) is the probability of generating the path via BRDF sampling
    p_brdf = scene.path_probability(x)
    
    # Sum of probabilities for differential vertices i=1 to N-1
    sum_diff = 0.0
    for i in range(1, N):
        # p(x, i) involves the probability of generating x with differential vertex i
        # This is derived from the geometric series terms (1-q)^{i-1}q
        # and the ratio of BSDF probabilities.
        # p(x, i) = p(x) * (1-q)^{i-1} * q * (p_partial_f(x_i) / p_f(x_i))
        # Note: Exact formula depends on implementation of p(x) and sampling strategy.
        # Based on Eq (37) structure for NEE, but adapted for standard path.
        # For standard path, the "switching" logic applies to where the derivative is taken.
        # The source implies p(x, i) is part of the mixture.
        # We assume p(x, i) is computed as the probability of generating x 
        # where the differential vertex is i.
        # In the context of MIS, this is often the probability of the path 
        # being generated by the "differential sampling" strategy.
        # However, the source Eq (41) suggests p(x, i) are distinct terms.
        # Let's use the structure from Eq (37) for p_nee as a guide for the form.
        # For standard path, the probability of generating x with diff vertex i
        # is likely p(x) * (1-q)^{i-1} * q * (p_partial_f(x_i) / p_f(x_i))
        # if the sampling strategy involves switching.
        # If not, p(x, i) might just be p(x) if the path is always generated 
        # and we just evaluate the derivative at i.
        # The source says "mixture probability ... via Eq (41)".
        # Eq (41) sums p(x), p(x,i), and p_nee terms.
        # This implies p(x) is the BRDF sampling prob, and p(x,i) are other strategies.
        # Given the complexity, we assume p(x, i) is computed as:
        # p(x, i) = p(x) * (1-q)^{i-1} * q * (p_partial_f(x_i) / p_f(x_i))
        # This is an inference based on Eq (37) and the geometric series.
        term = p_brdf * ((1 - q) ** (i - 1)) * q * (scene.partial_f_prob(x[i]) / scene.f_prob(x[i]))
        sum_diff += term
        
    # p_nee(x, x_{N-1}x_N) is the probability of generating x connected to emitter at N-1
    # This is a specific NEE path generation strategy.
    # p_nee(x, x_{N-1}x_N) = p(x) * (1-q)^{N-1} * q * (p_e(x_{N-1}x_N) / p_f(x_{N-1}))
    # Again, inferred from Eq (38) and (40).
    p_nee_last = p_brdf * ((1 - q) ** (N - 1)) * q * (scene.emitter_prob(x[N-1]) / scene.f_prob(x[N-1]))
    
    return p_brdf + sum_diff + p_nee_last

def compute_p_mix_nee(x, x_ee, j, N, q, scene):
    """
    Computes p_mix(x, x_j x_{e,j}) for an NEE path.
    """
    # p(x, x_j x_{e,j}) is the probability of generating this path via BRDF sampling
    # Note: x_ee shares the first j+1 vertices with x.
    p_brdf_ee = scene.path_probability(x_ee[:j+1]) # Probability of the path segment
    
    # Sum of probabilities for differential vertices i=1 to j
    sum_diff = 0.0
    for i in range(1, j + 1):
        # p(x, x_j x_{e,j}, i)
        # Based on Eq (39) and (37)
        # p_nee(x, x_j x_{e,j}, i) = [prod_{k=0}^{j-1} p_f(x_k) / p_f(x_i)] * p_partial_f(x_i) * (1-q)^{i-1} * q
        # But we need to adjust for the emitter connection.
        # Eq (39) gives p(x, x_j x_{e,j}, i) in terms of p_nee.
        # Let's use the direct form from Eq (37) adjusted for the path.
        # p(x, x_j x_{e,j}, i) = p_brdf_ee * (1-q)^{i-1} * q * (p_partial_f(x_i) / p_f(x_i))
        term = p_brdf_ee * ((1 - q) ** (i - 1)) * q * (scene.partial_f_prob(x[i]) / scene.f_prob(x[i]))
        sum_diff += term
        
    # p_nee(x, x_j x_{e,j}) is the probability of generating this path via NEE
    # Based on Eq (38)
    # p_nee(x, x_j x_{e,j}) = [prod_{k=0}^{j-1} p_f(x_k)] * p_e(x_j x_{e,j}) * (1-q)^{j-1}
    # This is the probability of the path being generated by the NEE strategy.
    p_nee = p_brdf_ee * (1 - q) ** (j - 1) # Assuming p_e is normalized into p_brdf_ee or handled separately
    
    # Sum of probabilities for differential vertices i=1 to j-1 for NEE strategy
    sum_diff_nee = 0.0
    for i in range(1, j):
        # p_nee(x, x_j x_{e,j}, i)
        # Based on Eq (37)
        # p_nee(x, x_j x_{e,j}, i) = [prod_{k=0}^{j-1} p_f(x_k) / p_f(x_i)] * p_partial_f(x_i) * (1-q)^{i-1} * q
        # This is the probability of generating the path via NEE with diff vertex i.
        term_nee = p_nee * (1 / scene.f_prob(x[i])) * scene.partial_f_prob(x[i]) * ((1 - q) ** (i - 1)) * q
        sum_diff_nee += term_nee
        
    return p_brdf_ee + sum_diff + p_nee + sum_diff_nee

def compute_differential_contribution_standard(x, i, theta, scene):
    """
    Computes g(x, i; theta) for a standard path.
    """
    # g(x, i; theta) is the measurement contribution with derivative at i.
    # For i=0, it's the derivative of the camera BSDF.
    # For i=N, it's the derivative of the emitter radiance.
    # For 0 < i < N, it's the derivative of the BSDF at x_i.
    if i == 0:
        return scene.derivative_camera_bsdf(x[0], theta) * scene.path_radiance(x)
    elif i == len(x) - 1:
        return scene.path_radiance(x) * scene.derivative_emitter_radiance(x[-1], theta)
    else:
        # Product of BSDFs except at i, multiplied by derivative of BSDF at i
        product_bsdf = 1.0
        for k in range(len(x)):
            if k != i:
                product_bsdf *= scene.bsdf(x[k], theta)
        return product_bsdf * scene.bsdf_derivative(x[i], theta) * scene.geometry_term(x, i) * scene.emitter_radiance(x[-1])

def compute_differential_contribution_nee(x, x_ee, j, i, theta, scene):
    """
    Computes g(x, x_j x_{e,j}, i; theta) for an NEE path.
    """
    # Based on Eq (43)
    # g(x, x_j x_{e,j}, i; theta) = [prod_{k=1}^{j-1} f_k(x)] * G(x_j, x_{e,j}) * rho_s(x_{j-1}, x_j, x_{e,j}) * (partial_f_i(x) / f_i(x)) * L_e(x_{e,j})
    
    # Product of BSDFs from 1 to j-1
    product_bsdf = 1.0
    for k in range(1, j):
        product_bsdf *= scene.bsdf(x[k], theta)
        
    # Geometry term between x_j and x_{e,j}
    G = scene.geometry_term(x[j], x_ee[-1])
    
    # Shadow term
    rho_s = scene.shadow_term(x[j-1], x[j], x_ee[-1])
    
    # Derivative ratio
    if i < j:
        ratio = scene.bsdf_derivative(x[i], theta) / scene.bsdf(x[i], theta)
    else:
        # i == j, derivative of the BSDF at x_j? Or is it the emitter connection?
        # Eq (43) uses partial_f_i(x) / f_i(x). If i=j, it's the BSDF at x_j.
        ratio = scene.bsdf_derivative(x[j], theta) / scene.bsdf(x[j], theta)
        
    # Emitter radiance
    L_e = scene.emitter_radiance(x_ee[-1])
    
    return product_bsdf * G * rho_s * ratio * L_e
```

**Parameters:**
*   $q$: Probability of switching to differential sampling at each vertex, used in geometric series terms $(1-q)^{i-1}q$ and $(1-q)^{j-1}$. Typical value: 0.5 [S1].
*   $M$: Number of independent samples drawn per pixel for the Monte Carlo estimator. Typical value: 16-64 [S1].

**Edge Cases and Pitfalls:**
*   **Zero Probability:** If $p_{\text{mix}}(x)$ or $p_{\text{mix}}(x, x_j x_{e,j})$ is zero, the path should be skipped or handled with a guard to avoid division by zero [G].
*   **Precision:** The geometric series terms $(1-q)^{i-1}q$ can become very small for large $i$, leading to underflow. Use log-probabilities if necessary [G].
*   **Boundary Conditions:** The formulas for $p_{\text{nee}}$ and $g$ assume $1 \le i < j$ or $i=j$. Ensure indices are within bounds [S1].

## Subtopics

### [Differential Vertex Sampling with MIS](6338-differential-vertex-sampling-with-mis.md)
Differential Vertex Sampling with MIS is a method to estimate the gradient of a loss function with respect to scene parameters by importance sampling an intermediate path vertex as the "differential vertex". It solves the problem of high variance in gradient estimation, particularly for specular materials, by combining differential vertex sampling with standard BRDF sampling using Multiple Importance Sampling (MIS). Unlike standard BRDF sampling, which treats all vertices identically, this method explicitly biases the sampling distribution to capture the boundary component of the gradient integral, significantly reducing variance for microfacet and mixture BRDFs. See [Differential Vertex Sampling with MIS](6338-differential-vertex-sampling-with-mis.md).

### [Differential Edge Sampling](6708-differential-edge-sampling.md)
Differential Edge Sampling is a method for computing the gradient of physically-based rendered images with respect to arbitrary scene parameters (camera, lights, materials, geometry) by explicitly sampling visibility discontinuities. It solves the problem that standard Monte Carlo estimators fail to capture derivatives at boundaries where visibility changes abruptly, such as object silhouettes and occlusion edges. Unlike its sibling [Differential Vertex Sampling with MIS](6338-differential-vertex-sampling-with-mis.md), which focuses on vertex-based contributions, this method decomposes the rendering derivative into a smooth interior integral (estimated via automatic differentiation) and a boundary integral (estimated by sampling triangle edges). See [Differential Edge Sampling](6708-differential-edge-sampling.md).

### Vectorized Total Derivative Estimation
This approach effectively shares all paths traced by Algorithm 1 to calculate derivatives with respect to all scene parameters simultaneously [S2, unofficial]. It executes Algorithm 1 once in a vectorized fashion to estimate the total scene derivative $d_\pi L$. It replaces all partial derivatives with their vector-valued total derivative counterparts and performs component-wise calculations between all total-derivative-related quantities [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The method requires tracing additional NEE paths for each vertex in the main path, leading to a time complexity of $O(K^2)$ per sample path of length $K$, compared to $O(K)$ for conventional VPT [S2, unofficial].
*   **Memory Complexity:** Requires storing the path and intermediate contributions. Memory usage is proportional to the path length $N$ [G].
*   **Measured Performance:** The method reduces variance for scenes with varying roughnesses compared to BRDF sampling [S1].
*   **Strengths:** Reduces variance for specular materials and complex lighting by combining BRDF and emitter sampling [S1]. Allows the differential vertex to be at any intermediate vertex [S1].
*   **Limitations:** Higher computational cost due to NEE path generation [S2, unofficial].
*   **When to Use:** Differentiable rendering, inverse rendering, and joint material/lighting estimation [S1].
*   **When Not to Use:** When computational resources are limited and variance is not a primary concern [G].

## Background
The method originates from the work of T. Su and I. Gkioulekas, "Path Sampling Methods for Differentiable Rendering" [S1]. It builds on the differential theory of radiative transfer [S2, unofficial] and previous work on differentiable Monte Carlo ray tracing [LADL18]. The use of MIS for differential contributions is a key innovation to handle the variance in gradient estimation [S1].

## Sources
[S1] Path Sampling Methods for Differentiable Rendering, https://doi.org/10.2312/sr.20241148, fetched 2026-10-29
[S2] PDF A Differential Theory of Radiative Transfer, https://cseweb.ucsd.edu/~ravir/diff_rendering.pdf, fetched 2026-10-29 (unofficial)
