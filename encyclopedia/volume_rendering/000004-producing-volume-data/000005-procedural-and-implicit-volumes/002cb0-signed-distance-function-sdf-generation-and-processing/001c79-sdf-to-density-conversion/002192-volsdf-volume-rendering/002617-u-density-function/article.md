# U-density Function

Also called: NeUDF rendering weight function, NeUDF weight, U-density, cumulative distribution function of U-density

## Overview
The U-density function is a rendering weight function designed for Unsigned Distance Fields (UDF) to ensure globally unbiased volume rendering [S1]. It solves the volume-surface gap inherent in naive extensions of Signed Distance Field (SDF) techniques to UDFs, specifically preventing redundant floating faces and irregular noises caused by improper weight distribution [S1]. Unlike the NeuS weight function, which violates the unbiased property for UDFs by allowing weight leakage to invisible surfaces, the U-density function concentrates rendering weight near the first surface intersection, ensuring the final color is derived solely from the visible surface geometry [S1].

## How it works
### Inputs and Outputs
*   **Input**: A ray defined by origin $o$ and direction $v$, sampled at discrete points $t_i$ along the ray [S1]. At each point, a UDF network $\Psi$ outputs a distance value $d = \Psi(p(t))$ where $p(t) = o + tv$ [S1].
*   **Output**: A rendering weight $w_r(t)$ for each sample point, used in alpha-compositing to compute the final pixel color [S1].

### Data Structures
*   **Ray Samples**: A list of distances $t$ along the ray [S1].
*   **UDF Values**: A list of scalar distances $d_i = \Psi(p(t_i))$ retrieved from the neural network [S1].
*   **Transformation Function**: $\varsigma_r(d)$, a parametric function mapping distance to cumulative probability [S1].

### Formulas and Definitions
The rendering weight $w_r(t)$ is defined as:
$$w_r(t) = \tau_r(t) e^{-\int_0^t \tau_r(u) du}$$
where $\tau_r(t)$ is the density derived from the UDF [S1].

The density $\tau_r(t)$ is calculated using the derivative of a transformed UDF:
$$\tau_r(t) = \left| \frac{\frac{\partial(\varsigma_r \circ \Psi \circ p)}{\partial t}(t)}{\varsigma_r \circ \Psi \circ p(t)} \right|$$
Here, $\circ$ denotes function composition, $\Psi$ is the UDF network, $p(t)$ is the ray position, and $\varsigma_r$ is the transformation function [S1].

The transformation function $\varsigma_r(d)$ must satisfy the following constraints for all $d > 0$:
1.  $\varsigma_r(0) = 0$ [S1]
2.  $\lim_{d \rightarrow +\infty} \varsigma_r(d) = 1$ [S1]
3.  $\varsigma'_r(d) > 0$ (strictly increasing) [S1]
4.  $\varsigma''_r(d) < 0$ (concave) [S1]

The specific form of $\varsigma_r(d)$ used in the source is:
$$\varsigma_r(d) = \frac{r d}{1 + r d}$$
where $r$ is a learnable parameter [S1].

### Parameters
*   **$r$**: A learnable parameter controlling the distribution of the density [S1].
    *   **Initial Value**: $0.05$ [S1].

