# Index-Based Upsampling with Recalibration

Also called: Index-based upsampling, recalibration upsampling

## Overview
Index-Based Upsampling with Recalibration is a technique to upsample low-resolution tensors to high resolution using an index map and a recalibration map, specifically for depth values [S1, unofficial]. It sets itself apart from standard bilinear upsampling by using explicit integer indices to gather low-resolution samples and a separate scaling map to adjust for resolution changes, ensuring that samples from similar depth planes contribute correctly to the final color [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Low-resolution tensor $T_{low}$, `index_map`, `recalib_map`.
*   **Output:** High-resolution tensor $T_{high}$.

**Data Structures:**
*   `index_map`: A tensor of shape $(H \times W, 2)$ containing integer indices mapping high-resolution pixels to low-resolution pixels [S1, unofficial].
*   `recalib_map`: A tensor of shape $(H \times W)$ containing scaling factors for recalibration [S1, unofficial].
*   $H, W$: Height and width of the high-resolution output.

**Steps:**
1.  **Gather:** For each pixel $(i, j)$ in the high-resolution grid, retrieve the corresponding low-resolution coordinates from `index_map`. Let these be $(u, v)$. Gather the value from $T_{low}$ at $(u, v)$.
2.  **Recalibrate:** Retrieve the scaling factor $s_{i,j}$ from `recalib_map` at $(i, j)$.
3.  **Scale:** Multiply the gathered value by $s_{i,j}$ to produce the high-resolution value.

**Formula:**
$$ T_{high}[i, j] = T_{low}[\text{index\_map}[i, j, 0], \text{index\_map}[i, j, 1]] \times \text{recalib\_map}[i, j] $$

**Parameters:**
*   Typical values for `ratio` (resolution scaling) are around 1.5 [S1, unofficial].
*   Thresholds for rendering steps may be set to $5 \times 10^{-5}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Precision issues may arise if index mapping is not integer-accurate.
*   Boundary conditions for `index_map` must be handled to avoid out-of-bounds access [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def index_based_upsampling_with_recalibration(T_low, index_map, recalib_map):
    """
    Upsamples low-resolution tensor T_low to high resolution using index_map and recalib_map.
    
    Args:
        T_low: Low-resolution tensor of shape (H_low, W_low, C).
        index_map: Tensor of shape (H_high, W_high, 2) with integer indices.
        recalib_map: Tensor of shape (H_high, W_high) with scaling factors.
        
    Returns:
        T_high: High-resolution tensor of shape (H_high, W_high, C).
    """
    H_high, W_high = recalib_map.shape
    C = T_low.shape[2] if len(T_low.shape) > 2 else 1
    
    # Initialize high-resolution tensor
    T_high = torch.zeros((H_high, W_high, C), dtype=T_low.dtype, device=T_low.device)
    
    # Extract indices
    u_indices = index_map[:, :, 0].long()
    v_indices = index_map[:, :, 1].long()
    
    # Gather values from low-resolution tensor
    # Assuming T_low is 2D or 3D; handling 2D case for simplicity if C=1
    if len(T_low.shape) == 2:
        gathered = T_low[u_indices, v_indices]
    else:
        gathered = T_low[u_indices, v_indices, :]
        
    # Apply recalibration scaling
    # recalib_map needs to be broadcasted to match gathered shape
    recalib_expanded = recalib_map.unsqueeze(-1) if len(T_low.shape) > 2 else recalib_map
    T_high = gathered * recalib_expanded
    
    return T_high
```

## Subtopics

### [Depth-Aware Upsampling](2777-depth-aware-upsampling.md)
Depth-aware upsampling reduces blue noise and performance costs in volumetric rendering by rendering clouds to a low-resolution RGBA texture and a distance buffer, then upsampling to high resolution using the depth buffer. It sets itself apart from standard bilinear upsampling by weighting the interpolation of low-resolution samples based on the depth difference between the high-resolution fragment and the low-resolution depth values, ensuring that samples from similar depth planes contribute more to the final color.

## Operating characteristics
*   **Time Complexity:** $O(H \times W)$, where $H$ and $W$ are the high-resolution dimensions, due to the linear scan of the index and recalibration maps.
*   **Memory Complexity:** $O(H \times W)$ for storing the index and recalibration maps.
*   **Strengths:** Explicit control over sampling locations via indices; recalibration allows for correction of resolution-induced artifacts.
*   **Limitations:** Requires pre-computation of index and recalibration maps; may introduce aliasing if indices are not carefully chosen [G].
*   **When to use:** When explicit control over upsampling is needed, particularly for depth values or when bilinear interpolation is insufficient [S1, unofficial].
*   **When not to use:** When real-time performance is critical and pre-computation is not feasible, or when smooth interpolation is preferred over explicit indexing [G].

## Background
Originating from the DIST-Renderer project, this technique is part of a broader effort to optimize volumetric rendering through multiresolution representations [S1, unofficial]. It builds on the concept of downscaled grid maps and index maps to efficiently upsample data [S1, unofficial].

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
