# Plausible Diversity Metric

Also called: PlausibleDiversity

## Overview
The Plausible Diversity Metric evaluates the quality of an estimated set of object poses (transforms) against a ground-truth set of "plausible" poses by measuring both plausibility and coverage [S1, unofficial]. Plausibility quantifies how close the estimated poses are to any plausible pose, while coverage quantifies how well the estimated poses represent the space of plausible poses [S1, unofficial]. It builds on [Pairwise Chamfer Distance via Relative Transform](13046-pairwise-chamfer-distance-via-relative-transform.md) to compute the underlying distance matrix [S1, unofficial].

## How it works
The metric operates on batches of estimated transforms and a reference set of plausible transforms, using mesh point sampling to define the distance metric.

**Inputs and Outputs:**
*   **Input:** A batch of $B$ estimated transforms $T_{est}$ (world-to-link) and a set of $P$ plausible transforms $T_p$ (world-to-link).
*   **Output:** A `PlausibleDiversityReturn` named tuple containing:
    *   `plausibility`: Scalar tensor representing average plausibility error.
    *   `coverage`: Scalar tensor representing average coverage error.
    *   `most_plausible_per_estimated`: Indices of the nearest plausible transform for each estimated transform.
    *   `most_covered_per_plausible`: Indices of the nearest estimated transform for each plausible transform.

**Data Structures:**
*   `PlausibleDiversityReturn`: A `NamedTuple` with fields `plausibility` (torch.tensor), `coverage` (torch.tensor), `most_plausible_per_estimated` (torch.tensor), and `most_covered_per_plausible` (torch.tensor) [S1, unofficial].
*   `errors_per_batch`: A 2D tensor of shape $(B, P)$ containing the pairwise Chamfer distances between each estimated transform and each plausible transform [S1, unofficial].

**Parameters:**
*   `num_model_points_eval`: Number of points sampled from the object mesh for evaluation. Default is 500 [S1, unofficial].
*   `bidirectional`: Boolean flag. If `True`, the metric is computed in both directions (estimated-to-plausible and plausible-to-estimated) and averaged [S1, unofficial].
*   `plausible_suboptimality`: Tolerance for SDF queries (used in underlying mesh operations) [S1, unofficial].

**Steps:**
1.  **Initialization:** If `model_points_eval` is not provided, sample `num_model_points_eval` (default 500) points from the object mesh [S1, unofficial].
2.  **Pairwise Error Calculation:** Compute the pairwise error matrix `errors_per_batch` of shape $(B, P)$ using the relative transforms between the estimated set $T_{est}$ and the plausible set $T_p$. This is done via `compute_tf_pairwise_error_per_batch(T_{est}^{-1}, T_p, scale)` [S1, unofficial].
3.  **Forward Evaluation:** Evaluate plausibility and coverage from `errors_per_batch` using `do_evaluate_plausible_diversity_on_pairwise_chamfer_dist` [S1, unofficial].
4.  **Bidirectional Averaging (Optional):** If `bidirectional` is `True`:
    *   Compute reverse errors `errors_per_batch_rev` using $T_p$ and $T_{est}^{-1}$ [S1, unofficial].
    *   Evaluate `ret2` from the reverse errors [S1, unofficial].
    *   Average the results:
        $$ \text{plausibility} = \frac{\text{ret.plausibility} + \text{ret2.coverage}}{2} $$
        $$ \text{coverage} = \frac{\text{ret.coverage} + \text{ret2.plausibility}}{2} $$
    *   Otherwise, use `ret` directly [S1, unofficial].

**Formulas:**
Let $E$ be the `errors_per_batch` matrix of shape $(B, P)$, where $E_{ij}$ is the distance between the $i$-th estimated transform and the $j$-th plausible transform [S1, unofficial].

*   **Plausibility:** The average minimum distance from each estimated transform to its nearest plausible transform [S1, unofficial]:
    $$ \text{plausibility} = \frac{1}{B} \sum_{i=1}^{B} \min_{j} E_{ij} $$
    In code: `sum(min(errors_per_batch, dim=1).values) / B` [S1, unofficial].

*   **Coverage:** The average minimum distance from each plausible transform to its nearest estimated transform [S1, unofficial]:
    $$ \text{coverage} = \frac{1}{P} \sum_{j=1}^{P} \min_{i} E_{ij} $$
    In code: `sum(min(errors_per_batch, dim=0).values) / P` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from typing import NamedTuple

class PlausibleDiversityReturn(NamedTuple):
    plausibility: torch.Tensor
    coverage: torch.Tensor
    most_plausible_per_estimated: torch.Tensor
    most_covered_per_plausible: torch.Tensor

