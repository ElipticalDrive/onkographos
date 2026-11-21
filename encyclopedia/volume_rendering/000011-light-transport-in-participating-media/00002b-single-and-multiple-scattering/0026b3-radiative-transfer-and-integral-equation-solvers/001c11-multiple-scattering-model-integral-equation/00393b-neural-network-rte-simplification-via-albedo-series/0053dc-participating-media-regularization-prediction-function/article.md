# Participating Media Regularization (Prediction Function)

Also called: Novel prediction function, Two-exponential prediction

## Overview
This method estimates the target radiance $R_t$ for a participating media scene given a rendered source radiance $R_s$, its gradient $G_s$ at a source albedo $\alpha_s$, and a target albedo $\alpha_t$ [S1, S2]. It solves the problem that linear and log-linear prediction functions fail to capture the complex, non-linear behavior of radiance with respect to albedo changes in participating media, particularly across varying mean free paths and extinction coefficients [S1, S2]. It sets itself apart by using a two-exponential formulation that adapts to both high/low albedo regimes and small/large mean free paths, providing a better fit to ground truth than simpler models [S1, S2].

## How it works
**Inputs:**
*   $R_s$: Source radiance (pixel value) at source albedo.
*   $G_s$: Source gradient (rate of change of radiance with respect to albedo) at source albedo.
*   $\alpha_s$: Source albedo.
*   $\alpha_t$: Target albedo.
*   $\sigma_t$: Extinction coefficient of the medium.

**Outputs:**
*   $R_t$: Predicted target radiance.

**Data Structures:**
No complex data structures are required; the method operates on scalar values per pixel or per light transport path segment.

**Parameters:**
*   $c_\beta$: A constant used to scale the mean free path ($\sigma_t^{-1}$). It must satisfy $0 < c_\beta \le \sigma_t \alpha_s$ to ensure $\beta \le 1$. In practice, $c_\beta$ is set to $0.5$ or searched for around $0.5$ [S1, S2].

**Formulas:**
The target radiance is computed as:
$$ R_t = R_s + G_s \beta \left( e^{(\alpha_t - \alpha_s)/\beta} - 1 \right) e^{-\gamma R_s (\alpha_t - \alpha_s)} $$

The parameters $\beta$ and $\gamma$ are defined as:
$$ \beta = \frac{c_\beta \ln(\sigma_t + 1)}{\sigma_t \alpha_s} $$
$$ \gamma = \frac{5}{1 - \alpha_s} $$

**Step-by-Step Procedure:**
1.  Calculate $\beta$ using the extinction coefficient $\sigma_t$, source albedo $\alpha_s$, and constant $c_\beta$.
2.  Calculate $\gamma$ using the source albedo $\alpha_s$.
3.  Compute the first exponential term component: $E_1 = e^{(\alpha_t - \alpha_s)/\beta} - 1$.
4.  Compute the second exponential term component: $E_2 = e^{-\gamma R_s (\alpha_t - \alpha_s)}$.
5.  Compute the predicted radiance: $R_t = R_s + G_s \cdot \beta \cdot E_1 \cdot E_2$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def predict_radiance(Rs, Gs, alpha_s, alpha_t, sigma_t, c_beta=0.5):
    """
    Predict target radiance Rt given source radiance Rs, gradient Gs,
    source albedo alpha_s, target albedo alpha_t, and extinction coefficient sigma_t.
    
    Args:
        Rs: Source radiance (float)
        Gs: Source gradient (float)
        alpha_s: Source albedo (float)
        alpha_t: Target albedo (float)
        sigma_t: Extinction coefficient (float)
        c_beta: Scaling constant for mean free path (float, default 0.5)
        
    Returns:
        Rt: Predicted target radiance (float)
    """
    # Calculate beta
    # beta = (c_beta * ln(sigma_t + 1)) / (sigma_t * alpha_s)
    # Note: Ensure sigma_t * alpha_s is not zero to avoid division by zero.
    # If alpha_s is 0, the behavior is undefined in this formulation.
    beta = (c_beta * math.log(sigma_t + 1)) / (sigma_t * alpha_s)
    
    # Calculate gamma
    # gamma = 5 / (1 - alpha_s)
    # Note: If alpha_s approaches 1, gamma approaches infinity.
    # The term (1 - alpha_s) in the exponent of the second exponential
    # helps control the amplification.
    gamma = 5.0 / (1.0 - alpha_s)
    
    # Calculate the first exponential term part
    # term1 = exp((alpha_t - alpha_s) / beta) - 1
    exponent1 = (alpha_t - alpha_s) / beta
    term1 = math.exp(exponent1) - 1.0
    
    # Calculate the second exponential term part
    # term2 = exp(-gamma * Rs * (alpha_t - alpha_s))
    exponent2 = -gamma * Rs * (alpha_t - alpha_s)
    term2 = math.exp(exponent2)
    
    # Calculate final radiance
    # Rt = Rs + Gs * beta * term1 * term2
    Rt = Rs + Gs * beta * term1 * term2
    
    return Rt
