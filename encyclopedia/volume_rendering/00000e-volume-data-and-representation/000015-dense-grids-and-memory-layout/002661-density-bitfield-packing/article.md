# Density Bitfield Packing

Also called: packbits

## Overview
Density Bitfield Packing compresses a floating-point density grid into a binary bitfield to accelerate volume ray marching by enabling efficient empty-space skipping [S1, S2, unofficial]. It differs from sibling methods like [Evaluation Cache](874-evaluation-cache.md) or [Bricking](914-bricking.md) by operating at the voxel level to create a compact spatial index of occupied vs. empty regions, rather than caching shader results or organizing data into hierarchical bricks [S1, S2, unofficial].

## How it works
**Inputs:**
*   `grid`: A floating-point density tensor of shape $[C, H, H, H]$ (flattened to $[C, H^3]$ in some implementations), where $C$ is the number of channels and $H$ is the spatial dimension. $H$ must be even ($H \% 2 == 0$) [S1, S2, unofficial].
*   `thresh`: A scalar float threshold used for binarization [S1, S2, unofficial].

**Outputs:**
*   `bitfield`: A uint8 tensor of shape $[C, H^3 / 8]$ (or $[C, H \times H \times H / 8]$) [S1, S2, unofficial].

**Data Structures:**
The output `bitfield` stores one bit per voxel. For each channel $c$, the $k$-th bit of the byte at index $\lfloor k/8 \rfloor$ corresponds to the voxel at linear index $k$ in the flattened grid. If the density value at voxel $k$ is greater than `thresh`, the bit is set to 1; otherwise, it is 0 [S1, S2, unofficial].

**Algorithm:**
1.  Ensure the input `grid` is a contiguous tensor in memory (e.g., a contiguous CUDA tensor) [S1, unofficial].
2.  Calculate the total number of output bytes $N$ for a single channel:
    $$ N = \frac{H^3}{8} $$
3.  Iterate over each channel $c$ from $0$ to $C-1$:
    a. Initialize the output bitfield slice for channel $c$ to zero.
    b. Iterate over each voxel index $k$ from $0$ to $H^3 - 1$:
        i. Retrieve density value $d = \text{grid}[c, k]$.
        ii. Determine the byte index $b = \lfloor k / 8 \rfloor$ and the bit offset $o = k \pmod 8$.
        iii. If $d > \text{thresh}$, set the $o$-th bit of byte $b$ in the bitfield to 1.
        iv. Otherwise, leave the bit as 0.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def packbits(grid: torch.Tensor, thresh: float) -> torch.Tensor:
    """
    Packs a floating-point density grid into a bitfield.
    
    Args:
        grid: torch.Tensor of shape [C, H, H, H] or [C, H^3]. dtype float32.
        thresh: float threshold for binarization.
        
    Returns:
        torch.Tensor of shape [C, H^3 / 8]. dtype uint8.
    """
    # Ensure contiguous memory layout [S1, unofficial]
    if not grid.is_contiguous():
        grid = grid.contiguous()
        
    # Determine shape
    # Handle both [C, H, H, H] and [C, H^3] inputs
    if grid.dim() == 4:
        C, H, _, _ = grid.shape
        grid_flat = grid.view(C, -1)
    else:
        C, H3 = grid.shape
        H = int(math.pow(H3, 1/3))
        grid_flat = grid
        
    # Validate H is even [S1, S2, unofficial]
    if H % 2 != 0:
        raise ValueError("H must be even for bitfield packing.")
        
    # Calculate output size [S1, S2, unofficial]
    N = H3 // 8
    bitfield = torch.zeros((C, N), dtype=torch.uint8, device=grid.device)
    
    # Binarize and pack [S1, S2, unofficial]
    # Create a mask where density > thresh
    mask = (grid_flat > thresh).to(torch.uint8) # Shape [C, H^3]
    
    # Reshape mask to [C, H^3/8, 8] to group bits
    mask_reshaped = mask.view(C, N, 8)
    
    # Convert bits to byte values
    # Bit 0 is LSB, Bit 7 is MSB (or vice versa depending on convention, 
    # standard bit packing usually aligns index k with bit k%8)
    # We compute the byte value by shifting bits to their correct positions
    positions = torch.arange(8, device=grid.device, dtype=torch.uint8)
    packed = (mask_reshaped << positions).sum(dim=2)
    
    bitfield = packed.to(torch.uint8)
    
    return bitfield
```

**Edge Cases and Pitfalls:**
*   **Precision:** The threshold comparison is strict ($>$). Values exactly equal to `thresh` are treated as empty (0) [S1, S2, unofficial].
*   **Boundary:** The method assumes $H$ is even. If $H$ is odd, the last voxel cannot be packed into a full byte without padding or special handling, which is not described in the sources [S1, S2, unofficial].
*   **Memory:** The input grid must be contiguous. Non-contiguous tensors may lead to incorrect packing if not explicitly flattened or copied [S1, unofficial].

## Subtopics

### Bitwise Density Computation
Calculates the number of filled voxels between a scene point and a light position to simulate translucency, avoiding expensive bit counting with textures by using an arithmetic dyadic approach [S3, unofficial]. This method is used in conjunction with solid voxelization for translucency effects [S3, unofficial]. It provides better solutions than taking a difference between front and back depth maps, which can contain artifacts [S3, unofficial]. On a Geforce 6, this approach exceeds 200 Hz framerate for models with 60,000 triangles [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(C \cdot H^3)$ for the packing process, as every voxel must be visited and compared against the threshold [S1, S2, unofficial].
*   **Memory Complexity:** The output bitfield uses $1/8$ the memory of the input float grid (assuming 8-bit floats per voxel) [S1, S2, unofficial].
*   **Performance:** The packing itself is a preprocessing step. The runtime benefit is realized during ray marching by skipping empty space. Specific ray marching performance metrics are not provided for the packing method itself, but related bitwise density computations achieve >200 Hz on older hardware (Geforce 6) for complex scenes [S3, unofficial].
*   **Strengths:** Significant reduction in memory bandwidth during ray traversal; simple to implement; effective for sparse or semi-sparse volumes [S1, S2, unofficial].
*   **Limitations:** Requires even spatial dimensions; threshold selection is critical and may require tuning; does not handle partial volume information (binary only) [S1, S2, unofficial].

## Background
Density Bitfield Packing is a technique used in modern neural rendering pipelines, specifically in implementations like ER-NeRF and IDDR-NGP, to optimize volume rendering performance [S1, S2, unofficial]. It builds on the concept of solid voxelization, where bitwise operations are used to accelerate spatial queries and shading calculations, such as translucency and CSG operations [S3, unofficial]. The specific implementation details (CUDA kernel, PyTorch custom function) are found in unofficial repositories [S1, S2, unofficial].

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S2] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S3] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
