# Surface Geometric Moments Metric


## Overview
The Surface Geometric Moments Metric (SMD) measures the geometric similarity between an original surface $S$ of an object $P$ and a simplified surface $S'$ by computing the sum of absolute differences of their geometric moments up to a maximal order $M$ [S1, unofficial]. It builds on [Geometric Consistency Metric](16683-geometric-consistency-metric.md) by providing a scale-normalized consistency measure that captures higher-order shape characteristics beyond simple area or volume differences [S1, unofficial]. Unlike the Volume Geometric Moments Metric, which evaluates the enclosed volume, SMD focuses exclusively on the boundary surface properties [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Original surface $S$ of object $P$, simplified surface $S'$ of object $P$, maximal moment order $M$.
- Output: Scalar value $SMD$ representing the geometric discrepancy.

**Data Structures:**
- Surface representation: Typically a triangle mesh or parametric surface allowing integration.
- Moment storage: A 3D array or flattened list storing moments $m_{p,q,r}$ for $0 \le p,q,r \le M$. The total number of moments is $\binom{M+3}{3}$.

**Definitions:**
The geometric moment of order $k_1+k_2+k_3$ for a surface $S$ is defined as [S1, unofficial]:
$$ m_{k_1,k_2,k_3}^S(P) = \int_S x^{k_1} y^{k_2} z^{k_3} dS $$
where the integral is performed over the surface of $P$, and $(x,y,z)$ are the coordinates on the surface [S1, unofficial].

The SMD is calculated as [S1, unofficial]:
$$ SMD = \sum_{p=0}^M \sum_{q=0}^M \sum_{r=0}^M |m_{p,q,r}^S(P) - m_{p,q,r}^{S'}(P)| $$
where $M$ is the maximal order of moments used [S1, unofficial].

**Parameters:**
- $M$: Maximal order of moments.
  - Typical values: Not specified in sources, but $M=0$ recovers the metric based solely on surface area (Park et al.) [S1, unofficial].
  - Higher $M$ captures more detailed shape information but increases computational cost [S1, unofficial].

**Steps:**
1. **Compute Original Moments:** Calculate all surface moments $m_{p,q,r}^S(P)$ for $0 \le p,q,r \le M$ for the original surface $S$.
   - *Note:* Direct calculation is computationally expensive. Use the fast surface moment estimation algorithm by Tuzikov et al. [8] to solve this [S1, unofficial].
2. **Compute Simplified Moments:** Calculate all surface moments $m_{p,q,r}^{S'}(P)$ for $0 \le p,q,r \le M$ for the simplified surface $S'$.
   - *Optimization:* During mesh simplification, estimate the evolution of the simplified surface's moments by only evaluating the impact of each local modification, avoiding full recalculation over the entire surface [S1, unofficial].
3. **Calculate Difference:** Compute the sum of absolute differences between corresponding moments of $S$ and $S'$ using the formula above.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_surface_moments(surface, max_order):
    """
    Computes geometric moments for a surface up to max_order.
    Uses Tuzikov et al. algorithm for efficiency [S1, unofficial].
    """
    # Initialize moments array
    # Dimensions: (max_order + 1) x (max_order + 1) x (max_order + 1)
    moments = zeros((max_order + 1, max_order + 1, max_order + 1))
    
    # Iterate over all combinations of p, q, r
    for p in range(max_order + 1):
        for q in range(max_order + 1):
            for r in range(max_order + 1):
                # Compute integral of x^p * y^q * z^r over the surface
                # Implementation relies on Tuzikov et al. fast estimation [S1, unofficial]
                moments[p, q, r] = integrate_surface_moment(surface, p, q, r)
                
    return moments

def surface_geometric_moments_metric(original_surface, simplified_surface, max_order):
    """
    Calculates the Surface Geometric Moments Metric (SMD).
    """
    # Step 1: Compute moments for original surface
    moments_original = compute_surface_moments(original_surface, max_order)
    
    # Step 2: Compute moments for simplified surface
    # Optimization: If simplified_surface is derived via local modifications,
    # update moments incrementally instead of full recalculation [S1, unofficial]
    moments_simplified = compute_surface_moments(simplified_surface, max_order)
    
    # Step 3: Calculate SMD
    smd = 0.0
    for p in range(max_order + 1):
        for q in range(max_order + 1):
            for r in range(max_order + 1):
                diff = abs(moments_original[p, q, r] - moments_simplified[p, q, r])
                smd += diff
                
    return smd
```

**Edge Cases and Pitfalls:**
- **Computational Cost:** Direct calculation of moments is expensive. The source explicitly states this problem is solved by using the fast surface moment estimation algorithm by Tuzikov et al. [8] [S1, unofficial].
- **Degenerate Input:** If $M=0$, the metric reduces to the absolute difference in surface area (Park et al. metric) [S1, unofficial].
- **Boundary Conditions:** The integral is performed over the surface; ensure the surface is closed or properly bounded for meaningful moment calculation [S1, unofficial].

## Subtopics

### Volume Geometric Moments Metric
A metric that measures the similarity between an original volume $V$ of an object $P$ and the volume $V'$ enclosed by a simplified surface $S'$ by computing the sum of absolute differences of their volume moments up to order $M$ [S1, unofficial]. It differs from SMD by integrating over the volume rather than the surface [S1, unofficial]. When restricted to $M=0$, it recovers the metric proposed by Alliez et al. (based solely on volume) [S1, unofficial]. Like SMD, it uses incremental updates during simplification and fast volume moment estimation algorithms [S1, unofficial]. The formula is $VMD = \sum_{p,q,r=0}^M |m_{p,q,r}^V(P) - m_{p,q,r}^{V'}(P)|$, where $m_{k_1,k_2,k_3}^V(P) = \int_P x^{k_1} y^{k_2} z^{k_3} dxdydz$ [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not explicitly given, but direct calculation is described as "assez longue à calculer" (quite long to calculate) [S1, unofficial]. Efficiency is achieved via Tuzikov et al.'s fast estimation algorithm and incremental updates during simplification [S1, unofficial].
- **Memory Complexity:** Not given.
- **Measured Performance:** Not given.
- **Scaling:** Not given.
- **Strengths:** Provides more information than metrics based solely on surface area or volume differences when $M > 0$ [S1, unofficial]. Efficient evaluation during simplification via local modification impact estimation [S1, unofficial].
- **Limitations:** Computationally expensive if not using the fast estimation algorithm [S1, unofficial].
- **When to use:** When evaluating mesh simplification quality where higher-order geometric consistency is important, and computational efficiency is maintained via incremental updates [S1, unofficial].
- **When not to use:** When only area or volume preservation is of interest (use $M=0$ variants or simpler metrics) [S1, unofficial].

## Background
The Surface Geometric Moments Metric was proposed as part of research into new medical volume visualization methods, specifically for kidney preoperative planning systems [S1, unofficial]. It builds on the idea of using geometric moments to measure similarity between original and simplified surfaces [S1, unofficial]. The work references Park et al. for the $M=0$ surface area metric and Alliez et al. for the $M=0$ volume metric [S1, unofficial]. The fast surface moment estimation algorithm by Tuzikov et al. [8] is cited as a key enabler for practical computation [S1, unofficial]. The evaluation scheme was inspired by [4] [S1, unofficial].

## Sources
[S1] Research of New Medical Volume Visualization Methods: Application in the Kidney Preoperative Planning System, https://theses.hal.science/tel-00355629v1, fetched 2026-10-26 (unofficial)