```

**Edge Cases and Pitfalls:**
*   **High Albedo and Small Mean Free Path:** In these cases, $\beta$ approaches 0. This causes the factor multiplying $G_s$ to become significantly large, amplifying small differences in $G_s$ among neighboring pixels and leading to discontinuities in the predicted result [S1]. The second exponential term (controlled by $\gamma$) is specifically designed to alleviate this amplification [S1].
*   **Albedo near 1:** As $\alpha_s \to 1$, $\gamma \to \infty$. However, the term $(1-\alpha_s)$ in the denominator of $\gamma$ is balanced by the $(\alpha_t - \alpha_s)$ term in the exponent of the second exponential. When $R_s$ is close to 0, $G_s$ is also close to 0, so the attenuation effect of the second exponential term is restrained [S2].
*   **Zero Extinction or Albedo:** The formula for $\beta$ involves division by $\sigma_t \alpha_s$. If $\alpha_s = 0$ or $\sigma_t = 0$, the formula is undefined. The sources do not specify a fallback for these degenerate cases [S1, S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel, as it involves a constant number of arithmetic operations and exponential evaluations.
*   **Memory Complexity:** $O(1)$ per pixel, requiring storage for $R_s, G_s, \alpha_s, \alpha_t, \sigma_t$ and intermediate variables.
*   **Performance:** The method is designed to work for a wide range of materials, from large mean free path (mfp) to small mfp [S1, S2]. It fits ground truth better than linear and log-linear prediction functions across varying mfp and extinction coefficients [S1, S2].
*   **Strengths:** Handles complex non-linear albedo behavior; adapts to different scales of mean free path; reduces discontinuities in high-albedo/small-mfp scenarios compared to simpler models.
*   **Limitations:** Requires accurate estimation of $G_s$ (gradient); performance depends on the choice of $c_\beta$; undefined for $\alpha_s = 0$ or $\sigma_t = 0$.
*   **When to use:** When rendering participating media with varying albedo and needing to predict radiance for a target albedo from a source albedo, especially when linear/log-linear models fail to capture the complexity.
*   **When not to use:** When $\alpha_s$ is 0 or $\sigma_t$ is 0; when computational cost of evaluating exponentials is prohibitive (though it is generally low).

## Background
The method was proposed in the context of efficient participating media rendering with differentiable volumetric path tracing [S1, S2]. It addresses the challenge that rendering participating media with high albedo is difficult due to slow light path attenuation, leading to noisy results [S2]. The prediction function is part of a regularization technique that uses source pixel values and gradients to predict target pixel values, avoiding the need for expensive re-rendering at the target albedo [S2]. The design was inspired by observations of radiance and gradient curves as functions of albedo for different pixels and extinction coefficients [S2]. The authors implemented their algorithm in Mitsuba 0.5 [S2].

## Sources
[S1] Efficient Participating Media Rendering with Differentiable ..., https://iccvm.org/2023/papers/s8-3-284-CVMJ.pdf, fetched 2026-10-29
[S2] Efficient participating media rendering with differentiable ..., https://www.sciopen.com/article/10.1007/s41095-023-0372-2?issn=2096-0433, fetched 2026-10-29
