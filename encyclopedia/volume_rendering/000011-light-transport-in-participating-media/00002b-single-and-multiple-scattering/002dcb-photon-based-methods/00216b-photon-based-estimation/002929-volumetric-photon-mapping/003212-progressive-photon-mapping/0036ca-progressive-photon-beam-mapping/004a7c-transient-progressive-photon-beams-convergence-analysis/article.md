# Transient Progressive Photon Beams Convergence Analysis

Also called: Asymptotic Analysis of Transient Progressive Photon Beams, TPPB Convergence

## Overview
This method provides the theoretical foundation for optimizing the convergence rate of the radiance estimator in Transient Progressive Photon Beams [S1]. It analyzes the asymptotic behavior of the estimator's variance and bias to derive the optimal parameters that minimize the Asymptotic Mean Squared Error (AMSE). It sets itself apart from its sibling [Transient Progressive Photon Beams](8973-transient-progressive-photon-beams.md) by providing the mathematical proof for *why* specific parameter schedules ($\alpha=2/3, \beta_T=1/2$) are chosen, rather than just describing the algorithmic steps.

## How it works
The analysis decomposes the Mean Squared Error (MSE) of the radiance estimator $\hat{L}_n$ after $n$ iterations into variance and bias components, then optimizes the kernel evolution parameters to minimize the asymptotic limit of this error.

### Inputs and Outputs
*   **Inputs**: The iteration count $n$, the variance decay parameter $\alpha$, and the kernel bandwidth decay parameter $\beta_T$.
*   **Outputs**: The optimal parameter values $\alpha^*$ and $\beta_T^*$ that minimize the AMSE, and the resulting convergence rate.

### Data Structures and Variables
The analysis relies on the following symbolic variables:
*   $\hat{L}_n$: The radiance estimator at iteration $n$.
*   $\Psi$: The photon weight or throughput factor.
*   $\epsilon_j$: The error term introduced at iteration $j$.
*   $\mathcal{T}_j$: The kernel bandwidth at iteration $j$.
*   $R_{bj}$: The kernel radius at iteration $j$.
*   $\mathcal{T}_1, R_{b1}$: Initial kernel bandwidth and radius.
*   $B(x, y)$: The Beta function.
*   $E[\cdot]$: Expectation operator.
*   $\text{Var}[\cdot]$: Variance operator.

### Formulas and Derivation

**1. Variance Decomposition**
Assuming the random variables $\Psi$ and $\epsilon_j$ are independent, the variance of the estimator is modeled as [KZ11]:

$$
\text{Var}[\hat{L}_n] = \frac{1}{n} \text{Var}[\Psi L] + \frac{1}{n^2} \sum_{j=1}^n \text{Var}[\Psi \epsilon_j]
$$

Expanding the second term using independence:

$$
\text{Var}[\hat{L}_n] = \frac{1}{n} \text{Var}[\Psi L] + \text{Var}[\Psi] \frac{1}{n^2} \sum_{j=1}^n \text{Var}[\epsilon_j] + \text{E}[\Psi]^2 \frac{1}{n^2} \sum_{j=1}^n \text{Var}[\epsilon_j] + \text{Var}[\Psi] \frac{1}{n^2} \sum_{j=1}^n \text{E}[\epsilon_j]^2 \quad (36)
$$

**2. Variance Approximation**
Following [KD13], the variance of the error term at iteration $n$ is approximated as a function of the first iteration's variance:

$$
\text{Var}[\epsilon_n] \approx \frac{\text{Var}[\epsilon_1]}{(2 - \alpha)n^\alpha} = O(n^{-\alpha}) \quad (37)
$$

Substituting this into the variance decomposition and applying asymptotic simplifications yields the asymptotic variance of the estimator:

$$
\text{Var}[\mathcal{L}_n] \approx \frac{1}{n} \text{Var}[\Psi L] + \text{E}[\Psi^2] \text{Var}[\epsilon_1] \approx \frac{1}{n} \text{Var}[\Psi L] + \frac{\text{Var}[\epsilon_1]}{(2 - \alpha)n^\alpha} = O(n^{-1}) + O(n^{-\alpha}) = O(n^{-\alpha}) \quad (38)
$$

