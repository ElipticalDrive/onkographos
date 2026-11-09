# Target Feature Density Calculation

Also called: Target feature density, $\rho^*(x)$

## Overview
Target feature density calculation defines a dynamic target distribution $\rho^*(x)$ used to steer the current feature density $\rho(x)$ during the training of adaptive grid-based scene representations [S1]. It solves the problem of allocating computational resources (feature grid density) efficiently by increasing density in regions with high reconstruction error and decreasing it in low-error regions [S1]. It sets itself apart from static density initialization methods by using a logarithmic scaling based on relative error, ensuring that regions with very large relative errors receive disproportionately more importance [S1].

## How it works
**Inputs:**
*   $x$: A spatial coordinate.
*   $h(x)$: The model's reconstruction error at location $x$.
*   $\bar{h}$: The average model error over a batch of coordinates.
*   $\rho(x)$: The current unscaled feature density at $x$.
*   $\varepsilon$: A small constant to prevent division by zero and numerical instability.

**Outputs:**
*   $\rho^*(x)$: The target feature density at $x$.

**Data Structures:**
*   The feature density $\rho(x)$ is typically stored in a grid or hash table structure associated with the scene representation.
*   $\rho_{\text{scaled}}(x)$ is a temporary normalized value derived from $\rho(x)$.

**Steps:**
1.  **Normalize Current Density:** Compute the scaled feature density $\rho_{\text{scaled}}(x)$ by dividing the current density $\rho(x)$ by the sum of densities over the entire set of coordinates $X$:
    $$ \rho_{\text{scaled}}(x) = \frac{\rho(x)}{\sum_{x' \in X} \rho(x')} $$
    This rescales the density to lie between 0 and 1 [S1].

2.  **Calculate Relative Error Factor:** Compute the ratio of the average error to the local error, adding $\varepsilon$ to the denominator:
    $$ \text{ratio}(x) = \frac{\bar{h}}{h(x) + \varepsilon} $$

3.  **Compute Logarithmic Scaling:** Calculate the logarithm of the scaled density plus $\varepsilon$:
    $$ \text{log\_density}(x) = \log(\rho_{\text{scaled}}(x) + \varepsilon) $$

4.  **Compute Target Density:** Exponentiate the product of the relative error factor and the log-density:
    $$ \rho^*(x) = \exp\left( \text{ratio}(x) \cdot \text{log\_density}(x) \right) $$
    Substituting the terms back:
    $$ \rho^*(x) = \exp\left( \frac{\bar{h}}{h(x) + \varepsilon} \log(\rho_{\text{scaled}}(x) + \varepsilon) \right) \quad (8) $$
    [S1]

**Parameters:**
*   $\varepsilon$: A small number to avoid dividing by zero. The exact value is not specified in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** The term $h(x) + \varepsilon$ prevents division by zero if the local error is zero. The term $\rho_{\text{scaled}}(x) + \varepsilon$ prevents $\log(0)$ [S1].
*   **Average Error:** If $h(x) = \bar{h}$, the ratio is 1, and $\rho^*(x) = \rho_{\text{scaled}}(x)$, meaning the target density is unchanged relative to the scaled current density [S1].
*   **High Error Regions:** If $h(x) \gg \bar{h}$, the ratio is small, but since $\log(\rho_{\text{scaled}}(x))$ is negative (as $\rho_{\text{scaled}} < 1$), the exponent becomes less negative, resulting in a higher $\rho^*(x)$ compared to $\rho_{\text{scaled}}(x)$. This increases density in high-error regions [S1].
*   **Low Error Regions:** If $h(x) \ll \bar{h}$, the ratio is large, making the exponent more negative, resulting in a lower $\rho^*(x)$. This decreases density in low-error regions [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_target_feature_density(h_local, h_avg, rho_current, rho_sum, epsilon):
    """
    Calculate the target feature density rho*(x) for a given coordinate.
    
    Args:
        h_local (float): Model error at location x, h(x).
        h_avg (float): Average model error over a batch, h_bar.
        rho_current (float): Current unscaled feature density, rho(x).
        rho_sum (float): Sum of feature densities over the entire set X.
        epsilon (float): Small constant to avoid numerical issues.
        
    Returns:
        float: Target feature density rho*(x).
    """
    # Step 1: Normalize current density
    # rho_scaled(x) = rho(x) / sum(rho)
    rho_scaled = rho_current / (rho_sum + epsilon)
    
    # Step 2 & 3 & 4: Compute target density
    # rho*(x) = exp( (h_bar / (h(x) + eps)) * log(rho_scaled(x) + eps) )
    
    # Avoid log(0) and division by zero
    log_rho_scaled = math.log(rho_scaled + epsilon)
    error_ratio = h_avg / (h_local + epsilon)
    
    target_density = math.exp(error_ratio * log_rho_scaled)
    
    return target_density
```
[G] `math.log` and `math.exp` are standard mathematical functions. `epsilon` is a hyperparameter, typically $10^{-6}$ or similar, though not explicitly given in the source [S1].

## Subtopics

### Feature Density Loss (KL Divergence)
A loss function that quantifies the difference between the current feature density and the target feature density by viewing it as a distribution-matching problem [S1]. It is calculated as the Kullback–Leibler (KL) divergence from the current scaled feature density to the target feature density [S1]. The formula is:
$$ \rho_{\text{density}} = \frac{1}{|X|} \sum_{x \in X} \rho^*(x) \log \left( \frac{\rho^*(x)}{\rho_{\text{scaled}}(x)} \right) \quad (9) $$
where $|X|$ is the number of coordinates in $X$ [S1]. This loss provides gradients to update the parameters of each feature grid to match the target density [S1].

## Operating characteristics
*   **Complexity:** The calculation of $\rho^*(x)$ is $O(1)$ per coordinate. The calculation of the total sum for normalization is $O(|X|)$. The KL divergence loss is $O(|X|)$.
*   **Performance:** The method effectively steers the feature grid to allocate resources where they are needed most, improving reconstruction quality in complex regions while maintaining efficiency in simple regions [S1].
*   **Strengths:** It dynamically adapts to the scene's complexity. It avoids over-allocating resources in uniform areas and under-allocating in detailed areas [S1].
*   **Limitations:** It requires computing the average error over a batch, which adds a synchronization step. The logarithmic scaling can be sensitive to the choice of $\varepsilon$ if errors are very small [S1].
*   **When to use:** When training adaptive grid-based scene representations where computational resources are limited and need to be allocated efficiently [S1].
*   **When not to use:** When the scene is static and uniform, or when the error metric is not reliable [S1].

## Background
The method was introduced in the paper "Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization" by [S1]. It is part of the APMGSRN training routine, specifically designed to work with flat-top Gaussian feature densities [S1]. The approach builds on the concept of using density-based allocation for adaptive grids, extending it with a differentiable loss function based on KL divergence [S1].

## Sources
[S1] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
