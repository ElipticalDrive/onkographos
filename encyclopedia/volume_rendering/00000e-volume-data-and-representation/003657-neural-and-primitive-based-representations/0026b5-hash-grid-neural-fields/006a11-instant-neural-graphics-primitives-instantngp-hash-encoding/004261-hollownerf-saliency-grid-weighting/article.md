# HollowNeRF Saliency Grid Weighting

Also called: Saliency Distribution Learning, Trainable Saliency Grid

## Overview
HollowNeRF Saliency Grid Weighting is a collision mitigation technique for hash grid-based neural radiance fields that learns a 3D probability distribution of feature presence to weight feature vectors fetched from a multi-level hashgrid [S1]. Unlike Instant-NGP, which distributes hash collisions evenly across voxels, this method learns to "shuffle" collisions by assigning higher saliency weights to important features, allowing them to dominate shared hash buckets during gradient updates while pruning less significant features [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D coordinate $\mathbf{x} = (x, y, z)$; a multi-level hashgrid encoding (as defined in [HashGrid Encoding](3866-hashgrid-encoding.md)) providing a raw feature vector $\mathbf{f}$; a trainable 3D saliency grid $G$.
*   **Output:** A weighted feature vector $\mathbf{v}$ suitable for input to the decoding MLP.

**Data Structures:**
*   **Saliency Grid ($G$):** A 3D tensor of resolution $T \times T \times T$. It stores scalar saliency values.
    *   **Initialization:** $G$ is initialized as an all-ones tensor [S1].
    *   **Size:** $T$ is typically much smaller than the resolution of the dense feature grid. Investigated values for $T$ are 64, 96, and 128 [S1].

**Algorithm Steps:**
1.  **Fetch Raw Feature:** Given input coordinate $\mathbf{x}$, retrieve the raw feature vector $\mathbf{f}$ from the multi-level hashgrid [S1].
2.  **Locate Voxels:** Identify the 8 voxels $(\mathbf{x}_1, \dots, \mathbf{x}_8)$ surrounding $\mathbf{x}$ in the 3D saliency grid $G$ [S1].
3.  **Quantize Indices:** For each voxel coordinate $\mathbf{x}_i = (x_i, y_i, z_i)$, quantize to integer indices $(\hat{x}_i, \hat{y}_i, \hat{z}_i)$ within the range $[0, T-1]$ [S1].
4.  **Fetch Saliency Values:** Retrieve the saliency value $g_i$ for each of the 8 voxels using the quantized indices:
    $$ g_i = G(\hat{x}_i, \hat{y}_i, \hat{z}_i) $$
5.  **Interpolate and Sigmoid:** Calculate the saliency weight $p$ at coordinate $\mathbf{x}$ by performing trilinear interpolation on the 8 saliency values $g_i$, followed by a sigmoid operator $\sigma(\cdot)$ [S1]. This ensures $0 < p < 1$ [S1].
6.  **Weight Feature:** Compute the final weighted feature vector $\mathbf{v}$ by multiplying the saliency weight $p$ with the raw feature vector $\mathbf{f}$ [S1]:
    $$ \mathbf{v} = p \mathbf{f} $$

**Mathematical Interpretation:**
The saliency distribution $P(\mathbf{x})$ is defined as the collection of saliency weights $p$ at every possible coordinate $\mathbf{x}$, such that $p = P(\mathbf{x})$ [S1]. The weighted feature $\mathbf{v}$ represents the expectation of the feature at $\mathbf{x}$ being either the fetched feature $\mathbf{f}$ (with probability $p$) or a zero vector $\mathbf{0}$ (with probability $1-p$) [S1]:
$$ \mathbf{v} = P(\mathbf{x})\mathbf{f} + (1-P(\mathbf{x}))\mathbf{0} = p\mathbf{f} $$
Training the saliency grid $G$ via gradient descent is equivalent to learning this probability distribution $P(\mathbf{x})$, which represents the likelihood that a given voxel contains a non-zero feature [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def hollownerf_weighting(x, hashgrid, saliency_grid, T):
    """
    x: Tensor of shape (..., 3), input coordinates.
    hashgrid: Function or object to fetch raw features f from multi-level hashgrid.
    saliency_grid: Tensor of shape (T, T, T), the trainable saliency grid G.
    T: Integer, resolution of the saliency grid.
    Returns: Weighted feature vector v.
    """
    # 1. Fetch raw feature vector f from the multi-level hashgrid
    f = hashgrid(x)
    
    # 2. Locate the 8 surrounding voxels in the saliency grid
    # Assuming x is normalized or mapped to the grid space [0, T-1]
    # We need to compute the grid indices for the 8 corners
    # Note: The exact quantization depends on the coordinate system mapping.
    # Based on S1, xi are quantized to integers between 0 and T-1.
    
    # Calculate lower and upper bounds for the 8 voxels
    # x_float is the coordinate in grid space
    x_float = x * (T - 1) # Assuming mapping to [0, T-1] range for indexing
    
    # Get integer indices for the "lower" corner (floor)
    ix_low = x_float.long()
    ix_high = ix_low + 1
    
    # Clamp indices to [0, T-1] to handle boundary conditions
    ix_low = torch.clamp(ix_low, 0, T - 1)
    ix_high = torch.clamp(ix_high, 0, T - 1)
    
    # Extract coordinates for the 8 voxels
    # Voxels are combinations of (ix_low/high, iy_low/high, iz_low/high)
    # We can use grid_sample or manual indexing. Manual indexing is clearer for 3D.
    
    # Create a grid of indices for the 8 corners
    # Shape: (..., 8, 3)
    # This part requires expanding dimensions to match x's batch/shape
    # For simplicity, assuming x is (N, 3)
    
    # Get fractional parts for interpolation weights
    fx = x_float - ix_low.float()
    fy = (x_float[:, 1:2] - ix_low[:, 1:2].float())
    fz = (x_float[:, 2:3] - ix_low[:, 2:3].float())
    
    # Fetch saliency values for the 8 corners
    # Indices: (x_low, y_low, z_low), (x_low, y_low, z_high), ...
    idx_list = [
        (ix_low[:, 0], ix_low[:, 1], ix_low[:, 2]),
        (ix_low[:, 0], ix_low[:, 1], ix_high[:, 2]),
        (ix_low[:, 0], ix_high[:, 1], ix_low[:, 2]),
        (ix_low[:, 0], ix_high[:, 1], ix_high[:, 2]),
        (ix_high[:, 0], ix_low[:, 1], ix_low[:, 2]),
        (ix_high[:, 0], ix_low[:, 1], ix_high[:, 2]),
        (ix_high[:, 0], ix_high[:, 1], ix_low[:, 2]),
        (ix_high[:, 0], ix_high[:, 1], ix_high[:, 2])
    ]
    
    g_values = []
    for ix, iy, iz in idx_list:
        # Gather from saliency_grid
        # saliency_grid is (T, T, T)
        # We need to index it with (ix, iy, iz)
        # Using advanced indexing
        g = saliency_grid[ix, iy, iz]
        g_values.append(g)
        
    g_tensor = torch.stack(g_values, dim=-1) # Shape: (N, 8)
    
    # 3. Trilinear interpolation of saliency values
    # Weights for interpolation
    w000 = (1 - fx) * (1 - fy) * (1 - fz)
    w001 = (1 - fx) * (1 - fy) * fz
    w010 = (1 - fx) * fy * (1 - fz)
    w011 = (1 - fx) * fy * fz
    w100 = fx * (1 - fy) * (1 - fz)
    w101 = fx * (1 - fy) * fz
    w110 = fx * fy * (1 - fz)
    w111 = fx * fy * fz
    
    # Sum of weights should be 1
    p_raw = (w000 * g_tensor[..., 0] +
             w001 * g_tensor[..., 1] +
             w010 * g_tensor[..., 2] +
             w011 * g_tensor[..., 3] +
             w100 * g_tensor[..., 4] +
             w101 * g_tensor[..., 5] +
             w110 * g_tensor[..., 6] +
             w111 * g_tensor[..., 7])
             
    # 4. Sigmoid operator
    p = torch.sigmoid(p_raw)
    
    # 5. Weight the feature vector
    v = p.unsqueeze(-1) * f
    
    return v
```

**Parameters:**
*   $T$: The resolution of the 3D saliency grid $G$. Typical values investigated are 64, 96, and 128 [S1].

**Edge Cases and Pitfalls:**
*   **Convergence to Zero:** If the saliency weight $p$ converges to 0, the feature is pruned with probability 1 and becomes a zero vector [S1]. This reduces interference in hash buckets but must be managed to avoid losing important information [S1].
*   **Boundary Conditions:** Quantization to indices $[0, T-1]$ requires clamping or handling out-of-bounds coordinates to prevent indexing errors [S1].

## Subtopics

### Instant-NGP
Uses hash encoding to distribute collisions evenly across voxels with non-zero density. Choose when uniform collision handling is sufficient and simpler implementation is preferred. [Instant-NGP](6604-instant-ngp.md)

### Progressive Training
Builds the hash grid from coarse to fine resolutions. Choose when memory constraints require starting with a small grid. [Progressive Training](9142-progressive-training.md)

### Dense Grid Encoding
Uses a fully dense 3D grid without hashing. Choose when memory is abundant and exact spatial locality is prioritized over hash efficiency. [Dense Grid Encoding](9335-dense-grid-encoding.md)

### ACORN
Uses octree-based sparse representation. Choose when the scene has large empty spaces and hierarchical sparsity is beneficial. [ACORN](9336-acorn.md)

### Multiresolution Grid Encoding
Similar to hash grids but uses structured multiresolution grids. Choose when structured memory access patterns are preferred over hashing. [Multiresolution Grid Encoding](9350-multiresolution-grid-encoding.md)

### RHINO
Uses radial basis functions for encoding. Choose when smooth, continuous encoding is required without discrete grid artifacts. [RHINO](10245-rhino.md)

## Operating characteristics
*   **Time Complexity:** Adds a constant-time lookup and interpolation step per coordinate query, proportional to the number of voxels (8) and the dimensionality of the feature vector.
*   **Memory Complexity:** Requires additional memory for the saliency grid $G$, which is $T^3$ floats. Since $T$ is typically much smaller than the feature grid resolution, this overhead is low [S1].
*   **Performance:** Mitigates hash collisions by allowing important features to dominate shared buckets, potentially improving reconstruction quality in complex scenes compared to uniform collision distribution [S1].
*   **Strengths:** Learnable collision mitigation; differentiable model compression opportunity [S1].
*   **Limitations:** Requires training an additional grid; risk of over-pruning if $p$ converges to 0 [S1].
*   **When to Use:** When hash collisions significantly degrade reconstruction quality and uniform distribution (as in Instant-NGP) is insufficient.
*   **When Not to Use:** When memory is extremely constrained (though overhead is low) or when training time is a critical bottleneck due to the additional gradient computation for $G$.

## Background
HollowNeRF was introduced in the paper "HollowNeRF: Pruning Hashgrid-Based NeRFs with Trainable Collision Mitigation" [S1]. The method addresses the issue of hash collisions in neural radiance fields by learning a saliency distribution to weight features, effectively pruning less important features to reduce interference in shared hash buckets [S1].

## Sources
[S1] HollowNeRF: Pruning Hashgrid-Based NeRFs with Trainable Collision Mitigation, https://doi.org/10.1109/iccv51070.2023.00322, fetched 2026-10-26
