# Monocular Depth Consistency Loss

Also called: Monocular Depth Loss

## Overview
Monocular Depth Consistency Loss enforces geometric consistency between the depth rendered from a neural radiance field (NeRF) and depth predicted by a pre-trained monocular depth estimator [S1]. It solves the problem of scale and shift ambiguity inherent in monocular depth priors by introducing learnable scale ($\eta$) and shift ($\epsilon$) parameters that align the rendered depth with the predicted depth [S1]. Unlike standard L2 depth losses that assume metric alignment, this method treats monocular depth as a relative cue, allowing the NeRF to optimize its geometry without requiring metric ground truth [S1]. It differs from [Multi-level Semantic Consistency Loss](7177-multi-level-semantic-consistency-loss.md) by focusing on geometric depth alignment rather than semantic feature consistency, and from [Optimal Transport Loss](7472-optimal-transport-loss.md) by using a closed-form least-squares alignment rather than distribution matching.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   $\hat{D}(\mathbf{r})$: Rendered depth for ray $\mathbf{r}$, obtained via differentiable volume rendering.
    *   $\bar{D}(\mathbf{r})$: Predicted monocular depth for the corresponding image pixel, obtained from a pre-trained network.
    *   $\boldsymbol{\theta}, \boldsymbol{\phi}$: NeRF parameters and auxiliary parameters (if any).
*   **Outputs:**
    *   $\mathcal{L}_{Mono}$: Scalar loss value.

**Data Structures:**
*   **Batch:** A set of rays $\{\mathbf{r}_i\}$ sampled for a single training step. The parameters $\eta$ and $\epsilon$ are computed per batch.
*   **Vectors:** $\hat{D}$ and $\bar{D}$ are treated as vectors of depth values for all pixels/rays in the current batch.

**Parameters:**
*   $\eta$: Scale parameter.
*   $\epsilon$: Shift parameter.
*   Typical values: Initialized to 1.0 and 0.0 respectively, or learned via gradient descent.

**Steps:**
1.  **Render Depth:** For each ray $\mathbf{r}$ in the current batch, compute the rendered depth $\hat{D}(\mathbf{r})$ using the current NeRF state.
2.  **Predict Monocular Depth:** Obtain the monocular depth prediction $\bar{D}(\mathbf{r})$ for the corresponding image coordinates.
3.  **Compute Alignment Parameters:** Calculate $\eta$ and $\epsilon$ for the current batch using the closed-form least-squares solution [S1]:
    $$ \eta = \left( \hat{D}(\mathbf{r})^T \hat{D}(\mathbf{r}) \right)^{-1} \hat{D}(\mathbf{r})^T \bar{D}(\mathbf{r}) $$
    $$ \epsilon = \bar{D}(\mathbf{r}) - \eta \hat{D}(\mathbf{r}) $$
    *Note: $\hat{D}(\mathbf{r})$ and $\bar{D}(\mathbf{r})$ here represent the vector of all depth values in the batch.*
4.  **Compute Loss:** Calculate the mean squared error between the aligned rendered depth and the monocular prediction [S1]:
    $$ \mathcal{L}_{Mono}(\boldsymbol{\theta}, \boldsymbol{\phi}) = \sum_i \mathbb{E}_{\mathbf{r} \sim \bar{D}_i} \left\| (\eta \hat{D}(\mathbf{r}) + \epsilon) - \bar{D}(\mathbf{r}) \right\|^2 $$
    In practice, this is implemented as the mean squared error over the batch pixels.
