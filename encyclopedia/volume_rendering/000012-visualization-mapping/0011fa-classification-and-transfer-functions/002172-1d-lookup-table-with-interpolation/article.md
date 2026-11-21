# 1D Lookup Table with Interpolation

Also called: HU -> mu mapping, Piecewise Linear HU to Attenuation Transfer Function, Piecewise Linear Transfer Function

## Overview
This method maps scalar voxel intensities, typically Hounsfield Units (HU), to optical properties such as attenuation coefficients ($\mu$), color, or opacity using a piecewise linear function defined by user-specified keypoints [S1, unofficial]. It builds on [Classification and transfer functions](38-classification-and-transfer-functions.md) by providing a concrete implementation of soft classification via linear interpolation between discrete control points [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume tensor $V$ of scalar values (e.g., HU) and a keypoints tensor $P$ of shape $(K, 2)$ [S1, unofficial].
*   **Output:** A 3D volume tensor $M$ of mapped values (e.g., attenuation coefficients $\mu$) [S1, unofficial].

**Data Structures:**
The transfer function is defined by a tensor of keypoints $P$ with shape $(K, 2)$, where $K$ is the number of keypoints [S1, unofficial].
*   Column 0: Input domain values (e.g., HU) [S1, unofficial].
*   Column 1: Output range values (e.g., attenuation $\mu$, opacity $\alpha$, or color components) [S1, unofficial].

**Steps:**
1.  **Define Keypoints:** Create the tensor $P$ of shape $(K, 2)$ [S1, unofficial].
2.  **Normalize Input Domain:** Map the raw input values to a normalized range $[0, 1]$ using an offset of 3500 and a scale of 7000 [S1, unofficial]:
    $$ x_p = \frac{x + 3500}{7000} $$
    where $x$ is the raw HU value [S1, unofficial].
3.  **Extract Tensors:** Extract the normalized input values $x_p$ and output values $y_p$ as 1D tensors of shape $(1, K)$ [S1, unofficial].
4.  **Interpolate:** Apply piecewise linear interpolation to the volume using `R.piecewise_linear_channelwise(vol, xp, yp)` to generate the attenuation map `mu_tf` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def apply_piecewise_linear_tf(volume: torch.Tensor, keypoints: torch.Tensor) -> torch.Tensor:
    """
    Applies piecewise linear transfer function to a volume.
    
    Args:
        volume: 3D tensor of shape (D, H, W) containing HU values.
        keypoints: 2D tensor of shape (K, 2). Column 0 is HU, Column 1 is output (e.g., mu).
        
    Returns:
        3D tensor of shape (D, H, W) with mapped values.
    """
    # 1. Extract input (x) and output (y) values from keypoints
    x_kp = keypoints[:, 0]  # HU values
    y_kp = keypoints[:, 1]  # Output values (e.g., attenuation)
    
    # 2. Normalize HU values to [0, 1] using offset 3500 and scale 7000 [S1, unofficial]
    x_norm_kp = (x_kp + 3500.0) / 7000.0
    vol_norm = (volume + 3500.0) / 7000.0
    
    # 3. Extract normalized input (xp) and output (yp) as 1D tensors of shape (1, K) [S1, unofficial]
    xp = x_norm_kp.unsqueeze(0)  # shape (1, K)
    yp = y_kp.unsqueeze(0)       # shape (1, K)
    
    # 4. Perform piecewise linear interpolation
    # Ensure keypoints are sorted by input value [G]
    sorted_indices = torch.argsort(xp[0])
    xp_sorted = xp[0][sorted_indices].unsqueeze(0)
    yp_sorted = yp[0][sorted_indices].unsqueeze(0)
    
    # Find interval indices for each voxel [G]
    indices = torch.searchsorted(xp_sorted[0], vol_norm.flatten(), side='right') - 1
    indices = torch.clamp(indices, 0, len(xp_sorted[0]) - 2)
    
    # Get bounds for the interval
    x_low = xp_sorted[0, indices]
    x_high = xp_sorted[0, indices + 1]
    y_low = yp_sorted[0, indices]
    y_high = yp_sorted[0, indices + 1]
    
    # Handle duplicate keypoints (denom == 0) to prevent division by zero [G]
    denom = x_high - x_low
    denom = torch.where(denom == 0, torch.ones_like(denom), denom)
    
    # Linear interpolation formula
    t = (vol_norm.flatten() - x_low) / denom
    result = y_low + t * (y_high - y_low)
    
    return result.view(volume.shape)
```

**Parameters:**
*   **Keypoints ($K$):** The number of control points. Typical values depend on the anatomy; e.g., 5 keypoints for air, lung/soft-tissue, soft tissue, cortical bone, and dense bone/metal [S1, unofficial].
*   **Normalization Offset/Scale:** Offset is 3500 and scale is 7000, mapping the HU range to $[0, 1]$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Out-of-Bounds Values:** Voxel values outside the range defined by the first and last keypoints must be clamped to the nearest endpoint value to prevent undefined behavior [G].
*   **Duplicate Keypoints:** If two keypoints have the same input value ($x_i = x_{i+1}$), the denominator in the interpolation formula becomes zero. This is handled by clamping the denominator to 1 in the pseudocode [G].
*   **Unsorted Keypoints:** The interpolation logic assumes keypoints are sorted by input value. If not, they must be sorted before processing [G].

## Subtopics

### Label-aware Interpolation Switching
Switches volume interpolation type to nearest-neighbor when labels are present to prevent band-crossing artifacts, otherwise uses linear interpolation [S2, unofficial]. It solves the problem of linear interpolation across label band boundaries producing values that fall into incorrect anatomical classes, which breaks label-scoped transfer functions [S2, unofficial]. Implementation: Check if labels are present via `hasLabels()`. If labels exist, call `volume.getProperty().setInterpolationTypeToNearest()`. Otherwise, call `volume.getProperty().setInterpolationTypeToLinear()` [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
The method is a specific instance of the broader [1D Transfer Function](11719-1d-transfer-function.md) family [G].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S2] kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, https://github.com/kevinkunkel98/reinforcement-learning-volume-rendering-transfer-function, fetched 2026-10-27 (unofficial)
