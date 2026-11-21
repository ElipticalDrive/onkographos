# Semantic Loss Computation
## Overview
Semantic Loss Computation computes the cross-entropy loss over coarse and fine semantic predictions for query rays to train a generalizable semantic field [S1]. It differs from standard photometric NeRF losses by enforcing semantic consistency across views via ray-based feature projection, allowing the model to learn contextual geometry for semantic information rather than just color [S1]. It is a core component of the Two-stage NeRF Rendering pipeline, specifically enabling the training of the semantic head alongside the geometry-aware network [S1].

## How it works
### Inputs and Outputs
**Inputs:**
1.  **Query Rays ($\mathcal{R}$):** A set of sample rays within a training batch, generated via two-stage (coarse and fine) ray sampling [S1].
2.  **Source Views:** Reference images and their associated features used for cross-reprojection [S1].
3.  **Ground Truth Labels:** Semantic segmentation masks for the query view [S1].
4.  **Network Outputs:**
    *   Coarse semantic logits ($\hat{p}_{lc}$) for each ray [S1].
    *   Fine semantic logits ($\hat{p}_{lf}$) for each ray [S1].
    *   Photometric color predictions (used in the total loss, though not in $L_{sem}$ itself) [S1].

**Outputs:**
*   Scalar value $L_{\text{sem}}$ representing the semantic cross-entropy loss [S1].

### Data Structures
*   **Ray Batch:** A list or tensor of rays $r \in \mathcal{R}$. Each ray contains sampling points and associated features.
*   **Semantic Logits:** Tensors of shape $(N, L)$, where $N$ is the number of rays in the batch and $L$ is the number of semantic classes.
*   **Ground Truth Probabilities:** Tensors of shape $(N, L)$, representing the one-hot or soft labels for the query view pixels corresponding to the rays.

### Steps
1.  **Ray Projection and Feature Fetching:** At training time, project each query ray (not just a single point) into each source view. Fetch the corresponding ray-based feature from the source views. This feature contains rich contextual information within each intra-view [S1].
2.  **Two-Stage Sampling:** Apply two-stage ray sampling (coarse and fine) to generate the sample rays within the training batch, consistent with the NeRF sampling strategy [S1].
3.  **Logit Generation:** Pass the sampled rays through the semantic network to obtain coarse semantic logits ($\hat{p}_{lc}$) and fine semantic logits ($\hat{p}_{lf}$) [S1].
4.  **Probability Conversion:** Convert the logits to probabilities using the softmax function (implied by the cross-entropy formulation) [S1].
5.  **Loss Calculation:** Compute the cross-entropy loss for both coarse and fine predictions against the ground truth probabilities [S1].

### Formulas
The semantic loss $L_{\text{sem}}$ is defined as [S1]:

$$
L_{\text{sem}} = -\sum_{r \in \mathcal{R}} \sum_{l=1}^{L} \left( p_l(r) \log \hat{p}_{lc}(r) + p_l(r) \log \hat{p}_{lf}(r) \right)
$$

Where:
*   $\mathcal{R}$ is the set of sample rays within a training batch [S1].
*   $1 \le l \le L$ is the class index, where $L$ is the total number of semantic classes [S1].
*   $p_l(r)$ is the multi-class probability at class $l$ of the ground truth for query ray $r$ [S1].
*   $\hat{p}_{lc}(r)$ is the multi-class probability at class $l$ of the coarse semantic logits for ray $r$ [S1].
*   $\hat{p}_{lf}(r)$ is the multi-class probability at class $l$ of the fine semantic logits for ray $r$ [S1].

The total loss used for training combines this with the photometric loss [S1]:

$$
L_{\text{total}} = \lambda_1 L_{\text{sem}} + \lambda_2 L_{\text{photometric}}
$$

Where $L_{\text{photometric}}$ is the color rendering loss from the geometry-aware network [S1].

