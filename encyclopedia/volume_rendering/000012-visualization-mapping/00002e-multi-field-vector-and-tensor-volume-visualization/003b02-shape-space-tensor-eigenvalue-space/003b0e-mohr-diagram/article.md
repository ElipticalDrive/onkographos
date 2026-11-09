# Mohr Diagram

Also called: Mohr's Circle

## Overview
The Mohr Diagram is a visualization technique for stress tensors that plots normal stress against shear stress to represent the most important invariants [S1, book]. It sets itself apart from scalar volume rendering by explicitly mapping the relationship between the three eigenvalues of a symmetric tensor to geometric circles, allowing for the inspection of maximum shear stress and the transition between tensile and compressive forces [S1, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A symmetric second-order tensor field. For each tensor, the three eigenvalues $\sigma_1, \sigma_2, \sigma_3$ must be computed and sorted such that $\sigma_1 \ge \sigma_2 \ge \sigma_3$ [S1, book].
*   **Output:** A 2D plot (or set of plots) where the horizontal axis represents normal stress and the vertical axis represents shear stress [S1, book].

**Data Structures:**
*   **Tensor:** A $3 \times 3$ symmetric matrix.
*   **Eigenvalues:** A sorted tuple $(\sigma_1, \sigma_2, \sigma_3)$.
*   **Mohr Circle:** Defined by a center $C$ and radius $R$.

**Formulas:**
For a tensor with sorted eigenvalues $\sigma_1 \ge \sigma_2 \ge \sigma_3$, three Mohr circles are constructed [S1, book]:
1.  **Outer Circle:** Connects $\sigma_1$ and $\sigma_3$.
    *   Center: $C_{outer} = \frac{\sigma_1 + \sigma_3}{2}$
    *   Radius: $R_{outer} = \frac{\sigma_1 - \sigma_3}{2}$
2.  **Middle Circle:** Connects $\sigma_1$ and $\sigma_2$.
    *   Center: $C_{mid1} = \frac{\sigma_1 + \sigma_2}{2}$
    *   Radius: $R_{mid1} = \frac{\sigma_1 - \sigma_2}{2}$
3.  **Inner Circle:** Connects $\sigma_2$ and $\sigma_3$.
    *   Center: $C_{mid2} = \frac{\sigma_2 + \sigma_3}{2}$
    *   Radius: $R_{mid2} = \frac{\sigma_2 - \sigma_3}{2}$

The region enclosed by the outer circle represents all possible combinations of normal and shear forces for any cutting plane orientation [S1, book]. A point within this region corresponds to the orientation of the plane’s normal [S1, book].

**Parameters:**
*   **Sign Convention:** Negative eigenvalues are interpreted as compressive forces (volume reduction); positive eigenvalues as tensile forces (volume expansion) [S1, book]. Note that some application areas use the reverse sign convention [S1, book].
*   **Clutter Reduction:** To reduce clutter around the origin (where high shear and tensile/compressive transitions occur), the diagram may be divided into three separate sub-diagrams based on anisotropic behavior (isotropic, linear, planar) [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mohr_circles(eigenvalues):
    """
    Computes the three Mohr circles for a symmetric tensor.
    Args:
        eigenvalues: Tuple (sigma1, sigma2, sigma3) sorted descending.
    Returns:
        List of dicts with 'center' and 'radius'.
    """
    sigma1, sigma2, sigma3 = eigenvalues
    
    # Outer circle: max and min eigenvalues
    c_outer = (sigma1 + sigma3) / 2.0
    r_outer = (sigma1 - sigma3) / 2.0
    
    # Middle circle: max and middle eigenvalues
    c_mid1 = (sigma1 + sigma2) / 2.0
    r_mid1 = (sigma1 - sigma2) / 2.0
    
    # Inner circle: middle and min eigenvalues
    c_mid2 = (sigma2 + sigma3) / 2.0
    r_mid2 = (sigma2 - sigma3) / 2.0
    
    return [
        {"center": c_outer, "radius": r_outer, "type": "outer"},
        {"center": c_mid1, "radius": r_mid1, "type": "mid1"},
        {"center": c_mid2, "radius": r_mid2, "type": "mid2"}
    ]

def classify_tensor(eigenvalues):
    """
    Classifies tensor anisotropy for diagram separation.
    Returns: 'isotropic', 'linear', or 'planar'.
    """
    sigma1, sigma2, sigma3 = eigenvalues
    # Implementation details for classification thresholds are not given in sources [G]
    # Typically based on ratios of eigenvalue differences
    pass
```

**Edge Cases:**
*   **Isotropic Tensors:** If $\sigma_1 \approx \sigma_2 \approx \sigma_3$, the circles collapse to a single point at the origin [S1, book].
*   **Precision:** Numerical stability in eigenvalue decomposition is critical for accurate circle placement [G].

## Subtopics

### Mohr Diagram (Extended)
An extension of the standard Mohr diagram designed to reduce clutter and analyze the relationship between three eigenvalues and their relative strength [S1, book]. It categorizes tensors by anisotropic behavior (isotropic, linear, or planar) and divides the diagram into three separate sub-diagrams to equalize circle distribution [S1, book]. Tensors with similar eigenvalue behavior are clustered using Euclidean distance in shape space as the similarity measure, and the number of occurrences in each cluster is encoded by color [S1, book]. Semi-circles are drawn in context regions to achieve a compact visualization without losing information [S1, book].

## Operating characteristics
*   **Time Complexity:** Computing eigenvalues is $O(1)$ for $3 \times 3$ tensors; plotting is linear in the number of selected tensors.
*   **Memory Complexity:** $O(1)$ per tensor for storing eigenvalues and circle parameters.
*   **Strengths:** Provides immediate visual insight into maximum shear stress (radius of the outer circle) and the balance between tensile and compressive forces (position on the x-axis) [S1, book].
*   **Limitations:** Best used after a selection has been specified in other diagram views, as it is not suitable for inspecting large, unselected datasets due to potential clutter [S1, book].
*   **Scaling:** The extended version scales better for dense datasets by using clustering and separation by anisotropy [S1, book].

## Background
The Mohr diagram originates from the analysis of stress tensors, where it represents the most important invariants [S1, book]. It is based on the classical Mohr's circle method from continuum mechanics [10] (cited in [S1, book]). The specific visualization techniques described, including the extended diagram with clustering and anisotropy-based separation, were proposed by A. Kratz, B. Meyer, and I. Hotz in 2011 [S1, book].

## Sources
[S1] A Visual Approach to Analysis of Stress Tensor Fields, https://doi.org/10.4230/dfu.vol2.sciviz.2011.188, fetched 2026-10-26 (book)
