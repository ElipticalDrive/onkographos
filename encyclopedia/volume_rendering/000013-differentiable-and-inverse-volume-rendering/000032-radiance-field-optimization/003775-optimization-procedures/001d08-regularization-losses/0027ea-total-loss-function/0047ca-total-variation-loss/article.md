# Total Variation Loss

Also called: BilagridProcessorModule, Bilateral Grid Processing with TV Loss, Bilateral Grid Splatting, TVLoss, Total Variation Loss for Bilateral Grids, Total Variation Loss module

## Overview
Total Variation (TV) Loss is a regularization constraint applied to explicit feature volumes or bilateral grid parameters to enforce spatial smoothness in learned color transformations [S1, S2, unofficial]. It differs from photometric reconstruction losses by penalizing high-frequency noise and discontinuities in the latent representation rather than matching pixel intensities [S1, unofficial]. It is typically combined with other regularization terms, such as Eikonal loss, within the total training objective [S1, unofficial].

## How it works
The method computes the sum of squared differences between adjacent elements in the spatial dimensions of a 4D tensor representing the bilateral grid or feature volume, normalized by the number of neighboring pairs and the batch size [S4, unofficial].

### Inputs and Outputs
- **Input**: A 4D tensor $x$ of shape $(B, C, H, W)$, where $B$ is the batch size, $C$ is the number of channels (or appearance groups), $H$ is the height, and $W$ is the width of the bilateral grid [S4, unofficial].
- **Output**: A scalar loss value $L_{TV}$ [S4, unofficial].

### Data Structures
The input tensor $x$ is a PyTorch tensor. The bilateral grid itself is initialized with dimensions determined by configuration parameters: `grid_x` (width), `grid_y` (height), and `grid_w` (depth/channels) [S3, unofficial]. The grid is indexed by appearance IDs [S3, unofficial].

### Parameters
- `TVLoss_weight` (or `tv_loss_weight`): Weight multiplier for the loss. Default value is 1.0 [S4, unofficial]. In some implementations, a specific weight of 10.0 is used [S3, unofficial].
- `batch_size` ($B$): Number of samples in the batch [S4, unofficial].
- `height` ($H$ or $h_x$): Spatial height dimension [S4, unofficial].
- `width` ($W$ or $w_x$): Spatial width dimension [S4, unofficial].
- `channels` ($C$): Number of channels or appearance groups [S4, unofficial].

### Formulas
The total variation loss for a tensor $x$ is defined as [S2, S4, unofficial]:

$$
L_{TV} = \frac{2 \cdot TVLoss\_weight}{B} \left( \frac{\sum_{i,j,k} (x_{i,j+1,k} - x_{i,j,k})^2}{N_h} + \frac{\sum_{i,j,k} (x_{i,j,k+1} - x_{i,j,k})^2}{N_w} \right)
$$

Where:
- $B$ is the batch size.
- $N_h = C \cdot (H-1) \cdot W$ is the count of horizontal neighbors [S4, unofficial].
- $N_w = C \cdot H \cdot (W-1)$ is the count of vertical neighbors [S4, unofficial].
- The factor of 2 arises from the specific implementation in [S4, unofficial], which sums horizontal and vertical components and multiplies by 2 before dividing by batch size. Note that [S2, unofficial] describes a more general formulation summing over all spatial dimensions $i$ from 2 to $D$, normalized by $N_i$, but [S4, unofficial] provides the specific 2D spatial implementation used in the bilateral grid context.

### Steps
1. **Extract Dimensions**: Determine batch size $B$, height $H$, and width $W$ from the input tensor $x$ [S4, unofficial].
2. **Calculate Counts**:
   - Compute $N_h = C \times (H-1) \times W$ [S4, unofficial].
   - Compute $N_w = C \times H \times (W-1)$ [S4, unofficial].
3. **Compute Horizontal Variation**:
   - If $N_h > 0$, calculate the squared difference between adjacent elements along the height dimension: $diff_h = (x[:, :, 1:, :] - x[:, :, :H-1, :])^2$ [S4, unofficial].
   - Sum the elements of $diff_h$ and divide by $N_h$ to get $h_{tv}$ [S4, unofficial].
4. **Compute Vertical Variation**:
   - If $N_w > 0$, calculate the squared difference between adjacent elements along the width dimension: $diff_w = (x[:, :, :, 1:] - x[:, :, :, :W-1])^2$ [S4, unofficial].
   - Sum the elements of $diff_w$ and divide by $N_w$ to get $w_{tv}$ [S4, unofficial].
