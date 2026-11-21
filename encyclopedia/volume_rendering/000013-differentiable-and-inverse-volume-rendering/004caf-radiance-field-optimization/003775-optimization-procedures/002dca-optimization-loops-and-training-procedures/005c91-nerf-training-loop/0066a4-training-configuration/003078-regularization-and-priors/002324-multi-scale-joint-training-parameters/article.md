# Multi-scale Joint Training Parameters

Also called: Multi-scale joint training loss weights

## Overview
Multi-scale joint training parameters refer to the specific hyperparameter configuration used to balance photometric and auxiliary losses across different stages or resolutions during the optimization of neural radiance fields. This method addresses the instability inherent in jointly optimizing complex, multi-component inverse rendering models by ensuring that gradient signals are appropriately weighted at each scale of the training hierarchy [S1]. It sets itself apart from sibling methods like [Surface Smoothness Regularization](12970-surface-smoothness-regularization.md) by explicitly tying loss weights to a multi-stage or multi-resolution progression rather than applying static priors to a single-stage network [S1].

## How it works
The method defines a composite loss function that aggregates errors from multiple network stages or resolution levels. The primary inputs are the rendered images from each stage $i$ and their corresponding ground truth images. The outputs are the scalar loss values used to update the network weights.

### Data Structures and Formulas
The total loss $\mathcal{L}$ is computed as a weighted sum over $D$ stages (where $D$ corresponds to the current number of stages or network growth depth) [S1]:

$$
\mathcal{L} = \sum_{i=1}^D \left( \beta_1 \mathcal{L}_{MSE}^i + \beta_2 \mathcal{L}_{unct}^i \right)
$$

Where:
*   $D$: The current number of stages in the training hierarchy. In dynamic growth schemes, $D-1$ represents the number of previous network growth events [S1].
*   $\mathcal{L}_{MSE}^i$: The Mean Squared Error (photometric) loss for the output image of stage $i$.
*   $\mathcal{L}_{unct}^i$: The uncertainty loss for the output image of stage $i$. This term typically derives from domain-specific assumptions about view-dependent noise or confidence in the rendered pixels [S1].
*   $\beta_1$: The weight for the photometric (MSE) loss component.
*   $\beta_2$: The weight for the uncertainty loss component.

### Parameters
*   $\beta_1 = 1.0$
*   $\beta_2 = 0.1$

These values are empirically determined to balance the contribution of pixel fidelity against the regularization provided by the uncertainty term [S1].

### Pseudocode
The following Python-like pseudocode illustrates the computation of the joint loss for a given batch of stages.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_multi_scale_joint_loss(rendered_images, ground_truth_images, uncertainty_maps, beta1=1.0, beta2=0.1):
    """
    Computes the multi-scale joint training loss.
    
    Args:
        rendered_images: List of tensors, shape (N, H, W, 3), for each stage i.
        ground_truth_images: List of tensors, shape (N, H, W, 3), for each stage i.
        uncertainty_maps: List of tensors, shape (N, H, W, 1), for each stage i.
        beta1: Weight for MSE loss (default 1.0).
        beta2: Weight for uncertainty loss (default 0.1).
        
    Returns:
        total_loss: Scalar tensor.
    """
    total_loss = 0.0
    num_stages = len(rendered_images)
    
    for i in range(num_stages):
        # Calculate MSE loss for stage i
        # Note: Standard MSE implementation
        mse_loss = torch.mean((rendered_images[i] - ground_truth_images[i]) ** 2)
        
        # Calculate uncertainty loss for stage i
        # Note: The exact formulation of L_unct is not specified in the source beyond its name.
        # In typical inverse rendering contexts, this may involve the variance or confidence
        # predicted by the network. Here we assume uncertainty_maps provides the necessary
        # scalar field for the loss term.
        # [G] Assuming a standard uncertainty regularization term proportional to the map values.
        unct_loss = torch.mean(uncertainty_maps[i])
        
        # Weighted sum for this stage
        stage_loss = beta1 * mse_loss + beta2 * unct_loss
        total_loss += stage_loss
        
    return total_loss
