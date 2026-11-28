# Ray-Adaptive Supersegment Generation

Also called: Automatic gamma determination, Determining the supersegment termination parameter gamma, Homogeneity-based supersegment generation, Ray-Adaptive Generation of Supersegments

## Overview
This method automatically determines the sensitivity parameter $\gamma$ for supersegment termination for each ray independently, ensuring a maximum number of supersegments $N_S$ per ray without manual tuning [S1]. It solves the problem of visual artifacts caused by a constant, manually-tuned $\gamma$ that is either too high (insufficient resolution) or too low (exhausting the budget and causing "smearing" artifacts) [S1]. Unlike [Adaptive Sampling](1385-adaptive-sampling.md) which adjusts sample density based on error metrics, this method adapts the structural granularity of pre-integrated volume data (supersegments) to fit a strict budget constraint [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Target maximum number of supersegments $N_S$, convergence threshold $\varepsilon$, tolerance margin $\delta$, and the volume data.
*   **Output:** The optimal sensitivity parameter $\gamma$ for the current ray.

**Data Structures:**
The algorithm maintains a search range for $\gamma$ using two floating-point variables `low` and `high`, and a current candidate $\gamma$. It also tracks iteration state with boolean flags `firstIteration` and `found`, and an integer counter `n` for the number of generated supersegments.

**Termination Criterion:**
A sample is merged into the current supersegment $\mathbb{S}$ unless it differs from the pre-multiplied color of $\mathbb{S}$ by more than $\gamma$ [S1]. The criterion is defined as:
$$ \tau : \gamma > \|C(\mathbb{S})\alpha(\mathbb{S}) - C'\alpha'\|_2 $$
where $C'$ and $\alpha'$ are the color and length-adjusted opacity of the next sample, and $C(\mathbb{S})\alpha(\mathbb{S})$ is the pre-multiplied color of the current supersegment [S1]. The distance metric is the $L_2$ norm between 3-element pre-multiplied color vectors [S1].

**Parameters:**
*   $\gamma$: Sensitivity parameter. Range: $[0, \sqrt{3}]$. The upper bound $\sqrt{3}$ is the maximum possible $L_2$ distance between two normalized 3D vectors [S1].
*   $\varepsilon$: Convergence threshold for the bisection search range. Typical value: $10^{-6}$ [S1].
*   $\delta$: Tolerance margin for the lower bound check. Empirically determined to be 15% of $N_S$ ($\delta = 0.15 N_S$) to balance performance and quality [S1].
*   Initial $\gamma$: $10^{-5}$, used to quickly identify homogeneous or empty rays [S1].

**Algorithm Steps:**
The method uses an iterative bisection search to find $\gamma$ such that the number of generated supersegments $n$ is close to $N_S$ but never exceeds it [S1].

1.  Initialize `low` = 0, `high` = $\sqrt{3}$, $\gamma$ = $10^{-5}$, `firstIteration` = TRUE, `found` = FALSE.
2.  While `!found`:
    a.  Sample the volume along the ray using the current $\gamma$ and count the number of generated supersegments, storing the result in `n`.
    b.  If `abs(high - low)` < $\varepsilon$:
        i.  Set `found` = TRUE.
        ii. Set $\gamma$ = (`n` == 0) ? `low` : `high`. This selects `low` if 0 supersegments are generated to prevent degenerate cases, otherwise `high` (which guarantees $n < N_S$) [S1].
    c.  Else if `n` > $N_S$:
        i.  Set `low` = `high`. (Note: The source algorithm sets `low = high` here, effectively narrowing the search or handling the case where the current $\gamma$ is too small. However, standard bisection would set `low = gamma`. The source code explicitly sets `low = high` [S1]).
    d.  Else if `n` < $N_S - \delta$:
        i.  Set `high` = `low`. (Similarly, this narrows the range based on the tolerance [S1]).
    e.  Else:
        i.  Set `found` = TRUE. (The count is within the acceptable tolerance $[N_S - \delta, N_S]$).
    f.  If `firstIteration`:
        i.  Set `firstIteration` = FALSE.
        ii. If `n` < $N_S$:
            1.  Set `found` = TRUE. (Optimization: if the initial small $\gamma$ yields fewer than $N_S$ supersegments, the ray is homogeneous/empty, and we can terminate early [S1]).
    g.  If `!found`:
        i.  Update $\gamma$ = (`low` + `high`) / 2.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def determine_gamma(N_S, epsilon=1e-6, delta=None):
    if delta is None:
        delta = 0.15 * N_S
    
    low = 0.0
    high = math.sqrt(3.0)
    gamma = 1e-5
    first_iteration = True
    found = False
    
    while not found:
        # Sample volume and count supersegments for current gamma
        n = num_supersegments_generated(gamma)
        
        if abs(high - low) < epsilon:
            found = True
            # If 0 supersegments, pick low to avoid degenerate 0-width segments
            # Otherwise pick high to ensure n < N_S (preventing smearing)
            if n == 0:
                gamma = low
            else:
                gamma = high
        elif n > N_S:
            low = high
        elif n < N_S - delta:
            high = low
        else:
            found = True
            
        if first_iteration:
            first_iteration = False
            if n < N_S:
                found = True
                
        if not found:
            gamma = (low + high) / 2.0
            
    return gamma
```

**Edge Cases and Pitfalls:**
*   **Smearing Artifacts:** If the supersegment budget $N_S$ is exhausted, the last supersegment must contain all remaining data, causing visual smearing [S1]. The algorithm prevents this by ensuring the search never allows more than $N_S$ supersegments (by selecting `high` when the range converges, as `high` produces fewer supersegments) [S1].
*   **Degenerate Zero Supersegments:** If the search converges to a $\gamma$ that produces 0 supersegments, the algorithm selects `low` instead of `high` to prevent this degenerate case [S1].
*   **Homogeneous/Empty Rays:** Initializing $\gamma$ to $10^{-5}$ allows the algorithm to quickly detect rays passing through empty or homogeneous regions. If the first iteration generates fewer than $N_S$ supersegments, the ray terminates immediately, freeing computational resources [S1].

## Operating characteristics
*   **Time Complexity:** The method requires multiple passes through the volume data for each ray (one per bisection iteration) to determine $\gamma$ [S1]. The number of iterations depends on the convergence threshold $\varepsilon$ and the initial range $[0, \sqrt{3}]$.
*   **Memory Complexity:** Minimal additional memory per ray, only storing the search state variables [S1].
*   **Performance:** Measured VDI generation times can be lower for larger $N_S$ in some cases because the $\gamma$ search converges faster (fewer iterations needed to find a suitable $\gamma$ that yields fewer supersegments) [S1].
*   **Strengths:** Eliminates manual tuning of $\gamma$, enabling remote VDI generation without user intervention [S1]. Generates more accurate VDIs by adapting resolution along each ray [S1].
*   **Limitations:** Requires multiple volume samples per ray during the initialization phase, increasing computational cost compared to a fixed $\gamma$ approach [S1].

## Background
This method was proposed by Frey et al. to address the limitations of the homogeneity criterion $\tau$ introduced in their previous work [10], where $\gamma$ was constant across all rays and required careful manual tuning [S1]. The method was published in "Efficient Raycasting of Volumetric Depth Images for Remote Visualization of Large Volumes at High Frame Rates" [S1].

## Sources
[S1] Efficient Raycasting of Volumetric Depth Images for Remote Visualization of Large Volumes at High Frame Rates, https://doi.org/10.1109/pacificvis56936.2023.00014, fetched 2026-10-24
