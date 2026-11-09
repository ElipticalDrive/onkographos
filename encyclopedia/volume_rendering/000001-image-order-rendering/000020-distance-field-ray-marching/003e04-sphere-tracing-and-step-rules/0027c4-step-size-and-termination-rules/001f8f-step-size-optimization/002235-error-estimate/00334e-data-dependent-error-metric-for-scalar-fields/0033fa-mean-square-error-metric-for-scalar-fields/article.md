# Mean-Square Error Metric for Scalar Fields

Also called: EMS, Tetrahedral MSE

## Overview
The Mean-Square Error (EMS) metric quantifies the approximation error of a scalar field by comparing the original continuous function to a piecewise linear interpolation over a tetrahedral element [S1]. It serves as a foundational error estimator for adaptive mesh refinement in volume rendering, determining whether a tetrahedron requires subdivision based on the deviation between sampled data and the interpolated surface [S1]. Unlike data-dependent metrics, EMS evaluates error globally within the element regardless of isovalue proximity, which can lead to unnecessary refinement in regions far from the target isosurface [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A tetrahedron $T$ defined by four vertices; a discrete scalar field $F$ sampled at specific locations; the set of sample points $x$ that lie within $T$.
*   **Output:** A scalar error value $EMS(T)$ representing the mean squared deviation.

**Data Structures**
The method operates on the geometric definition of the tetrahedron and the scalar values at its vertices and internal sample points. No complex data structures are required beyond standard vector/matrix representations for vertex positions and scalar arrays for field values.

**Steps**
1.  Identify the set of original sample points $\{x_i\}$ from the scalar field $F$ that lie within the volume of tetrahedron $T$.
2.  For each sample point $x_i$, determine its barycentric coordinates relative to the vertices of $T$.
3.  Compute the interpolated value $f(x_i)$ by linearly interpolating the scalar values at the four vertices of $T$ using the barycentric coordinates.
4.  Calculate the squared difference between the original field value $F(x_i)$ and the interpolated value $f(x_i)$.
5.  Sum these squared differences over all sample points $x_i$ within $T$.
6.  Normalize the sum by the volume of the tetrahedron $|T|$ to obtain the mean square error.

**Formula**
The error metric is defined as [S1]:

$$
EMS(T) = \frac{1}{|T|} \sum_{x \in T} (F(x) - f(x))^2
$$

Where:
*   $|T|$ is the volume spanned by tetrahedron $T$.
*   $F(x)$ is the original scalar field value at sample point $x$.
*   $f(x)$ is the value at $x$ linearly interpolated from the values at the vertices of $T$.
*   The summation is over all sample values of $F$ at vertices $x$ that lie in $T$.

**Parameters**
*   **Root-Mean-Square (RMS) Conversion:** If an RMS error is preferred instead of EMS, the parameter $\alpha$ (used in the context of the refinement threshold) can be doubled [S1].

**Pitfalls**
*   **Unnecessary Refinement:** Using EMS alone may cause unnecessary refinement near the viewpoint even if the isosurface is not present in that region, leading to wasted computation [S1]. This is because EMS measures geometric approximation error of the field, not the relevance of the field to a specific isovalue. Data-dependent metrics are required to solve this [S1].
*   **Screen-Space Projection:** The raw EMS is a volume-based metric. To use it for view-dependent rendering, it must be projected onto the screen space [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ems(tetrahedron, scalar_field, sample_points):
    """
    Computes the Mean-Square Error for a tetrahedron.
    
    Args:
        tetrahedron: Object containing 4 vertex positions and 4 vertex scalar values.
        scalar_field: Function or array providing F(x) for any point x.
        sample_points: List of points x lying within the tetrahedron.
        
    Returns:
        float: The EMS value.
    """
    # Calculate volume of the tetrahedron |T|
    # Volume = |dot(v0-v3, cross(v1-v3, v2-v3))| / 6
    v0, v1, v2, v3 = tetrahedron.vertices
    volume = abs(dot(v0 - v3, cross(v1 - v3, v2 - v3))) / 6.0
    
    if volume == 0:
        return 0.0
        
    squared_diff_sum = 0.0
    count = 0
    
    for x in sample_points:
        # Get original field value F(x)
        F_x = scalar_field(x)
        
        # Compute interpolated value f(x)
        # f(x) is linear interpolation from vertex values
        # Using barycentric coordinates or linear system solution
        # Let V_i be vertex positions, S_i be vertex scalar values
        # Solve for weights w_i such that sum(w_i * V_i) = x and sum(w_i) = 1
        # f(x) = sum(w_i * S_i)
        
        # Note: In practice, barycentric coordinates are computed relative to v3
        # x = v3 + u*(v0-v3) + v*(v1-v3) + w*(v2-v3)
        # f(x) = w*S0 + v*S1 + u*S2 + (1-u-v-w)*S3
        
        f_x = interpolate_linear(tetrahedron, x)
        
        squared_diff_sum += (F_x - f_x) ** 2
        count += 1
        
    # The formula sums over all sample points. 
    # The source defines EMS(T) = (1/|T|) * sum(...). 
    # It does not divide by the count of points, but by the volume.
    # However, standard MSE divides by N. The source text says:
    # "EMS(T) = 1/|T| sum_{x in T} (F(x) - f(x))^2"
    # This implies the normalization is by volume, not point count.
    
    ems = squared_diff_sum / volume
    
    return ems
```

## Subtopics

### Standard Deviation Model (SDM) for Error Quantification
A local measurement model used to quantify errors in low-resolution approximations of the original data set [S2]. It calculates the standard deviation of attribute values around a weighted average [S2]. Formula: $e = \sqrt{\frac{1}{n}[(V_1 - V)^2 + (V_2 - V)^2 + \cdots + (V_n - V)^2]}$, where $e$ is the local error measurement, $n$ is the number of grid points, $V_1 \dots V_n$ are attribute values, and $V$ is the weighted average [S2]. It is applied to regular grids and used for uncertainty visualization with marching cubes [S2].

## Operating characteristics
*   **Complexity:** The complexity is linear with respect to the number of sample points within the tetrahedron. Calculating the volume is constant time.
*   **Strengths:** Provides a rigorous mathematical measure of the deviation between the original data and the linear approximation [S1].
*   **Limitations:** It does not account for the visual importance of the error relative to a specific isovalue. As noted, it can lead to over-refinement in empty space or regions far from the isosurface of interest [S1].
*   **Scaling:** The metric is local to each tetrahedron, making it suitable for parallel evaluation during adaptive refinement.

## Background
The Mean-Square Error metric was introduced in the context of adaptive multi-valued volume data visualization [S1]. It was proposed as a typical error metric for comparing original scalar fields to downsampled approximations [S1]. The work highlights the limitation of using EMS alone for fly-through exploration, motivating the development of data-dependent error metrics that restrict refinement to regions near a chosen isovalue [S1].

## Sources
[S1] Adaptive Multi-Valued Volume Data Visualization using Data-Dependent Error Metrics, https://doi.org/10.2316/journal.205.2005.2.205-4291, fetched 2026-10-26
[S2] Visualizing uncertainty in multi-resolution volumetric data using marching cubes, https://doi.org/10.1145/2254556.2254649, fetched 2026-11-05
