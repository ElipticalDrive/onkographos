# 2D Object Matching and Supervision

Also called: Hungarian Algorithm Association, sIoU/CES Cost Calculation, 2D Object Supervision with Hungarian Matching, Optimal Association and Supervision

## Overview
This method supervises projected 2D object predictions against 2D ground truth labels using the Hungarian algorithm to handle variable object counts and ordering across views [S1, unofficial]. It associates predicted 2D object masks with ground truth masks based on Soft Intersection-over-Union (sIoU) and Cross-Entropy Score (CES) costs, enabling effective supervision for object decomposition in 3D scenes [S1, unofficial]. Unlike standard per-pixel loss, this approach establishes a one-to-one correspondence between predicted and ground truth instances, ensuring multi-view consistency for the object fields defined in [2D Mask Matching](10596-2d-mask-matching.md).

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $L$ images of 2D object predictions $\{I_1, \dots, I_l, \dots, I_L\}$, where $I_l \in \mathbb{R}^{U \times V \times (H+1)}$. The last dimension $H+1$ consists of $H$ solid object predictions and 1 background/empty slot.
    *   $L$ images of 2D ground truth object labels $\{\bar{I}_1, \dots, \bar{I}_l, \dots, \bar{I}_L\}$, where $\bar{I}_l \in \mathbb{R}^{U \times V \times T}$. $T$ is the number of ground truth objects in the scene.
    *   $H$: Predefined number of object slots in the prediction.
    *   $T$: Number of ground truth objects.
*   **Outputs:**
    *   The 2D object loss $\ell_{2d\_obj}$.
    *   Reordered predicted masks aligned with ground truth indices.

**Data Structures**
*   **Prediction Mask Tensor ($M$):** Reshaped from the first $H$ dimensions of $I_l$. Shape: $\mathbb{R}^{N \times H}$, where $N = U \times V$. $M^n_h$ is the value at pixel $n$ for object slot $h$.
*   **Ground Truth Mask Tensor ($\bar{M}$):** Reshaped from $\bar{I}_l$. Shape: $\mathbb{R}^{N \times T}$. $\bar{M}^n_t$ is the value at pixel $n$ for ground truth object $t$.

**Steps**
1.  **Reshape:** For each image pair $l$, extract the first $H$ dimensions of $I_l$ and reshape to $M \in \mathbb{R}^{N \times H}$. Reshape $\bar{I}_l$ to $\bar{M} \in \mathbb{R}^{N \times T}$.
2.  **Cost Matrix Construction:** Compute the cost matrix $C \in \mathbb{R}^{H \times T}$ where each entry $C_{h,t}$ is the sum of sIoU and CES costs between predicted mask $h$ and ground truth mask $t$.
3.  **Association:** Feed $M$ and $\bar{M}$ into the Hungarian algorithm (Kuhn, 1955) to find the optimal one-to-one association between predicted and ground truth masks based on $C$ [S1, unofficial].
4.  **Reordering:** Reorder the predicted object masks in $M$ to align with the $T$ ground truth masks according to the association result.
5.  **Loss Calculation:** Compute the final loss by averaging the costs of the associated pairs across all $L$ images.

**Formulas**
The sIoU cost between predicted mask $h$ and ground truth mask $t$ is defined as [S1, unofficial]:

$$
\boldsymbol {C}^{sIoU}_{h,t} = \frac { - \sum _{n=1}^N(M^n_h * \bar {M}^n_t)}{ \sum _{n=1}^N M^n_h+\sum _{n=1}^{N}\bar {M}^n_t- \sum _{n=1}^{N}(M^n_h *\bar {M}^n_t)}
$$

*Note: One source defines the numerator as positive [S1, unofficial], but the loss minimization context and the other source [S1, unofficial] use the negative sign to represent a cost (lower is better for overlap). The formula above follows the minimization convention where higher overlap yields a lower (more negative) cost.*

