# Backprojection via Shepard's Interpolation

Also called: Scale Field Construction, Wendland Polynomial Interpolation

## Overview
This method constructs a continuous scale field $S(\mathbf{x})$ from a discrete set of Laplacian of Gaussian (LoG) extrema by interpolating representative scales using basis functions, enabling smooth size-based transfer functions in volume rendering [S1, unofficial]. It distinguishes itself from nearest-neighbor approaches by providing $C^2$ continuity via Wendland polynomials and from simple maximum blending by allowing weighted averaging that preserves global smoothness, though it requires a visibility test to prevent scale bleeding from large features into small embedded ones [S1, unofficial].

## How it works
**Inputs:** A discrete set of LoG extrema $N = \{(\mathbf{x}_i, t_i)\}$, where $\mathbf{x}_i$ is the spatial position and $t_i$ is the representative scale (radius) at that location. A support parameter multiplier $k$ (typically 1).
**Outputs:** A continuous scalar field $S(\mathbf{x})$ representing the interpolated scale at any point $\mathbf{x}$ in the volume.

**Data Structures:**
*   $N$: List of tuples $(\mathbf{x}_i, t_i)$. $\mathbf{x}_i \in \mathbb{R}^3$, $t_i \in \mathbb{R}^+$.

**Parameters:**
*   $k$: Multiplier for local support. Typical value: $1$.
*   $h_i$: Local support radius for extremum $i$. Defined as $h_i = k \cdot t_i$.

**Formulas:**
The scale field is computed using Shepard's interpolation:
$$ S(\mathbf{x}) = \sum_{\mathbf{x}_i \in N} \Theta(\|\mathbf{x} - \mathbf{x}_i\|) t_i $$
where $\Theta(d)$ is the basis function. The method uses the fourth-degree Wendland polynomial with compact support:
$$ \Theta(d) = \left[ 1 - \frac{d}{h} \right]_0^1 \right)^4 \left( 4\frac{d}{h} + 1 \right) $$
Here, $[u]_0^1$ is the clamping function $\max(0, \min(1, u))$. The parameter $h$ controls the local support; increasing $h$ increases overlap and smoothness but reduces distinction between different scales [S1, unofficial].

**Alternative Blending:**
To prioritize larger features and avoid smoothing away scale variation, the maximum operator can be used instead of summation:
$$ S(\mathbf{x}) = \max_{\mathbf{x}_i \in N} \{ \Theta(\|\mathbf{x} - \mathbf{x}_i\|) t_i \} $$
This variant is useful when large features must dominate the scale assignment [S1, unofficial].

**Pitfalls and Edge Cases:**
*   **Scale Bleeding:** A small feature embedded within the support radius of a large feature may inherit the large scale due to the weighted sum. This is mitigated by a visibility test that modulates the weight $\Theta$ based on occlusion [S1, unofficial].
*   **Precision:** The clamping function $[ \cdot ]_0^1$ ensures $\Theta(d) = 0$ for $d \ge h$, providing compact support and computational efficiency.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clamp(value, min_val, max_val):
    """Clamps value between min_val and max_val."""
    if value < min_val:
        return min_val
    if value > max_val:
        return max_val
    return value

def wendland_basis(d, h):
    """
    Computes the fourth-degree Wendland polynomial basis function.
    d: distance from the extremum.
    h: local support radius.
    """
    if h <= 0:
        return 0.0
    ratio = d / h
    clamped = clamp(1.0 - ratio, 0.0, 1.0)
    return (clamped ** 4) * (4.0 * ratio + 1.0)

def backproject_shepard(extrema, query_point, k=1.0, use_max=False):
    """
    Computes the interpolated scale S(x) at query_point.
    extrema: List of tuples (x_i, t_i).
    query_point: The 3D position x where scale is evaluated.
    k: Multiplier for support radius.
    use_max: If True, use maximum blending; otherwise, use weighted sum.
    """
    if use_max:
        max_val = -1.0
        for (x_i, t_i) in extrema:
            dist = distance(query_point, x_i)
            h = k * t_i
            weight = wendland_basis(dist, h)
            val = weight * t_i
            if val > max_val:
                max_val = val
        return max_val
    else:
        total_scale = 0.0
        for (x_i, t_i) in extrema:
            dist = distance(query_point, x_i)
            h = k * t_i
            weight = wendland_basis(dist, h)
            total_scale += weight * t_i
        return total_scale

def distance(p1, p2):
    """Euclidean distance between two 3D points."""
    return ((p1[0]-p2[0])**2 + (p1[1]-p2[1])**2 + (p1[2]-p2[2])**2)**0.5
```
Note: The visibility test mentioned in the source is not detailed in the provided text, so it is omitted from the core algorithm [S1, unofficial].

## Subtopics

### [Scale-space Backprojection with Visibility Test](21330-scale-space-backprojection-with-visibility-test.md)
Constructs a continuous scale field from LoG extrema using Shepard interpolation but adds a visibility test to modulate weights, preventing small features from inheriting scales from larger occluding blobs. Choose this when accurate scale representation for embedded structures is critical.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per query point, where $N$ is the number of extrema. The compact support of the Wendland polynomial allows for early termination if a spatial index is used to cull distant extrema, but the basic implementation iterates all $N$.
*   **Memory Complexity:** $O(N)$ to store the extrema set.
*   **Performance:** The method is fast enough for GPU implementation using hardware-supported blending [S1, unofficial]. The compact support reduces the number of non-zero contributions per point compared to Gaussian kernels.
*   **Strengths:** Provides $C^2$ continuity, intuitive size representation, and compact support for efficiency.
*   **Limitations:** Prone to scale bleeding without additional visibility tests. The choice of $k$ significantly impacts the smoothness vs. detail trade-off.
*   **When to use:** When a smooth, continuous size-based transfer function is needed from sparse LoG extrema.
*   **When not to use:** When strict isolation of small features from large surrounding scales is required without additional visibility logic.

## Background
Originated in the work of Correa and Ma (2007) on Size-based Transfer Functions [S1, unofficial]. The method was developed to provide a continuous representation of scale for volume exploration, addressing the limitations of local descriptors that decrease away from extrema. It builds on scattered data interpolation techniques, specifically selecting the Wendland polynomial for its compact support properties [S1, unofficial].

## Sources
[S1] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