**3. Kernel Evolution**
The kernel bandwidth $\mathcal{T}_j$ and radius $R_{bj}$ evolve according to the following schedules, where $B(\alpha, j)$ is the Beta function:

$$
\mathcal{T}_j = \mathcal{T}_1(j \alpha B(\alpha, j))^{-\beta_{\mathcal{T}}} \quad (39)
$$

$$
R_{bj} = R_{b1}(j \alpha B(\alpha, j))^{-\beta_{Rb}} \quad (40)
$$

**4. Bias Analysis**
The expected error $E[\epsilon_j]$ at iteration $j$ is modeled as a function of initial kernel parameters and constants $E[\Phi], p_{R_b}, C_T^{ij}, C_1^{ij}$:

$$
E[\epsilon_j] = E[\Phi]p_{R_b}C_T^{ij}\mathcal{T}_1^{-\alpha}\Theta(j^{-\alpha})^{-2\beta\tau} + E[\Phi]p_TC_1^{ij}R_b^2\Theta(j^{-\alpha})^{-2\beta_{Rb}} + E[\Phi]C_T^{ij}C_1^{ij}\mathcal{T}_1^{-\alpha}R_b^2\Theta(j^{-\alpha})^{-2(\beta\tau+\beta_{Rb})} \quad (41)
$$

Using the summation property $\sum_{j=1}^n \Theta(j^x) = O(n^{1+x})$, the asymptotic bias $E[\epsilon_n]$ is derived:

$$
E[\epsilon_n] = O(n^{1-\alpha})^{-2\beta\tau} + O(n^{1-\alpha})^{-2\beta_{Rb}} + O(n^{1-\alpha})^{-2(\beta\tau + \beta_{Rb})}
$$

By applying the constraint $\beta_{Rb} = 1 - \beta_T$, this simplifies to:

$$
E[\epsilon_n] = O(n^{1-\alpha})^{-2\beta_T} + O(n^{1-\alpha})^{2\beta_T - 2} \quad (42)
$$

**5. AMSE Optimization**
The Asymptotic Mean Squared Error (AMSE) is the sum of asymptotic variance and squared bias:

$$
AMSE(\hat{L}_n) = O(n^{-\alpha}) + \left( O(n^{1-\alpha})^{-2\beta_T} + O(n^{1-\alpha})^{2\beta_T - 2} \right)^2 \quad (43)
$$

To minimize AMSE, we first optimize $\beta_T$. Since the variance term is independent of $\beta_T$, we differentiate the bias term in Eq. 42 with respect to $\beta_T$, apply asymptotic simplifications, and equate to zero. This yields the optimal bandwidth decay parameter:

$$
\beta_T = 1/2
$$

Substituting $\beta_T = 1/2$ into Eq. 43:

$$
AMSE(\hat{L}_n) = O(n^{-\alpha}) + O(n^{-2(1-\alpha)}) \quad (44)
$$

Finally, we optimize $\alpha$. The optimal convergence rate is achieved when the variance and bias terms decay at the same rate, or by balancing the exponents. The analysis identifies the optimal parameters as:

$$
\alpha = 2/3, \quad \beta_T = 1/2
$$

Resulting in the optimal convergence rate:

$$
AMSE(\hat{L}_n) = O(n^{-2/3}) \quad (45)
$$

### Pseudocode
The following Python-like pseudocode calculates the optimal parameters and convergence rate based on the derived formulas.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def beta_function(x, y):
    """
    Computes the Beta function B(x, y) = Gamma(x) * Gamma(y) / Gamma(x + y).
    Using standard math library for Gamma.
    """
    return math.gamma(x) * math.gamma(y) / math.gamma(x + y)