### Numbered Steps
1.  **Sample Ray**: Discretize the ray into $N$ points $t_1, \dots, t_N$ [S1].
2.  **Query UDF**: For each point $t_i$, compute the unsigned distance $d_i = \Psi(p(t_i))$ [S1].
3.  **Transform Distance**: Compute the transformed value $s_i = \varsigma_r(d_i) = \frac{r d_i}{1 + r d_i}$ [S1].
4.  **Compute Density**: Calculate the density $\tau_r(t_i)$ using the chain rule: $\tau_r(t_i) = \left| \frac{\varsigma'_r(d_i) \cdot \frac{\partial d_i}{\partial t_i}}{\varsigma_r(d_i)} \right|$ [S1]. The term $\frac{\partial d_i}{\partial t_i}$ is the gradient of the UDF along the ray direction, computed via automatic differentiation of $\Psi$ with respect to $p(t_i)$ projected onto $v$ [S1].
5.  **Compute Weight**: Calculate the weight $w_r(t_i)$ using the exponential integral approximation [S1].
6.  **Composite**: Accumulate colors using standard alpha compositing with the computed weights [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigma_r(d, r):
    """Transformation function: s = rd / (1 + rd)"""
    return (r * d) / (1 + r * d)

def sigma_r_prime(d, r):
    """Derivative: s' = r / (1 + rd)^2"""
    denom = 1 + r * d
    return r / (denom * denom)

def compute_tau(d, grad_along_ray, r):
    """Compute density tau_r(t) = | (d/dt sigma(d)) / sigma(d) |"""
    s = sigma_r(d, r)
    s_prime = sigma_r_prime(d, r)
    numerator = abs(s_prime * grad_along_ray)
    # Guard against division by zero at d=0 [G]
    if s < 1e-8:
        return 1e6
    return numerator / s

def u_density_render(rays_o, rays_v, udf_network, r, num_samples, near, far):
    """
    Discretized volume rendering using U-density weight function.
    Implements alpha compositing: w_r(t) = tau(t) * exp(-integral_0^t tau(u) du).
    """
    results = []
    for o, v in zip(rays_o, rays_v):
        T = 1.0
        accumulated_color = 0.0
        delta_t = (far - near) / num_samples
        for i in range(num_samples):
            t = near + (far - near) * (i + 0.5) / num_samples
            p = o + t * v
            d = udf_network(p)
            grad_d = udf_network.gradient(p) @ v
            tau = compute_tau(d, abs(grad_d), r)
            alpha = 1.0 - math.exp(-tau * delta_t)
            c = udf_network.color(p)
            accumulated_color += alpha * T * c
            T *= (1.0 - alpha)
        results.append(accumulated_color)
    return results
```

### Edge Cases and Pitfalls
*   **Division by Zero**: $\varsigma_r(0) = 0$. If a sample point lies exactly on the surface ($d=0$), the denominator in $\tau_r(t)$ becomes 0, causing the density to concentrate sharply near the surface [S1].
*   **Naive Sigmoid Extension**: Using a standard Sigmoid function (as in NeuS) violates $\varsigma_r(0)=0$ because $\text{Sigmoid}(0) = 0.5$, which leads to bias in rendering weights and results in redundant floating faces and irregular noises [S1].
*   **Negative Distances**: UDFs are non-negative. The constraints apply for $d > 0$ [S1].

## Operating characteristics
*   **Time Complexity**: Dominated by the neural network inference for $\Psi$ and its gradient computation [S1].
*   **Memory Complexity**: $O(1)$ auxiliary memory per ray sample [S1].
*   **Strengths**: Guarantees global unbiased rendering for UDFs and prevents color mixing from invisible surfaces ($c_m, c_n$) [S1].
*   **Limitations**: Requires gradient computation of the UDF network, which can be computationally expensive [S1].
*   **When to Use**: When rendering with Unsigned Distance Fields and unbiased surface reconstruction is required [S1].
*   **When Not to Use**: When using Signed Distance Fields (SDFs) where other methods may be more appropriate [S1].

## Background
The U-density function was introduced in the paper "NeUDF: Learning Neural Unsigned Distance Fields with Volume Rendering" [S1]. It was developed to address the limitations of applying SDF-based neural rendering techniques directly to UDFs [S1]. The authors observed that naive extensions violate the unbiased property due to the different value domains of SDFs and UDFs [S1]. The method builds on the concept of volume rendering but adapts the weight function to the specific properties of UDFs, ensuring that the rendering process converges to the correct surface geometry without artifacts [S1].

## Sources
[S1] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
