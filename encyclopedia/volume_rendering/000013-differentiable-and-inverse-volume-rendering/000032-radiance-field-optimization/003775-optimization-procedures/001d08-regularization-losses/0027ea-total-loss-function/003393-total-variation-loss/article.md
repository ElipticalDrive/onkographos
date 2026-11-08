# Total Variation Loss

Also called: TVLoss, Total Variation Loss for Bilateral Grids, Total Variation Loss module

## Overview
Total Variation Loss is a regularization term applied to the parameters of bilateral grids to encourage spatial smoothness in learned color transformations [S1, S2, unofficial]. It computes the sum of squared differences between adjacent elements in the spatial dimensions of the grid tensor, normalized by the element count and batch size [S2, unofficial]. It is distinct from gradient-based Eikonal losses by operating directly on the discrete grid parameters rather than continuous field gradients [S2, unofficial].

## How it works
**Inputs and Outputs**
- Input: A 4D tensor $x$ of shape $(B, C, H, W)$ representing bilateral grid parameters, where $B$ is batch size, $C$ is channels, $H$ is height, and $W$ is width [S2, unofficial].
- Output: A scalar loss value [S2, unofficial].

**Data Structures**
- The bilateral grid is stored as a dense 4D PyTorch tensor [S2, unofficial]. No sparse structures or bit-packing are used in the standard implementation [S2, unofficial].

**Parameters**
- `TVLoss_weight`: Weight multiplier for the loss. Default value is 1 [S2, unofficial].
- `config.bilgrid_tv_loss_mult`: Multiplier applied when summing losses across multiple grids in a model [S1, unofficial].

**Algorithm**
1. Extract batch size $B$, height $H$, and width $W$ from the input tensor $x$ [S2, unofficial].
2. Initialize total loss accumulator to 0 [S2, unofficial].
3. Compute horizontal variation:
   - Calculate count $N_h = C \times (H-1) \times W$ [S2, unofficial].
   - If $N_h > 0$:
     - Compute squared differences between adjacent elements along the height dimension: $diff_h = (x[:,:,1:,:] - x[:,:,:H-1,:])^2$ [S2, unofficial].
     - Sum the differences: $sum_h = \sum diff_h$ [S2, unofficial].
     - Normalize: $h_{tv} = sum_h / N_h$ [S2, unofficial].
     - Add to total: $total \leftarrow total + h_{tv}$ [S2, unofficial].
4. Compute vertical variation:
   - Calculate count $N_w = C \times H \times (W-1)$ [S2, unofficial].
   - If $N_w > 0$:
     - Compute squared differences between adjacent elements along the width dimension: $diff_w = (x[:,:,:,1:] - x[:,:,:,:W-1])^2$ [S2, unofficial].
     - Sum the differences: $sum_w = \sum diff_w$ [S2, unofficial].
     - Normalize: $w_{tv} = sum_w / N_w$ [S2, unofficial].
     - Add to total: $total \leftarrow total + w_{tv}$ [S2, unofficial].
5. Compute final loss: $L_{TV} = \frac{2 \times total}{B} \times TVLoss\_weight$ [S2, unofficial].

**Formula**
The total variation loss for a tensor $x$ is defined as [S1, S2, unofficial]:

$$
L_{TV} = \frac{2 \cdot TVLoss\_weight}{B} \left( \frac{\sum_{c,j,k} (x_{c,j+1,k} - x_{c,j,k})^2}{C(H-1)W} + \frac{\sum_{c,j,k} (x_{c,j,k+1} - x_{c,j,k})^2}{CH(W-1)} \right)
$$

Where:
- $B$ is the batch size.
- $C$ is the number of channels.
- $H$ is the height dimension.
- $W$ is the width dimension.
- $x_{c,j,k}$ represents the element at channel $c$, height $j$, and width $k$.
- The first term corresponds to horizontal neighbors (height dimension), and the second to vertical neighbors (width dimension) [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def total_variation_loss(x, tv_loss_weight=1.0):
    """
    Computes the Total Variation Loss for a 4D tensor.
    
    Args:
        x (torch.Tensor): Input tensor of shape (B, C, H, W).
        tv_loss_weight (float): Weight multiplier for the loss. Default 1.0.
        
    Returns:
        torch.Tensor: Scalar loss value.
    """
    batch_size = x.size(0)
    h_x = x.size(2)
    w_x = x.size(3)
    
    # Calculate counts for normalization
    # Count for horizontal neighbors: channels * (height-1) * width
    count_h = x.size(1) * (h_x - 1) * w_x
    # Count for vertical neighbors: channels * height * (width-1)
    count_w = x.size(1) * h_x * (w_x - 1)
    
    total = 0.0
    
    if count_h > 0:
        # Horizontal TV: difference between x[:,:,1:,:] and x[:,:,:h_x-1,:]
        h_tv = torch.pow(x[:,:,1:,:] - x[:,:,:h_x-1,:], 2).sum()
        h_tv /= count_h
        total += h_tv
        
    if count_w > 0:
        # Vertical TV: difference between x[:,:,:,1:] and x[:,:,:,:w_x-1]
        w_tv = torch.pow(x[:,:,:,1:] - x[:,:,:,:w_x-1], 2).sum()
        w_tv /= count_w
        total += w_tv
        
    # Final formula: 2 * (h_tv + w_tv) / batch_size * TVLoss_weight
    return (2.0 * total / batch_size) * tv_loss_weight
```

**Edge Cases and Pitfalls**
- If $H=1$ or $W=1$, the corresponding count becomes 0, and that dimension's contribution is skipped to avoid division by zero [S2, unofficial].
- The loss is sensitive to the scale of the bilateral grid parameters; large values may dominate the optimization if not weighted appropriately relative to photometric losses [S1, unofficial].
- The implementation assumes a 4D tensor structure; inputs with different dimensions may require adaptation [S2, unofficial].

## Operating characteristics
- **Time Complexity**: $O(B \cdot C \cdot H \cdot W)$, as it iterates over all elements to compute differences [S2, unofficial].
- **Memory Complexity**: $O(B \cdot C \cdot H \cdot W)$ for storing the input tensor and intermediate difference tensors [S2, unofficial].
- **Strengths**: Encourages local smoothness in bilateral grid parameters, reducing noise and artifacts in color transformations [S1, unofficial].
- **Limitations**: Only considers adjacent neighbors, so it does not enforce global smoothness [S2, unofficial]. The squared difference formulation may be sensitive to outliers compared to L1-based TV [S2, unofficial].
- **When to use**: When training models that use bilateral grids for color correction or tone mapping, such as in Gaussian splatting pipelines [S1, unofficial].
- **When not to use**: When the grid parameters are expected to have sharp discontinuities or when computational resources are extremely limited [S2, unofficial].

## Background
Total Variation Loss for Bilateral Grids was introduced in the context of differentiable rendering and inverse graphics, specifically within the Joint-TensoRF project [S2, unofficial]. It is used to regularize the bilateral grid parameters in models that learn color transformations for radiance fields [S1, S2, unofficial]. The method is part of the broader family of regularization losses used in optimization procedures for radiance-field optimization [S1, unofficial].

## Sources
[S1] nerfstudio-project/gsplat, https://github.com/nerfstudio-project/gsplat, fetched 2026-10-27 (unofficial)
[S2] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