def analyze_tppb_convergence():
    """
    Analyzes the convergence of Transient Progressive Photon Beams.
    Returns optimal parameters and convergence rate.
    """
    # Step 1: Determine optimal beta_T
    # Derived from minimizing the bias term in Eq. 42.
    # The bias terms are O(n^(1-alpha)^(-2*beta_T)) and O(n^(1-alpha)^(2*beta_T-2)).
    # Equating the exponents for balanced decay or differentiating yields beta_T = 0.5.
    beta_T_opt = 0.5
    
    # Step 2: Determine optimal alpha
    # Substitute beta_T into AMSE (Eq. 44): AMSE = O(n^-alpha) + O(n^-2(1-alpha))
    # We want to minimize the dominant term. The optimal balance occurs when
    # the derivative of the combined error is minimized.
    # The source explicitly states alpha = 2/3 yields O(n^-2/3).
    alpha_opt = 2.0 / 3.0
    
    # Step 3: Calculate Convergence Rate
    # AMSE = O(n^-alpha) + O(n^-2(1-alpha))
    # With alpha = 2/3:
    # Term 1: O(n^-2/3)
    # Term 2: O(n^-2(1-2/3)) = O(n^-2(1/3)) = O(n^-2/3)
    # Both terms decay at rate 2/3.
    convergence_rate = alpha_opt
    
    return {
        "alpha": alpha_opt,
        "beta_T": beta_T_opt,
        "convergence_rate": convergence_rate,
        "amse_order": f"O(n^{-convergence_rate})"
    }

# Example usage
if __name__ == "__main__":
    results = analyze_tppb_convergence()
    print(f"Optimal Alpha: {results['alpha']}")
    print(f"Optimal Beta_T: {results['beta_T']}")
    print(f"Convergence Rate: {results['amse_order']}")
```

### Parameters and Typical Values
*   $\alpha$: Kernel radius decay parameter. Optimal value: $2/3$ [S1].
*   $\beta_T$: Kernel bandwidth decay parameter. Optimal value: $1/2$ [S1].
*   $\mathcal{T}_1$: Initial kernel bandwidth. Not given in the sources.
*   $R_{b1}$: Initial kernel radius. Not given in the sources.

### Edge Cases and Pitfalls
*   **Precision**: The asymptotic analysis assumes $n$ is sufficiently large for the $O(\cdot)$ notation to hold. For small $n$, the actual convergence may deviate from the theoretical rate.
*   **Independence Assumption**: The variance decomposition (Eq. 36) assumes $\Psi$ and $\epsilon_j$ are independent. If this assumption is violated in specific scene configurations, the variance estimate may be inaccurate.
*   **Beta Function Calculation**: Computing $B(\alpha, j)$ for large $j$ may require careful numerical implementation to avoid overflow or underflow, though the source does not specify a particular numerical method.

## Operating characteristics
*   **Time Complexity**: The analysis itself is $O(1)$ as it involves closed-form mathematical derivations. The application of these parameters in the rendering algorithm follows the complexity of [Transient Progressive Photon Beams](8973-transient-progressive-photon-beams.md).
*   **Memory Complexity**: The analysis requires no additional memory beyond storing the symbolic parameters.
*   **Strengths**: Provides a rigorous theoretical basis for parameter selection, ensuring optimal convergence rates ($O(n^{-2/3})$) for the radiance estimator.
*   **Limitations**: The analysis is asymptotic. It does not provide guidance for early iterations where the asymptotic behavior has not yet stabilized. It relies on the validity of the independence assumption between photon weights and error terms.

## Background
The convergence analysis for Transient Progressive Photon Beams was presented in the paper "Progressive Transient Photon Beams" [S1]. The variance approximation technique follows the work of Kaplanyan and Dachsbacher (2013) [KD13], which was originally developed for Progressive Photon Mapping [KZ11]. The base method, Progressive Photon Beams, was introduced by Jarosz et al. (2011) [JNT*11]. This analysis extends those concepts to the transient (time-of-flight) domain.

## Sources
[S1] Progressive Transient Photon Beams, https://doi.org/10.1111/cgf.13600, fetched 2026-10-27
