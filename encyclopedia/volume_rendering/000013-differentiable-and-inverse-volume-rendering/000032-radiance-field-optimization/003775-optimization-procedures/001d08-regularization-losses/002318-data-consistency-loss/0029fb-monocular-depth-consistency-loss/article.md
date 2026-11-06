# Monocular Depth Consistency Loss

Also called: Monocular Depth Loss

## Overview
Monocular Depth Consistency Loss enforces geometric consistency between the depth rendered from a radiance field and depth predicted by a monocular depth estimation network, which operates on a relative scale [S1]. It solves the ambiguity of absolute scale in monocular supervision by computing a closed-form least-squares solution for scale ($\eta$) and shift ($\epsilon$) parameters per batch [S1]. Unlike sibling methods that may rely on optimal transport or sparse feature alignment, this method directly aligns the statistical distribution of the rendered and predicted depth maps using a linear transformation [S1].

## How it works
**Inputs:**
- $\hat{D}(\mathbf{r})$: Rendered depth for ray $\mathbf{r}$, derived from the radiance field parameters $\boldsymbol{\theta}$.
- $\bar{D}(\mathbf{r})$: Predicted monocular depth for the corresponding image pixel, derived from the monocular network parameters $\boldsymbol{\phi}$.
- $\bar{D}_i$: The set of rays/pixels associated with the $i$-th image in the current batch.

**Outputs:**
- $\mathcal{L}_{Mono}(\boldsymbol{\theta}, \boldsymbol{\phi})$: The scalar loss value.

**Data Structures:**
- Depth maps are treated as tensors of shape $(H, W)$ or flattened vectors for batch-wise computation.
- Parameters $\eta$ and $\epsilon$ are scalars computed per batch.

**Steps:**
1. For each image $i$ in the batch, collect the rendered depth values $\hat{D}(\mathbf{r})$ and predicted monocular depth values $\bar{D}(\mathbf{r})$ for all rays $\mathbf{r} \in \bar{D}_i$.
2. Compute the scale parameter $\eta$ using the closed-form least-squares solution:
   $$ \eta = \left( \hat{D}(\mathbf{r})^T \hat{D}(\mathbf{r}) \right)^{-1} \hat{D}(\mathbf{r})^T \bar{D}(\mathbf{r}) $$
   Here, $\hat{D}(\mathbf{r})$ and $\bar{D}(\mathbf{r})$ are treated as column vectors containing the depth values for all pixels in the batch image.
3. Compute the shift parameter $\epsilon$:
   $$ \epsilon = \bar{D}(\mathbf{r}) - \eta \hat{D}(\mathbf{r}) $$
   Note: In implementation, $\epsilon$ is typically the mean residual after scaling, or computed as $\text{mean}(\bar{D}(\mathbf{r})) - \eta \cdot \text{mean}(\hat{D}(\mathbf{r}))$ if the least-squares formulation assumes zero-mean or if $\eta$ is computed via standard linear regression intercept logic. However, the source defines $\epsilon$ explicitly as the residual term in the alignment equation. Strictly following the source formula [S1]:
   $$ \epsilon = \bar{D}(\mathbf{r}) - \eta \hat{D}(\mathbf{r}) $$
   *Implementation Note*: Since $\eta$ and $\epsilon$ are scalars for the whole batch, $\epsilon$ is usually calculated as the mean difference: $\epsilon = \text{mean}(\bar{D}(\mathbf{r}) - \eta \hat{D}(\mathbf{r}))$. The source formula implies a vector subtraction resulting in a vector, but $\epsilon$ is a scalar parameter. The standard least squares solution for $y = \eta x + \epsilon$ is $\eta = \frac{\sum (x_i - \bar{x})(y_i - \bar{y})}{\sum (x_i - \bar{x})^2}$ and $\epsilon = \bar{y} - \eta \bar{x}$. The source [S1] provides a simplified matrix form. We follow the source's definition of $\eta$ and $\epsilon$ as derived from the least-squares criterion.