5. **Aggregate**: Sum $h_{tv}$ and $w_{tv}$ [S4, unofficial].
6. **Normalize and Weight**: Multiply the sum by $2 \cdot TVLoss\_weight$ and divide by $B$ [S4, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def total_variation_loss(x, tv_loss_weight=1.0):
    """
    Computes the Total Variation loss for a 4D tensor x.
    
    Args:
        x (torch.Tensor): Input tensor of shape (B, C, H, W).
        tv_loss_weight (float): Weight multiplier for the loss.
        
    Returns:
        torch.Tensor: Scalar loss value.
    """
    batch_size = x.size(0)
    h_x = x.size(2)
    w_x = x.size(3)
    
    # Calculate counts for normalization
    # Count of horizontal neighbors: channels * (height-1) * width
    count_h = x.size(1) * (h_x - 1) * w_x
    # Count of vertical neighbors: channels * height * (width-1)
    count_w = x.size(1) * h_x * (w_x - 1)
    
    total = 0.0
    
    # Horizontal TV
    if count_h > 0:
        # Difference between x[:,:,1:,:] and x[:,:,:h_x-1,:]
        h_diff = (x[:, :, 1:, :] - x[:, :, :h_x-1, :]) ** 2
        h_tv = h_diff.sum() / count_h
        total += h_tv
        
    # Vertical TV
    if count_w > 0:
        # Difference between x[:,:,:,1:] and x[:,:,:,:w_x-1]
        w_diff = (x[:, :, :, 1:] - x[:, :, :, :w_x-1]) ** 2
        w_tv = w_diff.sum() / count_w
        total += w_tv
        
    # Final loss calculation as per [S4, unofficial]
    loss = (2.0 * tv_loss_weight * total) / batch_size
    
    return loss
```

### Edge Cases and Pitfalls
- **Zero Dimensions**: If $H=1$ or $W=1$, the corresponding count ($N_h$ or $N_w$) becomes 0. The implementation must check for this to avoid division by zero [S4, unofficial].
- **Batch Size**: The loss is normalized by batch size, so it is independent of the batch size magnitude [S4, unofficial].
- **Precision**: Squaring differences can lead to large values if the grid parameters are not normalized. Ensure inputs are in a reasonable range [G].

## Subtopics

### [Loss Function Configuration](6782-loss-function-configuration.md)
Defines how TV loss weight is configured alongside other losses.

### [Sparse Depth Consistency Loss](7294-sparse-depth-consistency-loss.md)
Another regularization term often used in conjunction with TV loss.

### [Discriminator Loss with Gradient Penalty](7695-discriminator-loss-with-gradient-penalty.md)
Alternative regularization method using adversarial training.

### [Loss Weight Configuration](7883-loss-weight-configuration.md)
Details on setting the `tv_loss_weight` parameter.

### [Photometric and Image Reconstruction Losses](9789-photometric-and-image-reconstruction-losses.md)
Primary reconstruction loss, distinct from TV regularization.

### [Specific Method Implementations](9794-specific-method-implementations.md)
Implementation details for various loss functions.

### [Numerical Gradient Eikonal Loss](9954-numerical-gradient-eikonal-loss.md)
Gradient-based regularization for radiance fields.

### [CF Loss](10573-cf-loss.md)
Consistency loss for feature volumes.

### [Overall Training Loss](10621-overall-training-loss.md)
Aggregation of all loss components including TV loss.

## Operating characteristics
- **Time Complexity**: $O(B \cdot C \cdot H \cdot W)$, as it requires a single pass over the spatial dimensions to compute differences [S4, unofficial].
- **Memory Complexity**: $O(B \cdot C \cdot H \cdot W)$ for storing the input tensor and intermediate difference tensors [S4, unofficial].
- **Strengths**: Effectively smooths out noise in learned feature volumes and bilateral grids, preventing overfitting to high-frequency artifacts [S1, S2, unofficial].
- **Limitations**: May oversmooth fine details if the weight is too high [G]. Computationally inexpensive but adds to the overall training time [S4, unofficial].
- **When to Use**: When training models with explicit feature volumes or bilateral grids that are prone to noise [S1, S2, unofficial].
- **When Not to Use**: When preserving high-frequency details is critical and the model is not overfitting [G].

## Background
Total Variation Loss was introduced as a regularization technique for inverse problems and image processing [S1, unofficial]. In the context of volume rendering and radiance fields, it has been adopted to enforce smoothness in explicit feature volumes and bilateral grids [S1, S2, unofficial]. It is commonly used in conjunction with other losses such as photometric loss and Eikonal loss [S1, unofficial]. The specific implementation for bilateral grids in Gaussian Splatting variants is detailed in recent open-source projects [S2, S3, unofficial].

## Sources
[S1] PDRF: Progressively Deblurring Radiance Field for Fast and Robust Scene Reconstruction from Blurry Images, https://doi.org/10.48550/arxiv.2208.08049, fetched 2026-10-26 (unofficial)
[S2] nerfstudio-project/gsplat, https://github.com/nerfstudio-project/gsplat, fetched 2026-10-27 (unofficial)
[S3] yzslab/gaussian-splatting-lightning, https://github.com/yzslab/gaussian-splatting-lightning, fetched 2026-10-27 (unofficial)
[S4] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