```

### Edge Cases and Pitfalls
*   **Precision:** The uncertainty loss $\mathcal{L}_{unct}$ requires careful handling of numerical stability, particularly if the uncertainty maps contain near-zero values that could lead to division by zero or vanishing gradients if used in a probabilistic likelihood formulation [S1].
*   **Stage Count ($D$):** In dynamic growth scenarios, $D$ changes as the network expands. The loss function must dynamically adjust the summation range to include only active stages [S1].

## Subtopics

### Cross-intrinsic Guidance
A coarse-to-fine training scheme that uses different camera intrinsic parameters to construct an image pyramid, enhancing multi-scale feature learning [S2]. It sets itself apart by modifying the camera projection matrix $K$ at each scale rather than just the image resolution, thereby enforcing multi-view consistency through geometric constraints [S2]. Choose this method when training multi-view Gaussian Splatting or NeRF variants where view-dependent noise is significant and multi-view consistency is critical [S2].

*   **Definition:** Constructs a 4-layer image pyramid with downsampling factors $S = \{2^{k-1} \mid k = 4, 3, 2, 1\}$ [S2].
*   **Step 1:** For each layer $s \in S$, reconfigure the intrinsic matrix $K$ by dividing the focal length $f$ and principal point coordinates $c_x, c_y$ by $s$:
    $$
    K'_s = \begin{bmatrix} f/s & 0 & c_x/s \\ 0 & f/s & c_y/s \\ 0 & 0 & 1 \end{bmatrix}
    $$
*   **Step 2:** Assign matched multi-view settings $M_s = \{M_4, M_3, M_2, M_1\}$ to each layer. Larger downsampling factors (coarser layers) use more views to provide stronger multi-view constraints [S2].
*   **Step 3:** Run only a few thousand iterations per stage for the first three coarse stages ($s=8, 4, 2$). Then start fine training ($s=1$) to refine details [S2].
*   **Parameters:** $S$ is the set of downsampling factors $\{1, 2, 4, 8\}$. The largest factor $s=8$ is empirically sufficient; $s=1$ means no downsampling [S2].
*   **Pitfalls:** Excessively large downsampling factors lead to blurry ground truth, decreasing fidelity and increasing refinement difficulty [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of stages $D$ and the number of views per stage. In dynamic growth schemes, the cost increases as $D$ grows [S1].
*   **Memory Complexity:** Proportional to the storage of intermediate rendered images and uncertainty maps for all active stages [S1].
*   **Strengths:** Stabilizes training in complex inverse rendering models by decoupling the optimization of geometry, appearance, and lighting across scales [S1]. Reduces view-dependent noise [S2].
*   **Limitations:** Requires careful tuning of stage counts and loss weights. Coarse stages may not capture sufficient detail, necessitating a fine-tuning phase [S2].
*   **When to use:** When training neural radiance fields or Gaussian Splatting models that exhibit instability due to interdependent modules or multi-view inconsistencies [S1, S2].

## Background
The concept of multi-scale joint training parameters originates from Recursive-NeRF, which introduced an efficient and dynamically growing NeRF architecture [S1]. The specific weights $\beta_1=1.0$ and $\beta_2=0.1$ were proposed to balance MSE and uncertainty losses in this dynamic growth framework [S1]. The related Cross-intrinsic Guidance variant was introduced in MVGS (Multi-view-regulated Gaussian Splatting) to address multi-view consistency issues by leveraging camera intrinsic parameter variations [S2].

## Sources
[S1] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S2] MVGS: Multi-view-regulated Gaussian Splatting for Novel View Synthesis, https://doi.org/10.48550/arXiv.2410.02103, fetched 2026-11-03