5.  **Backpropagate:** Compute gradients with respect to $\boldsymbol{\theta}$ (and $\boldsymbol{\phi}$ if applicable). Note that $\eta$ and $\epsilon$ are typically treated as fixed constants for the gradient computation of the NeRF parameters in a single step, or updated via a separate optimization loop, as they are derived from the current batch statistics [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def monocular_depth_consistency_loss(rendered_depth, mono_depth):
    """
    Computes the Monocular Depth Consistency Loss for a batch.
    
    Args:
        rendered_depth (Tensor): Shape (N,), rendered depth values for N rays/pixels.
        mono_depth (Tensor): Shape (N,), monocular depth predictions for N rays/pixels.
        
    Returns:
        Tensor: Scalar loss value.
    """
    # Step 3: Compute scale and shift via closed-form least squares
    # eta = (D_hat^T D_hat)^-1 D_hat^T D_bar
    # epsilon = D_bar - eta * D_hat
    
    # Dot products
    dot_dd = torch.dot(rendered_depth, rendered_depth)
    dot_dm = torch.dot(rendered_depth, mono_depth)
    
    # Avoid division by zero
    if dot_dd < 1e-8:
        eta = torch.tensor(1.0)
    else:
        eta = dot_dm / dot_dd
        
    epsilon = torch.mean(mono_depth) - eta * torch.mean(rendered_depth)
    # Note: The source formula for epsilon is vector-based: epsilon = D_bar - eta * D_hat.
    # In implementation, if eta is a scalar, epsilon is often computed as the mean residual
    # or per-pixel. The source implies a global shift for the batch.
    # Using the vector definition from source:
    # epsilon_vector = mono_depth - eta * rendered_depth
    # However, the loss formula uses a single epsilon. 
    # Standard least squares for y = ax + b gives b = mean(y) - a * mean(x).
    # The source formula epsilon = D_bar - eta D_hat suggests a vector result, 
    # but the loss uses a scalar epsilon. We assume epsilon is the scalar bias term 
    # derived from the least squares fit.
    
    # Re-evaluating source formula: epsilon = D_bar(r) - eta D_hat(r). 
    # This looks like a residual vector. But the loss is || (eta D_hat + epsilon) - D_bar ||^2.
    # If epsilon is a vector, this is just || eta D_hat - (D_bar - epsilon) ||^2.
    # Usually, in such losses, epsilon is a scalar bias. 
    # Let's stick to the scalar interpretation for stability, or follow the exact vector math if implied.
    # Given "calculated separately for each batch", it implies global batch parameters.
    # Least squares solution for y = ax + b:
    # a = cov(x,y)/var(x)
    # b = mean(y) - a * mean(x)
    
    # Let's implement the standard least squares scalar parameters.
    mean_d_hat = torch.mean(rendered_depth)
    mean_d_bar = torch.mean(mono_depth)
    
    # Covariance and Variance
    cov = torch.mean((rendered_depth - mean_d_hat) * (mono_depth - mean_d_bar))
    var = torch.var(rendered_depth, unbiased=False)
    
    if var < 1e-8:
        eta = torch.tensor(1.0)
        epsilon = mean_d_bar - mean_d_hat
    else:
        eta = cov / var
        epsilon = mean_d_bar - eta * mean_d_hat

    # Step 4: Compute Loss
    aligned_depth = eta * rendered_depth + epsilon
    loss = torch.mean((aligned_depth - mono_depth) ** 2)
    
    return loss
```

**Edge Cases and Pitfalls:**
*   **Batch Variance:** Predicted depth maps can vary in scale and shift across different batches. $\eta$ and $\epsilon$ must be recalculated for each batch [S1].
*   **Degenerate Input:** If rendered depth has zero variance (e.g., all pixels are the same depth), the least-squares solution for $\eta$ is undefined. A guard value (e.g., $\eta=1$) is required [G].
*   **Outliers:** Monocular depth estimators can produce significant errors in texture-less or reflective regions. This loss is sensitive to such outliers unless combined with robust loss functions or filtering (see [Pseudo-depth Supervised Loss](7487-pseudo-depth-supervised-loss.md)) [S1][S2, unofficial].

## Subtopics

### [Reliable Depth Estimation via Reprojection](7293-reliable-depth-estimation-via-reprojection.md)
This method provides a data consistency mechanism for monocular depth estimation in radiance fields by leveraging geometric reprojection to identify reliable depth estimates and filter outliers. It solves the problem of noisy or incorrect depth predictions in sparse-view settings by verifying consistency across multiple camera viewpoints. It differs from [Monocular Depth Consistency Loss](4668-monocular-depth-consistency-loss.md) by using reprojection error of image intensities to determine reliability rather than a closed-form least-squares scale/shift fit, and from [Pseudo-depth Supervised Loss](7487-pseudo-depth-supervised-loss.md) by explicitly filtering unreliable pixels via multi-view geometric checks.

### [Pseudo-depth Supervised Loss](7487-pseudo-depth-supervised-loss.md)
Pseudo-depth Supervised Loss is a family of regularization and consistency terms used in differentiable volume rendering to constrain the geometry of a neural radiance field (NeRF) using external depth priors. It addresses the problem of geometry error degradation and unreliable depth estimation in texture-less regions by filtering predictions based on confidence scores or by using robust loss functions that are less sensitive to outliers than standard L2 losses. The method encourages the rendered depth to align with pseudo-depth targets (derived from Structure-from-Motion or monocular estimators) while retaining freedom for inaccurate geometry through gradient suppression at large errors.

### MonoSDF
MonoSDF uses monocular depth and normal constraints to address weak texture area reconstruction in neural implicit surface reconstruction [S2, unofficial]. It extends the concept of monocular consistency by incorporating normal maps derived from monocular estimators to provide stronger geometric cues in texture-less regions [S2, unofficial].

### Scale Alignment of Depth Priors
This variant aligns the scale of a predicted monocular depth map with sparse COLMAP depths to enable supervision during NeRF optimization [S3]. It uses a log-space scale factor $\alpha$ computed by averaging the difference between the log of the predicted depth and the log of the sparse depth over valid pixels [S3]:
$$ \alpha (D^i_p, D^i_{Sparse}) = \frac {1}{n} \sum _j(\log D^i_p(j) - \log D^i_{Sparse}(j)) $$
where $D^i_p$ is the predicted depth, $D^i_{Sparse}$ is the sparse depth, and $n$ is the number of valid pixels [S3].

### Self-Supervised Depth Losses
This approach uses two auxiliary losses ($L_{smooth}$ and $L_{std}$) to constrain the estimated depth and confidence interval in a self-supervised manner using monocular depth estimates and image gradients [S4]. It relies on the smoothness prior of natural images and the consistency of depth gradients with image gradients [S4].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per batch, where $N$ is the number of pixels/rays. The least-squares calculation is linear in the number of elements.
*   **Memory Complexity:** $O(N)$ to store the depth vectors for the batch.
*   **Measured Performance:** Effective in monocular endoscopic sequences where metric depth is unavailable, improving reconstruction quality over standard depth losses [S1].
*   **Scaling:** Scales linearly with batch size.
*   **Strengths:** Handles scale and shift ambiguity; does not require metric ground truth; simple to implement.
*   **Limitations:** Sensitive to outliers in monocular predictions; requires careful handling of batch statistics; may not enforce absolute metric scale without additional constraints.
*   **When to Use:** Monocular NeRF optimization, endoscopic surgery reconstruction, scenarios with no metric depth ground truth.
*   **When Not to Use:** When metric scale is critical and no sparse metric points are available; when monocular depth predictions are known to be highly unreliable in specific regions (consider filtering first).

## Background
The Monocular Depth Consistency Loss was introduced in the context of efficient deformable tissue reconstruction via orthogonal neural planes, specifically for endoscopic sequences [S1]. It addresses the challenge that depth in endoscopic videos is often a relative cue due to the lack of metric information, yet relative depth information is beneficial for reconstruction [S1]. The method replaces the standard depth loss $L_D$ with $L_{Mono}$ during training for monocular endoscopic sequences [S1].

## Sources
[S1] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S3] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S4] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
