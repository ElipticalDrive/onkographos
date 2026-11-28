# MIMO Objective
## Overview
The MIMO objective is the total training loss for MIMO-NeRF, combining pixel-wise reconstruction errors with a 3D consistency regularization term across $M$ reformulated multi-input multi-output (MIMO) neural radiance fields [S1]. It enforces consistency between parallel MLPs trained with different sampling configurations, enabling faster rendering without sacrificing quality [S1].

## How it works
The method trains $M$ reformulated MIMO MLPs, each using a different shift parameter $s$ (or variation reduction configuration) and a specific repetition count $R_m$ [S1]. The total number of samples ("# Run") across all $M$ MLPs is constrained to not exceed the sample count of a single original SISO (Single-Input Single-Output) MLP [S1].

### Inputs and Outputs
**Inputs:**
*   $M$: The number of reformulated MIMO MLPs.
*   $\mathbf{C}$: The ground-truth RGB color for the current pixel/view.
*   $\hat{\mathbf{C}}^m$: The final composited predicted RGB color from the $m$-th MIMO MLP.
*   $\mathbf{c}_i^m$: The color of the $i$-th sample along the ray for the $m$-th MLP.
*   $\alpha_i^m$: The alpha (opacity) of the $i$-th sample along the ray for the $m$-th MLP.
*   $N$: The number of samples along the ray.
*   $R_m$: The repetition count (or reliability metric) for the $m$-th MLP.
*   $R_{\max}$: The maximum $R$ value among all $M$ MLPs ($R_{\max} = \max_{m \in \{1, \dots, M\}} R_m$).
*   $\lambda$: Hyperparameter balancing pixel-wise and 3D consistency losses.

**Outputs:**
*   $\mathcal{L}_{\text{MIMO}}$: The scalar total loss value.

### Data Structures
Not given in the sources.

### Formulas
The total loss is defined as [S1]:
$$
\mathcal{L}_{\text{MIMO}} = \mathcal{L}_{\text{pixel}}^{\text{MIMO}} + \lambda \mathcal{L}_{3D}
$$

1.  **Pixel-wise Loss**: Sum of squared errors for each MLP's final rendered color against the ground truth [S1]:
    $$
    \mathcal{L}_{\text{pixel}}^{\text{MIMO}} = \sum_{m=1}^M \|\hat{\mathbf{C}}^m - \mathbf{C}\|_2^2
    $$

2.  **3D Consistency Loss**: Encourages the $M$ MLPs to produce consistent colors and alphas at the sample level [S1]. It consists of color and alpha components [S1]:
    $$
    \mathcal{L}_{3D} = \mathcal{L}_{3D}^{\text{color}} + \mathcal{L}_{3D}^{\text{alpha}}
    $$

    The **Color 3D Consistency Loss** is [S1]:
    $$
    \mathcal{L}_{3D}^{\text{color}} = \sum_{m_1=1}^{M-1} \sum_{m_2=m_1+1}^M \frac{1}{N} \sum_{i=1}^N \left[ \mu_{m_1}^{m_2} \|\mathbf{c}_i^{m_1} - \text{sg}(\mathbf{c}_i^{m_2})\|_2^2 + \mu_{m_2}^{m_1} \|\text{sg}(\mathbf{c}_i^{m_1}) - \mathbf{c}_i^{m_2}\|_2^2 \right]
    $$

    The **Alpha 3D Consistency Loss** is [S1]:
    $$
    \mathcal{L}_{3D}^{\text{alpha}} = \sum_{m_1=1}^{M-1} \sum_{m_2=m_1+1}^M \frac{1}{N} \sum_{i=1}^N \left[ \mu_{m_1}^{m_2} \|\alpha_i^{m_1} - \text{sg}(\alpha_i^{m_2})\|_2^2 + \mu_{m_2}^{m_1} \|\text{sg}(\alpha_i^{m_1}) - \alpha_i^{m_2}\|_2^2 \right]
    $$

    Here, $\text{sg}(\cdot)$ denotes the stop-gradient operation, which prevents gradients from flowing through the target branch in the consistency term [S1].

3.  **Asymmetric Weight**: The weight $\mu_{m_j}^{m_i}$ is defined based on the reliability of the MLPs [S1]. MLPs with greater $R_m$ have lower ambiguity and are considered more reliable, thus their influence on the consistency loss is reduced [S1]. The explicit formula provided in the source is [S1]:
    $$
    \mu_{m_j}^{m_i} = \sqrt{m_j} \sqrt{R_{\max}} \sqrt{R_{m_i}}
    $$
    *Note: The source text explicitly provides this formula. However, the accompanying description states that weights are derived from $\sqrt{R_{m_j}} / \sqrt{R_{\max}}$ to reduce the effect of reliable MLPs. The use of the index $m_j$ in the numerator of the source formula is inconsistent with the description of weighting by $R$ [S1].*

### Parameters
*   $\lambda$: Hyperparameter balancing pixel-wise loss and 3D consistency loss. Typical values are not specified in the sources [S1].
*   $M$: Number of reformulated MLPs.
*   $R_m$: Repetition count for MLP $m$.
*   $R_{\max}$: Maximum $R$ value across all MLPs.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def stop_gradient(x):
    """Detaches x from the computation graph. Framework-specific behavior [G]."""
    return x

