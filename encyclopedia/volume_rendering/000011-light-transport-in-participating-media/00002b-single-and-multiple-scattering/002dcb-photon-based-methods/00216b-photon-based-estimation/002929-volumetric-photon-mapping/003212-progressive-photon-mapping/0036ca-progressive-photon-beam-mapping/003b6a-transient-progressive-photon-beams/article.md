# Transient Progressive Photon Beams


## Overview
Transient Progressive Photon Beams is a volumetric light transport simulation method that uses photon beams to estimate transient radiance in participating media [S1]. It builds on [Progressive Photon Beam Mapping](7795-progressive-photon-beam-mapping.md) by optimizing the convergence rate of the estimator through analytical minimization of the Asymptotic Mean Squared Error (AMSE) [S1].

## How it works
### Inputs and Outputs
Not given in the sources.

### Data Structures
The method relies on the small photon maps generated iteratively by the parent method [G]. Each photon beam carries position, direction, energy, time, and weight [G].

### Parameters and Optimal Values
The convergence is governed by three decay parameters and two initial bandwidths [S1]:
1. $\alpha$: Iteration count scaling exponent. Optimal value: $\alpha = 2/3$ [S1].
2. $\beta_T$: Temporal kernel bandwidth decay exponent. Optimal value: $\beta_T = 1/2$ [S1].
3. $\beta_{Rb}$: Spatial kernel radius decay exponent. Constrained by $\beta_{Rb} = 1 - \beta_T$, yielding $\beta_{Rb} = 1/2$ [S1].
4. $\mathcal{T}_1$: Initial temporal bandwidth [S1].
5. $R_{b1}$: Initial spatial radius [S1].

### Step-by-Step Method
1. **Initialization**: Set $\alpha = 2/3$, $\beta_T = 1/2$, $\beta_{Rb} = 1/2$, and initial bandwidths $\mathcal{T}_1$, $R_{b1}$ [S1].
2. **Iteration Loop** (for $n = 1$ to $N$):
   *   **Update Kernel Bandwidths**: Calculate current temporal bandwidth $\mathcal{T}_n$ and spatial radius $R_{bn}$ using the decay formulas [S1]:
       $$ \mathcal{T}_n = \mathcal{T}_1 (n \alpha \text{B}(\alpha, n))^{-\beta_T} $$
       $$ R_{bn} = R_{b1} (n \alpha \text{B}(\alpha, n))^{-\beta_{Rb}} $$
       where $\text{B}(x, y)$ is the Beta function [S1].
   *   **Photon Transport & Storage**: Shoot photon beams from the light source and store them in the current small photon map [G].
   *   **Estimate Radiance**: Accumulate contributions from all stored photon beams using the current bandwidths $\mathcal{T}_n$ and $R_{bn}$ for spatial and temporal smoothing [G].
3. **Convergence**: The estimator $\hat{L}_n$ converges to the true transient radiance with an optimal AMSE rate of $O(n^{-2/3})$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def beta_function(x, y):
    """Compute the Beta function B(x, y) = Gamma(x)Gamma(y)/Gamma(x+y)."""
    return math.exp(math.lgamma(x) + math.lgamma(y) - math.lgamma(x + y))

def transient_progressive_photon_beams(N, alpha, beta_T, beta_Rb, T1, Rb1):
    """
    Implements the parameter updates and AMSE analysis for Transient Progressive Photon Beams.
    Photon shooting and radiance accumulation are handled by the parent method [G].
    """
    # Optimal parameters derived from AMSE minimization [S1]
    alpha = 2/3
    beta_T = 1/2
    beta_Rb = 1/2
    
    for n in range(1, N + 1):
        # Update kernel bandwidths according to Equations 39 and 40 [S1]
        beta_val = beta_function(alpha, n)
        T_n = T1 * (n * alpha * beta_val) ** (-beta_T)
        R_b_n = Rb1 * (n * alpha * beta_val) ** (-beta_Rb)
        
        # Photon transport and storage step [G]
        # photon_beams = shoot_photon_beams()
        # store_in_map(photon_beams)
        
        # Radiance estimation step [G]
        # L_n = estimate_radiance(query_points, T_n, R_b_n)
        
        # Asymptotic variance and bias bounds [S1]
        # Var[L_n] approx O(n^-alpha) + O(n^-1)
        # E[epsilon_n] approx O(n^(1-alpha))^(-2*beta_T) + O(n^(1-alpha))^(2*beta_T - 2)
        # AMSE[L_n] approx O(n^-alpha) + O(n^(-2*(1-alpha)))
