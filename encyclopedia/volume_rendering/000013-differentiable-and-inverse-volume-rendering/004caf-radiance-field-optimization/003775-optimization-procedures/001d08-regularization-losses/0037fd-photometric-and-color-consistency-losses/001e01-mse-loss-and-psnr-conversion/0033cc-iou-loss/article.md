# IoU Loss

Also called: Intersection over Union (IoU) Loss

## Overview
IoU Loss is a binary segmentation metric that computes the ratio of overlapping voxels above a threshold to the total union of predicted and target voxels [S1, unofficial]. It is used to optimize occupancy or instance fields in radiance-field training, addressing fragmentation caused by inconsistent 2D masks across views [S1, unofficial]. It differs from its parent, [MSE Loss and PSNR Conversion](1651-mse-loss-and-psnr-conversion.md), by operating on binary occupancy/instance probabilities rather than continuous RGB values, providing structural supervision for geometry or instance segmentation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Predicted voxel values $\hat{v}$ (scalar field, e.g., density or instance probability), target voxel values $t_v$ (binary or scalar field), and a threshold $\tau$.
- Output: A scalar loss value $L_{IoU} = 1 - \text{IoU}$.

**Data Structures:**
- $\hat{v}$: Tensor of predicted values, shape $(N,)$ or $(D, H, W)$, containing continuous probabilities or densities.
- $t_v$: Tensor of target values, shape matching $\hat{v}$, containing ground-truth labels (0 or 1) or soft targets.
- $\tau$: Scalar threshold for binarization.

**Formula:**
The Intersection over Union is defined as [S1, unofficial]:
$$ \text{IoU} = \frac{|\{v \mid \hat{v} > \tau\} \cap \{v \mid t_v > 0\}|}{|\{v \mid \hat{v} > \tau\} \cup \{v \mid t_v > 0\}|} $$
The loss is typically $L_{IoU} = 1 - \text{IoU}$.

**Parameters:**
- $\tau$: Prediction threshold for binarization. Default value is 0.9 [S1, unofficial].

**Steps:**
1. Create a binary mask for predictions: $M_{pred} = (\hat{v} > \tau)$.
2. Create a binary mask for targets: $M_{target} = (t_v > 0)$.
3. Compute the intersection: $I = \sum (M_{pred} \land M_{target})$.
4. Compute the union: $U = \sum (M_{pred} \lor M_{target})$.
5. Compute IoU: $\text{IoU} = I / U$.
6. Compute Loss: $L = 1 - \text{IoU}$.

**Edge Cases and Pitfalls:**
- Division by zero: If both masks are empty ($U=0$), the IoU is undefined. A common guard is to return 0 loss or a small epsilon to the denominator [G].
- Threshold sensitivity: The choice of $\tau$ significantly affects the gradient signal. A high threshold (e.g., 0.9) makes the loss sparse and less informative for early training [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iou_loss(predicted: Tensor, target: Tensor, threshold: float = 0.9) -> Tensor:
    """
    Computes the IoU loss for binary segmentation.
    
    Args:
        predicted: Tensor of predicted values (probabilities/densities).
        target: Tensor of target values (ground truth).
        threshold: Threshold for binarizing predictions.
        
    Returns:
        Scalar IoU loss.
    """
    # Binarize predictions and targets
    pred_mask = predicted > threshold
    target_mask = target > 0
    
    # Compute intersection and union
    intersection = (pred_mask & target_mask).sum()
    union = (pred_mask | target_mask).sum()
    
    # Handle empty case to avoid division by zero
    if union == 0:
        return 0.0
        
    iou = intersection / union
    return 1.0 - iou
```

## Subtopics

### Mean Intersection over Union (mIoU)
mIoU is a segmentation evaluation metric computed as the mean of IoU across all classes [S2, unofficial]. It is defined as $mIoU = \frac{1}{N_c} \sum_{c=1}^{N} IoU_c$, where $N_c$ is the number of classes, and $IoU_c = \frac{TP_c}{TP_c + FP_c + FN_c}$ for class $c$, with TP, FP, and FN being true positives, false positives, and false negatives [S2, unofficial]. It is used when multi-class instance segmentation is required, extending the binary IoU Loss to handle multiple object categories [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels, as it requires a single pass to compute sums.
- **Memory Complexity:** $O(N)$ for storing masks if not computed in-place.
- **Strengths:** Provides a scale-invariant measure of overlap, robust to class imbalance compared to MSE [S1, unofficial].
- **Limitations:** Non-differentiable at the threshold boundary; gradients are zero almost everywhere unless soft-IoU variants are used [G]. Sensitive to the choice of threshold $\tau$ [S1, unofficial].
- **When to use:** When optimizing for geometric occupancy or instance segmentation masks where overlap quality is more important than pixel-wise intensity accuracy.
- **When not to use:** When fine-grained intensity reconstruction is needed, or when the target is not binary/occupancy-based.

## Background
IoU Loss originates from computer vision segmentation tasks and has been adapted for inverse rendering in volumetric additive manufacturing and radiance fields [S1, unofficial]. It is used to train instance branches of radiance fields to ensure multi-view consistency of instance masks [S1, unofficial]. The metric is standard in 2D segmentation and has been extended to 3D voxel-based representations [S2, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
[S2] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
