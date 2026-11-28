# Jaccard Index Evaluation
Also called: Intersection over Union, IoU for Volumes, Jaccard Index

## Overview
A metric used to evaluate voxel segmentation quality by computing the Jaccard index between target and predicted voxel grids [S3, unofficial]. It measures spatial overlap within a specified region, differing from correlation-based metrics by requiring exact voxel-wise agreement rather than statistical correlation [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
- Inputs: `target` (ground-truth voxel grid), `pred` (predicted voxel grid), `region` (spatial mask restricting evaluation) [S1, S3, unofficial].
- Output: A scalar float representing the Jaccard index (IoU) for each class or the mean across classes [S3, unofficial].

**Data Structures:**
- `target.voxel`: A boolean or integer array of shape matching the evaluation grid [S3, unofficial].
- `pred.voxel`: A boolean or integer array of identical shape and lattice spacing to `target.voxel` [S2, S3, unofficial].
- `region`: A boolean array defining the active evaluation volume [S1, unofficial].

**Steps:**
1. For each class $c \in \{0, \dots, C-1\}$, extract binary masks $A_c$ (target) and $B_c$ (predicted) within the `region` mask [S1, S3, unofficial].
2. Compute the intersection count by counting non-zero elements in the bitwise AND of $A_c$, $B_c$, and `region` [S1, unofficial].
3. Compute the union count by counting non-zero elements in the bitwise OR of $A_c$ and $B_c$, masked by `region` [S1, unofficial].
4. Calculate the ratio of the intersection count to the union count [S2, unofficial].
5. If the union count is zero, return 1.0 because two empty sets are identical [S2, unofficial].
6. Aggregate the per-class scores, typically by computing the mean [S3, unofficial].

**Formulas:**
The Jaccard index is defined as $J = \frac{|A \cap B|}{|A \cup B|} = \frac{\text{shared}}{n_a + n_b - \text{shared}}$ [S2, unofficial].
where $n_a$ and $n_b$ are the counts of non-zero elements in $A$ and $B$, and $\text{shared}$ is the intersection count [S2, unofficial].

**Parameters:**
- `num_classes`: An integer $C$ specifying the number of distinct classes [S3, unofficial].

**Edge Cases and Pitfalls:**
- Empty sets are handled by returning 1.0 when the union is zero [S2, unofficial].
- The two sets must live on the same lattice with identical grid origin and spacing, as coordinates are compared as given [S2, unofficial].
- Use integer arithmetic for intersection and union counts to avoid floating-point errors in large grids [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_jaccard_index(target, pred, region, num_classes):
    """Compute mean Jaccard Index (IoU) for voxel grids within a region."""
    total_iou = 0.0
    count = 0
    for c in range(num_classes):
        # Extract binary masks for class c
        mask_a = [1 if v == c else 0 for v in target]
        mask_b = [1 if v == c else 0 for v in pred]
        active = [1 if r else 0 for r in region]
        
        # Apply region mask
        active_a = [a & r for a, r in zip(mask_a, active)]
        active_b = [b & r for b, r in zip(mask_b, active)]
        
        # Compute intersection and union counts
        intersection = [a & b for a, b in zip(active_a, active_b)]
        union = [a | b for a, b in zip(active_a, active_b)]
        
        n_intersection = sum(intersection)
        n_union = sum(union)
        
        # Handle empty sets
        if n_union == 0:
            iou = 1.0
        else:
            iou = n_intersection / n_union
            
        total_iou += iou
        count += 1
        
    return total_iou / count if count > 0 else 0.0
```

## Subtopics

### Modified Jaccard Index with Spatial Tolerance
A metric to measure the spatial overlap between a set of voxels with the highest variances and a set of voxels with the highest errors, relaxing the strict intersection requirement of the standard Jaccard index [S4]. It differs from the standard variant by expanding the top-error voxel set with one-voxel neighbors of its original members before computing the intersection and union [S4]. Choose this when evaluating uncertainty quantification in scene representation networks, specifically to assess whether high-variance voxels are spatially adjacent to high-error voxels rather than requiring exact coincidence [S4]. It typically evaluates the top 1% or 5% ranked variances and errors [S4].

## Operating characteristics
- **Time Complexity:** $O(N \cdot C)$ for the standard metric, where $N$ is the number of voxels and $C$ is the number of classes [S3, unofficial]. The modified variant adds $O(N \log N)$ overhead for sorting voxels by variance and error [S4].
- **Memory Complexity:** $O(N)$ to store masks and intermediate sums [G].
- **Strengths:** Provides a clear, interpretable overlap score [S2, unofficial]. The modified variant accounts for spatial proximity when exact voxel coincidence is not guaranteed [S4].
- **Limitations:** Sensitive to class imbalance and requires identical grid spacing for both masks [S2, unofficial]. The modified variant requires computing variance and error maps, which are not available for non-uncertain models [S4].
- **When to Use:** For voxel segmentation tasks evaluating predicted voxel grids against ground truth [S3, unofficial]. Use the modified variant when evaluating models that output variance estimates alongside predictions, particularly in super-resolution or reconstruction tasks [S4].

## Background
The Jaccard Index, also known as Intersection over Union (IoU), is a classic set similarity metric [S2, unofficial]. In volume rendering and voxel segmentation, it has been adopted to evaluate the quality of predicted voxel grids against ground truth [S3, unofficial]. The Modified Jaccard Index with Spatial Tolerance was proposed in the context of Regularized Multi-Decoder Ensembles for error-aware scene representation, specifically to evaluate the correlation between prediction variance and reconstruction error in super-resolution networks [S4].

## Sources
[S1] EPFL-LAPD/Antagonistic-Dual-Wavelength-Tomographic-Volumetric-Additive-Manufacturing, https://github.com/EPFL-LAPD/Antagonistic-Dual-Wavelength-Tomographic-Volumetric-Additive-Manufacturing, fetched 2026-10-26 (unofficial)
[S2] navis-org/sparse-cubes, https://github.com/navis-org/sparse-cubes, fetched 2026-10-26 (unofficial)
[S3] cvlab-epfl/voxel2mesh, https://github.com/cvlab-epfl/voxel2mesh, fetched 2026-10-27 (unofficial)
[S4] Regularized Multi-Decoder Ensemble for an Error-Aware Scene Representation Network, https://doi.org/10.1109/tvcg.2024.3456357, fetched 2026-10-26
