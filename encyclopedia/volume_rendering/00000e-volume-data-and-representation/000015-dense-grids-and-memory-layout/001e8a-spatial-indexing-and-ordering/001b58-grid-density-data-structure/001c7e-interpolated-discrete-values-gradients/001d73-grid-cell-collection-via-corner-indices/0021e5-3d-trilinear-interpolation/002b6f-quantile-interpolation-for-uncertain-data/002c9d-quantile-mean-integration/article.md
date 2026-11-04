# Quantile Mean Integration

Also called: Quantile Mean Technique

## Overview
Quantile Mean Integration is a volume rendering integration scheme for Direct Volume Rendering (DVR) that computes the expected fragment color by evaluating the Transfer Function (TF) at the mean intensity of each quantile, weighted by the discrete probability of that quantile [S1]. It differs from [Quantile Range Integration](5246-quantile-range-integration.md) by approximating the continuous distribution within each quantile with a single point mass (the mean) rather than integrating the TF over the quantile's intensity interval [S1]. This method is chosen when computational efficiency is prioritized over the accuracy of capturing intensity variance within quantiles, as it reduces the integration of a function over an interval to a single function evaluation per quantile [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A set of $q$ quantiles $Q_1, \dots, Q_q$ for an interpolated random variable $X$. Each quantile $Q_j$ is associated with a weight $w_j$ (representing the width or probability mass contribution) and a scalar value $qval$ (a constant scaling factor for the probability mass) [S1].
- **Input:** A Transfer Function $\text{TF}(x)$ mapping intensity $x$ to color/opacity [S1].
- **Output:** The expected fragment color $E(\text{TF}(X))$ [S1].

**Data Structures:**
- **Quantile List:** An array of $q$ elements. Each element $j$ stores:
  - $w_j$: The width/probability weight of the $j$-th quantile.
  - $m_j$: The mean intensity of the $j$-th quantile.
- **Constants:**
  - $qval$: The quantile value scaling factor (typically $1/q$ for uniform quantiles, but treated as a general parameter here) [S1].

**Steps:**
1. **Calculate Mean Intensities:** Determine the mean intensity $m_j$ for each quantile $Q_j$. This value $m_j$ represents the expected intensity value within the $j$-th quantile [S1].
2. **Compute Discrete Probabilities:** Calculate the unnormalized probability mass $p_j$ for each quantile using the formula:
   $$ p_j = qval \cdot w_j $$
   Compute the normalization constant $p$ by summing the unnormalized masses:
   $$ p = \sum_{k=1}^{q} qval \cdot w_k $$
   Calculate the discrete probability $\Pr(x = m_j)$ for each mean intensity:
   $$ \Pr(x = m_j) = \frac{p_j}{p} = \frac{qval \cdot w_j}{\sum_{k=1}^{q} qval \cdot w_k} $$
   Note: If $qval$ is constant across all quantiles, it cancels out in the normalization, simplifying to $\Pr(x = m_j) = \frac{w_j}{\sum_{k=1}^{q} w_k}$ [S1].
3. **Evaluate Transfer Function:** Sample the Transfer Function at each mean intensity $m_j$ to get $\text{TF}(m_j)$ [S1].
4. **Weight and Sum:** Compute the expected fragment color by summing the product of the TF value and the discrete probability for each quantile:
   $$ E(\text{TF}(X)) = \sum_{j=1}^{q} \text{TF}(m_j) \Pr(x = m_j) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quantile_mean_integration(quantiles, transfer_function):
    """
    Compute expected fragment color using Quantile Mean Integration.
    
    Args:
        quantiles: List of tuples (w_j, m_j) where w_j is the weight/width 
                   and m_j is the mean intensity of the j-th quantile.
        transfer_function: A function TF(x) returning color/opacity for intensity x.
        
    Returns:
        Expected fragment color.
    """
    q = len(quantiles)
    if q == 0:
        return 0.0  # Or default background color [G]
    
    # Step 1 & 2: Compute probabilities
    # qval is assumed to be a constant scaling factor. 
    # In many implementations, qval is uniform (e.g., 1/q), but we keep it general.
    # If qval is not provided explicitly, it is often implicit in w_j or normalized out.
    # Based on [S1], p_j = qval * w_j. Let's assume qval is passed or derived.
    # For standard uniform quantiles, qval = 1/q. If w_j are normalized probabilities, qval=1.
    # We assume qval is a scalar constant provided or 1.0 if not specified.
    qval = 1.0  # [G] Default assumption if not specified; adjust based on context
    
    numerator = 0.0
    weighted_tf_sum = 0.0
    
    for j in range(q):
        w_j, m_j = quantiles[j]
        
        # Unnormalized probability mass
        p_j = qval * w_j
        
        # Accumulate normalization constant
        numerator += p_j
        
        # Accumulate weighted TF evaluation
        # We store TF(m_j) * p_j to divide by p later
        weighted_tf_sum += transfer_function(m_j) * p_j
        
    # Normalization constant p
    p = numerator
    
    if p == 0:
        return 0.0  # [G] Avoid division by zero for empty or zero-weight quantiles
        
    # Step 3 & 4: Final expected value
    expected_color = weighted_tf_sum / p
    
    return expected_color
