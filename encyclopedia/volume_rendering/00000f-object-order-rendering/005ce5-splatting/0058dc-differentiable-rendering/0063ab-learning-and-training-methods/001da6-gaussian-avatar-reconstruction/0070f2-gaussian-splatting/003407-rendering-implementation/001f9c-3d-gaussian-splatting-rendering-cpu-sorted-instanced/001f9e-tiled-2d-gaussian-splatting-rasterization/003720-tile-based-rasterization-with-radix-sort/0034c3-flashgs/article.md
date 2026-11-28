# FlashGS
## Overview
FlashGS is a multi-GPU rendering method for 3D Gaussian Splatting that accelerates the forward pass by partitioning the screen into finer tiles and scheduling them across a multi-GPU pipeline [S1, unofficial].
## How it works
FlashGS builds on [Tile-based Rasterization with Radix Sort](7921-tile-based-rasterization-with-radix-sort.md) by distributing the rasterization workload across multiple GPUs.
**Inputs:** $N$ projected 2D Gaussians, image resolution, tile grid dimensions, available GPU devices.
**Outputs:** Rendered image buffer, per-Gaussian gradients.
**Data structures:** Global list of $(\text{splat}, \text{tile})$ pairs; 64-bit sort keys; per-GPU tile assignment queues; per-tile sorted splat lists.
**Steps:**
1. **Finer Tile Partitioning:** The screen is divided into a grid of tiles with higher granularity than standard single-GPU implementations [S1, unofficial].
2. **Multi-GPU Scheduling:** The partitioned tiles are distributed and scheduled across available GPUs for parallel execution [S1, unofficial].
3. **Local Rasterization:** Each GPU processes its assigned tiles using the standard tile-based rasterization pipeline [Tile-based Rasterization with Radix Sort](7921-tile-based-rasterization-with-radix-sort.md). This covers projecting Gaussians, assigning them to tiles, packing 64-bit sort keys, performing a global radix sort, and executing front-to-back compositing per tile [S1, unofficial].
4. **Gradient Computation:** The same sorted lists are reused for backpropagation to compute gradients for position, shape, opacity, and color without additional sorting [S1, unofficial].
**Formulas:** Sort key packing follows the parent method: $\text{key} = (\text{tile\_id} \ll 32) \mid (\text{uint32\_t})Z_c$, where $\text{tile\_id}$ is the 1D tile index ($y \times \text{grid\_width} + x$) and $Z_c$ is the splat depth cast to a 32-bit unsigned integer [Sibling: Tile-based Rasteriser].
**Parameters:** Tile dimensions are finer than the standard $16 \times 16$ pixels, though exact values are not given in the sources.
**Edge cases:** GPU memory limits may require dynamic tile redistribution; synchronization overhead between GPUs must be minimized to maintain throughput [G].
**Pseudocode:** # No pseudocode provided in sources for the scheduling logic [S1, unofficial].
## Subtopics

### [System Level Primitive Sharding](15077-system-level-primitive-sharding.md)
A distributed strategy that partitions the Gaussian model across GPUs rather than the screen. It assigns Gaussians via index parity and distributes cameras, eliminating boundary waste and merge steps. Choose this for city-scale scenes with hundreds of millions of primitives where model size exceeds single-GPU memory. FlashGS focuses on screen partitioning for rendering acceleration.

## Operating characteristics
FlashGS accelerates the forward process by leveraging multiple GPUs [S1, unofficial]. Specific time and memory complexity figures are not given in the sources [S1, unofficial]. It is designed for high-performance rendering on multi-GPU setups [S1, unofficial].
## Background
FlashGS was introduced in the context of surveying 3D Gaussian Splatting rasterization techniques [S1, unofficial]. It is cited alongside other optimization methods like gsplat and Mobile-GS [S1, unofficial].
## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
