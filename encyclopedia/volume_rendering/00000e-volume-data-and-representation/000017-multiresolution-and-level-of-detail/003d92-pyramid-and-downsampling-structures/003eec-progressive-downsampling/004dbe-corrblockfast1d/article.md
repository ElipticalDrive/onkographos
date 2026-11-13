# CorrBlockFast1D

Also called: CorrBlock1D, PytorchAlternateCorrBlock1D

## Overview
CorrBlockFast1D is a 1D correlation block implementation for stereo matching that computes correlations using grid sampling on downsampled feature maps, supporting a multi-level pyramid structure [S1, unofficial]. It pre-computes a correlation pyramid using all-pairs correlation and bilinear sampling for efficient inference [S1, unofficial]. Unlike standard correlation layers that compute pairwise similarities on the fly, this method constructs a pyramid of correlation volumes between two feature maps to enable efficient multi-scale 1D correlation sampling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two feature maps `fmap1` and `fmap2` of shape $(B, C, H, W)$, where $B$ is batch size, $C$ is feature dimension, $H$ is height, and $W$ is width. Parameters: `num_levels` (integer, default 4) and `radius` (integer, default 4) [S1, unofficial].
*   **Outputs:** A concatenated correlation volume from all pyramid levels, permuted to shape $(B, C_{out}, H, W)$, where $C_{out}$ depends on the number of levels and radius [S1, unofficial].

**Data Structures:**
*   `corr_pyramid`: A list of tensors, each representing the correlation volume at a specific pyramid level.
*   `coords`: Normalized coordinates for sampling, shape $(B, H, W, 2)$ containing $(x, y)$ components in range $[-1, 1]$ [S1, unofficial].

**Step-by-Step Method:**

1.  **Initialization:** Store `fmap1`, `fmap2`, `num_levels`, and `radius`. Initialize an empty list `corr_pyramid` [S1, unofficial].
2.  **All-Pairs Correlation:** Compute the raw correlation between `fmap1` and `fmap2` using Einstein summation:
    $$ \text{corr}_{raw} = \sum_{k=1}^{C} \text{fmap1}_{b,i,j,k} \cdot \text{fmap2}_{b,i,l,k} $$
    Implemented as `torch.einsum('aijk,aijh->ajkh', fmap1, fmap2)`, resulting in shape $(B, H, W_1, 1, W_2)$ [S1, unofficial]. Note: The source notes indicate $W_1$ and $W_2$ are derived from input widths, but typically in 1D correlation for stereo, one dimension is collapsed or treated as the disparity axis. The source reshapes this to $(B \cdot H \cdot W_1, 1, 1, W_2)$ for pyramid construction [S1, unofficial].
3.  **Normalization:** Normalize the correlation by dividing by $\sqrt{D}$, where $D$ is the feature dimension $C$:
    $$ \text{corr}_{norm} = \frac{\text{corr}_{raw}}{\sqrt{C}} $$
    [S1, unofficial].
4.  **Pyramid Construction:**
    *   Reshape `corr_norm` to $(B \cdot H \cdot W_1, 1, 1, W_2)$ and append to `corr_pyramid` [S1, unofficial].
    *   For each level $i$ from $1$ to `num_levels` - 1:
        *   Downsample the current correlation volume using average pooling with kernel $[1, 2]$ and stride $[1, 2]$ along the width (disparity) dimension:
            $$ \text{corr}_{lvl} = \text{F.avg\_pool2d}(\text{corr}_{prev}, \text{kernel\_size}=[1, 2], \text{stride}=[1, 2]) $$
        *   Append $\text{corr}_{lvl}$ to `corr_pyramid` [S1, unofficial].
        *   *Note on Feature Map Downsampling:* The source also mentions downsampling `fmap2` using `F.avg_pool2d` with kernel $[1, 2]$ and stride $[1, 2]$ for the next level in the context of coordinate generation, implying the feature maps themselves are downsampled to match the correlation pyramid resolution [S1, unofficial].
