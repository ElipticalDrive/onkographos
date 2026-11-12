# IoU Loss

Also called: Intersection over Union (IoU) Loss

## Overview
IoU Loss is a binary segmentation metric used to supervise the occupancy or density fields in differentiable volume rendering by measuring the overlap between predicted and target voxel sets [S1, unofficial]. It sets itself apart from pixel-wise photometric losses like MSE Loss by operating on the discrete 3D structural consistency rather than RGB color fidelity, making it suitable for enforcing geometric boundaries [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: Predicted scalar field values $\hat{v}$ (e.g., density or occupancy probability) and target binary mask values $t_v$ (ground truth occupancy) for a set of voxels.
- Output: A scalar loss value $L_{IoU} = 1 - \text{IoU}$, where IoU is the ratio of intersection to union.

**Data structures:**
- $\hat{v}$: Tensor of predicted values, shape $(N,)$, where $N$ is the number of voxels. Values are typically in $[0, 1]$.
- $t_v$: Tensor of target values, shape $(N,)$, binary (0 or 1).

**Parameters:**
- $\tau$: Prediction threshold for binarization. Default value is 0.9 [S1, unofficial].

**Formula:**
The Intersection over Union is calculated as:
$$ \text{IoU} = \frac{|\{v \mid \hat{v} > \tau\} \cap \{v \mid t_v > 0\}|}{|\{v \mid \hat{v} > \tau\} \cup \{v \mid t_v > 0\}|} $$
where $|\cdot|$ denotes the cardinality (count) of the set [S1, unofficial].

**Steps:**
1. Binarize the predicted field $\hat{v}$ using the threshold $\tau$: create a mask $\hat{m}$ where $\hat{m}_i = 1$ if $\hat{v}_i > \tau$, else $0$.
2. Use the target mask $t_v$ directly as the ground truth mask $m_{gt}$.
3. Compute the intersection count: $I = \sum (\hat{m} \cdot m_{gt})$.
4. Compute the union count: $U = \sum (\hat{m} + m_{gt} - \hat{m} \cdot m_{gt})$.
5. Compute IoU: $\text{IoU} = I / U$.
6. Compute Loss: $L_{IoU} = 1 - \text{IoU}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iou_loss(predicted: Tensor, target: Tensor, threshold: float = 0.9) -> Tensor:
    """
    Computes the IoU loss between predicted and target voxel fields.
    
    Args:
        predicted: Tensor of shape (N,) with predicted density/occupancy values.
        target: Tensor of shape (N,) with binary ground truth occupancy.
        threshold: Float threshold for binarizing predictions.
        
    Returns:
        Scalar tensor representing the IoU loss (1 - IoU).
    """
    # Step 1: Binarize predictions
    pred_mask = (predicted > threshold).float()
    
    # Step 2: Target is already binary, ensure float type
    target_mask = target.float()
    
    # Step 3: Compute intersection
    intersection = torch.sum(pred_mask * target_mask)
    
    # Step 4: Compute union
    # Union = sum(pred) + sum(target) - sum(intersection)
    union = torch.sum(pred_mask) + torch.sum(target_mask) - intersection
    
    # Step 5: Handle edge case where union is zero (empty prediction and target)
    if union == 0:
        return torch.tensor(0.0) # Or a large penalty depending on implementation preference
        
    # Step 6: Compute IoU and Loss
    iou = intersection / union
    loss = 1.0 - iou
    
    return loss
```

**Edge cases and pitfalls:**
- If both predicted and target masks are empty (union is 0), the IoU is undefined. A common convention is to return 0 loss or a specific penalty [G].
- The choice of threshold $\tau$ significantly impacts the gradient signal. A high threshold (e.g., 0.9) makes the loss sparse and less sensitive to partial overlaps [S1, unofficial].
- The loss is non-differentiable at the threshold boundary due to binarization. In practice, it is often used as a metric or with straight-through estimators, or combined with differentiable soft-IoU variants [G].

## Subtopics

### Mean Intersection over Union (mIoU)
mIoU extends IoU to multi-class segmentation by computing the mean IoU across all classes [S2, unofficial]. It is defined as $mIoU = \frac{1}{N_c} \sum_{c=1}^{N} IoU_c$, where $N_c$ is the number of classes and $IoU_c = \frac{TP_c}{TP_c + FP_c + FN_c}$ for class $c$ [S2, unofficial]. It is used when the radiance field or instance field predicts multiple semantic categories rather than binary occupancy [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels, due to element-wise operations and summations.
- **Memory Complexity:** $O(N)$ to store the masks and intermediate sums.
- **Strengths:** Directly optimizes for structural overlap, robust to class imbalance compared to MSE, and provides a clear geometric interpretation of segmentation quality [S1, unofficial].
- **Limitations:** Non-differentiable at the threshold boundary, making gradient-based optimization difficult without approximations. Sensitive to the choice of threshold $\tau$ [S1, unofficial].
- **When to use:** When supervising occupancy or density fields with binary ground truth masks is required, especially in instance segmentation or shape reconstruction tasks [S1, unofficial].
- **When not to use:** When fine-grained color or photometric consistency is the primary goal, or when the ground truth is probabilistic rather than binary.

## Background
IoU Loss originates from computer vision segmentation tasks and has been adapted for inverse rendering and volumetric additive manufacturing to supervise 3D occupancy fields [S1, unofficial]. It is used in frameworks like Dr. TVAM to enforce consistency between predicted volumetric structures and tomographic ground truth [S1, unofficial]. The metric is standard in evaluating segmentation performance, with mIoU being a common aggregate metric [S2, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
[S2] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
