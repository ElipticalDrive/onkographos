# Normal-L2 Score Computation

Also called: Masked Normal L2, Normal Error Metric

## Overview
The Normal-L2 Score computes the mean squared error between predicted surface normals and ground-truth normals to evaluate geometric fidelity in radiance-field optimization. It differs from sibling metrics like [PSNR Metric Calculation](2742-psnr-metric-calculation.md) by focusing on shading-critical surface orientation rather than color intensity, and from [IoU Loss](7110-iou-loss.md) by measuring local geometric accuracy rather than global silhouette alignment.

## How it works
**Inputs:**
- Predicted surface normals tensor $N_{pred}$ (shape: $H \times W \times 3$).
- Ground-truth surface normals tensor $N_{gt}$ (shape: $H \times W \times 3$), interpolated from a reference mesh.
- Predicted binary mask $M_{pred}$ (shape: $H \times W$).
- Ground-truth binary mask $M_{gt}$ (shape: $H \times W$).

**Outputs:**
- Scalar Normal-L2 Score.

**Data Structures:**
- Masks are boolean or float tensors where 1 indicates the object silhouette and 0 indicates background.
- Normals are unit vectors.

**Steps:**
1. **Mask Intersection:** Compute the intersection mask $M_{int}$ where both predicted and ground-truth masks are active:
   $$ M_{int} = M_{pred} \land M_{gt} $$
   This ensures error is only computed on regions where the object is present in both prediction and ground truth, separating geometric error from segmentation error [S1].

2. **Normal Sampling:** For each pixel $(u, v)$ in the image grid:
   - If $M_{int}[u, v] == 1$, retrieve $N_{pred}[u, v]$ and $N_{gt}[u, v]$.
   - If $M_{int}[u, v] == 0$, discard the pixel.

3. **Error Calculation:** Compute the squared L2 distance for each valid pixel:
   $$ E_{uv} = \| N_{pred}[u, v] - N_{gt}[u, v] \|_2^2 $$
   Since normals are unit vectors, this simplifies to:
   $$ E_{uv} = 2 - 2 (N_{pred}[u, v] \cdot N_{gt}[u, v]) $$

4. **Aggregation:** Average the errors over all pixels in $M_{int}$:
   $$ \text{Score} = \frac{1}{|M_{int}|} \sum_{u,v \in M_{int}} E_{uv} $$

**Parameters:**
- Camera positions: 32 evenly distributed fixed positions using a spherical Fibonacci sequence with radius 4 [S1].
- Render resolution: $512 \times 512$ [S1].
- Samples per pixel: 4 [S1].

**Edge Cases:**
- If $|M_{int}| = 0$, the score is undefined or set to infinity, indicating complete silhouette mismatch [G].
- Normals must be normalized before comparison to ensure valid dot products [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_normal_l2_score(N_pred, N_gt, M_pred, M_gt):
    # M_pred, M_gt: H x W boolean tensors
    # N_pred, N_gt: H x W x 3 float tensors, normalized
    
    # 1. Compute intersection mask
    M_int = M_pred & M_gt
    
    # 2. Get valid indices
    valid_indices = torch.nonzero(M_int, as_tuple=True)
    num_valid = len(valid_indices[0])
    
    if num_valid == 0:
        return float('inf') # Or handle as error
        
    # 3. Gather normals for valid pixels
    N_pred_valid = N_pred[valid_indices]
    N_gt_valid = N_gt[valid_indices]
    
    # 4. Compute squared L2 distance
    # ||a - b||^2 = ||a||^2 + ||b||^2 - 2(a . b)
    # Assuming normalized normals: ||a||^2 = 1, ||b||^2 = 1
    dot_product = torch.sum(N_pred_valid * N_gt_valid, dim=-1)
    squared_diff = 2.0 - 2.0 * dot_product
    
    # 5. Average
    score = torch.mean(squared_diff)
    return score
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(H \times W)$ per camera view, dominated by mask intersection and element-wise operations.
- **Memory Complexity:** $O(H \times W)$ to store intermediate masks and normal tensors.
- **Strengths:** Directly correlates with shading quality; sensitive to local geometric errors like bumps or dents.
- **Limitations:** Sensitive to normal orientation flips (sign ambiguity) if not handled; requires accurate ground-truth normals from a reference mesh.
- **When to use:** When shading fidelity is critical and silhouette alignment is handled separately (e.g., via iIoU) [S1].

## Background
Originated in the context of Neural Geometric Level of Detail (Neural GLOD) for real-time rendering with implicit 3D shapes [S1]. It is used to evaluate the quality of predicted surface normals against interpolated normals from a reference mesh, ensuring that the rendered geometry produces correct shading [S1].

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