5.  **Inference Query (`__call__`):**
    *   **Coordinate Normalization:** Given input coordinates `coords` (shape $(B, H, W, 2)$), normalize them to $[-1, 1]$:
        $$ x_{norm} = \frac{2 \cdot x_{grid}}{W - 1} - 1 $$
        $$ y_{norm} = \frac{2 \cdot y_{grid}}{H - 1} - 1 $$
        Concatenate along the last dimension to form `grid` [S1, unofficial].
    *   **Level-wise Sampling:** For each level $i$ in `range(num_levels)`:
        *   Retrieve $\text{corr}_{lvl}$ from `corr_pyramid` [S1, unofficial].
        *   Create delta offsets for the disparity (x) dimension: $\Delta x = \text{linspace}(-r, r, 2r+1)$ and $\Delta y = 0$ [S1, unofficial].
        *   Create a meshgrid of $\Delta y$ and $\Delta x$ [S1, unofficial].
        *   Scale the centroid coordinates by $2^i$: $\text{coords}_{scaled} = \text{coords} / 2^i$ [S1, unofficial].
        *   Add the delta offsets to the scaled centroid coordinates to get sampling coordinates $\text{coords}_{lvl}$ [S1, unofficial].
        *   Sample the correlation volume $\text{corr}_{lvl}$ at $\text{coords}_{lvl}$ using bilinear sampling (`bilinear_sampler` or `F.grid_sample`) [S1, unofficial].
        *   Reshape the sampled output to $(B, H, W, -1)$ and append to `out_pyramid` [S1, unofficial].
    *   **Final Output:** Concatenate all tensors in `out_pyramid` along the last dimension, permute to $(B, C_{out}, H, W)$, make contiguous, and cast to float [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

class CorrBlockFast1D:
    def __init__(self, fmap1, fmap2, num_levels=4, radius=4):
        self.fmap1 = fmap1
        self.fmap2 = fmap2
        self.num_levels = num_levels
        self.radius = radius
        self.corr_pyramid = []
        self._build_pyramid()

    def _build_pyramid(self):
        # All-pairs correlation
        # einsum: (B, C, H, W1) x (B, C, H, W2) -> (B, H, W1, W2)
        # Note: Source uses 'aijk,aijh->ajkh' which implies specific dim ordering.
        # Assuming standard NCHW: fmap1=(B,C,H,W), fmap2=(B,C,H,W)
        # The source note 'aijk,aijh->ajkh' suggests dims: a=batch, i=height, j=width1, k=channel, h=width2?
        # Let's stick to the explicit reshape provided in source:
        # corr = torch.einsum('aijk,aijh->ajkh', fmap1, fmap2)
        # This implies fmap1 shape (B, H, W1, C) and fmap2 shape (B, H, W2, C) if 'a' is batch, 'i' is H, 'j' is W1, 'k' is C, 'h' is W2.
        # However, PyTorch standard is NCHW. The source code context shows permute operations.
        # We will assume the source's einsum string is correct for the internal tensor layout.
        corr = torch.einsum('aijk,aijh->ajkh', self.fmap1, self.fmap2)
        
        # Normalize
        D = self.fmap1.shape[-1] # Assuming last dim is C based on einsum 'k'
        corr = corr / torch.sqrt(torch.tensor(D, dtype=corr.dtype))
        
        # Reshape for pyramid: (B*H*W1, 1, 1, W2)
        B, H, W1, W2 = corr.shape[0], corr.shape[1], corr.shape[2], corr.shape[3]
        corr = corr.view(B * H * W1, 1, 1, W2)
        self.corr_pyramid.append(corr)
        
        # Downsample for subsequent levels
        for i in range(1, self.num_levels):
            # Average pool along the last dimension (width/disparity)
            corr = F.avg_pool2d(corr, kernel_size=[1, 2], stride=[1, 2])
            self.corr_pyramid.append(corr)

    def __call__(self, coords):
        # coords shape: (B, H, W, 2)
        out_pyramid = []
        
        for i in range(self.num_levels):
            corr_lvl = self.corr_pyramid[i]
            
            # Scale coordinates by 2**i
            coords_scaled = coords / (2 ** i)
            
            # Create delta offsets
            r = self.radius
            dx = torch.linspace(-r, r, 2 * r + 1, device=coords.device)
            dy = torch.zeros_like(dx)
            
            # Meshgrid
            dy_grid, dx_grid = torch.meshgrid(dy, dx, indexing='ij')
            # Stack to form offsets (H_offset, W_offset, 2) -> (2r+1, 2r+1, 2)
            offsets = torch.stack([dx_grid, dy_grid], dim=-1)
            
            # Add offsets to scaled coordinates
            # coords_scaled: (B, H, W, 2)
            # offsets: (2r+1, 2r+1, 2)
            # We need to broadcast offsets to (B, H, W, 2r+1, 2r+1, 2)
            # And coords_scaled to (B, H, W, 1, 1, 2)
            coords_lvl = coords_scaled.unsqueeze(3).unsqueeze(4) + offsets.unsqueeze(0).unsqueeze(0).unsqueeze(0)
            
            # Sample correlation
            # corr_lvl shape: (B*H*W1, 1, 1, W2)
            # This requires a custom bilinear sampler or grid_sample adapted for this specific shape.
            # The source mentions `bilinear_sampler`. We assume it handles the (B*H*W1, 1, 1, W2) tensor.
            # For implementation clarity, we note that grid_sample expects (N, C, H, W) and (N, H', W', 2).
            # Here, the "spatial" dimensions are complex.
            # We will write a comment as the exact sampler implementation is not fully detailed in the source text alone.
            sampled_corr = bilinear_sampler(corr_lvl, coords_lvl) # [G: Placeholder for specific bilinear sampler logic]
            
            # Reshape
            sampled_corr = sampled_corr.view(coords.shape[0], coords.shape[1], coords.shape[2], -1)
            out_pyramid.append(sampled_corr)
            
        # Concatenate and permute
        out = torch.cat(out_pyramid, dim=-1)
        out = out.permute(0, 3, 1, 2)
        out = out.contiguous()
        out = out.float()
        return out

def bilinear_sampler(corr, coords):
    # [G] Implementation detail: This function must sample the correlation volume
    # at the specified coordinates using bilinear interpolation.
    # Given the source's specific tensor shapes, this likely involves reshaping
    # corr to (B, 1, H, W_disparity) and coords to (B, H, W, 2) appropriately.
    # Since the source does not provide the full bilinear_sampler code, we mark it as a gap.
    # A standard grid_sample approach would be:
    # corr_reshaped = corr.view(B, 1, H, W_disparity)
    # coords_reshaped = ...
    # return F.grid_sample(corr_reshaped, coords_reshaped, align_corners=True)
    raise NotImplementedError("Bilinear sampler implementation depends on specific tensor reshaping not fully detailed in source.")
```

**Edge Cases and Pitfalls:**
*   **Precision:** Division by $\sqrt{D}$ requires $D > 0$.
*   **Boundaries:** Coordinate normalization assumes $W > 1$ and $H > 1$ to avoid division by zero.
*   **Degenerate Input:** If `fmap1` or `fmap2` have zero channels, the correlation is undefined.
*   **Memory:** The all-pairs correlation step creates a large intermediate tensor $(B, H, W_1, W_2)$. For high-resolution images, this can exceed GPU memory.

## Operating characteristics
*   **Time Complexity:** The all-pairs correlation step is $O(B \cdot H \cdot W_1 \cdot W_2 \cdot C)$. Pyramid construction is $O(\sum_{i=0}^{L} H_i W_i)$, where $L$ is the number of levels. Sampling is $O(B \cdot H \cdot W \cdot L \cdot (2r+1)^2)$.
*   **Memory Complexity:** Dominated by the initial correlation volume $O(B \cdot H \cdot W_1 \cdot W_2)$.
*   **Performance:** Efficient for inference due to pre-computed pyramid. The source does not provide specific measured performance numbers on hardware/resolution.
*   **Strengths:** Multi-scale support via pyramid; efficient sampling.
*   **Limitations:** High memory usage for large images; requires custom bilinear sampler for the specific tensor layout.

## Background
CorrBlockFast1D is part of the GPS+ project, an unofficial implementation for stereo matching and human modeling [S1, unofficial]. It is designed to work within a progressive downsampling framework for efficient correlation computation [S1, unofficial].

## Sources
[S1] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