```

**Edge Cases and Pitfalls:**
- **Zero Weights:** If all $w_j = 0$, the normalization constant $p$ is zero, leading to division by zero. Guard against this [S1, G].
- **Precision:** Summing many small probabilities can lead to floating-point precision issues. Use Kahan summation or similar techniques if $q$ is large [G].
- **Mean Calculation:** The accuracy of the result depends entirely on the accuracy of $m_j$. If $m_j$ is poorly estimated (e.g., from a skewed distribution within the quantile), the integration will be biased [S1].

## Subtopics

### Quantile Range Integration
An integration scheme for DVR where all intensities contained within a quantile are integrated with the Transfer Function (TF) [S1]. Unlike Quantile Mean Integration, which uses a single point (the mean), this method computes the integral of the TF over the entire interval $[start_j, end_j]$ of each quantile [S1]. The expected fragment color is calculated as:
$$ E(\text{TF}(X)) = \sum_{j=1}^{q} \frac{qval}{w_j} \int_{start_j}^{end_j} \text{TF}(x) dx $$
where $w_j$ is the width of the $j$-th quantile [S1]. This is equivalent to averaging the TFs for each quantile and performing a weighted sum where the weight is $qval$ [S1]. It is more computationally expensive due to the integration step but provides a more accurate representation of the intensity distribution within each quantile [S1].

## Operating characteristics
- **Time Complexity:** $O(q)$ per sample, where $q$ is the number of quantiles. This is significantly faster than Quantile Range Integration if the TF evaluation is cheaper than integration, as it avoids numerical integration over intervals [S1].
- **Memory Complexity:** The memory consumed by input data for the nonparametric DVR framework is $L \times M \times N \times (q+1)$, where $L \times M \times N$ is the grid size and $q+1$ represents the number of boundaries defining quantiles at each grid vertex [S1]. The integration step itself requires $O(1)$ additional memory per sample [S1].
- **Strengths:** Computationally efficient; simple to implement; suitable for real-time rendering where integration over intervals is too slow [S1].
- **Limitations:** Approximates the distribution within each quantile with a single value, potentially losing information about variance and skewness within the quantile [S1]. Less accurate than Quantile Range Integration for highly non-linear Transfer Functions [S1].
- **When to use:** When performance is critical and the Transfer Function is relatively linear or smooth within quantile intervals [S1].
- **When not to use:** When high fidelity in representing uncertainty is required, or when the Transfer Function has sharp discontinuities within quantile intervals [S1].

## Background
Quantile Mean Integration was proposed in the context of Direct Volume Rendering with Nonparametric Models of Uncertainty [S1]. It is part of a framework that uses Kernel Density Estimation (KDE) to model uncertainty at grid vertices and reduces continuous distributions to quantile representations for efficient rendering [S1]. The method was introduced to provide a computationally viable alternative to integrating the full probability density function over the Transfer Function, balancing accuracy and performance [S1].

## Sources
[S1] Direct Volume Rendering with Nonparametric Models of Uncertainty, https://doi.org/10.1109/tvcg.2020.3030394, fetched 2026-10-26