The Cross-Entropy Score (CES) cost is defined as [S1, unofficial]:

$$
\boldsymbol {C}^{CES}_{h,t} =-\frac {1}{N} \sum _{n=1}^{N} \left [\bar {M}^n_t\log M^n_h + (1 - \bar {M}^n_t)\log (1-M^n_h)\right ]
$$

The total cost for a pair is $C_{h,t} = C^{sIoU}_{h,t} + C^{CES}_{h,t}$.

After association, the loss for image $l$ is computed by averaging the costs of the matched pairs (where index $t$ corresponds to the matched predicted index) [S1, unofficial]:

$$
sIoU_l = \frac {1}{T}\sum _{t=1}^T(\boldsymbol {C}^{sIoU}_{t,t}), \quad CES_l = \frac {1}{T}\sum _{t=1}^T(\boldsymbol {C}^{CES}_{t,t})
$$

The final 2D object loss is [S1, unofficial]:

$$
\ell _{2d\_obj} = \frac {1}{L}\sum _{l=1}^L (sIoU_l + CES_l)
$$

**Parameters**
*   $H$: Predefined number of object predictions. Typical value: Not specified in sources, but must be $\ge$ max possible objects in scene.
*   $T$: Number of ground truth objects. Varies per scene.
*   $L$: Number of images in the batch/view set.
*   $N$: Total pixels ($U \times V$).

**Edge Cases and Pitfalls**
*   **Denominator Zero:** If both predicted and ground truth masks are empty for a pair, the sIoU denominator is zero. [G] Implement a guard to return 0 or a large penalty cost if $\sum M + \sum \bar{M} - \sum (M * \bar{M}) = 0$.
*   **Variable Object Counts:** $H$ and $T$ may differ. The Hungarian algorithm handles rectangular cost matrices, but unused slots in $M$ (if $H > T$) or unmatched ground truths (if $T > H$) must be handled. The sources assume $T$ is fixed for notation, but in practice, if $T > H$, some ground truths cannot be matched. [G] Typically, $H$ is set to the maximum expected objects, or a dummy "background" class is used.
*   **Log Domain:** Ensure $M^n_h$ and $\bar{M}^n_t$ are clamped to $[\epsilon, 1-\epsilon]$ before log operations to avoid $-\infty$.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.optimize import linear_sum_assignment

