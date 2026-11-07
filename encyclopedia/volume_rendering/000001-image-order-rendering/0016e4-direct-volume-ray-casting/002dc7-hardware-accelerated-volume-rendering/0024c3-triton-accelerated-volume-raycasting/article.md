# Triton-accelerated Volume Raycasting

Also called: FlashDRR Triton Kernel, Triton-based Volume Raycasting, fused volume rendering

## Overview
Triton-accelerated Volume Raycasting is a fused GPU rendering technique that uses the Triton programming language to perform volume raycasting, avoiding the materialization of intermediate ray-sample tensors in High Bandwidth Memory (HBM) to reduce memory footprint and increase speed [S1, unofficial]. It sets itself apart from standard PyTorch implementations by fusing CUDA kernels, which reduces iteration time and memory overhead, particularly at high resolutions where standard execution degrades super-linearly due to memory bandwidth limits and cache thrashing [S1, unofficial].

## How it works
The method implements the emission-absorption optical model defined in [Direct volume ray casting](19-direct-volume-ray-casting.md). It fuses the ray generation, sampling, classification, and compositing steps into a single Triton kernel to eliminate intermediate global memory writes.

**Inputs and Outputs:**
*   **Input:** A 5D tensor representing the volume data (batch, channels, depth, height, width), a view matrix, and a coordinate transformation matrix (RAS to IJK) [S1, unofficial].
*   **Output:** Rendered image tensors (RGB, depth, opacity) [S1, S2, unofficial].
*   **Parameters:**
    *   `triton`: Boolean flag. If `True`, uses the Triton kernel; if `False`, uses the standard PyTorch implementation [S1, unofficial].
    *   `ray_samples`: Number of samples per ray (e.g., 384 or 256) [S1, unofficial].
    *   `resolution`: Output image resolution (e.g., 1024x1024, 2048x2048) [S1, unofficial].

**Data Structures:**
*   **Per-ray Registers:** Accumulators for RGB, depth, and opacity are stored in per-thread registers (stack-allocated) rather than global memory [S2, unofficial].
*   **Output Tensors:** Final accumulated values are written exactly once per ray to output tensors (`outRGB`, `outDepth`, `outOpacity`) [S2, unofficial].

