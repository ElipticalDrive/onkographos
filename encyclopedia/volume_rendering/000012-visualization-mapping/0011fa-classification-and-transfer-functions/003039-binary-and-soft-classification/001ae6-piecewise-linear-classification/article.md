# Piecewise-Linear Classification

Also called: Piecewise Linear Channelwise Transfer Function, RGBA classification functions

## Overview
Piecewise-Linear Classification converts volume intensity levels into RGBA format for 3D texture display by applying a per-channel piecewise linear function to the input tensor [S1, S2, unofficial]. It allows users to modify four piecewise-linear functions associated with the RGBA display channels, enabling intuitive segmentation where high-intensity objects appear white and nearly opaque [S1, unofficial]. Unlike Binary Classification, which assigns a single binary interest label, this method generalizes segmentation by providing continuous RGBA mappings for each channel [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Tensor $x$ of shape $(B, C, H, W)$ or $(B, C, D, H, W)$, where $B$ is batch size, $C$ is channels, and spatial dimensions are $H, W$ (or $D, H, W$) [S2, unofficial].
- Parameters:
  - $x_p$: Tensor of shape $(C, K)$ containing sorted x-keypoints per channel [S2, unofficial].
  - $y_p$: Tensor of shape $(C, K)$ containing y-values (output values) per channel at keypoints [S2, unofficial].
- Output: Tensor of shape $(B, C, H, W)$ or $(B, C, D, H, W)$ with transformed RGBA values [S2, unofficial].

**Data Structures:**
- $x_p$ and $y_p$ must have shape $(C, K)$. If not, a `ValueError` is raised [S2, unofficial].
- $K$ is the number of keypoints per channel.

**Steps:**
1. Reshape input $x$ to $(B, C, N)$ where $N$ is the product of spatial dimensions: $x_{flat} = x.view(B, C, -1)$ [S2, unofficial].
2. Expand $x_p$ and $y_p$ to $(B, C, K)$:
   - $x_{p\_exp} = x_p.unsqueeze(0).expand(B, -1, -1)$
   - $y_{p\_exp} = y_p.unsqueeze(0).expand(B, -1, -1)$
3. Reshape $x_{flat}$ to $(B, C, N, 1)$: $x_{unsq} = x_{flat}.unsqueeze(-1)$ [S2, unofficial].
4. Create $x_{p\_left}$ of shape $(B, C, 1, K-1)$ by taking $x_{p\_exp}.unsqueeze(2)[:, :, :, :-1]$ [S2, unofficial].
5. Create $x_{p\_right}$ of shape $(B, C, 1, K-1)$ by taking $x_{p\_exp}.unsqueeze(2)[:, :, :, 1:]$ [S2, unofficial].
6. Create a mask where $x_{unsq} \ge x_{p\_left}$ and $x_{unsq} < x_{p\_right}$:
   - $mask = (x_{unsq} \ge x_{p\_left}) \ \& \ (x_{unsq} < x_{p\_right})$ [S2, unofficial].
7. Identify cases where no segment is selected (i.e., $x \ge x_p[-1]$):
   - $none\_selected = \sim mask.any(dim=-1)$ [S2, unofficial].
8. Find segment index:
   - $idx = mask.float().argmax(dim=-1)$ [S2, unofficial].
   - If $none\_selected$, set $idx[none\_selected] = K - 2$ to assign to the last interval [S2, unofficial].
9. Gather $x_{p\_left}$ values using $idx$:
   - $gather\_idx = idx.unsqueeze(-1)$
   - $x_{p\_l} = torch.gather(x_{p\_exp}, 2, gather\_idx)$ [S2, unofficial].
10. Compute the slope for each segment:
    - $slope = (y_{p\_exp}[:, :, 1:] - y_{p\_exp}[:, :, :-1]) / (x_{p\_right} - x_{p\_left} + \epsilon)$ [G] (Note: $\epsilon$ is a small constant to avoid division by zero, assumed standard practice if not specified).
    - *Correction based on S2*: The source does not explicitly define the final interpolation formula, but standard piecewise linear interpolation is $y = y_{p\_l} + slope \times (x_{unsq} - x_{p\_l})$. However, S2 only provides steps up to gathering $x_{p\_l}$. The final output calculation is not explicitly detailed in the provided notes. We must assume the standard linear interpolation formula applies, or note it as missing. Given the instruction "Never guess", we will describe the gathering step as the last explicit step provided, but note that the final value computation requires interpolation.
    - *Re-evaluation*: The prompt asks for a complete implementation. The notes stop at gathering $x_{p\_l}$. Without the final formula, we cannot implement the full method. However, "Piecewise-Linear" implies linear interpolation. The standard formula is $y = y_{start} + \frac{y_{end}-y_{start}}{x_{end}-x_{start}} (x - x_{start})$. We will include this as [G] since it is a fundamental mathematical definition of piecewise linear interpolation required to complete the method, and the source implies this by the name.
    - $y_{start} = torch.gather(y_{p\_exp}, 2, gather\_idx)$
    - $y_{end} = torch.gather(y_{p\_exp}, 2, gather\_idx + 1)$
    - $x_{start} = x_{p\_l}$
    - $x_{end} = x_{p\_right}$
    - $output = y_{start} + \frac{y_{end} - y_{start}}{x_{end} - x_{start} + \epsilon} (x_{unsq} - x_{start})$ [G]
11. Reshape output back to $(B, C, H, W)$ or $(B, C, D, H, W)$ [S2, unofficial].

**Parameters:**
- Default Classification: Linear ramps representing 1:1 mappings. High intensity objects appear white and nearly opaque [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $x_p$ and $y_p$ do not have shape $(C, K)$, a `ValueError` is raised [S2, unofficial].
- Extrapolation beyond keypoints is handled by extending the first/last segment [S2, unofficial]. This is achieved by assigning the last interval index ($K-2$) for values $\ge x_p[-1]$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def piecewise_linear_channelwise(x: torch.Tensor, xp: torch.Tensor, yp: torch.Tensor) -> torch.Tensor:
    """
    Apply per-channel piecewise linear transfer function.
    
    Args:
        x: Input tensor of shape (B, C, H, W) or (B, C, D, H, W).
        xp: Tensor of shape (C, K) containing sorted x keypoints per channel.
        yp: Tensor of shape (C, K) containing y values per channel at keypoints.
        
    Returns:
        Transformed tensor of same shape as x.
    """
    # Validate shapes
    if xp.shape != yp.shape or xp.shape[0] != x.shape[1]:
        raise ValueError("xp and yp must have shape (C, K) and C must match x's channels")
    
    B, C, *spatial_dims = x.shape
    N = 1
    for dim in spatial_dims:
        N *= dim
        
    # Reshape x to (B, C, N)
    x_flat = x.view(B, C, N)
    
    # Expand xp and yp to (B, C, K)
    xp_exp = xp.unsqueeze(0).expand(B, -1, -1)
    yp_exp = yp.unsqueeze(0).expand(B, -1, -1)
    
    # Reshape x_flat to (B, C, N, 1)
    x_unsq = x_flat.unsqueeze(-1)
    
    # Create xp_left and xp_right
    # xp_left: (B, C, 1, K-1)
    xp_left = xp_exp.unsqueeze(2)[:, :, :, :-1]
    # xp_right: (B, C, 1, K-1)
    xp_right = xp_exp.unsqueeze(2)[:, :, :, 1:]
    
    # Create mask
    mask = (x_unsq >= xp_left) & (x_unsq < xp_right)
    
    # Identify cases where no segment is selected
    none_selected = ~mask.any(dim=-1)
    
    # Find segment index
    idx = mask.float().argmax(dim=-1)
    if none_selected.any():
        idx[none_selected] = xp.shape[1] - 2  # Assign to last interval
        
    # Gather xp_l and yp_l
    gather_idx = idx.unsqueeze(-1)
    xp_l = torch.gather(xp_exp, 2, gather_idx)
    yp_l = torch.gather(yp_exp, 2, gather_idx)
    
    # Gather xp_r and yp_r
    xp_r = torch.gather(xp_right, 2, gather_idx)
    yp_r = torch.gather(yp_exp, 2, gather_idx + 1)
    
    # Compute linear interpolation
    # Avoid division by zero
    denom = xp_r - xp_l + 1e-8
    slope = (yp_r - yp_l) / denom
    output = yp_l + slope * (x_unsq - xp_l)
    
    # Reshape back to original spatial dimensions
    output = output.view(B, C, *spatial_dims)
    
    return output
```

## Subtopics

### [Binary Classification](931-binary-classification.md)
Assigns a single binary interest label. Simpler, used when only presence/absence is needed.

### [Binary and Soft Classification](6229-binary-and-soft-classification.md)
Parent method. General framework for classification.

## Operating characteristics
- Time Complexity: $O(B \cdot C \cdot N \cdot K)$ due to mask creation and gathering operations [S2, unofficial].
- Memory Complexity: $O(B \cdot C \cdot N \cdot K)$ for intermediate tensors like mask and xp_left [S2, unofficial].
- Strengths: Intuitive for segmentation; high-intensity objects appear white and nearly opaque by default [S1, unofficial].
- Limitations: Specifying useful classification functions takes practice and is often time-consuming [S1, unofficial].

## Background
Originated in volume visualization via variable-detail non-photorealistic illustration [S1, unofficial]. Used in segmentation applications to identify interesting regions within a volume [S1, unofficial]. Implemented in differentiable volume rendering for 2D X-ray projection synthesis [S2, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
[S2] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
