# Feature Query

Also called: Function Field Feature Query, Range Query

## Overview
Feature Query identifies spatial regions in a function field where 1-dimensional functions are similar [S1, unofficial]. It pares down large scientific datasets by identifying regions of interest via user-specified selection queries, constraining computational complexity to the number of returned items rather than processing the whole dataset [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A dataset containing $m$ samples per function, where each spatial point $p$ has an associated 1-dimensional function $f^p$ [S1, unofficial].
*   **Output:** A binary mask or subset of spatial points $p$ that satisfy the query constraints [S1, unofficial].

**Data Structures**
The query is defined by a set of constraints over a closed interval $I$ [S1, unofficial]. The specific layout includes three modifiable curves:
1.  **Minimum Constraint Curve:** Defines the lower bound for each sample index [S1, unofficial].
2.  **Maximum Constraint Curve:** Defines the upper bound for each sample index [S1, unofficial].
3.  **Sample Weights Curve:** Defines the importance of each sample index [S1, unofficial].
Control points for these curves can be added, removed, and manipulated by the user [S1, unofficial].

**Algorithm Steps**
1.  **Exploration:** The user explores the function field using distance field renderings to identify potential features [S1, unofficial].
2.  **Constraint Definition:** The user sketches a pair of curves to define the feature query's minimum and maximum constraints [S1, unofficial]. Let $Q_i$ be the minimum-maximum interval constraint for sample index $i$, where $i = 1, \dots, m$ [S1, unofficial].
3.  **Evaluation:** For each point $p$ in the dataset with function $f^p$, evaluate the condition:
    $$ f^p_i \in Q_i \quad \forall i \in \{1, \dots, m\} $$
    A point $p$ is part of the feature if and only if its function value at every sample index $i$ falls within the corresponding interval $Q_i$ [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def feature_query(dataset, m, min_curve, max_curve):
    """
    dataset: List of point objects, each with a .function attribute (list of m floats).
    m: Number of samples per function.
    min_curve: Callable or list returning the lower bound Q_min[i] for sample index i.
    max_curve: Callable or list returning the upper bound Q_max[i] for sample index i.
    
    Returns:
    List of point objects that satisfy the query constraints.
    """
    matching_points = []
    
    # Pre-calculate constraint intervals for all sample indices
    # Q_i corresponds to the closed interval [Q_min[i], Q_max[i]]
    constraints = []
    for i in range(m):
        q_min = min_curve(i) if callable(min_curve) else min_curve[i]
        q_max = max_curve(i) if callable(max_curve) else max_curve[i]
        constraints.append((q_min, q_max))
        
    for point in dataset:
        is_match = True
        for i in range(m):
            f_i = point.function[i]
            q_min, q_max = constraints[i]
            
            # Check if f^p_i is within the closed interval Q_i
            if not (q_min <= f_i <= q_max):
                is_match = False
                break
                
        if is_match:
            matching_points.append(point)
            
    return matching_points
```

**Parameters**
*   $m$: The number of samples per function in the dataset [S1, unofficial].
*   $Q_i$: The minimum-maximum interval constraint for sample $i$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** The constraints are defined over closed intervals, so implementation must handle floating-point comparisons carefully to ensure boundary inclusion matches the mathematical definition $f^p_i \in Q_i$ [S1, unofficial].
*   **Degenerate Input:** If the minimum curve exceeds the maximum curve at any index $i$, the interval $Q_i$ is empty, and no points will match for that index [G].

## Operating characteristics
*   **Time Complexity:** The computational complexity is constrained to the number of items returned by the query, rather than the size of the entire dataset [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Measured Performance:** Tested on a 2.6 GHz Mobile Pentium 4-M laptop with 1.0 GB RAM and nVidia GeForce 4200 Go graphics card [S1, unofficial]:
    *   Hyperspectral image query execution times were 320ms (Water), 218ms (Golf Courses), 257ms (Evaporation Ponds), and 23ms (High SO4 Concentration) [S1, unofficial].
    *   Distance fields were generated at ~6 per second for hyperspectral images and ~25 per second for CRPAQS [S1, unofficial].
*   **Strengths:** Allows for the extraction of reusable query constraints across multiple datasets and enables rapid identification and visualization of regions of interest [S1, unofficial].
*   **Limitations:** Performance depends on the number of samples $m$ and the dataset size during the initial exploration phase (distance field generation) [S1, unofficial].
*   **When to use/not:** Use when intelligently paring down large scientific datasets to analyze smaller, meaningful subsets is required; avoid when processing the entire dataset simultaneously is necessary [S1, unofficial].

## Background
The concept of coupling visualization with high-performance query technology was first presented by Stockinger et al., who introduced the DEX software system utilizing the FastBit indexing infrastructure to identify regions of interest [S1, unofficial]. Feature Query applies these Query-Driven Visualization (QDV) techniques specifically to extract features from function fields [S1, unofficial]. The method was evaluated on datasets including AVIRIS hyperspectral images of Moffett Field and the San Francisco Bay area, and air quality simulations from the California Regional Particulate Air Quality Study (CRPAQS) [S1, unofficial].

## Sources
[S1] Scientific visualization techniques for volume fraction data and function fields, https://escholarship.org/uc/item/41x9x1gm, fetched 2026-10-27 (unofficial)