**Algorithm Steps:**
1.  **Tile Generation:** The image is tiled over height and width. For each tile, the corresponding volume grid slice is extracted [S1, unofficial].
2.  **Ray Casting Loop:** For each pixel in the tile, a ray is generated. The ray is discretized into `ray_samples` steps [S1, unofficial].
3.  **Fused Sampling and Compositing:**
    *   At each sample point, the scalar volume value is interpolated (typically trilinear) [S1, S6, unofficial].
    *   The transfer function maps the scalar value to color $C_i$ and opacity $\alpha_i$ [S1, S2, unofficial].
    *   Accumulation occurs in per-thread registers:
        $$
        C'_{reg} = C'_{reg} + (1 - \alpha'_{reg}) \alpha_i C_i
        $$
        $$
        \alpha'_{reg} = \alpha'_{reg} + (1 - \alpha'_{reg}) \alpha_i
        $$
    *   This avoids reading and writing to global memory for every sample [S2, unofficial].
4.  **Final Write:** After the ray marching loop, the final accumulated RGB, depth, and opacity values are written from registers to the global output tensors exactly once [S2, unofficial].
5.  **Backward Pass (Training):** If training is enabled, gradients are computed. Note that the Triton kernel uses `tl.atomic_add`, which is order-nondeterministic [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import triton
import triton.language as tl

@triton.jit
def triton_volume_kernel(
    volume_ptr, view_mat_ptr, ras2ijk_ptr,
    out_rgb_ptr, out_depth_ptr, out_opacity_ptr,
    H, W, D, ray_samples,
    BLOCK_H: tl.constexpr, BLOCK_W: tl.constexpr
):
    # Tile coordinates
    pid = tl.program_id(axis=0)
    num_tiles = tl.cdiv(H, BLOCK_H)
    tile_idx = pid // num_tiles
    row_idx = pid % num_tiles
    
    # Block offsets
    y_off = tl.arange(0, BLOCK_H)
    x_off = tl.arange(0, BLOCK_W)
    
    # Determine pixel coordinates for this block
    # Note: Simplified indexing; real implementation handles bounds and stride
    y = tile_idx * BLOCK_H + y_off
    x = pid * BLOCK_W + x_off # Simplified for single tile logic
    
    # Initialize per-thread registers
    acc_rgb = tl.zeros((3,), dtype=tl.float32)
    acc_alpha = tl.float32(0.0)
    acc_depth = tl.float32(0.0)
    
    # Ray generation and marching loop
    # ... (Ray generation logic using view_mat and ras2ijk) ...
    
    for s in range(ray_samples):
        # Sample volume at current ray position
        # Interpolate scalar value V
        V = sample_trilinear(volume_ptr, ray_pos)
        
        # Apply transfer function to get C_i, alpha_i
        C_i, alpha_i = apply_transfer_function(V)
        
        # Update registers
        transmittance = 1.0 - acc_alpha
        acc_rgb = acc_rgb + transmittance * alpha_i * C_i
        acc_alpha = acc_alpha + transmittance * alpha_i
        acc_depth = acc_depth + transmittance * alpha_i * ray_t
        
        # Early termination if fully opaque
        if acc_alpha >= 1.0:
            break
            
    # Write final results to global memory exactly once
    tl.store(out_rgb_ptr, acc_rgb)
    tl.store(out_depth_ptr, acc_depth)
    tl.store(out_opacity_ptr, acc_alpha)
```

**Edge Cases and Pitfalls:**
*   **Determinism:** The use of `tl.atomic_add` (if used for gradient accumulation or specific parallel reductions) introduces order-nondeterminism. Gradient checks require looser tolerances (`atol=1e-4`, `rtol=1e-3`) and `nondet_tol=1e-6` [S1, unofficial].
*   **Precision:** The method supports both fp32 and fp16/bf16. Matches between Triton and standard PyTorch are verified with `atol=1e-4` [S1, unofficial].

## Subtopics

### Per-ray register accumulation for volume rendering
A forward pass optimization where RGB, depth, and opacity are accumulated in per-thread registers (stack-allocated) rather than global memory, writing each per-ray output exactly once to eliminate per-sample global-memory read-modify-write operations [S2, unofficial]. It differs from the baseline by eliminating per-sample global-memory read-modify-write on `outRGB`/`outDepth`/`outOpacity`, reducing global memory traffic [S2, unofficial]. It is chosen when global memory bandwidth is the bottleneck.

## Operating characteristics
*   **Time Complexity:** Triton maintains predictable linear scaling proportional to total rays rendered [S1, unofficial]. Standard PyTorch execution time degrades super-linearly beyond 1024x1024 resolution [S1, unofficial].
*   **Memory Complexity:** Peak VRAM usage scales to a near-constant footprint below 430 MiB, achieving a 99.5% memory reduction at high workloads compared to standard PyTorch which scales linearly with total samples (Rays × Samples/Ray) [S1, unofficial].
*   **Measured Performance:**
    *   At 2048x2048 resolution with 256 samples: Triton iteration time is 9.16 ms vs. 1,804.32 ms for standard PyTorch (197x speedup) [S1, unofficial].
    *   At 1024x1024 resolution with 512 samples: Triton evaluation memory is 110.7 MiB vs. ~12.4 GB for standard PyTorch [S1, unofficial].
    *   At 1024x1024 resolution with 512 samples: Triton training step time is 15.4 ms [S1, unofficial].
*   **Strengths:** Significant reduction in memory footprint and iteration time at high resolutions; avoids cache thrashing [S1, unofficial].
*   **Limitations:** Order-nondeterminism in gradients requires adjusted tolerance settings; requires Triton runtime support [S1, unofficial].

## Background
The method originates from the FlashDRR project, which implements a fused GPU rendering technique using Triton to perform volume raycasting [S1, unofficial]. It was developed to address the memory bandwidth limits and cache thrashing issues inherent in standard PyTorch-based volume rendering implementations at high resolutions [S1, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S2] Optimizations for `volume_render` and move its autograd layer to Python, https://github.com/openvdb/fvdb-core/pull/640, fetched 2026-10-27 (unofficial)