def compute_mimo_loss(colors, alphas, gt_color, R_vals, lam):
    """
    Computes the MIMO training objective.
    
    Args:
        colors: List of M lists, each containing N tuples of 3 floats. colors[m][i] is c_i^m.
        alphas: List of M lists, each containing N floats. alphas[m][i] is alpha_i^m.
        gt_color: Tuple of 3 floats. Ground truth RGB for the pixel.
        R_vals: List of M floats. R_m for each MLP.
        lam: Float. Balances pixel and 3D consistency losses.
        
    Returns:
        total_loss: Float. The MIMO objective value.
    """
    M = len(R_vals)
    N = len(colors[0])
    R_max = max(R_vals)

    # 1. Pixel-wise Loss
    pixel_loss = 0.0
    for m in range(M):
        c_hat_m = colors[m][-1]  # Final composited color for MLP m
        pixel_loss += sum((c_hat_m[k] - gt_color[k])**2 for k in range(3))

    # 2. 3D Consistency Loss
    l_3d_color = 0.0
    l_3d_alpha = 0.0

    for m1 in range(M - 1):
        for m2 in range(m1 + 1, M):
            # Compute weights using 1-based indexing as per source notation
            mj = m1 + 1
            mi = m2 + 1
            mu_12 = math.sqrt(mj) * math.sqrt(R_max) * math.sqrt(R_vals[mi - 1])
            mu_21 = math.sqrt(mi) * math.sqrt(R_max) * math.sqrt(R_vals[mj - 1])

            # Color consistency
            for i in range(N):
                c1 = colors[m1][i]
                c2 = colors[m2][i]
                c2_sg = stop_gradient(c2)
                c1_sg = stop_gradient(c1)

                diff1 = [c1[k] - c2_sg[k] for k in range(3)]
                diff2 = [c2[k] - c1_sg[k] for k in range(3)]
                norm1 = sum(d**2 for d in diff1)
                norm2 = sum(d**2 for d in diff2)
                l_3d_color += (mu_12 * norm1 + mu_21 * norm2) / N

            # Alpha consistency
            for i in range(N):
                a1 = alphas[m1][i]
                a2 = alphas[m2][i]
                a2_sg = stop_gradient(a2)
                a1_sg = stop_gradient(a1)

                diff_a1 = a1 - a2_sg
                diff_a2 = a2 - a1_sg
                l_3d_alpha += (mu_12 * diff_a1**2 + mu_21 * diff_a2**2) / N

    l_3d = l_3d_color + l_3d_alpha
    return pixel_loss + lam * l_3d
```

### Edge Cases and Pitfalls
*   **Formula Ambiguity**: The source provides the formula $\mu_{m_j}^{m_i} = \sqrt{m_j} \sqrt{R_{\max}} \sqrt{R_{m_i}}$ but describes the weight as being based on reliability ($R_m$) where higher $R$ reduces ambiguity [S1]. The use of the index $m_j$ in the numerator of the source formula is inconsistent with the description of weighting by $R$ [S1]. Implementers should verify if $\sqrt{m_j}$ is a typo for $\sqrt{R_{m_j}}$ or if the index is intended [S1].
*   **Stop-Gradient**: Forgetting to apply the stop-gradient (`sg`) operation to the target branch in the consistency loss will cause gradients to flow through both branches, potentially destabilizing training or negating the regularization effect [S1].

## Subtopics

### Unified Metric Aggregation
Aggregates various evaluation metrics into a single score. It is used for final model assessment, not training optimization. [Unified Metric Aggregation](3644-unified-metric-aggregation.md)

### PSNR Evaluation Pipeline
Calculates Peak Signal-to-Noise Ratio for quality assessment. It is a post-training or validation step. [PSNR Evaluation Pipeline](4315-psnr-evaluation-pipeline.md)

### Masked Evaluation
Computes metrics only on valid pixels (e.g., excluding background). It is used for evaluation, not training. [Masked Evaluation](7531-masked-evaluation.md)

## Operating characteristics
*   **Time Complexity**: The 3D consistency loss involves pairwise comparisons between all $M$ MLPs and summation over $N$ samples, leading to $O(M^2 \cdot N)$ complexity for the consistency term [G]. The pixel-wise loss is $O(M)$ [G].
*   **Memory Complexity**: Requires storing predictions for all $M$ MLPs across $N$ samples, $O(M \cdot N)$ [G].
*   **Strengths**: Enables faster rendering by allowing fewer samples per MLP while maintaining quality through consistency regularization [S1]. It effectively regularizes the 3D space, which pixel-wise loss alone cannot do [S1].
*   **Limitations**: Increased computational overhead during training due to the pairwise consistency calculations [S1]. The importance of the weighting scheme is investigated via an ablation study [S1].
*   **When to use**: When training MIMO-NeRF or similar multi-output NeRF variants where speed-quality trade-offs are managed via sample reduction and consistency [G].
*   **When not to use**: For standard single-MLP NeRF training, or when evaluation metrics are needed (use sibling methods) [G].

## Background
The MIMO objective was introduced in the paper "MIMO-NeRF: Fast Neural Rendering with Multi-Input Multi-Output Neural Radiance Fields" [S1]. The method addresses the need for fast neural rendering by reformulating the NeRF into multiple MLPs with different sampling configurations, ensuring the total computational cost does not exceed that of a standard SISO MLP while improving rendering speed and quality through consistency constraints [S1].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-Input Multi-Output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
