# Kernel-Reflection Sequences


## Overview
Kernel-Reflection Sequences (KRS) are a procedural modeling technique for generating complex, fractal-like geometries using Signed Distance Functions (SDFs) and reflection transformations [S1]. It solves the problem of evaluating global distance functions for recursive structures without recursion, enabling real-time sphere tracing on parallel hardware like GPUs [S1]. Unlike standard Iterated Function Systems (IFS) which require recursive traversal and cause divergent control flow, KRS uses a fixed operation count logarithmic time algorithm, allowing efficient parallelization [S1].

## How it works
**Inputs:**
- A world-space point $\mathbf{x}_w$ [S1].
- A KRS defined by two finite sequences: kernel sets $K_0, \dots, K_n$ where each $K_i$ is defined by a distance function $k_i(\mathbf{x})$, and reflection operators $\mathcal{R}_0, \dots, \mathcal{R}_{n-1}$ [S1]. Each $\mathcal{R}_i$ corresponds to a reflection plane defined by $\mathbf{u}_i \cdot \mathbf{x} - w_i = 0$, where $\mathbf{u}_i$ is the unit normal and $w_i$ is the distance from the origin [S1].

**Outputs:**
- The signed distance $d$ from $\mathbf{x}_w$ to the KRS geometry [S1].

**Data Structures:**
- The KRS is stored as two arrays of length $n+1$ and $n$ respectively: `kernels` containing functions or data structures for $k_0, \dots, k_n$, and `reflections` containing structures for $\mathcal{R}_0, \dots, \mathcal{R}_{n-1}$ where each structure holds the plane normal $\mathbf{u}_i$ and plane offset $w_i$ [S1].

**Algorithm:**
1. Initialize $\mathbf{x} \leftarrow \mathbf{x}_w$ [S1].
2. Initialize distance $d \leftarrow k_n(\mathbf{x})$ (distance to the top-level kernel) [S1].
3. Loop for $i$ from $n-1$ down to $0$: [S1].
4. If $\mathbf{x} \cdot \mathbf{u}_i - w_i > 0$ (point is behind the mirror), transform $\mathbf{x} \leftarrow \mathcal{R}_i \mathbf{x}$ [S1].
5. Update $d \leftarrow \min(d, k_i(\mathbf{x}))$ [S1].
6. Return $d$ [S1].

The recursive definition of expansion level geometries $L_i$ is: $L_{i+1} = K_{i+1} \cup L_i \cup \mathcal{R}_i L_i$, with base case $L_0 = K_0$ [S1]. The distance function $l_{i+1}(\mathbf{x})$ for the KRS is computed iteratively using the reflection plane equation $\mathbf{u}_i \cdot \mathbf{x} - w_i = 0$ [S1]. If $\mathbf{u}_i \cdot \mathbf{x} - w_i \leq 0$, then $l_{i+1}(\mathbf{x}) = \min(k_{i+1}(\mathbf{x}), l_i(\mathbf{x}))$ [S1]. If $\mathbf{u}_i \cdot \mathbf{x} - w_i > 0$, then $l_{i+1}(\mathbf{x}) = \min(k_{i+1}(\mathbf{x}), l_i(\mathcal{R}_i(\mathbf{x})))$ [S1]. This can be evaluated in a single pass from the top level down to the base level without recursion [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distance_to_krs(x_w, kernels, reflections):
    """
    Computes the distance from x_w to the KRS.
    
    Args:
        x_w: World space point (vector).
        kernels: List of distance functions [k_0, ..., k_n].
        reflections: List of reflection plane definitions [(u_0, w_0), ..., (u_{n-1}, w_{n-1})].
        
    Returns:
        d: Distance to the KRS geometry.
    """
    n = len(kernels) - 1
    x = x_w
    
    # Initialize distance to the top-level kernel K_n
    d = kernels[n](x)
    
    # Iterate from level n-1 down to 0
    for i in range(n - 1, -1, -1):
        u_i, w_i = reflections[i]
        
        # Check if point is behind the mirror plane
        if dot(u_i, x) - w_i > 0:
            # Transform point to the reflected space
            dist = dot(u_i, x) - w_i
            x = x - 2.0 * dist * u_i
        
        # Update distance with the current level's kernel
        d = min(d, kernels[i](x))
        
    return d
```

**Parameters:**
- **Sphere tracing termination threshold:** Inversely proportional to camera depth, set to assure that the final unbounding sphere projected onto the viewport is smaller than a pixel [S1]. A higher threshold makes kernel geometries appear thicker [S1].

**Edge Cases and Pitfalls:**
- **Intersection with Mirror Plane:** If a previous level geometry $A_i$ is not entirely on the negative side of the mirror plane but intersects it, the formula still applies [S1]. This is equivalent to discarding the part of the geometry on the positive side, providing an underestimation for the distance to the mirrored geometry, which remains suitable for sphere tracing [S1].
- **Lipschitz Constant:** Standard KRS transformations have a Lipschitz constant of unity [S1]. The method can handle any Lipschitz transformation by sacrificing performance [S1].

## Subtopics

### Multiple KRS Union Sphere Tracing
A technique to combine multiple KRSs to produce natural disorder and hide symmetry by taking the union of their geometries [S1]. It maintains the free distance along the ray for all KRS components, always choosing the minimal one, and advancing the ray with the radius of the unbounding sphere of the respective component [S1]. The search ends when the step size for the minimal component meets the error threshold [S1]. Performance does not depend on the number of KRS components, but rather on the number of sphere tracing steps required, with a slight overhead of maintaining a small sorted index of free distances for minimum selection [S1]. The error threshold is inversely proportional to camera depth and set to assure that the final unbounding sphere projected onto the viewport is smaller than a pixel [S1]. Higher threshold levels make kernel geometries appear thicker and cause distant geometries to merge into smoother formations, losing sub-pixel details [S1].

## Operating characteristics
- **Time Complexity:** Logarithmic time with respect to the number of levels $n$, due to the fixed operation count per level and non-recursive evaluation [S1].
- **Memory Complexity:** Independent of the number of levels, requiring only a few registers of read-write memory for the iterative evaluation [S1].
- **Performance:** Efficient for GPU sphere tracing due to the lack of recursion and divergent control flow [S1]. Performance depends on the number of sphere tracing steps required, not the number of KRS components in the union case [S1].
- **Strengths:** Real-time rendering capability, continuous level-of-detail, automatic minification filtering, and suitability for parallel hardware [S1].
- **Limitations:** Symmetry may be visible if only a single KRS is used, requiring multiple KRSs or other techniques to hide symmetry [S1]. Higher thresholds can cause distant geometries to merge, losing sub-pixel details [S1].

## Background
Kernel-Reflection Sequences were proposed in a paper titled "Kernel-Reflection Sequences" (2016) [S1]. The approach is inspired by Iterated Function Systems (IFS) but modifies them to allow for real-time rendering without concessions in visible detail [S1]. It was developed to model natural phenomena in uncompromising detail and extents without obvious clues of symmetry [S1]. The method extends sphere tracing to conformally transformed geometries and includes a GPU load balancing scheme [S1].

## Sources
[S1] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26
