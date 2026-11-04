# FBM Sphere Tracing with Implicit BVH

Also called: Dynamic Bounding Volume Hierarchy for FBM, Progressive evaluation of octaves

## Overview
This method accelerates the sphere tracing of isosurfaces defined by Fractal Brownian Motion (FBM) noise by treating the cascaded sum of octaves as a dynamic, implicit Bounding Volume Hierarchy (BVH) [S1]. It progressively evaluates octaves while bounding the remaining potential contribution to determine if the total function value can exceed the isosurface threshold, allowing for early termination and empty space skipping [S1]. Unlike naive sphere tracing which evaluates the full FBM at every step, this approach reduces the average evaluation cost in empty space from $O(N)$ to $O(k)$, where $k \ll N$ is the number of octaves required to prove emptiness [S1]. It builds on [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md) by replacing the explicit spatial tree with a procedural hierarchy derived from the FBM structure itself.

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- FBM parameters: octave count $N$, amplitude $A_i$ and frequency $F_i$ for each octave $i \in [0, N-1]$.
- Base noise function $n(\mathbf{p})$ with range $[-1, 1]$.
- Isosurface threshold $v$ (typically $0$).

**Outputs:**
- Distance to the nearest surface intersection or empty space indicator.
- Accumulated density (for volume rendering).

**Data Structures:**
- No explicit tree nodes. The hierarchy is implicit in the loop order of octave evaluation.
- Variables: `val` (current accumulated FBM value), `rem` (remaining maximum possible contribution of uncomputed octaves).

**Algorithm:**
The core optimization occurs within the FBM evaluation loop. Instead of computing all $N$ octaves, the algorithm maintains a bound on the maximum possible value the remaining octaves can add. If the current value plus this maximum bound cannot reach the threshold $v$, the point is guaranteed to be outside the object (assuming the object exists where $f(\mathbf{p}) > v$), and the loop terminates early [S1].

1.  Initialize `val` = 0.
2.  Initialize `rem` = $\sum_{i=0}^{N-1} A_i$. This represents the worst-case contribution if all remaining noise samples were at their maximum positive value ($1$).
3.  Iterate through octaves $i$ from $0$ to $N-1$:
    a.  Compute noise sample $n_i = n(\mathbf{p} \cdot F_i)$.
    b.  Update `val`: `val` = `val` + $A_i \cdot n_i$.
    c.  Update `rem`: `rem` = `rem` - $A_i$. Since $n_i \in [-1, 1]$, the maximum possible contribution of the *next* octave $i+1$ is $A_{i+1}$. By subtracting $A_i$ from `rem` after processing octave $i$, `rem` correctly reflects the sum of amplitudes of all *unprocessed* octaves $j > i$.
    d.  **Early Termination Check:** If `val` + `rem` < $v$, terminate the loop immediately. The point is outside the object.
4.  If the loop completes, `val` is the full FBM value.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fbm_with_implicit_bvh(p, octaves, threshold):
    """
    Computes FBM with early termination based on implicit BVH bounds.
    
    Args:
        p: 3D point position.
        octaves: List of tuples (amplitude, frequency) for each octave.
        threshold: The isosurface threshold (typically 0).
        
    Returns:
        The FBM value if fully computed, or the partial value if early terminated.
        Note: The caller must handle the 'early terminated' state to skip ray marching steps.
    """
    val = 0.0
    # Initialize rem as the sum of all amplitudes (worst-case potential of all octaves)
    rem = sum(a for a, f in octaves)
    
    for i, (amplitude, frequency) in enumerate(octaves):
        # Evaluate base noise at scaled position
        # Assuming base noise returns values in [-1, 1]
        noise_val = base_noise(p * frequency)
        
        # Add current octave contribution
        val += amplitude * noise_val
        
        # Update remaining bound: remove current amplitude from potential max
        # The remaining octaves can contribute at most their sum of amplitudes
        rem -= amplitude
        
        # Check if it's impossible to reach the threshold
        # If val + rem < threshold, even if all remaining noise samples are 1,
        # the total will be below the threshold.
        if val + rem < threshold:
            return val, True  # True indicates early termination
            
    return val, False  # False indicates full computation
