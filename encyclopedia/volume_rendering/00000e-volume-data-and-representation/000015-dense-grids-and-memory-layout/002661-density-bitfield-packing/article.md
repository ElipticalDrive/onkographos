# Density Bitfield Packing

Also called: packbits

## Overview
Density Bitfield Packing compresses a floating-point density grid into a binary bitfield to accelerate volume ray marching by enabling efficient empty-space skipping [S1, S2, unofficial]. Unlike standard dense grid storage which samples every voxel, this method allows the renderer to jump over large empty regions by interpreting the bitfield as a spatial index of occupied voxels [S1, S2, unofficial]. It differs from [Evaluation Cache](874-evaluation-cache.md) by operating on the raw density field rather than precomputed ray integrals, and from [Bricking](914-bricking.md) by focusing on binary occupancy rather than hierarchical tiling for caching [S1, S2, unofficial].

## How it works
**Inputs:**
*   `grid`: A 3D density field represented as a contiguous tensor of shape $[C, H, H, H]$ or flattened to $[C, H^3]$, where $C$ is the number of channels (e.g., density, color) and $H$ is the grid dimension [S1, S2, unofficial]. $H$ must be even ($H \% 2 == 0$) [S1, S2, unofficial].
*   `thresh`: A scalar float threshold used for binarization [S1, S2, unofficial].

**Outputs:**
*   `bitfield`: A binary occupancy map stored as a `uint8` tensor of shape $[C, H^3 / 8]$ [S1, S2, unofficial].

**Data Structures:**
The output `bitfield` packs 8 voxels into a single byte. For a grid of size $H \times H \times H$, the total number of voxels is $V = H^3$. The number of bytes required is $N = V / 8 = C \cdot H^3 / 8$ [S1, unofficial]. Each byte corresponds to a $2 \times 2 \times 2$ block of voxels in the input grid [G]. The $k$-th bit of the $n$-th byte (where $n = \lfloor k/8 \rfloor$ and $b = k \% 8$) represents the occupancy of the $k$-th voxel in the flattened grid [G].

**Algorithm:**
1.  Ensure the input `grid` is contiguous in memory (e.g., a contiguous CUDA tensor or flattened NumPy array) [S1, unofficial].
2.  Calculate the total number of output bytes $N = C \cdot H^3 / 8$ [S1, unofficial].
3.  Iterate through each voxel $v$ in the flattened `grid` with index $k \in [0, H^3 - 1]$.
4.  For each voxel, compare its density value $d_k$ against the threshold `thresh`:
    $$ b_k = \begin{cases} 1 & \text{if } d_k \ge \text{thresh} \\ 0 & \text{otherwise} \end{cases} $$
5.  Pack the bits $b_k$ into the corresponding position in the `bitfield` byte. If the voxel index is $k$, the target byte index is $n = \lfloor k/8 \rfloor$ and the bit position is $p = k \% 8$ [G]. Set the $p$-th bit of `bitfield[n]` to $b_k$ [G].
6.  Return the `bitfield` [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def packbits(grid: torch.Tensor, thresh: float) -> torch.Tensor:
    """
    Packs a floating-point density grid into a bitfield.
    
    Args:
        grid: Float tensor of shape [C, H, H, H] or [C, H^3]. H must be even.
        thresh: Float threshold for binarization.
        
    Returns:
        uint8 tensor of shape [C, H^3 / 8].
    """
    # Ensure contiguous memory layout
    if not grid.is_contiguous():
        grid = grid.contiguous()
        
    # Flatten grid to [C, H^3] if not already
    if grid.dim() == 4:
        C, H, _, _ = grid.shape
        grid = grid.view(C, -1)
    else:
        C, _ = grid.shape
        
    # Check H is even (H^3 is divisible by 8 if H is even)
    # Note: H is derived from grid.shape[1] if 2D view, or inferred.
    # Assuming input is [C, H^3], we need H.
    # If input was [C, H, H, H], H = int(sqrt(sqrt(grid.shape[1])))
    # For simplicity, assume we know H or infer from total size.
    # Let's assume grid is [C, H^3].
    total_voxels = grid.shape[1]
    H = int(round(total_voxels ** (1/3)))
    
    # Binarize
    # Create mask: 1 if grid >= thresh, else 0
    mask = (grid >= thresh).float()
    
    # Reshape to [C, H^3/8, 8] to pack bits
    # H^3 is divisible by 8 because H is even
    num_bytes = total_voxels // 8
    mask_reshaped = mask.view(C, num_bytes, 8)
    
    # Initialize bitfield
    bitfield = torch.zeros((C, num_bytes), dtype=torch.uint8, device=grid.device)
    
    # Pack bits
    # For each byte, combine 8 bits
    for bit_idx in range(8):
        # Shift the bit to the correct position
        # bit_idx 0 is LSB, bit_idx 7 is MSB (or vice versa, depends on convention)
        # Standard packing: bit 0 of voxel k goes to bit (k%8) of byte
        bitfield |= (mask_reshaped[:, :, bit_idx].to(torch.uint8) << bit_idx)
        
    return bitfield
```

**Edge Cases and Pitfalls:**
*   **Grid Dimensions:** $H$ must be even to ensure $H^3$ is divisible by 8. If $H$ is odd, padding or irregular packing is required, which is not supported by the standard implementation [S1, S2, unofficial].
*   **Precision:** The threshold comparison is exact. Floating-point noise in density values near the threshold may cause jitter in the bitfield if the grid is dynamic [G].
*   **Memory:** The bitfield reduces memory usage by a factor of 8 compared to the original float grid, but requires an additional pass to generate [S1, S2, unofficial].

## Subtopics

### Bitwise Density Computation
Calculates the number of filled voxels between a scene point and a light position to simulate translucency, avoiding expensive bit counting with textures by using an arithmetic dyadic approach [S3, unofficial]. This method provides better solutions than taking a difference between front and back depth maps, which can contain artifacts [S3, unofficial]. On a Geforce 6, this approach exceeds 200 Hz for models with 60,000 triangles [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(C \cdot H^3)$ for the packing step, as every voxel must be visited once [S1, S2, unofficial].
*   **Memory Complexity:** The output bitfield uses $1/8$ the memory of the input float grid [S1, S2, unofficial].
*   **Performance:** The packing itself is a simple linear pass. The benefit is realized during ray marching, where empty-space skipping reduces the number of density evaluations [S1, S2, unofficial].
*   **Strengths:** Significant memory reduction; fast empty-space skipping during rendering [S1, S2, unofficial].
*   **Limitations:** Loss of density information (binary only); requires even grid dimensions [S1, S2, unofficial].
*   **When to use:** When the volume has large empty regions and binary occupancy is sufficient for culling [S1, S2, unofficial].
*   **When not to use:** When partial volume effects or smooth density gradients are required for the culling logic [G].

## Background
Density Bitfield Packing is a technique used in GPU-based volume rendering and neural radiance fields (NeRF) to optimize ray marching performance [S1, S2, unofficial]. It is often implemented as a custom CUDA kernel or PyTorch function to leverage parallel processing [S1, S2, unofficial]. The concept of using bitfields for occupancy is related to solid voxelization techniques, where bitwise operations are used for efficient spatial queries [S3, unofficial].

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S2] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S3] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
