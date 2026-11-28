# Orientation-Selective Sampling Strategies

Also called: Steerable Pyramid Sampling Strategies

## Overview
Orientation-Selective Sampling Strategies optimize steerable pyramid volume rendering by modulating the evaluation of orientation bins based on the alignment between the viewing ray direction and the kernel's frequency modulation direction [S1]. Unlike deterministic approaches that evaluate all orientations or uniform sampling that ignores geometric alignment, this method uses the alignment metric to cull or probabilistically select bins, balancing rendering quality against performance [S1]. It is specifically designed for steerable pyramid representations where orientation selectivity is governed by the central frequency $f_0$ [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray direction vector $\vec{v}$, set of $K$ orientation bins $\{\mathbf{o}_1, \dots, \mathbf{o}_K\}$, central frequency $f_0$, alignment threshold $\delta$, and power-law parameters $a, b, \beta$ [S1].
*   **Output:** A subset of orientation bins to evaluate, or a probability distribution for stochastic selection [S1].

**Data Structures:**
*   **Orientation Bins:** A list of $K$ unit vectors $\mathbf{o}_i$ representing the preferred orientations of the steerable pyramid filters [S1].
*   **Alignment Metric:** A scalar $a_i$ for each bin, computed as the absolute dot product $a_i = |\vec{v} \cdot \mathbf{o}_i|$ [S1].

**Contribution Weight Formula:**
The contribution weight $w_i$ for each orientation bin is determined by its alignment with the ray and the central frequency:
$$
w_i = \exp\left(-\frac{f_0^2 a_i^2}{2}\right)
$$
The total weight is $W = \sum_{i=1}^K w_i$ [S1].

**Sampling Strategies:**
The method defines several strategies for selecting which bins to render. Let $K$ be the total number of bins.

1.  **Deterministic:**
    *   **Action:** Evaluate all $K$ bins.
    *   **Probability:** $1$ (for the set).
    *   **Weight:** $1$ [S1].

2.  **Threshold Culling:**
    *   **Action:** Evaluate only bins where $a_i \leq \delta$. Bins with $a_i > \delta$ are culled.
    *   **Probability:** Undefined (deterministic subset).
    *   **Weight:** $1$ [S1].
    *   **Note:** Lower $\delta$ results in more aggressive culling, potentially degrading to evaluating only the single best-aligned bin [S1].

3.  **Uniform Sampling:**
    *   **Action:** Select one bin $i$ uniformly at random.
    *   **Probability:** $P(i) = K^{-1}$.
    *   **Weight:** $K$ [S1].

4.  **Importance Sampling:**
    *   **Action:** Select one bin $i$ with probability proportional to $w_i$.
    *   **Probability:** $P(i) = \frac{w_i}{W}$.
    *   **Weight:** $\frac{W}{w_i}$ [S1].

5.  **Threshold + Uniform:**
    *   **Action:** Deterministically render all bins where $a_i \leq \delta$. For bins where $a_i > \delta$, uniformly sample one bin.
    *   **Probability:** For a selected bin $j$ in the "above-threshold" set, $P(j) = N_{\text{above}}^{-1}$, where $N_{\text{above}}$ is the count of bins with $a_i > \delta$.
    *   **Weight:** $N_{\text{above}}$ [S1].
    *   **Control Variate Note:** In "Threshold + Control Variate" mode, bins below the threshold ($a_i \leq \delta$) are assigned a fixed weight of $1$ [S1].

6.  **Power-law Weighted:**
    *   **Action:** Sample bins proportionally to a power-law function of the alignment.
    *   **Probability:** $P(x) \propto x^{(1-\beta)^{-1}}$, where $x$ is the alignment metric.
    *   **Weight:** $\left(b^{(1-\beta)} - a^{(1-\beta)}\right)^{-1}$ [S1].
    *   **Parameters:** $a, b$ are bounds for the power-law distribution; $\beta$ is the tuning parameter [S1].

7.  **Uniform + Control Variate:**
    *   **Action:** Use Gaussian-based control variates for certain levels; sample remaining levels uniformly.
    *   **Probability:** $(\mathcal{P} - 1)^{-1}$, where $\mathcal{P}$ is the total number of levels.
    *   **Weight:** $\mathcal{P} - 1$ [S1].

8.  **Power-law + Control Variate:**
    *   **Action:** Use Gaussians as control variates; sample other levels via power-law.
    *   **Probability:** $x^{(1-\beta)^{-1}}$.
    *   **Weight:** $\left(b^{(1-\beta)} - a^{(1-\beta)}\right)^{-1}$ [S1].

9.  **Power-law + CV (Accumulative):**
    *   **Action:** Sample level $k$ via power-law; render all levels $0$ to $k$.
    *   **Probability:** $x^{(1-\beta)^{-1}}$.
    *   **Weight:** $\left(1 - \left(\frac{j}{\beta}\right)^{1-\beta}\right)^{-1}$ [S1].

**Pseudocode:**
The following Python-like pseudocode implements the computation of alignment weights and the selection logic for the primary strategies.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def compute_alignment_weights(ray_direction, orientation_bins, f0):
    """
    Computes the contribution weight w_i for each orientation bin.
    
    Args:
        ray_direction (list): 3D vector [vx, vy, vz]
        orientation_bins (list): List of K 3D vectors [o1, o2, ..., oK]
        f0 (float): Central frequency of the steerable pyramid
    
    Returns:
        list: Weights w_i for each bin
    """
    weights = []
    for o in orientation_bins:
        # Compute alignment a_i = |v . o|
        dot_product = ray_direction[0]*o[0] + ray_direction[1]*o[1] + ray_direction[2]*o[2]
        a_i = abs(dot_product)
        
        # Compute weight w_i = exp(-f0^2 * a_i^2 / 2)
        w_i = math.exp(-(f0**2 * a_i**2) / 2)
        weights.append(w_i)
    return weights