```

**Parameters:**
- **Threshold $v$**: The density threshold defining the object boundary. Default is $0$. Object exists where $f(\mathbf{p}) > v$ [S1].
- **Octave count $N$**: The number of noise layers. Higher counts increase complexity and the Lipschitz constant [S1].
- **Amplitude $A_i$**: Typically decreases with octave index (e.g., $A_i = A_0 \cdot \text{lacunarity}^{-i}$).

**Edge Cases and Pitfalls:**
- **Precision**: The accumulation of `val` and `rem` must be done with sufficient precision to avoid false positives in the early termination check, especially when `rem` is small.
- **Noise Range**: The method assumes the base noise function is strictly bounded in $[-1, 1]$. If the noise function has a different range, `rem` must be scaled accordingly [S1].
- **Gradient Calculation**: For sphere tracing step sizes, gradients are still required. If early termination occurs, the gradient is undefined or incomplete. The algorithm must ensure that sphere tracing steps are only taken when the full FBM (or a sufficiently accurate approximation) is available, or use the partial value with caution [S1].

## Subtopics

### [Implicit BVH for Sum of Bounded Functions](2039-implicit-bvh-for-sum-of-bounded-functions.md)
This method accelerates the ray marching of implicit surfaces defined by a sum of bounded functions $F_n(\mathbf{p}) = \sum_{i=1}^n f_i(\mathbf{p})$ by exploiting the known value ranges of individual terms. It constructs an implicit hierarchy where partial sums serve as bounding volumes for the final sum, allowing early termination when the potential range of the total function excludes the isosurface threshold. It differs from explicit BVH approaches by computing bounds on-the-fly from pre-sorted function ranges rather than traversing a spatial tree structure.

### Hypertexture Modulation with FBM
A technique to render hypertextures by modulating a smooth main shape function with FBM noise, while preserving the ability to use the implicit BVH and gradient-based sphere tracing optimizations [S1]. It defines a smooth base shape function $S(\mathbf{p})$ (e.g., a sphere or plane) and computes the FBM noise $N(\mathbf{p})$ [S1]. Unlike naive hypertexture rendering which might require full evaluation of complex modulated functions, this method keeps the structure compatible with the progressive octave evaluation and gradient-based stepping [S1].

## Operating characteristics
- **Time Complexity**: Reduces average evaluation cost per step in empty space from $O(N)$ to $O(k)$, where $k \ll N$ is the number of octaves needed to prove emptiness [S1]. In dense regions, the cost remains $O(N)$ as all octaves must be evaluated.
- **Memory Complexity**: $O(1)$ auxiliary memory, as no explicit tree structure is stored. The hierarchy is implicit in the evaluation order.
- **Measured Performance**: Speedups up to 12-fold on opaque or constant semi-transparent scenes compared to classical sphere tracing, and up to 2-fold on non-constant density volumetric scenes [S1]. Gains of up to $\times 1.8$ are attributed specifically to the progressive octave evaluation [S1].
- **Strengths**: Real-time performance on large, detailed scenes; no precomputation or storage overhead; compatible with animated content due to proceduralism [S1].
- **Limitations**: Effectiveness depends on the distribution of the noise; if the object is dense and fills most of the scene, early termination opportunities are reduced. The method assumes the base noise is bounded in $[-1, 1]$ [S1].

## Background
Originating from the paper "Fast sphere tracing of procedural volumetric noise for very large and detailed scenes" [S1], this method addresses the high rendering cost of procedural 3D noise in real-time applications. Precomputing volumes is storage-prohibitive for large open worlds and prevents animation. The method leverages the structure of FBM, treating cascaded sums as nested bounding volumes to enable efficient empty space skipping [S1]. It is part of a broader effort to accelerate sphere tracing for general Signed Distance Functions (SDF) by combining progressive evaluation with Lipschitz-based stepping [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
