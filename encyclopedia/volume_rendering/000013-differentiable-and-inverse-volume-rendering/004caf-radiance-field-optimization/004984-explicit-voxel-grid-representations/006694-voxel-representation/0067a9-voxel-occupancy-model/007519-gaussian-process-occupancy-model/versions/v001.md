# Gaussian Process Occupancy Model
Also called: GP occupancy, Probabilistic implicit surface

## Overview
The Gaussian Process Occupancy Model defines voxel occupancy as the probability that a point lies on or inside an object, derived from a Gaussian Process (GP) over a geometric classification function [S1]. It differs from deterministic implicit models by providing a probabilistic interpretation of occupancy, enabling robust optimization through orientation assignment based on the expected gradient of the mean function [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A grid-based texture storing the mean implicit function values $\mu(\mathbf{x})$ at voxel centers.
*   **Output:** Occupancy probability $\alpha(\mathbf{x})$ for any point $\mathbf{x}$, and the background surface $\bar{S}$ defined by the zero-level set of $\mu(\mathbf{x})$.

**Data Structures**
The model relies on a 3D grid (texture) where each cell stores a scalar value $\mu(\mathbf{x})$. In the cited implementation, cubic interpolation is used to query $\mu(\mathbf{x})$ at arbitrary positions [S1].

**Parameters**
*   $\sigma$: Constant variance for the Gaussian Process. The sources assume $\sigma(\mathbf{x}) = \sigma$ for all $\mathbf{x}$ [S1].
*   $\gamma$: Parameter for the squared exponential kernel, used to define spatial correlation. However, this parameter is not used in the core occupancy calculation or surface extraction described [S1].

**Formulas**
The geometric classification function $\Phi(\mathbf{x})$ is modeled as a Gaussian random variable:
$$ \Phi(\mathbf{x}) \sim \mathcal{N}(\mu(\mathbf{x}), \sigma^2) $$
The occupancy $\alpha(\mathbf{x})$ is the probability that $\Phi(\mathbf{x}) \leq 0$:
$$ \alpha(\mathbf{x}) = \text{Pr}\{\Phi(\mathbf{x}) \leq 0\} = \frac{1}{2} \left[ 1 - \operatorname{erf} \left( \frac{\mu(\mathbf{x})}{\sqrt{2\sigma^2}} \right) \right] $$
The orientation $\beta(\mathbf{x})$ at point $\mathbf{x}$ is defined by the normalized expected gradient of the mean:
$$ \beta(\mathbf{x}) = \frac{\nabla\mu(\mathbf{x})}{\|\nabla\mu(\mathbf{x})\|} $$
The background surface $\bar{S}$ is the isosurface where $\mu(\mathbf{x}) = 0$ (equivalently $\alpha(\mathbf{x}) = 0.5$) [S1].

**Steps**
1.  **Query Mean:** For a given point $\mathbf{x}$, retrieve $\mu(\mathbf{x})$ from the grid using cubic interpolation [S1].
2.  **Compute Occupancy:** Calculate $\alpha(\mathbf{x})$ using the error function formula above with the constant $\sigma$ [S1].
3.  **Compute Orientation (Optional):** Calculate $\nabla\mu(\mathbf{x})$ (e.g., via finite differences on the grid) and normalize it to get $\beta(\mathbf{x})$. This improves optimization robustness [S1].
4.  **Surface Extraction (Optional):** To visualize or interact with the background surface $\bar{S}$, apply Marching Cubes to the grid of $\mu(\mathbf{x})$ values [S1].

**Pitfalls and Edge Cases**
*   **Gradient Vanishing:** If $\nabla\mu(\mathbf{x}) = \mathbf{0}$, the orientation $\beta(\mathbf{x})$ is undefined. This occurs at critical points of the implicit function.
*   **Spatial Correlation Ignored:** The implementation does not model the auto-correlation $\text{corr}(\Phi(\mathbf{x}), \Phi(\mathbf{y})) = \exp(-\gamma\|\mathbf{x} - \mathbf{y}\|^2)$ because it only requires pointwise probability evaluations and surface extraction, not sampling concrete surface realizations [S1].
*   **Variance Assumption:** The model assumes constant variance $\sigma$. If the true uncertainty varies spatially, this assumption may lead to inaccurate occupancy probabilities [S1].

## Operating characteristics
*   **Complexity:** Querying occupancy is $O(1)$ per point given the grid. Computing gradients adds a small constant factor.
*   **Strengths:** The orientation assignment $\beta(\mathbf{x})$ significantly improves the robustness and speed of optimizations [S1]. The probabilistic formulation allows for a natural interpretation of uncertainty.
*   **Limitations:** The assumption of constant variance may not capture complex uncertainty structures. The model does not inherently handle spatial correlations between points, which might be necessary for tasks involving sampling multiple surface realizations [S1].
*   **When to use:** When robust optimization of implicit surfaces is needed and a probabilistic interpretation of occupancy is beneficial.
*   **When not to use:** When spatial correlation between points is critical (e.g., sampling realistic surface variations) or when variance varies significantly across the scene [S1].

## Background
The Gaussian Process Occupancy Model is part of the "Many-Worlds Inverse Rendering" framework [S1]. It was introduced to handle the ambiguity in inverse rendering by treating the scene geometry as a distribution of surfaces rather than a single deterministic surface. The use of Gaussian Processes allows for a principled way to define occupancy probabilities and orientations, which are crucial for stable optimization in inverse rendering scenarios [S1].

## Sources
[S1] Many-Worlds Inverse Rendering, https://doi.org/10.1145/3767318, fetched 2026-10-26
