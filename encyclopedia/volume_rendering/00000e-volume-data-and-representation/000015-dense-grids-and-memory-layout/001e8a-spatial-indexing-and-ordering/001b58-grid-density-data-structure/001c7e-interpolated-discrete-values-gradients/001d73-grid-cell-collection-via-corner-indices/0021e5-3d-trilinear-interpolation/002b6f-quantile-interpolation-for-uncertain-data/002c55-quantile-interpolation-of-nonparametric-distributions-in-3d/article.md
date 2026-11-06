# Quantile Interpolation of Nonparametric Distributions in 3D

Also called: 3D Quantile Interpolation, Quantile Interpolation of Histograms in 3D

## Overview
This method computes the interpolated Probability Density Function (PDF) of a random variable at an arbitrary 3D position within a grid cell by interpolating quantile representations from the eight vertices of the containing 3D cell [S1]. It extends 1D quantile interpolation to 3D using a bilinear interpolation logic, sequentially interpolating along edges, then faces, then the volume [S1]. It operates on probability density functions rather than scalar values, preserving the statistical nature of uncertain data [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The PDFs of the $j$-th quantile at the 8 vertices of a 3D grid cell, denoted $Pr_1$ through $Pr_8$ [S1]. The spatial distance parameters $\alpha, \beta, \gamma \in [0,1]$ define the query point's position relative to the cell vertices along the three dimensions [S1].
*   **Output:** The interpolated probability density $Pr(Q_j)$ for the $j$-th quantile at the query point [S1].

**Data Structures:**
The method assumes the PDFs at vertices are pre-computed piecewise constant functions where each quantile $j$ has a constant probability density $Pr_i$ at vertex $i$ [S1].

**Steps:**
1.  **Edge Interpolation:** Compute interpolated PDFs along the cell edges by applying the 1D quantile interpolation formula to the PDFs at grid vertices using parameter $\alpha$ [S1].
2.  **Face Interpolation:** Compute interpolated PDFs in the interior of cell faces by applying the 1D quantile interpolation formula to the edge-interpolated PDFs using parameter $\beta$ [S1].
3.  **Volume Interpolation:** Compute the final interpolated PDF in the interior of the 3D cell by applying the 1D quantile interpolation formula to the face-interpolated PDFs using parameter $\gamma$ [S1].

**Formulas:**
The final interpolated PDF for the $j$-th quantile is given by [S1]:

$$Pr(Q_j) = \frac{Pr_1 Pr_2 Pr_3 Pr_4 Pr_5 Pr_6 Pr_7 Pr_8}{t_1 t_2 t_3 t_4 t_5 t_6 t_7}$$

where the intermediate terms are defined as [S1]:

$$t_1 = \alpha Pr_1 + (1-\alpha)Pr_2$$
$$t_2 = \alpha Pr_3 + (1-\alpha)Pr_4$$
$$t_3 = \alpha Pr_5 + (1-\alpha)Pr_6$$
$$t_4 = \alpha Pr_7 + (1-\alpha)Pr_8$$
$$t_5 = \beta \frac{Pr_1 Pr_2}{t_1} + (1-\beta) \frac{Pr_3 Pr_4}{t_2}$$
$$t_6 = \beta \frac{Pr_5 Pr_6}{t_3} + (1-\beta) \frac{Pr_7 Pr_8}{t_4}$$
$$t_7 = \gamma \frac{Pr_1 Pr_2 Pr_3 Pr_4}{t_1 t_2 t_5} + (1-\gamma) \frac{Pr_5 Pr_6 Pr_7 Pr_8}{t_3 t_4 t_6}$$

**Parameters:**
*   $Pr_i$: The probability density of the $j$-th quantile for the random variable at the $i$-th vertex of the 3D grid cell ($i=1..8$) [S1].
*   $\alpha, \beta, \gamma$: Spatial distance parameters for the vertices of the 3D grid cell along the three dimensions, each in the range $[0,1]$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quantile_interpolation_3d(Pr, alpha, beta, gamma):
    """
    Computes the interpolated PDF for a specific quantile in a 3D cell.
    
    Args:
        Pr (list of 8 floats): Probability densities Pr_1 to Pr_8 at the 8 vertices.
        alpha (float): Spatial parameter along the first dimension [0, 1].
        beta (float): Spatial parameter along the second dimension [0, 1].
        gamma (float): Spatial parameter along the third dimension [0, 1].
        
    Returns:
        float: The interpolated probability density Pr(Q_j).
    """
    Pr1, Pr2, Pr3, Pr4, Pr5, Pr6, Pr7, Pr8 = Pr
    
    # Step 1: Edge interpolation (along dimension 1)
    t1 = alpha * Pr1 + (1 - alpha) * Pr2
    t2 = alpha * Pr3 + (1 - alpha) * Pr4
    t3 = alpha * Pr5 + (1 - alpha) * Pr6
    t4 = alpha * Pr7 + (1 - alpha) * Pr8
    
    # Step 2: Face interpolation (along dimension 2)
    t5 = beta * (Pr1 * Pr2 / t1) + (1 - beta) * (Pr3 * Pr4 / t2)
    t6 = beta * (Pr5 * Pr6 / t3) + (1 - beta) * (Pr7 * Pr8 / t4)
    
    # Step 3: Volume interpolation (along dimension 3)
    t7 = gamma * (Pr1 * Pr2 * Pr3 * Pr4 / (t1 * t2 * t5)) + (1 - gamma) * (Pr5 * Pr6 * Pr7 * Pr8 / (t3 * t4 * t6))
    
    # Final result
    numerator = Pr1 * Pr2 * Pr3 * Pr4 * Pr5 * Pr6 * Pr7 * Pr8
    denominator = t1 * t2 * t3 * t4 * t5 * t6 * t7
    
    return numerator / denominator
```

**Edge Cases and Pitfalls:**
If any intermediate term $t_k$ evaluates to zero (e.g., due to zero probability densities at vertices), the formula is undefined; a small epsilon or clamping is required [G]. The parameters $\alpha, \beta, \gamma$ must be strictly within $[0,1]$ to maintain the convex combination structure of the interpolation [S1].

## Subtopics

### Quantile Interpolation for Uncertain Data
The parent method defines the 1D quantile interpolation formula and represents PDFs as piecewise constant quantile functions. It provides the foundational 1D formula used in the edge, face, and volume steps. [Quantile Interpolation for Uncertain Data](5008-quantile-interpolation-for-uncertain-data.md)

## Operating characteristics
Time complexity: Not given in the sources.
Memory complexity: Not given in the sources.
Measured performance: Not given in the sources.
Strengths: Not given in the sources.
Limitations: Not given in the sources.
When to use: Not given in the sources.
When not to use: Not given in the sources.

## Background
The method was extended from 2D by Hollister and Pang, who presented a closed-form solution for bilinear interpolation of histograms/nonparametric distributions on a 2D grid using quantile interpolation [S1]. The 3D extension follows a similar approach, applying the 1D quantile interpolation formula sequentially along edges, faces, and the volume [S1].

## Sources
[S1] Direct Volume Rendering with Nonparametric Models of Uncertainty, https://doi.org/10.1109/tvcg.2020.3030394, fetched 2026-10-26
