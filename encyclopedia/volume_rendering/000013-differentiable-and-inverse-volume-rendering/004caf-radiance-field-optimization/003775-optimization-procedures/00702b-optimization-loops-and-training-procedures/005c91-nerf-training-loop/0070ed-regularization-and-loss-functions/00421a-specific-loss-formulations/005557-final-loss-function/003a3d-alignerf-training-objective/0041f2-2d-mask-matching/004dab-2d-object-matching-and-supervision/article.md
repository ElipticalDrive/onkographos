# 2D Object Matching and Supervision

Also called: Hungarian Algorithm Association, sIoU/CES Cost Calculation, 2D Object Supervision with Hungarian Matching, Optimal Association and Supervision

## Overview
This method supervises projected 2D object predictions against 2D ground truth labels using the Hungarian algorithm to handle variable object counts and ordering across views [S1, unofficial]. It associates predicted 2D object masks with ground truth masks based on Soft Intersection-over-Union (sIoU) and Cross-Entropy Score (CES) costs, enabling effective supervision for object decomposition in 3D scenes [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   $L$ images of 2D object predictions $\{I_1, \dots, I_L\}$, where $I_l \in \mathbb{R}^{U \times V \times (H+1)}$ [S1, unofficial]. The last dimension contains $H$ solid object predictions and 1 background/empty slot [S1, unofficial].
    *   $L$ images of 2D ground truth object labels $\{\bar{I}_1, \dots, \bar{I}_L\}$, where $\bar{I}_l \in \mathbb{R}^{U \times V \times T}$ [S1, unofficial].
    *   $H$: Predefined number of object slots in the prediction [S1, unofficial].
    *   $T$: Number of ground truth objects [S1, unofficial].
*   **Outputs:**
    *   The 2D object loss $\ell_{2d\_obj}$ [S1, unofficial].

**Data Structures**
*   **Prediction Mask Tensor ($M$):** Reshaped from the first $H$ dimensions of $I_l$ to $M \in \mathbb{R}^{N \times H}$, where $N = U \times V$ [S1, unofficial]. $M^n_h$ is the value at pixel $n$ for object slot $h$.
*   **Ground Truth Mask Tensor ($\bar{M}$):** Reshaped from $\bar{I}_l$ to $\bar{M} \in \mathbb{R}^{N \times T}$ [S1, unofficial]. $\bar{M}^n_t$ is the value at pixel $n$ for ground truth object $t$.

**Steps**
1.  **Reshape:** For each image pair $l$, extract the first $H$ dimensions of $I_l$ and reshape to $M \in \mathbb{R}^{N \times H}$. Reshape $\bar{I}_l$ to $\bar{M} \in \mathbb{R}^{N \times T}$ [S1, unofficial].
2.  **Cost Matrix Construction:** Compute the cost matrix $C \in \mathbb{R}^{H \times T}$ where each entry $C_{h,t} = C^{sIoU}_{h,t} + C^{CES}_{h,t}$ [S1, unofficial].
3.  **Association:** Feed $M$ and $\bar{M}$ into the Hungarian algorithm (Kuhn, 1955) to associate every ground truth 2D object mask with a unique predicted 2D object mask based on $C$ [S1, unofficial].
4.  **Reordering:** Reorder the predicted object masks in $M$ to align with the $T$ ground truth masks according to the association result [S1, unofficial].
5.  **Loss Calculation:** Compute the final loss by averaging the costs of the associated pairs across all $L$ images [S1, unofficial].

**Formulas**
The sIoU cost between predicted mask $h$ and ground truth mask $t$ is defined as [S1, unofficial]:

$$
\boldsymbol {C}^{sIoU}_{h,t} = \frac { - \sum _{n=1}^N(M^n_h \bar {M}^n_t)}{ \sum _{n=1}^N M^n_h+\sum _{n=1}^{N}\bar {M}^n_t- \sum _{n=1}^{N}(M^n_h \bar {M}^n_t)}
$$

*Note: One equation in the source uses a positive numerator [S1, unofficial], but the negative version is applied here to represent a cost for minimization.*

The Cross-Entropy Score (CES) cost is defined as [S1, unofficial]:

$$
\boldsymbol {C}^{CES}_{h,t} =-\frac {1}{N} \sum _{n=1}^{N} \left [\bar {M}^n_t\log M^n_h + (1 - \bar {M}^n_t)\log (1-M^n_h)\right ]
$$

The total cost for a pair is $C_{h,t} = C^{sIoU}_{h,t} + C^{CES}_{h,t}$ [S1, unofficial].

After association, the loss for image $l$ is computed by averaging the costs of the matched pairs (where index $t$ corresponds to the matched predicted index) [S1, unofficial]:

$$
sIoU_l = \frac {1}{T}\sum _{t=1}^T(\boldsymbol {C}^{sIoU}_{t,t}), \quad CES_l = \frac {1}{T}\sum _{t=1}^T(\boldsymbol {C}^{CES}_{t,t})
$$

The final 2D object loss is [S1, unofficial]:

$$
\ell _{2d\_obj} = \frac {1}{L}\sum _{l=1}^L (sIoU_l + CES_l)
$$

**Parameters**
*   $H$: Predefined number of object predictions [S1, unofficial]. Not given in the sources.
*   $T$: Number of ground truth objects [S1, unofficial]. Not given in the sources.
*   $L$: Number of images [S1, unofficial]. Not given in the sources.
*   $N$: Total pixels ($U \times V$) [S1, unofficial]. Not given in the sources.

**Edge Cases and Pitfalls**
*   **Denominator Zero:** If both predicted and ground truth masks are empty for a pair, the sIoU denominator is zero [G]. Implement a guard to return a large penalty cost or zero if $\sum M + \sum \bar{M} - \sum (M * \bar{M}) = 0$.
*   **Variable Object Counts:** $H$ and $T$ may differ [S1, unofficial]. The source notes that ground truth object counts usually differ per image, but a fixed $T$ is used to avoid notation abuse [S1, unofficial]. The Hungarian algorithm handles rectangular matrices, but if $H < T$, some ground truths remain unmatched [G].
*   **Log Domain:** $M^n_h$ and $\bar{M}^n_t$ must be clamped to $[\epsilon, 1-\epsilon]$ before log operations to avoid $-\infty$ [G].

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
        pred_mask = predictions[l][:, :, :H].reshape(N, H)
        gt_mask = ground_truths[l].reshape(N, T)
        
        # 2. Compute Cost Matrix
        cost_matrix = np.zeros((H, T))
        s_iou_matrix = np.zeros((H, T))
        ces_matrix = np.zeros((H, T))
        
        for h in range(H):
            for t in range(T):
                m_h = pred_mask[:, h]
                m_t = gt_mask[:, t]
                
                # sIoU Cost
                intersection = np.sum(m_h * m_t)
                union = np.sum(m_h) + np.sum(m_t) - intersection
                if union == 0:
                    s_iou_val = 0.0
                else:
                    s_iou_val = -intersection / union
                s_iou_matrix[h, t] = s_iou_val
                
                # CES Cost
                m_h_c = np.clip(m_h, epsilon, 1 - epsilon)
                m_t_c = np.clip(m_t, epsilon, 1 - epsilon)
                ce_term = m_t_c * np.log(m_h_c) + (1 - m_t_c) * np.log(1 - m_h_c)
                ces_val = -np.mean(ce_term)
                ces_matrix[h, t] = ces_val
                
                cost_matrix[h, t] = s_iou_val + ces_val
        
        # 3. Hungarian Algorithm
        row_ind, col_ind = linear_sum_assignment(cost_matrix)
        
        # 4. Reordering & Loss Calculation
        # The source defines loss over T ground truths.
        # We assume H >= T so that every GT has a match.
        s_iou_l = 0.0
        ces_l = 0.0
        matched_count = 0
        
        for r, c in zip(row_ind, col_ind):
            if c < T:
                s_iou_l += s_iou_matrix[r, c]
                ces_l += ces_matrix[r, c]
                matched_count += 1
                
        if T > 0:
            s_iou_l /= T
            ces_l /= T
        else:
            s_iou_l = 0.0
            ces_l = 0.0
            
        total_loss += (s_iou_l + ces_l)
        
    return total_loss / L
```

## Operating characteristics
*   **Time Complexity:** $O(L \cdot H \cdot T \cdot N)$ for cost matrix computation, where $N=U \times V$ [G]. The Hungarian algorithm takes $O(H^2 T)$ or $O(H T^2)$ depending on implementation [G].
*   **Memory Complexity:** $O(N \cdot (H+T))$ to store the reshaped masks [G].
*   **Strengths:** Handles variable numbers of objects and ordering issues [S1, unofficial]. Provides instance-level supervision [S1, unofficial].
*   **Limitations:** Requires ground truth masks [S1, unofficial]. The Hungarian algorithm is non-differentiable, so gradients flow through the cost calculation but not the assignment itself [G].
*   **When to use:** When decomposing 3D scenes into distinct objects and multi-view consistency of object identities is required [S1, unofficial].

## Background
Originated in the DM-NeRF paper (Yang et al., 2023) for 3D scene geometry decomposition [S1, unofficial]. It adapts the Optimal Association and Supervision strategy from 3D-BoNet (Yang et al., 2019b) to the context of NeRF-based object fields [S1, unofficial]. The method addresses the issue of inconsistent object ordering and variable counts across different views in 3D reconstruction [S1, unofficial].

## Sources
[S1] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
