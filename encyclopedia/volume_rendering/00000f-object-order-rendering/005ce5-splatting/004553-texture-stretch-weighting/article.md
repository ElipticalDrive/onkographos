# Texture Stretch Weighting

Also called: wTS Calculation

## Overview
Texture Stretch Weighting ($w_{TS}$) is a weighting term used in point-based neural rendering to compensate for artifacts introduced by weighted average pooling when handling view-dependent effects [S1]. It specifically penalizes pixels at grazing angles, thereby increasing the influence of more front-facing views in the final composite [S1]. It is distinct from sibling methods like EWA Splatting, which focuses on spatial footprint adaptation, by operating as a scalar weight modifier rather than a geometric projection technique [S1, 1069-ewa-splatting.md].

## How it works
**Inputs and Outputs:**
*   **Input:** The covariance matrix $\Sigma$ of the splat kernel $G$ for a point reprojected into the novel view [S1].
*   **Output:** A scalar weight $w_{TS} \in [0, 1]$ representing the penalty for texture stretch [S1].

**Data Structures:**
The method relies on the eigenvalues of the 2D covariance matrix $\Sigma$ derived from the projection of the point's 3D covariance into the image plane. Let $\lambda_1$ and $\lambda_2$ be the eigenvalues of $\Sigma$, such that $\lambda_1 \ge \lambda_2 \ge 0$ [S1].

**Formula:**
The weight is calculated as the ratio of the smaller eigenvalue to the larger eigenvalue:

$$ w_{TS} = \frac{\lambda_2}{\lambda_1} $$

Where:
*   $\lambda_1$ is the largest eigenvalue of the splat covariance matrix [S1].
*   $\lambda_2$ is the smallest eigenvalue of the splat covariance matrix [S1].

**Steps:**
1.  Compute the 2D covariance matrix $\Sigma$ for the splat kernel $G$ at the current pixel location in the novel view [S1].
2.  Calculate the eigenvalues $\lambda_1$ and $\lambda_2$ of $\Sigma$ [S1].
3.  Compute $w_{TS}$ using the ratio $\lambda_2 / \lambda_1$ [S1].
4.  Combine $w_{TS}$ with the covariance smoothing weight $w_{CS}$ and the probabilistic depth weight $w_{PD}$ to form the final weight $w$ [S1]:
    $$ w = w_{CS} \cdot w_{TS} \cdot w_{PD} $$

**Parameters:**
*   No explicit tuning parameters are defined for $w_{TS}$ itself; it is derived directly from the geometry [S1].

**Edge Cases and Pitfalls:**
*   **Degenerate Splat:** If $\lambda_1 = 0$ (a point with zero extent), the ratio is undefined. In practice, a small epsilon should be added to the denominator to prevent division by zero [G].
*   **Grazing Angles:** As the view becomes grazing, the splat stretches, causing $\lambda_1 \gg \lambda_2$. This drives $w_{TS}$ toward 0, effectively discarding the contribution of that point/view, which is the intended behavior [S1].

## Subtopics

### Final View Weighting
This is not a separate algorithmic variant but the aggregation step where $w_{TS}$ is multiplied with other weights. The final weight $w$ is the product of the covariance smoothing weight $w_{CS}$, the texture stretch weight $w_{TS}$, and the probabilistic depth weight $w_{PD}$ [S1]. This product is used in the weighted average pooling operation to encode inductive bias for visibility and allow for temporal smoothing [S1].

## Operating characteristics
**Time and Memory Complexity:**
The computation of eigenvalues for a 2x2 matrix is $O(1)$ and requires minimal memory overhead [S1].

**Strengths and Limitations:**
*   **Strengths:** It effectively mitigates artifacts in weighted average pooling caused by view-dependent effects by down-weighting stretched (grazing) splats [S1]. It improves temporal stability in interactive scenarios [S1].
*   **Limitations:** It relies on the accuracy of the covariance estimation. It does not handle occlusion directly (that is the role of $w_{PD}$) [S1].

**When to use it:**
Use this method when performing point-based neural rendering with per-view optimization where view-dependent effects are present and weighted average pooling is used to aggregate features [S1].

## Background
The method was introduced by Georgios Kopanas, Julien Philip, Tobias Leimkühler, and George Drettakis in the paper "Point-Based Neural Rendering with Per-View Optimization" [S1]. It addresses the issue that standard weighted average pooling can lead to artifacts when view-dependent effects are present, by introducing a penalty for texture stretch [S1].

## Sources
[S1] Point-Based Neural Rendering with Per-View Optimization, https://www.youtube.com/watch?v=TupOewenSAk, fetched 2026-10-31
