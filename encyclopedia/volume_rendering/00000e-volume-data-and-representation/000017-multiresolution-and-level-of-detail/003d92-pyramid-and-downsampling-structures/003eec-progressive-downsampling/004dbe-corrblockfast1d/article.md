# CorrBlockFast1D

Also called: CorrBlock1D, PytorchAlternateCorrBlock1D

## Overview
CorrBlockFast1D is a 1D correlation block implementation for stereo matching that computes correlations using grid sampling on downsampled feature maps, supporting a multi-level pyramid structure [S1, unofficial]. It pre-computes a correlation pyramid using all-pairs correlation and bilinear sampling for efficient inference [S1, unofficial]. Unlike standard correlation layers that compute pairwise similarities on the fly, this method constructs a pyramid of correlation volumes between two feature maps to enable efficient multi-scale 1D correlation sampling [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two feature maps `fmap1` and `fmap2`, `num_levels` (integer, default 4), and `radius` (integer, default 4) [S1, unofficial].
*   **Outputs:** A concatenated correlation volume from all pyramid levels, permuted to shape $(B, C, H, W)$ and cast to float [S1, unofficial].

**Data Structures:**
*   `corr_pyramid`: A list of tensors, each representing the correlation volume at a specific pyramid level.
*   `coords`: Normalized coordinates for sampling, shape $(B, H, W, 2)$ containing $(x, y)$ components in range $[-1, 1]$ [S1, unofficial].

**Step-by-Step Method:**

1.  **Initialization:** Store `fmap1`, `fmap2`, `num_levels`, and `radius`. Initialize an empty list `corr_pyramid` [S1, unofficial].
2.  **All-Pairs Correlation:** Compute the raw correlation between `fmap1` and `fmap2` using Einstein summation:
    $$ \text{corr}_{\text{raw}} = \text{torch.einsum}(\text{'aijk,aijh->ajkh'}, \text{fmap1}, \text{fmap2}) $$
    resulting in shape $(B, H, W_1, W_2)$ [S1, unofficial]. Reshape to $(B, H, W_1, 1, W_2)$ [S1, unofficial].
3.  **Normalization:** Normalize the correlation by dividing by $\sqrt{D}$, where $D$ is the feature dimension [S1, unofficial].
4.  **Pyramid Construction:** Reshape `corr_norm` to $(B \cdot H \cdot W_1, 1, 1, W_2)$ and append to `corr_pyramid` [S1, unofficial]. For each subsequent level $i$ from 1 to `num_levels`-1, downsample the current correlation volume using average pooling with kernel $[1, 2]$ and stride $[1, 2]$ along the width (disparity) dimension, then append to `corr_pyramid` [S1, unofficial].
5.  **Inference Query (`__call__`):**
    *   **Coordinate Normalization:** Given input coordinates `coords` (shape $(B, H, W, 2)$), normalize them to $[-1, 1]$:
        $$ x_{\text{norm}} = \frac{2 \cdot x_{\text{grid}}}{W - 1} - 1 $$
        $$ y_{\text{norm}} = \frac{2 \cdot y_{\text{grid}}}{H - 1} - 1 $$
        Concatenate along the last dimension to form `grid` [S1, unofficial].
    *   **Level-wise Sampling:** For each level $i$ in `range(num_levels)`:
        *   Retrieve $\text{corr}_{\text{lvl}}$ from `corr_pyramid` [S1, unofficial].
        *   Scale the centroid coordinates by $2^i$: $\text{coords}_{\text{scaled}} = \text{grid} / 2^i$ [S1, unofficial].
        *   Create delta offsets for the disparity (x) dimension: $\Delta x = \text{linspace}(-r, r, 2r+1)$ and $\Delta y = 0$ [S1, unofficial].
        *   Compute sampling coordinates by adding $\Delta x$ to the x-component of $\text{coords}_{\text{scaled}}$ and setting the y-component offset to zero: $\text{coords}_{\text{lvl}} = \text{cat}([\text{coords}_{\text{scaled}}[..., 0:1] + \Delta x, \text{coords}_{\text{scaled}}[..., 1:2]], \text{dim}=-1)$ [S1, unofficial].
        *   Sample the correlation volume $\text{corr}_{\text{lvl}}$ at $\text{coords}_{\text{lvl}}$ using bilinear sampling [S1, unofficial].
        *   Reshape the sampled output to $(B, H, W, -1)$ and append to `out_pyramid` [S1, unofficial].
    *   **Final Output:** Concatenate all tensors in `out_pyramid` along the last dimension, permute to $(B, C, H, W)$, make contiguous, and cast to float [S1, unofficial].

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
        # All-pairs correlation: einsum 'aijk,aijh->ajkh'
        # Assumes input tensors match the einsum dimension mapping (a=batch, i=height, j=width1, k=channel, h=width2)
        corr = torch.einsum('aijk,aijh->ajkh', self.fmap1, self.fmap2)
        # corr shape: (B, H, W1, W2)
        corr = corr.unsqueeze(3)  # (B, H, W1, 1, W2)
        D = self.fmap1.shape[-1]
        corr = corr / torch.sqrt(torch.tensor(D, dtype=corr.dtype))
        # Reshape for pyramid: (B*H*W1, 1, 1, W2)
        B, H, W1, W2 = corr.shape[0], corr.shape[1], corr.shape[2], corr.shape[4]
        corr = corr.view(B * H * W1, 1, 1, W2)
        self.corr_pyramid.append(corr)
        # Downsample for subsequent levels
        for _ in range(1, self.num_levels):
            corr = F.avg_pool2d(corr, kernel_size=[1, 2], stride=[1, 2])
            self.corr_pyramid.append(corr)

    def __call__(self, coords):
        # coords shape: (B, H, W, 2) in pixel coordinates
        # Normalize coordinates to [-1, 1]
        x_grid, y_grid = coords[..., 0:1], coords[..., 1:2]
        W, H = coords.shape[2], coords.shape[1]
        x_norm = 2 * x_grid / (W - 1) - 1
        y_norm = 2 * y_grid / (H - 1) - 1
        grid = torch.cat([x_norm, y_norm], dim=-1)

        out_pyramid = []
        for i in range(self.num_levels):
            corr_lvl = self.corr_pyramid[i]
            # Scale coordinates by 2^i
            coords_scaled = grid / (2 ** i)
            # Create 1D delta offsets for x (disparity), y offset is zero
            r = self.radius
            dx = torch.linspace(-r, r, 2 * r + 1, device=coords.device)
            # coords_scaled: (B, H, W, 2)
            # dx: (2r+1,)
            # Add dx to x-component, keep y-component unchanged (offset 0)
            coords_lvl = torch.cat([coords_scaled[..., 0:1] + dx, coords_scaled[..., 1:2]], dim=-1)
            # coords_lvl shape: (B, H, W, 2r+1, 2)
            
            # Prepare for grid_sample: (N, C, H', W') and (N, H', W', 2)
            B_s, H_s, W_s, D_s, _ = coords_lvl.shape
            N = B_s * H_s * W_s
            corr_reshaped = corr_lvl.view(N, 1, 1, corr_lvl.shape[-1])
            coords_reshaped = coords_lvl.view(N, 1, D_s, 2)
            
            # Sample correlation using bilinear interpolation
            sampled = F.grid_sample(corr_reshaped, coords_reshaped, align_corners=True, mode='bilinear')
            # sampled shape: (N, 1, 1, 2r+1)
            sampled = sampled.view(B_s, H_s, W_s, -1)
            out_pyramid.append(sampled)

        out = torch.cat(out_pyramid, dim=-1)
        out = out.permute(0, 3, 1, 2)
        out = out.contiguous()
        return out.float()
```

**Edge Cases and Pitfalls:**
*   **Precision:** Division by $\sqrt{D}$ requires $D > 0$ [G].
*   **Boundaries:** Coordinate normalization assumes $W > 1$ and $H > 1$ to avoid division by zero [G].
*   **Coordinate Scaling:** Off-by-one errors can occur if `coords` are not properly normalized to $[-1, 1]$ before division by $2^i$ [S1, unofficial].
*   **Tensor Layout:** The `einsum` string `'aijk,aijh->ajkh'` assumes a specific dimension ordering; mismatched input layouts will produce incorrect correlation shapes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Efficient for inference due to pre-computed pyramid. The source does not provide specific measured performance numbers on hardware/resolution [S1, unofficial].
*   **Strengths:** Multi-scale support via pyramid; efficient sampling [S1, unofficial].
*   **Limitations:** High memory usage for large images; requires custom bilinear sampler for the specific tensor layout [S1, unofficial].

## Background
CorrBlockFast1D is part of the GPS+ project, an unofficial implementation for stereo matching and human modeling [S1, unofficial]. It is designed to work within a progressive downsampling framework for efficient correlation computation [S1, unofficial].

## Sources
[S1] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