```

### Mathematical Derivation of Optimal Parameters
The optimality of the parameters is derived from the Asymptotic Mean Squared Error (AMSE) [S1].

**Variance Analysis**:
The variance of the estimator $\hat{L}_n$ is decomposed into signal variance and noise variance [S1]:
$$ \text{Var}[\hat{L}_n] = \frac{1}{n} \text{Var}[\Psi L] + \frac{1}{n^2} \sum_{j=1}^n \text{Var}[\Psi \epsilon_j] $$
Expanding this, and assuming independence between the signal $\Psi$ and noise $\epsilon_j$:
$$ \text{Var}[\hat{L}_n] = \frac{1}{n} \text{Var}[\Psi L] + \text{Var}[\Psi] \frac{1}{n^2} \sum_{j=1}^n \text{Var}[\epsilon_j] + \text{E}[\Psi]^2 \frac{1}{n^2} \sum_{j=1}^n \text{Var}[\epsilon_j] + \text{Var}[\Psi] \frac{1}{n^2} \sum_{j=1}^n \text{E}[\epsilon_j]^2 $$
The variance of the noise term $\epsilon_n$ at iteration $n$ is approximated as [S1]:
$$ \text{Var}[\epsilon_n] \approx \frac{\text{Var}[\epsilon_1]}{(2 - \alpha)n^\alpha} = O(n^{-\alpha}) $$
The asymptotic variance of the final estimator $\mathcal{L}_n$ is dominated by the noise term [S1]:
$$ \text{Var}[\mathcal{L}_n] \approx \frac{1}{n} \text{Var}[\Psi L] + \text{E}[\Psi^2] \text{Var}[\epsilon_1] \approx \frac{1}{n} \text{Var}[\Psi L] + \frac{\text{Var}[\epsilon_1]}{(2 - \alpha)n^\alpha} = O(n^{-1}) + O(n^{-\alpha}) = O(n^{-\alpha}) $$

**Bias Analysis**:
The expected bias $E[\epsilon_j]$ at iteration $j$ is a sum of three terms involving photon flux $E[\Phi]$, probabilities $p_{R_b}$ and $p_T$, constants $C_T^{ij}, C_1^{ij}$, initial bandwidths $\mathcal{T}_1, R_b$, and decay factors [S1]:
$$ E[\epsilon_j] = E[\Phi]p_{R_b}C_T^{ij}\mathcal{T}_1^{-\alpha}\Theta(j^{-\alpha})^{-2\beta_T} + E[\Phi]p_TC_1^{ij}R_b^2\Theta(j^{-\alpha})^{-2\beta_{Rb}} + E[\Phi]C_T^{ij}C_1^{ij}\mathcal{T}_1^{-\alpha}R_b^2\Theta(j^{-\alpha})^{-2(\beta_T+\beta_{Rb})} $$
The asymptotic expected bias $E[\epsilon_n]$ for the final estimator is [S1]:
$$ E[\epsilon_n] = O(n^{1-\alpha})^{-2\beta_T} + O(n^{1-\alpha})^{-2\beta_{Rb}} + O(n^{1-\alpha})^{-2(\beta_T + \beta_{Rb})} $$
Using the equality $\beta_{Rb} = 1 - \beta_T$, the bias simplifies to [S1]:
$$ E[\epsilon_n] = O(n^{1-\alpha})^{-2\beta_T} + O(n^{1-\alpha})^{2\beta_T - 2} $$

**AMSE Minimization**:
The Asymptotic Mean Squared Error (AMSE) is the sum of variance and squared bias [S1]:
$$ AMSE(\hat{L}_n) = O(n^{-\alpha}) + \left( O(n^{1-\alpha})^{-2\beta_T} + O(n^{1-\alpha})^{2\beta_T - 2} \right)^2 $$
To minimize AMSE, we first optimize $\beta_T$. Since the variance is independent of $\beta_T$, we differentiate the bias term with respect to $\beta_T$ and equate to zero, yielding the optimal value [S1]:
$$ \beta_T = 1/2 $$
Substituting $\beta_T = 1/2$ into the AMSE equation gives [S1]:
$$ AMSE(\hat{L}_n) = O(n^{-\alpha}) + O(n^{-2(1-\alpha)}) $$
Finally, finding the minimum of this expression with respect to $\alpha$ yields the optimal parameter [S1]:
$$ \alpha = 2/3 $$
This results in the optimal convergence rate [S1]:
$$ AMSE(\hat{L}_n) = O(n^{-2/3}) $$

### Edge Cases and Pitfalls
*   **Precision**: The use of the Beta function $\text{B}(\alpha, n)$ for large $n$ may require careful numerical implementation to avoid underflow or overflow [G].
*   **Boundary Conditions**: The transient nature of the simulation requires careful handling of time boundaries; photons arriving outside the simulation window must be discarded or handled appropriately [G].
*   **Degenerate Input**: If the initial bandwidths $\mathcal{T}_1$ or $R_{b1}$ are set too small, the estimator may suffer from high variance initially; if set too large, the bias may be high [G].

## Operating characteristics
*   **Convergence Rate**: The method achieves an optimal convergence rate of $O(n^{-2/3})$ for the AMSE [S1].
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Strengths**: Optimal convergence rate for transient radiance estimation in participating media [S1].
*   **Limitations**: Requires careful tuning of initial bandwidths and relies on asymptotic independence assumptions [G].
*   **When to use**: Suitable for transient light transport simulation where optimal convergence is prioritized [S1].
*   **When not to use**: Not given in the sources.

## Background
Transient Progressive Photon Beams was introduced in the paper "Progressive Transient Photon Beams" [S1]. It extends the concept of Progressive Photon Beam Mapping to the transient domain, addressing the challenges of time-resolved light transport [S1].

## Sources
[S1] Progressive Transient Photon Beams, https://arxiv.org/abs/1805.09562v1, fetched 2026-10-27
