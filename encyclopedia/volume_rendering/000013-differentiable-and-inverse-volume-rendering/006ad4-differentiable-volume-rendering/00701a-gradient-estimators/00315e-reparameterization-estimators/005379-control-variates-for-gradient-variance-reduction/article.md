# Control Variates for Gradient Variance Reduction

Also called: Gradient control variates, Variance reduction via control variates

## Overview
Control variates for gradient variance reduction reduces the variance of gradient estimators in differentiable volume rendering by exploiting the zero expected gradient of the reparameterization weight term [S1]. It differs from sibling methods like shift maps by explicitly constructing a correlated control estimator from the Jacobian weights rather than relying solely on path correlation through random number reuse [S1]. It targets the high variance inherent in the gradient of the weight term when the integrand is smooth or constant [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene parameters $\theta$, reference parameters $\theta_0$, sampling points $y_i$ from a fixed reference domain, transformation $T(y_i, \theta)$, integrand $f(T(y_i, \theta), \theta)$, and probability density $k$ [S1].
*   **Outputs:** An unbiased gradient estimator $\frac{\partial E'}{\partial \theta}$ with reduced variance [S1].

**Data Structures:**
*   **Weights $w_i(\theta)$:** A scalar per sample $i$, defined as the ratio of the probability density at the transformed point under current parameters to that under reference parameters:
    $$ w_i(\theta) = \frac{k(T(y_i, \theta), \theta)}{k(T(y_i, \theta_0), \theta_0)} $$
*   **Control variate parameter $\alpha$:** A scalar coefficient that must be independent of the current weights $w_i(\theta)$ to avoid bias [S1].

**Numbered Steps:**
1.  **Sample and Transform:** For each sample $i \in \{1, \dots, N\}$, generate $y_i$ from the fixed reference distribution and compute the transformed position $x_i = T(y_i, \theta)$ [S1].
2.  **Compute Weights:** Calculate the weight $w_i(\theta)$ for the current parameters $\theta$ and note that $w_i(\theta_0) = 1$ by definition [S1].
3.  **Evaluate Integrand:** Compute $f_i = f(T(y_i, \theta), \theta)$ [S1].
4.  **Construct Control Variate Estimator:** Form the intermediate estimator $E' = \frac{1}{N} \sum_{i=1}^N [ f_i w_i(\theta) + \alpha(w_i(\theta) - w_i(\theta_0)) ]$, which is unbiased because $\mathbb{E}[w_i(\theta) - w_i(\theta_0)] = 0$ [S1].
5.  **Differentiate:** Compute the gradient of $E'$ with respect to $\theta$. Since $w_i(\theta_0)$ is constant, its derivative vanishes, yielding $\frac{\partial E'}{\partial \theta} = \frac{1}{N} \sum_{i=1}^N \frac{\partial}{\partial \theta} [ f_i w_i(\theta) + \alpha w_i(\theta) ]$ [S1].
6.  **Return Gradient:** Return the computed sum as the gradient estimate [S1].

**Formulas:**
*   **Weight:** $w_i(\theta) = \frac{k(T(y_i, \theta), \theta)}{k(T(y_i, \theta_0), \theta_0)}$ [S1].
*   **Optimal $\alpha$:** $\alpha = -\frac{\text{Cov}(E, F)}{\text{Var}(F)}$, where $E$ is the original gradient estimator and $F$ is the control variate estimator $\frac{\partial F}{\partial \theta} = \frac{1}{N} \sum \frac{\partial w_i(\theta)}{\partial \theta}$ [S1]. In practice, $\alpha$ is approximated (e.g., as $-c$ if $f(x)=c$, or as a running average of $f$) [S1].
*   **Gradient Estimator:** $\frac{\partial E'}{\partial \theta} = \frac{1}{N} \sum \frac{\partial}{\partial \theta} [ f(T(y_i, \theta), \theta) w_i(\theta) + \alpha w_i(\theta) ]$ [S1].

**Parameters:**
*   **$\alpha$:** The control variate coefficient. Optimal value is $-\frac{\text{Cov}(E, F)}{\text{Var}(F)}$. Practical approximation: average of $f$ or constant $c$ if $f$ is constant [S1].
*   **$\theta_0$:** Reference parameters. Often set to $\theta$ for the primal pass, but for gradient estimation, it serves as the baseline for the weight ratio [S1].

**Edge Cases and Pitfalls:**
*   **Bias from $\alpha$:** The parameter $\alpha$ must be independent of the weights $w_i(\theta)$ to avoid introducing bias in the gradient of $E'$ [S1]. If $\alpha$ is computed using the same samples $w_i(\theta)$, the estimator becomes biased [S1]. Cross-weighting schemes or using uncorrelated estimates are required to estimate $\alpha$ without bias [S1].
*   **Constant Integrand:** If $f(x) = c$, the variance of the gradient can be completely eliminated if $\alpha = -c$ [S1].
*   **Smoothness:** The variance reduction is most effective when $f$ is a smooth function with small gradients with respect to $\theta$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient_control_variates(samples, theta, theta_0, T, f, k, alpha):
    """
    Compute gradient of rendering integral using control variates.
    
    Args:
        samples: List of N reference samples y_i.
        theta: Current scene parameters (vector).
        theta_0: Reference scene parameters (vector).
        T: Transformation function T(y, theta).
        f: Integrand function f(x, theta).
        k: Probability density function k(x, theta).
        alpha: Control variate coefficient (must be independent of current weights).
        
    Returns:
        Gradient estimate dE'/dtheta.
    """
    N = len(samples)
    grad_dim = len(theta)
    grad_sum = [0.0] * grad_dim
    
    for i in range(N):
        y_i = samples[i]
        x_i = T(y_i, theta)
        x_i_ref = T(y_i, theta_0)
        
        k_curr = k(x_i, theta)
        k_ref = k(x_i_ref, theta_0)
        
        w_i = k_curr / k_ref if k_ref != 0 else 0.0
        # w_i(theta_0) is identically 1 by definition
        f_i = f(x_i, theta)
        
        # Compute gradients of f and w with respect to theta
        df_dtheta = compute_gradient(f, x_i, theta)
        dw_dtheta = compute_gradient_weight(w_i, T, k, y_i, theta, theta_0)
        
        # Accumulate: df/dtheta * w + (f + alpha) * dw/dtheta
        for d in range(grad_dim):
            grad_sum[d] += df_dtheta[d] * w_i + (f_i + alpha) * dw_dtheta[d]
            
    return [g / N for g in grad_sum]

def compute_gradient_weight(w_i, T, k, y_i, theta, theta_0):
    # The sources do not provide the analytical derivative of k(T(y_i, theta), theta) w.r.t theta.
    # In practice, this is computed via automatic differentiation [G].
    return compute_gradient(lambda th: k(T(y_i, th), th), theta)

def compute_gradient(func, x, params):
    # Compute gradient of func with respect to params using automatic differentiation [G].
    # Returns a vector of partial derivatives.
    pass # Implementation depends on the AD framework [G]
```

## Subtopics

### Variation Reduction for Gradient Estimators
A variant that reduces variance by tracing pairs of correlated paths that reuse certain random numbers and combining their estimates using control variates [S1]. It is similar to shift maps [Kettunen et al. 2015; Lehtinen et al. 2013] in sampling correlated paths, and similar to Rousselle et al. [2016] in applying control variates to gradient estimates [S1].

## Operating characteristics
*   **Time Complexity:** Adds minimal overhead to the standard reparameterization estimator, primarily the computation of the additional term $\alpha \frac{\partial w_i}{\partial \theta}$ and the storage of $\alpha$ [S1].
*   **Memory Complexity:** Requires storing the reference parameters $\theta_0$ and potentially precomputed weights $w_i(\theta_0)$ if not computed on the fly [S1].
*   **Performance:** Significant variance reduction is achieved with low performance overhead [S1]. For constant $f$, variance can be eliminated with optimal $\alpha$ [S1].
*   **Strengths:** Effective for smooth integrands; reduces variance of the weight gradient term which is often the dominant source of variance in reparameterization estimators [S1].
*   **Limitations:** Requires careful choice of $\alpha$ to avoid bias; $\alpha$ must be independent of the current weights [S1]. Performance gain depends on the correlation between $E$ and $F$ [S1].
*   **When to use:** When gradient variance is high due to the Jacobian weight term, especially for smooth scenes or constant integrands [S1].
*   **When not to use:** When $f$ has large gradients or discontinuities that break the smoothness assumption, as the variance reduction relies on $f$ being smooth with small gradients with respect to $\theta$ [S1].

## Background
Originated in the context of differentiable rendering, specifically in the paper "Reparameterizing discontinuous integrands for differentiable rendering" [S1]. It builds on the general control variates method from statistics [S1] and applies it to the gradient of reparameterized rendering integrals. It is related to shift maps [Kettunen et al. 2015; Lehtinen et al. 2013] and Rousselle et al. [2016] who applied control variates to image-space gradients [S1].

## Sources
[S1] Reparameterizing discontinuous integrands for differentiable rendering, https://doi.org/10.1145/3355089.3356510, fetched 2026-10-26
