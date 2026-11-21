# Grid Embedding

Also called: Learnable 4D Tensor Lookup Table, Shape-specific latent codes grid

## Overview
Grid Embedding is a method that stores shape-specific latent codes indexed by model ID in a dense, learnable 4D tensor lookup table [S1, unofficial]. It builds on [Voxel Grid Indexing](1719-voxel-grid-indexing.md) by replacing coordinate-based hashing with direct model-ID indexing into a fixed volumetric latent space.

## How it works
**Inputs:** A tensor of model indices with shape $(n_{\text{models}})$ [S1, unofficial].
**Outputs:** A tensor of shape $(n_{\text{models}}, \text{grid\_size}+1, \text{grid\_size}+1, \text{grid\_size}+1, \text{latent\_dim})$ containing the retrieved latent codes [S1, unofficial].

**Data Structure:**
The embedding is stored as a learnable parameter tensor $E$ with shape $(n, \text{grid\_size}+1, \text{grid\_size}+1, \text{grid\_size}+1, \text{latent\_dim})$, where $n$ is the number of models, $\text{grid\_size}$ defines the spatial resolution of the latent code volume, and $\text{latent\_dim}$ is the feature dimensionality [S1, unofficial]. The tensor is stored as `nn.Parameter` to enable gradient tracking during backpropagation [S1, unofficial].

**Steps:**
1. **Gather:** Given an input tensor of model indices $I$ of shape $(n_{\text{models}})$, retrieve the corresponding embeddings from the lookup table via $E[I]$ [S1, unofficial].
2. **Reshape:** Reshape the gathered tensor to $(n_{\text{models}}, \text{grid\_size}+1, \text{grid\_size}+1, \text{grid\_size}+1, -1)$ to preserve the 4D grid structure across the batch dimension [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class GridEmbedding(nn.Module):
    def __init__(self, n_models, grid_size, latent_dim):
        super().__init__()
        # Shape: (n_models, grid_size+1, grid_size+1, grid_size+1, latent_dim)
        # Stored as nn.Parameter for gradient tracking [S1, unofficial]
        # Initialization method is not specified in the sources [G]
        self.grid_size = grid_size
        self.embeddings = nn.Parameter(torch.empty(n_models, grid_size + 1, grid_size + 1, grid_size + 1, latent_dim))

    def forward(self, indices):
        """
        Args:
            indices: Tensor of shape (n_models,) containing model IDs.
        Returns:
            Tensor of shape (n_models, grid_size+1, grid_size+1, grid_size+1, latent_dim)
        """
        # Gather embeddings via self.embeddings[indices] [S1, unofficial]
        gathered = self.embeddings[indices]
        
        # Reshape to (n_models, grid_size+1, grid_size+1, grid_size+1, -1)
        # to preserve the 4D grid structure across the batch dimension [S1, unofficial]
        n = indices.shape[0]
        output = gathered.view(n, self.grid_size + 1, self.grid_size + 1, self.grid_size + 1, -1)
        return output
```

**Parameters:**
*   $n_{\text{models}}$: Number of distinct models/shapes [S1, unofficial].
*   $\text{grid\_size}$: Spatial resolution of the latent code volume. Typical values are not given in the sources [S1, unofficial].
*   $\text{latent\_dim}$: Dimensionality of the feature vector per voxel [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Index Out of Bounds:** If an index in $I$ exceeds $n_{\text{models}}-1$, the gather operation will fail. Input validation is required [G].
*   **Measured Performance:** Not given in the sources [S1, unofficial].

## Subtopics

### Trainable 3D Saliency Grid
A trainable 3D tensor data structure $G$ with resolution $T \times T \times T$ that stores trainable values representing the saliency of features near each grid voxel, used to guide feature pruning [S2]. It differs from the standard Grid Embedding by operating on spatial coordinates to modulate feature importance (saliency) rather than storing shape-specific latent codes [S2]. Choose this when aiming to reduce space complexity and improve accuracy by pruning internal occluded regions in hashgrid-based NeRFs [S2].

**Definition:** A 3D tensor $G$ of size $T \times T \times T$ initialized as an all-ones tensor, storing trainable floats [S2].
**Steps:**
1. Split space coarsely into $T \times T \times T$ uniform grids [S2].
2. For input coordinate $x$, locate the 8 voxels $(x_1 \dots x_8)$ surrounding $x$ in grid $G$ [S2].
3. Quantize coordinates $x_i$ to $(\hat{x}_i, \hat{y}_i, \hat{z}_i)$ between $0$ and $T-1$ to index saliency values $g_i$ [S2].
4. Interpolate values associated to the 8 surrounding voxels to get smoothed saliency distribution $P$ [S2].
**Formulas:**
*   Saliency value: $g_i = G(\hat{x}_i, \hat{y}_i, \hat{z}_i)$ [S2].
*   Saliency weight $p$ at coordinate $x$ is calculated by trilinear interpolation of $g_i$ followed by a sigmoid operator, ensuring $0 < p < 1$ [S2].
*   Final weighted feature vector: $v = p \cdot f$, where $f$ is the feature vector fetched from the hashgrid [S2].
**Parameters:**
*   $T$: Resolution of the saliency grid. Typically much smaller than the resolution of the dense feature grid [S2].
**Pitfalls:**
*   Precision in quantization and interpolation affects the smoothness of the saliency distribution [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Measured Performance:** Not given in the sources [S1, unofficial].
*   **Strengths:** Not given in the sources [S1, unofficial].
*   **Limitations:** Not given in the sources [S1, unofficial].
*   **When to use:** Not given in the sources [S1, unofficial].
*   **When not to use:** Not given in the sources [S1, unofficial].

## Background
Grid Embedding is described in VC-SDFGen, an unofficial repository [S1, unofficial]. It is used to store and optimize shape-specific latent codes for 3D models [S1, unofficial]. The Trainable 3D Saliency Grid variant originates from HollowNeRF, which proposes pruning hashgrid-based NeRFs by learning sparsity patterns to reclaim capacity spent on internal occluded regions [S2].

## Sources
[S1] VC-SDFGen, https://github.com/TurtleJayWang/VC-SDFGen, fetched 2026-10-28 (unofficial)
[S2] HollowNeRF: Pruning Hashgrid-Based NeRFs with Trainable Collision Mitigation, https://doi.org/10.1109/iccv51070.2023.00322, fetched 2026-10-26