def plausible_diversity_metric(
    T_est_inv: torch.Tensor,  # Shape: (B, 4, 4) or similar transform representation
    T_p: torch.Tensor,        # Shape: (P, 4, 4) or similar transform representation
    scale: float = 1.0,
    bidirectional: bool = False,
    num_model_points_eval: int = 500
) -> PlausibleDiversityReturn:
    """
    Computes the Plausible Diversity Metric.
    
    Args:
        T_est_inv: Inverse estimated transforms (world-to-link inverse). Shape (B, ...).
        T_p: Plausible transforms (world-to-link). Shape (P, ...).
        scale: Scale factor for the mesh.
        bidirectional: If True, average forward and reverse metrics.
        num_model_points_eval: Number of points sampled from mesh for distance calculation.
        
    Returns:
        PlausibleDiversityReturn with plausibility, coverage, and indices.
    """
    # Step 1: Initialization (implicit in setup, points sampled if needed)
    # Note: Point sampling is handled internally by the ObjectFactory or mesh loader
    # if model_points_eval is None. Here we assume the distance function handles
    # the mesh point evaluation.

    # Step 2: Compute pairwise error matrix
    # errors_per_batch shape: (B, P)
    # This function computes the Chamfer distance between mesh points transformed
    # by the relative transform between each pair (i, j).
    errors_per_batch = compute_tf_pairwise_error_per_batch(T_est_inv, T_p, scale)

    # Step 3: Evaluate forward direction
    ret = do_evaluate_plausible_diversity_on_pairwise_chamfer_dist(errors_per_batch)

    if bidirectional:
        # Step 4: Compute reverse direction
        # Swap roles: plausible becomes "estimated", estimated becomes "plausible"
        # Note: T_p is used as the first arg, T_est_inv as the second
        errors_per_batch_rev = compute_tf_pairwise_error_per_batch(T_p, T_est_inv, scale)
        ret2 = do_evaluate_plausible_diversity_on_pairwise_chamfer_dist(errors_per_batch_rev)

        # Average plausibility and coverage
        plausibility = (ret.plausibility + ret2.coverage) / 2.0
        coverage = (ret.coverage + ret2.plausibility) / 2.0
        
        # Indices are not averaged, but typically one direction is preferred or
        # the forward direction is used for indices. The source does not specify
        # how to handle indices in bidirectional mode. We return the forward indices.
        most_plausible_per_estimated = ret.most_plausible_per_estimated
        most_covered_per_plausible = ret.most_covered_per_plausible
    else:
        plausibility = ret.plausibility
        coverage = ret.coverage
        most_plausible_per_estimated = ret.most_plausible_per_estimated
        most_covered_per_plausible = ret.most_covered_per_plausible

    return PlausibleDiversityReturn(
        plausibility=plausibility,
        coverage=coverage,
        most_plausible_per_estimated=most_plausible_per_estimated,
        most_covered_per_plausible=most_covered_per_plausible
    )

def compute_tf_pairwise_error_per_batch(T1: torch.Tensor, T2: torch.Tensor, scale: float) -> torch.Tensor:
    """
    Computes pairwise Chamfer distance between two sets of transforms.
    Returns a tensor of shape (len(T1), len(T2)).
    Implementation details depend on the specific mesh point sampling and SDF query logic.
    """
    # Placeholder for the actual implementation which involves:
    # 1. Sampling points from the mesh.
    # 2. Transforming points by relative transforms.
    # 3. Querying SDF or closest point distances.
    # 4. Aggregating distances into a (B, P) matrix.
    # This is a helper function assumed to be available in the library.
    raise NotImplementedError("Implementation details for compute_tf_pairwise_error_per_batch are in the source library.")

def do_evaluate_plausible_diversity_on_pairwise_chamfer_dist(errors: torch.Tensor) -> PlausibleDiversityReturn:
    """
    Computes plausibility and coverage from a pairwise error matrix.
    
    Args:
        errors: Tensor of shape (B, P).
        
    Returns:
        PlausibleDiversityReturn.
    """
    B, P = errors.shape
    
    # Plausibility: min over P (dim=1) for each B
    min_errors_est, most_plausible_per_estimated = torch.min(errors, dim=1)
    plausibility = torch.sum(min_errors_est) / B
    
    # Coverage: min over B (dim=0) for each P
    min_errors_p, most_covered_per_plausible = torch.min(errors, dim=0)
    coverage = torch.sum(min_errors_p) / P
    
    return PlausibleDiversityReturn(
        plausibility=plausibility,
        coverage=coverage,
        most_plausible_per_estimated=most_plausible_per_estimated,
        most_covered_per_plausible=most_covered_per_plausible
    )
```

## Operating characteristics
*   **Complexity:** The complexity is dominated by the pairwise Chamfer distance calculation, which is $O(B \cdot P \cdot N)$, where $N$ is the number of mesh points sampled for evaluation. The final aggregation is $O(B \cdot P)$.
*   **Performance:** Performance depends on the efficiency of `compute_tf_pairwise_error_per_batch`, which involves mesh point transformations and SDF/closest-point queries. Using GPU-accelerated operations (PyTorch) is recommended for large batches [S1, unofficial].
*   **Strengths:** Provides a comprehensive view of pose estimation quality by considering both accuracy (plausibility) and diversity/coverage [S1, unofficial].
*   **Limitations:** Computationally expensive for large $B$ and $P$. The metric is sensitive to the scale of the mesh and the number of sampled points [S1, unofficial].

## Background
The Plausible Diversity Metric is part of the `pytorch_volumetric` library developed by the University of Michigan Robotics Lab (UM-ARM-Lab) [S1, unofficial]. It is used in robotics and computer vision applications to evaluate the quality of estimated object poses against a set of plausible ground-truth or reference poses [S1, unofficial].

## Sources
[S1] GitHub - UM-ARM-Lab/pytorch_volumetric: Volumetric structures such as ..., https://github.com/UM-ARM-Lab/pytorch_volumetric, fetched 2026-10-30 (unofficial)