def compute_2d_object_loss(predictions, ground_truths, H, T, L, N, epsilon=1e-7):
    """
    predictions: List of L tensors, shape (U, V, H+1).
    ground_truths: List of L tensors, shape (U, V, T).
    H: Predefined number of object slots.
    T: Number of ground truth objects.
    L: Number of images.
    N: Total pixels (U * V).
    """
    total_loss = 0.0
    
    for l in range(L):
        # 1. Reshape
        # Take first H dimensions of prediction
        pred_mask = predictions[l][:, :, :H] 
        pred_mask = pred_mask.reshape(N, H)
        
        # Reshape ground truth
        gt_mask = ground_truths[l].reshape(N, T)
        
        # 2. Compute Cost Matrix
        cost_matrix = np.zeros((H, T))
        
        for h in range(H):
            for t in range(T):
                m_h = pred_mask[:, h]
                m_t = gt_mask[:, t]
                
                # sIoU Cost
                intersection = np.sum(m_h * m_t)
                union = np.sum(m_h) + np.sum(m_t) - intersection
                
                if union == 0:
                    # Guard against division by zero
                    s_iou_cost = 0.0 
                else:
                    # Negative sign for minimization (higher overlap -> lower cost)
                    s_iou_cost = -intersection / union
                
                # CES Cost
                # Clamp for log stability
                m_h_clamped = np.clip(m_h, epsilon, 1 - epsilon)
                m_t_clamped = np.clip(m_t, epsilon, 1 - epsilon)
                
                ce_term1 = m_t_clamped * np.log(m_h_clamped)
                ce_term2 = (1 - m_t_clamped) * np.log(1 - m_h_clamped)
                ces_cost = -np.mean(ce_term1 + ce_term2)
                
                cost_matrix[h, t] = s_iou_cost + ces_cost
        
        # 3. Hungarian Algorithm
        row_ind, col_ind = linear_sum_assignment(cost_matrix)
        
        # 4. Reorder and Compute Loss
        # The loss is computed on the matched pairs
        # Note: If H != T, linear_sum_assignment returns min(H, T) pairs.
        # The sources define loss over T ground truths. 
        # If H < T, some GTs are unmatched. If H > T, some preds are unmatched.
        # Assuming H >= T for full supervision as per typical setup.
        
        s_iou_l = 0.0
        ces_l = 0.0
        matched_count = 0
        
        for r, c in zip(row_ind, col_ind):
            # Only count matches if we are matching against all T GTs
            # If H > T, we only care about the T matches corresponding to GTs
            if c < T:
                s_iou_l += cost_matrix[r, c] # This includes both sIoU and CES
                # To separate them for the final formula structure:
                # We need to re-calculate or store them. 
                # However, the formula sums C_t,t. 
                # Let's just sum the costs directly.
                matched_count += 1
        
        # The source formula divides by T. 
        # If matched_count < T, we divide by T as per source definition.
        if T > 0:
            s_iou_l = s_iou_l / T
            ces_l = ces_l / T # This is approximate if we summed combined costs
            
        # More precise separation for the final loss:
        # Re-calculate or store sIoU and CES separately for matched pairs
        s_iou_sum = 0.0
        ces_sum = 0.0
        for r, c in zip(row_ind, col_ind):
            if c < T:
                m_h = pred_mask[:, r]
                m_t = gt_mask[:, c]
                
                intersection = np.sum(m_h * m_t)
                union = np.sum(m_h) + np.sum(m_t) - intersection
                if union == 0:
                    s_iou_val = 0.0
                else:
                    s_iou_val = -intersection / union
                
                m_h_c = np.clip(m_h, epsilon, 1 - epsilon)
                m_t_c = np.clip(m_t, epsilon, 1 - epsilon)
                ces_val = -np.mean(m_t_c * np.log(m_h_c) + (1 - m_t_c) * np.log(1 - m_h_c))
                
                s_iou_sum += s_iou_val
                ces_sum += ces_val
        
        if T > 0:
            s_iou_l = s_iou_sum / T
            ces_l = ces_sum / T
        else:
            s_iou_l = 0.0
            ces_l = 0.0
            
        total_loss += (s_iou_l + ces_l)
        
    return total_loss / L
```

## Operating characteristics
*   **Time Complexity:** $O(L \cdot H \cdot T \cdot N)$ for cost matrix computation, where $N=U \times V$. The Hungarian algorithm takes $O(H^2 T)$ or $O(H T^2)$ depending on implementation. For typical values ($H, T \ll N$), the cost matrix computation dominates.
*   **Memory Complexity:** $O(N \cdot (H+T))$ to store the reshaped masks.
*   **Strengths:** Handles variable numbers of objects and ordering issues. Provides instance-level supervision rather than just pixel-level.
*   **Limitations:** Requires ground truth masks. The Hungarian algorithm is non-differentiable, so gradients flow through the cost calculation but not the assignment itself. Performance depends on the quality of the initial predictions.
*   **When to use:** When decomposing 3D scenes into distinct objects and multi-view consistency of object identities is required.

## Background
Originated in the DM-NeRF paper (Aliev et al., 2022/2023) for 3D scene geometry decomposition [S1, unofficial]. It adapts the Optimal Association and Supervision strategy from 3D-BoNet (Yang et al., 2019b) to the context of NeRF-based object fields. The method addresses the issue of inconsistent object ordering and variable counts across different views in 3D reconstruction.

## Sources
[S1] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