### Parameters
*   **$\lambda_1$ (Semantic Weight):** Weighting factor for semantic loss. Typical value: 1.0 for main experiments [S1].
*   **$\lambda_2$ (Photometric Weight):** Weighting factor for photometric loss. Typical value: 1.0 for main experiments [S1].
*   **Ablation Settings:** $\lambda_1 = 0, \lambda_2 = 1$ is used for ablation studies to isolate photometric training [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_semantic_loss(query_rays, ground_truth_labels, coarse_logits, fine_logits, num_classes):
    """
    Computes the semantic loss L_sem for a batch of rays.
    
    Args:
        query_rays: List of rays in the batch.
        ground_truth_labels: Tensor of shape (num_rays, num_classes) containing 
                             ground truth probabilities p_l(r).
        coarse_logits: Tensor of shape (num_rays, num_classes) containing 
                       coarse semantic logits.
        fine_logits: Tensor of shape (num_rays, num_classes) containing 
                     fine semantic logits.
        num_classes: Integer L, the number of semantic classes.
        
    Returns:
        Scalar loss value L_sem.
    """
    # Convert logits to probabilities using softmax
    coarse_probs = softmax(coarse_logits, dim=-1)
    fine_probs = softmax(fine_logits, dim=-1)
    
    # Initialize loss accumulator
    loss = 0.0
    
    # Iterate over rays in the batch
    for r in query_rays:
        # Get ground truth probabilities for this ray
        p_l = ground_truth_labels[r]
        
        # Get coarse and fine probabilities for this ray
        p_lc = coarse_probs[r]
        p_lf = fine_probs[r]
        
        # Compute cross-entropy for coarse prediction
        # Sum over classes l=1 to L
        coarse_loss = -sum(p_l[l] * log(p_lc[l]) for l in range(num_classes))
        
        # Compute cross-entropy for fine prediction
        fine_loss = -sum(p_l[l] * log(p_lf[l]) for l in range(num_classes))
        
        # Accumulate loss
        loss += coarse_loss + fine_loss
        
    return loss
```

### Edge Cases and Pitfalls
*   **Zero Probabilities:** If $\hat{p}_{lc}(r)$ or $\hat{p}_{lf}(r)$ is zero, the log term becomes undefined. Standard numerical stability techniques (e.g., clipping probabilities to a small epsilon value like $1e-7$) should be applied [G].
*   **Class Imbalance:** The loss is a simple sum over classes. If certain classes are rare, the loss may be dominated by frequent classes. The sources do not specify class weighting [S1].
*   **Ray Projection Accuracy:** The method relies on projecting query rays to source views. Incorrect projection (e.g., due to inaccurate camera poses or depth) will fetch wrong features, leading to incorrect semantic predictions [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays in the batch and the number of semantic classes [S1]. The cost includes projecting rays to source views and fetching features, which adds overhead compared to point-based sampling [S1].
*   **Memory Complexity:** Depends on the number of source views and the size of the ray-based features fetched. The sources mention high GPU memory usage for dense attention mechanisms in related modules (CRA), but do not specify the exact memory footprint of the loss computation itself [S1].
*   **Performance Impact:** The semantic loss drives the model to learn contextual geometry. On ScanNet generalization settings, varying $\lambda_1/\lambda_2$ ratios significantly affects PSNR and mIoU [S1]:
    *   $\lambda_1=1, \lambda_2=0$: 17.49 PSNR / 55.10 mIoU
    *   $\lambda_1=0.75, \lambda_2=0.25$: 25.26 PSNR / 56.51 mIoU
    *   $\lambda_1=0.5, \lambda_2=0.5$: 25.35 PSNR / 57.15 mIoU
    *   $\lambda_1=0.25, \lambda_2=0.75$: 26.24 PSNR / 58.12 mIoU
    *   $\lambda_1=0, \lambda_2=1$: 26.57 PSNR / 3.62 mIoU [S1]
*   **Strengths:** Enables generalization to unseen semantic scenarios. Semantic loss alone can drive the model to learn reasonable contextual geometry [S1].
*   **Limitations:** Requires accurate ray projection and feature fetching. High computational cost and GPU memory usage associated with the cross-reprojection attention module [S1].

## Background
Semantic Loss Computation was introduced in the paper "Semantic Ray: Learning a Generalizable Semantic Field with Cross-Reprojection Attention" by Richard Zhang et al. (CVPR 2023) [S1]. It builds upon the NeRF rendering pipeline and Semantic-NeRF [63], adapting the loss function to work with ray-based features and two-stage sampling. The method was designed to improve generalization to unseen semantic scenarios by leveraging cross-view attention and ray-based feature projection [S1].

## Sources
[S1] Semantic Ray: Learning a Generalizable Semantic Field with Cross-Reprojection Attention, https://doi.org/10.1109/cvpr52729.2023.01668, fetched 2026-10-29