def select_orientation_bins(strategy, ray_direction, orientation_bins, f0, delta, beta=0.5, a_bound=0.0, b_bound=1.0):
    """
    Selects orientation bins based on the specified strategy.
    
    Args:
        strategy (str): One of 'deterministic', 'threshold_culling', 'uniform', 
                        'importance', 'threshold_uniform', 'power_law', 
                        'uniform_cv', 'power_law_cv', 'power_law_cv_accum'
        ray_direction (list): 3D vector
        orientation_bins (list): List of K 3D vectors
        f0 (float): Central frequency
        delta (float): Alignment threshold for culling
        beta (float): Power-law tuning parameter
        a_bound (float): Lower bound for power-law
        b_bound (float): Upper bound for power-law
        
    Returns:
        list: Indices of selected bins
    """
    K = len(orientation_bins)
    weights = compute_alignment_weights(ray_direction, orientation_bins, f0)
    W = sum(weights)
    
    if strategy == 'deterministic':
        return list(range(K))
    
    elif strategy == 'threshold_culling':
        selected = []
        for i, o in enumerate(orientation_bins):
            dot_product = ray_direction[0]*o[0] + ray_direction[1]*o[1] + ray_direction[2]*o[2]
            a_i = abs(dot_product)
            if a_i <= delta:
                selected.append(i)
        return selected
    
    elif strategy == 'uniform':
        idx = random.randint(0, K - 1)
        return [idx]
    
    elif strategy == 'importance':
        # Sample proportionally to weights
        idx = random.choices(range(K), weights=weights, k=1)[0]
        return [idx]
    
    elif strategy == 'threshold_uniform':
        selected = []
        above_threshold_indices = []
        for i, o in enumerate(orientation_bins):
            dot_product = ray_direction[0]*o[0] + ray_direction[1]*o[1] + ray_direction[2]*o[2]
            a_i = abs(dot_product)
            if a_i <= delta:
                selected.append(i)
            else:
                above_threshold_indices.append(i)
        
        if above_threshold_indices:
            idx = random.choice(above_threshold_indices)
            selected.append(idx)
        return selected
    
    elif strategy == 'power_law':
        # Sample based on power-law of alignment
        # Note: Exact implementation of power-law sampling on alignment metric
        # requires normalizing the distribution over the range [a_bound, b_bound]
        # For simplicity, we assume alignment metrics are normalized or mapped.
        # This is a placeholder for the specific power-law sampling logic.
        # In practice, one would sample x from the power-law distribution and
        # find the bin with closest alignment.
        # Given the complexity, this strategy is often approximated or precomputed.
        # Here we return an empty list to indicate complexity not fully detailed in sources.
        return [] 
    
    elif strategy == 'uniform_cv':
        # Simplified: Return all but one, or handle control variate logic
        # Sources indicate sampling remaining levels uniformly.
        # Assuming P is total levels, sample P-1 uniformly.
        # This requires knowledge of level structure not present in basic bin list.
        return []
        
    elif strategy == 'power_law_cv':
        return []
        
    elif strategy == 'power_law_cv_accum':
        return []
        
    else:
        raise ValueError(f"Unknown strategy: {strategy}")
```

**Parameters:**
*   $K$: Number of orientation bins [S1].
*   $\delta$: Alignment threshold for culling. Lower values result in more aggressive dropping of orientation bins [S1].
*   $\beta$: Decay rate parameter for power-law weighting [S1].
*   $f_0$: Central frequency of the steerable pyramid. High $f_0$ increases orientation selectivity [S1].

**Edge Cases and Pitfalls:**
*   **Degenerate Culling:** If $\delta$ is set too low, the threshold culling strategy may degenerate into evaluating only the single highest-contributing bin, potentially losing necessary frequency information [S1].
*   **High Frequency Volumes:** Volumes with high $f_0$ exhibit high orientation selectivity. For these volumes, threshold culling is particularly effective, but requires careful tuning of $\delta$ to avoid artifacts [S1].

## Subtopics

### [Stochastic Orientation-selective Path Tracing](22760-stochastic-orientation-selective-path-tracing.md)
Applies importance sampling to angular bins, stochastically selecting orientations proportional to their contribution to reduce traversal time at the cost of increased variance.

## Operating characteristics
*   **Time Complexity:** Depends on the strategy. Deterministic is $O(K)$. Threshold culling is $O(K)$ to compute alignments but reduces rendering cost by a factor proportional to the culling rate. Importance sampling is $O(K)$ to compute weights and $O(1)$ to sample.
*   **Memory Complexity:** $O(K)$ to store orientation bins and weights [S1].
*   **Performance:** Volumes with high $f_0$ benefit significantly from threshold culling due to high orientation selectivity [S1].
*   **Strengths:** Direct trade-off between quality and performance via $\delta$; reduces redundant evaluations of misaligned orientations [S1].
*   **Limitations:** Stochastic strategies introduce variance; deterministic culling introduces bias if $\delta$ is too aggressive [S1].

## Background
The method originates from the "Gabor Fields: Orientation-Selective Level-of-Detail for Volume Rendering" paper [S1]. It is used in steerable pyramid volume rendering to optimize the evaluation of orientation-selective filters [S1].

## Sources
[S1] Gabor Fields: Orientation-Selective Level-of-Detail for Volume Rendering, https://arxiv.org/abs/2602.05081, fetched 2026-10-27
