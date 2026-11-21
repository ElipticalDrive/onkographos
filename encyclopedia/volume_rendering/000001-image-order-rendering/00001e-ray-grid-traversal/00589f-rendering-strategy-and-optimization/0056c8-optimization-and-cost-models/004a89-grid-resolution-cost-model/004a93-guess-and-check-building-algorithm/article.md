# Guess and Check Building Algorithm
Also called: Sampling-based building algorithm
## Overview
The Guess and Check Building Algorithm is a method for determining the optimal grid resolution parameter $m$ in volume rendering by sampling a cost function over a theoretically bounded interval to minimize average traversal time [S1]. It distinguishes itself from exhaustive search methods by guaranteeing linear overall building time while leveraging theoretical bounds to restrict the search space [S1].
## How it works
**Inputs and Outputs:**
- Input: The number of geometric primitives $N$ in the scene.
- Output: The optimal subdivision factor $m$ that minimizes the cost function $T$.

**Data Structures:**
No complex data structures are required. The algorithm operates on scalar values for $m$ and the cost function $T$.

**Steps:**
1.  **Determine the Search Interval:** Calculate the lower and upper bounds for $m$ using theoretical results derived from the cost model assumptions. For typical constants where $T_{setup} \approx 1.0$, $T_{step} \approx 0.1$, and $T_{inter} \approx 0.5$, the interval is:
    $$ m \in \left( (10N)^{1/3}, 5N^{2/3} \right) $$
    [S1]
2.  **Sample the Cost Function:** Select a set of discrete values for $m$ within the bounded interval. For each candidate $m$, evaluate the cost function $T$. The cost function $T$ depends on the number of occupied cells $N_m$, which can be evaluated in linear time for a given $m$ by performing primitive counting during the build [S1].
3.  **Select the Minimum:** Compare the evaluated cost values and select the $m$ that yields the minimum cost $T$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def guess_and_check_build(N, T_setup, T_step, T_inter):
    """
    Determine optimal grid resolution m for N primitives.
    """
    # Step 1: Determine bounds based on theoretical analysis [S1]
    # Lower bound: (10 * N)^(1/3)
    # Upper bound: 5 * N^(2/3)
    lower_bound = (10 * N) ** (1/3)
    upper_bound = 5 * (N ** (2/3))
    
    # Step 2: Sample the cost function
    # A naive approach samples a constant number of points in the interval [S1]
    # The cost function T is minimized. T depends on N_m (occupied cells).
    # N_m is evaluated in linear time for a given m via primitive counting [S1].
    
    # Define a simple cost function structure based on standard grid traversal models
    # T = T_setup + T_step * (N_m / N) + T_inter * (something related to empty space)
    # Note: The exact formula for T is not provided in the sources, only that it is minimized [S1].
    # We assume a generic evaluation function exists.
    
    best_m = None
    min_cost = float('inf')
    
    # Sample points in the interval. The number of samples is constant [S1].
    num_samples = 10 # Example constant, not specified in sources
    step_size = (upper_bound - lower_bound) / num_samples
    
    for i in range(num_samples + 1):
        m_candidate = lower_bound + i * step_size
        if m_candidate < lower_bound or m_candidate > upper_bound:
            continue
            
        # Evaluate cost for this m
        # N_m can be computed in linear time for this m [S1]
        # This requires a primitive counting pass or similar linear operation
        N_m = compute_occupied_cells(N, m_candidate) # [G] Implementation detail for primitive counting
        
        # Compute cost T
        # Formula not explicitly given in sources, but T is minimized [S1]
        # Assuming a standard SAH-like cost structure for grids
        cost = T_setup + T_step * (N_m / N) + T_inter * (1 - N_m / N) # [G] Placeholder for actual T formula
        
        if cost < min_cost:
            min_cost = cost
            best_m = m_candidate
            
    return best_m
```

**Edge Cases and Pitfalls:**
- The theoretical bounds assume specific values for $T_{setup}$, $T_{step}$, and $T_{inter}$. If these constants differ significantly in the target hardware or scene, the bounds may need adjustment [S1].
- The algorithm assumes that $N_m$ can be evaluated in linear time for any given $m$ [S1].

## Operating characteristics
- **Time Complexity:** Linear overall building time $O(N)$ [S1]. This is achieved because the number of samples is constant, and evaluating the cost for each sample (including primitive counting for $N_m$) takes linear time [S1].
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Guarantees linear build time while finding a near-optimal resolution [S1].
- **Limitations:** Relies on theoretical bounds that may not be tight for all scene types or hardware configurations [S1].

## Background
The algorithm is described in the context of grid-based ray tracing, specifically as a method to optimize grid resolution [S1]. It is compared to similar sampling-based approaches used in SAH kd-tree building, such as the conservatively sub-sampling method by Popov et al. [PGSS06] and the adaptive error-bounded heuristic by Hunt et al. [HMS06] [S1]. The work is attributed to Biagio Cosenza in "A Survey on Exploiting Grids for Ray Tracing" [S1].

## Sources
[S1] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S2] Grids in Ray Tracing: A Comprehensive Survey, https://www.scribd.com/document/349616186/GG-400, fetched 2026-10-25 (unofficial)
