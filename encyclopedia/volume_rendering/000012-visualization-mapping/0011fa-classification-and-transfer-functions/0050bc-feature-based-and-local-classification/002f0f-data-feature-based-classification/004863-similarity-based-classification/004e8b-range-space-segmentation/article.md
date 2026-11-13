# Range-Space Segmentation

Also called: Function Field Visualization via Range-Space Segmentation, Function-space distance segmentation, Range-Space Segmentation Visualization

## Overview
Range-Space Segmentation is a method for visualizing and analyzing function field data by projecting high-dimensional functions at each spatial location into scalar fields based on their distance to a set of user-defined exemplar functions [S1, unofficial]. It solves the problem of intelligently segmenting complex datasets (such as hyperspectral imagery or time-varying pollution data) by creating a classification field that assigns each spatial point to the nearest exemplar in function space [S1, unofficial]. Unlike Multidimensional Scaling (MDS), which loses spatial layout, this method preserves the original spatial structure of the data while enabling direct volume rendering or isosurfacing of feature boundaries [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A function field defined over a domain $\mathbb{R}^n$, where each point $p$ has an associated function $f_p$. The function $f$ is typically represented by a discrete set of $m$ samples (e.g., spectral bands) [S1, unofficial].
*   **Input:** An ordered set of $m$ exemplar function samples $M = (f_1, f_2, \dots, f_m)$, selected by the user via spatial probes or derived from data statistics [S1, unofficial].
*   **Input:** A similarity metric $\| \cdot \|$ defined over the space of functions [S1, unofficial].
*   **Output:** A multi-function scalar distance field $S^*$, where each point stores the minimal distance to any exemplar in $M$ [S1, unofficial].
*   **Output:** An integer-valued classification field $L$, where each point stores the index $i$ of the closest exemplar $f_i$ [S1, unofficial].

**Data Structures:**
1.  **Function Field:** A grid of size $N$ (number of spatial points). Each point $p$ stores a vector of function samples.
2.  **Exemplar Set $M$:** A list of $m$ vectors, each of the same dimension as the function samples.
3.  **Distance Field $S_{f_i}$:** A scalar field of size $N$ for each exemplar $f_i$.
4.  **Classification Field $L$:** An integer field of size $N$, storing indices $1 \dots m$.
5.  **Multi-function Distance Field $S^*$:** A scalar field of size $N$, storing $\min_i \| f_i - f_p \|$.

**Steps:**

1.  **Define Metric:** Choose a distance metric $\| f - g \|$ to measure similarity between functions. The weighted Euclidean metric is commonly used [S1, unofficial]. For a function $f$ defined discretely by samples $(f_1, \dots, f_m)$ with weights $(w_1, \dots, w_m)$:
    $$ \| f - g \| = \left( \sum_{k=1}^m w_k (f_k - g_k)^2 \right)^{1/2} $$
    If weights are constant, this corresponds to a Voronoi tessellation in range-space [S1, unofficial].

2.  **Select Exemplars:** The user specifies $m$ exemplar functions $M = (f_1, \dots, f_m)$. These can be selected by placing spatial probes $p \in \mathbb{R}^n$ in the data domain, where the exemplar $f_i$ is the function value $f_p$ at that probe location [S1, unofficial]. For time-varying fields, probes are points in $\mathbb{R}^n \times T$ [S1, unofficial].

3.  **Compute Single-Function Distance Fields:** For each exemplar $f_i \in M$ and for each spatial point $p$ with function $f_p$, compute the scalar distance field $S_{f_i}$:
    $$ S_{f_i}(p) = \| f_i - f_p \| $$
    This is done for all $i \in \{1, \dots, m\}$ [S1, unofficial].

4.  **Compute Multi-Function Distance Field $S^*$:** For each point $p$, compute the minimum distance across all exemplars:
    $$ S^*(p) = \min_{i \in \{1, \dots, m\}} S_{f_i}(p) = \min_{i \in \{1, \dots, m\}} \| f_i - f_p \| $$
    This can be computed by calculating all $S_{f_i}$ first and then taking the element-wise minimum, or by tracking the minimum sequentially for each point [S1, unofficial].

5.  **Compute Classification Field $L$:** For each point $p$, determine the index of the exemplar that achieved the minimum distance:
    $$ L(p) = \arg \min_{i \in \{1, \dots, m\}} \| f_i - f_p \| $$
    The value $L(p)$ is an integer in $\{1, \dots, m\}$ [S1, unofficial].

6.  **Visualization:**
    *   **Volume Rendering:** Associate a 1D transfer function $t_i$ with each exemplar $i$. The color at point $p$ is determined by applying the transfer function corresponding to its classification label to its distance value:
        $$ \text{color}(p) = t_{L(p)}(S^*(p)) $$
        Opacity can be modified as part of the transfer function [S1, unofficial].
    *   **Isosurfacing:** To extract segmenting surfaces between features, perform surface extraction on the classification field $L$. For $m=2$, $L$ is binary (labels 0 or 1). Extract the surface where $L(x) = v$ (e.g., $v=0.5$) using algorithms like Marching Cubes [S1, unofficial]. For $m > 2$, use multi-label segmentation algorithms such as multi-label Marching Cubes, Dual Contouring, or Nielson and Franke's method on an implicit tetrahedrization [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def range_space_segmentation(function_field, exemplars, weights=None):
    """
    Computes range-space segmentation for a function field.
    
    Args:
        function_field: np.ndarray of shape (N, D), where N is number of spatial points
                        and D is the number of function samples (e.g., spectral bands).
        exemplars: np.ndarray of shape (m, D), the m exemplar functions.
        weights: np.ndarray of shape (D,), optional weights for the distance metric.
                 If None, assumes uniform weights (standard Euclidean).
                 
    Returns:
        S_star: np.ndarray of shape (N,), the multi-function scalar distance field.
        L: np.ndarray of shape (N,), the integer-valued classification field.
    """
    N, D = function_field.shape
    m = exemplars.shape[0]
    
    # Initialize distance matrix: rows are points, cols are exemplars
    # Shape: (N, m)
    distances = np.zeros((N, m))
    
    # Step 3: Compute single-function distance fields
    for i in range(m):
        # Compute difference between all points and exemplar i
        diff = function_field - exemplars[i]
        
        # Apply weights if provided
        if weights is not None:
            diff = diff * weights
            
        # Compute weighted Euclidean norm along the function sample axis (axis=1)
        # || f_i - f_p || = sqrt(sum(w_k * (f_ik - f_pk)^2))
        distances[:, i] = np.sqrt(np.sum(diff**2, axis=1))
        
    # Step 4 & 5: Compute S* and L
    # S* is the minimum distance across exemplars
    S_star = np.min(distances, axis=1)
    
    # L is the index of the exemplar that achieved the minimum
    L = np.argmin(distances, axis=1)
    
    return S_star, L

def render_volume(S_star, L, transfer_functions):
    """
    Renders the volume using the classification field and distance field.
    
    Args:
        S_star: np.ndarray of shape (N,), distance values.
        L: np.ndarray of shape (N,), classification labels (0 to m-1).
        transfer_functions: list of m callable functions, where transfer_functions[i]
                            takes a scalar distance and returns a color (or RGBA).
                            
    Returns:
        colors: np.ndarray of shape (N, 3) or (N, 4), the computed colors.
    """
    m = len(transfer_functions)
    colors = np.zeros((S_star.shape[0], 3)) # Assuming RGB output
    
    for i in range(m):
        # Find points belonging to class i
        mask = (L == i)
        if np.any(mask):
            # Apply the specific transfer function for this class
            # Note: In a real renderer, this would be done per-voxel during ray casting
            # Here we compute the final color for visualization
            dists_for_class = S_star[mask]
            colors[mask] = np.array([tf(d) for tf, d in zip([transfer_functions[i]]*np.sum(mask), dists_for_class)])
            
    return colors
```

**Parameters:**
*   $m$: The number of function samples (exemplars). Typically large, leading to tens or hundreds of samples per data point [S1, unofficial].
*   Transfer functions: 1D transfer functions are used during volume rendering, though 2D, multidimensional, and local transfer functions may be applied to emphasize local structures [S1, unofficial].
*   Probes: User-specified points in the data domain $p \in \mathbb{R}^n$, controlled by a full space cursor. The function sample associated with the probe is the function $f_p$ at point $p$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   If the weight function or vector is constant, the segmentation corresponds to a Voronoi tessellation of the range-space [S1, unofficial].
*   When modifying a single function sample (e.g., repositioning a probe), only one of the $m$ distance fields changes; the remaining $m-1$ fields remain unchanged. The latency experienced is $S + S^*$, where $S$ is the time for one single-function distance field and $S^*$ is the time to combine all distance fields [S1, unofficial].

## Subtopics

### [Feature Query](13751-feature-query.md)
Feature Query is a method for identifying spatial regions in a function fields where 1-dimensional functions are similar to a user-defined exemplar. It solves the problem of intelligently paring down large scientific datasets by identifying regions of interest based on user-specified selection queries, allowing for the analysis of smaller, more meaningful subsets rather than processing the entire dataset. Unlike traditional visualization which processes the whole dataset, this approach constrains computational complexity to the number of items returned by a query.

## Operating characteristics
**Time and Memory Complexity:**
*   Computing the distance fields requires $O(N \cdot m \cdot D)$ operations, where $N$ is the number of spatial points, $m$ is the number of exemplars, and $D$ is the dimension of the function samples [S1, unofficial].
*   Finding the minimum and argmin for $S^*$ and $L$ requires $O(N \cdot m)$ operations [S1, unofficial].
*   Memory usage is $O(N \cdot m)$ to store the distance matrix during computation, or $O(N)$ if computed sequentially and minimized on the fly [S1, unofficial].

**Measured Performance:**
Tests on an Apple MacBook Pro (2.33 GHz Intel Core 2 Duo, 2 GB memory, ATI Radeon X1600) showed interactive performance:
*   Hyperspectral Imagery segmentation: 336 ms total (80 ms for $S$, 16 ms for $S^*$) [S1, unofficial].
*   H2O Aerosol (National): 51 ms total (20 ms for $S$, 11 ms for $S^*$) [S1, unofficial].
*   SO4 Aerosol (CRPAQS): 104 ms total (27 ms for $S$, 23 ms for $S^*$) [S1, unofficial].
*   Latency for modifying a single probe is significantly lower, as only one distance field needs recomputation [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Preserves original spatial layout (unlike MDS); allows interactive feedback; flexible visualization via per-class transfer functions; supports time-varying fields [S1, unofficial].
*   **Limitations:** Performance scales with the number of exemplars $m$; choice of distance metric affects segmentation results; requires user input for exemplar selection unless automated [S1, unofficial].

**When to Use:**
*   Use for hyperspectral imagery and particulate pollution data [S1, unofficial].
*   Use when the goal is to segment data based on functional similarity rather than raw intensity or gradient magnitude [S1, unofficial].

## Background
Range-Space Segmentation was introduced in the context of scientific visualization techniques for volume fraction data and function fields [S1, unofficial]. It addresses the challenge of visualizing high-dimensional function fields (such as hyperspectral images or time-varying simulations) by reducing them to scalar fields suitable for standard rendering techniques [S1, unofficial]. The method builds on the concept of function-space distance metrics and Voronoi tessellations in range-space [S1, unofficial]. It is used for analyzing hyperspectral imagery and particulate pollution data (CRPAQS) [S1, unofficial].

## Sources
[S1] Scientific visualization techniques for volume fraction data and function fields, https://escholarship.org/uc/item/41x9x1gm, fetched 2026-10-27 (unofficial)