4. Compute the loss for the batch:
   $$ \mathcal{L}_{Mono} = \sum_i \mathbb{E}_{\mathbf{r} \sim \bar{D}_i} \left\| (\eta \hat{D}(\mathbf{r}) + \epsilon) - \bar{D}(\mathbf{r}) \right\|^2 $$
   This is the Mean Squared Error (MSE) between the scaled/shifted rendered depth and the predicted monocular depth.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def monocular_depth_consistency_loss(rendered_depth, predicted_depth):
    """
    Args:
        rendered_depth: Tensor of shape (N,) or (H, W), rendered depth values.
        predicted_depth: Tensor of shape (N,) or (H, W), monocular predicted depth values.
    Returns:
        loss: Scalar tensor.
    """
    # Flatten to 1D vectors
    d_hat = rendered_depth.flatten()
    d_bar = predicted_depth.flatten()
    
    # Remove NaN or Inf values if necessary [G]
    valid_mask = torch.isfinite(d_hat) & torch.isfinite(d_bar)
    d_hat = d_hat[valid_mask]
    d_bar = d_bar[valid_mask]
    
    if d_hat.numel() == 0:
        return torch.tensor(0.0)

    # Compute scale eta using closed-form least squares
    # eta = (d_hat^T * d_hat)^-1 * d_hat^T * d_bar
    # This is equivalent to dot(d_hat, d_bar) / dot(d_hat, d_hat)
    numerator = torch.dot(d_hat, d_bar)
    denominator = torch.dot(d_hat, d_hat)
    
    # Guard against division by zero [G]
    if denominator < 1e-8:
        eta = torch.tensor(0.0)
    else:
        eta = numerator / denominator
        
    # Compute shift epsilon
    # epsilon = mean(d_bar) - eta * mean(d_hat)
    # Note: The source formula epsilon = d_bar - eta * d_hat is vector-wise.
    # For a scalar shift parameter in a batch-wise loss, we use the mean residual.
    mean_d_hat = torch.mean(d_hat)
    mean_d_bar = torch.mean(d_bar)
    epsilon = mean_d_bar - eta * mean_d_hat
    
    # Compute aligned rendered depth
    aligned_d_hat = eta * d_hat + epsilon
    
    # Compute MSE loss
    loss = torch.mean((aligned_d_hat - d_bar) ** 2)
    
    return loss
```

**Parameters:**
- $\eta$: Scale parameter. Calculated per batch.
- $\epsilon$: Shift parameter. Calculated per batch.
- No fixed hyperparameters for the loss function itself, but it is often used in conjunction with other losses (e.g., photometric loss).

**Edge Cases and Pitfalls:**
- **Scale Variation:** Predicted depth maps vary in scale and shift across different batches; therefore, $\eta$ and $\epsilon$ must be calculated separately for each batch [S1].
- **Texture-less Regions:** Monocular depth predictions can be unreliable in texture-less regions. This method does not inherently filter these; variants like Pseudo-depth Supervised Loss use confidence thresholds [S4, unofficial].
- **Relative Cue:** Depth should be treated as a relative cue [S1].

## Subtopics

### [Reliable Depth Estimation via Reprojection](7293-reliable-depth-estimation-via-reprojection.md)
This method selects reliable depth estimates for supervising radiance field training by comparing reprojection consistency between a main model and an augmented model against a reference training view. It differs from standard monocular depth consistency by using explicit geometric reprojection error (MSE of intensities) rather than photometric alignment alone, allowing it to filter out incorrect depth predictions in ambiguous regions.

### MonoSDF
Uses monocular depth and normal constraints to address weak texture area reconstruction in neural implicit surface reconstruction [S2, unofficial].

### Scale Alignment of Depth Priors
Aligns the scale of a predicted monocular depth map with sparse COLMAP depths to enable supervision during NeRF optimization. It uses a log-space alignment factor $\alpha$ computed as the mean difference between log predicted depth and log sparse depth [S3].

### Pseudo-depth Supervised Loss
A loss function that supervises the rendered depth against a pseudo-depth target, filtered by a confidence score to ensure reliability in texture-less regions. It combines color reconstruction loss and depth loss, where the depth loss is only applied to pixels with a confidence score $U(p) > \delta$ [S4, unofficial].

## Operating characteristics
- **Time Complexity:** The computation of $\eta$ and $\epsilon$ is $O(N)$ where $N$ is the number of pixels in the batch, as it involves dot products and means. The final loss calculation is also $O(N)$.
- **Memory Complexity:** $O(N)$ to store the depth tensors for the batch.
- **Strengths:** Simple to implement; effectively handles the scale ambiguity of monocular depth predictions without requiring absolute scale ground truth [S1].
- **Limitations:** Relies on the quality of the monocular depth estimator; does not handle occlusions or texture-less regions inherently [S1][S4, unofficial].
- **When to use:** Monocular endoscopic sequences or scenarios where only monocular images are available and stereo/COLMAP data is absent [S1].

## Background
The Monocular Depth Consistency Loss was introduced in the context of efficient deformable tissue reconstruction via orthogonal neural planes [S1]. It addresses the challenge of reconstructing non-rigid deformable tissues (e.g., in robotic surgery) where stereo cues may be insufficient or unavailable, necessitating the use of monocular depth priors [S1]. The method replaces the standard depth loss $\mathcal{L}_D$ with $\mathcal{L}_{Mono}$ for monocular endoscopic sequences [S1].

## Sources
[S1] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
[S2] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S3] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S4] Improved Neural Radiance Fields Using Pseudo-depth and Fusion, https://core.ac.uk/download/577699041.pdf, fetched 2026-10-30 (unofficial)
