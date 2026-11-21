# FlashGS
## Overview
FlashGS is a multi-GPU rendering method for 3D Gaussian Splatting that accelerates the forward pass by partitioning the screen into finer tiles and scheduling them on a multi-GPU pipeline [S1, unofficial].
## How it works
FlashGS extends the tile-based rasterization with radix sort pipeline by distributing tile processing across multiple GPUs [Tile-based Rasterization with Radix Sort](7921-tile-based-rasterization-with-radix-sort.md). The method operates as follows:
1. **Tile Partitioning:** The screen is divided into a grid of tiles with finer granularity than standard single-GPU implementations [S1, unofficial].
2. **Multi-GPU Scheduling:** The partitioned tiles are distributed and scheduled across available GPUs for parallel execution [S1, unofficial].
3. **Local Rasterization:** Each GPU processes its assigned tiles using the standard tile-based rasterization pipeline [Tile-based Rasterization with Radix Sort](7921-tile-based-rasterization-with-radix-sort.md). This involves projecting Gaussians, assigning them to tiles, packing 64-bit sort keys (tile ID in high bits, depth in low bits), performing a global radix sort, and executing front-to-back compositing per tile [Tile-based Rasterization with Radix Sort](7921-tile-based-rasterization-with-radix-sort.md).
4. **Gradient Computation:** The same sorted lists are reused for backpropagation to compute gradients for position, shape, opacity, and color without additional sorting [Tile-based Rasterization with Radix Sort](7921-tile-based-rasterization-with-radix-sort.md).
No specific pseudocode is provided in the sources for the scheduling logic itself [S1, unofficial].
## Subtopics

### [System Level Primitive Sharding](15077-system-level-primitive-sharding.md)
A distributed strategy that partitions the Gaussian model across GPUs rather than the screen. It assigns Gaussians via index parity and distributes cameras, eliminating boundary waste and merge steps. Choose this for city-scale scenes with hundreds of millions of primitives where model size exceeds single-GPU memory. FlashGS focuses on screen partitioning for rendering acceleration.

## Operating characteristics
FlashGS accelerates the forward process by leveraging multiple GPUs [S1, unofficial]. Specific time and memory complexity figures are not given in the sources [S1, unofficial]. It is designed for high-performance rendering on multi-GPU setups [S1, unofficial].
## Background
FlashGS was introduced in the context of surveying 3D Gaussian Splatting rasterization techniques [S1, unofficial]. It is cited alongside other optimization methods like gsplat and Mobile-GS [S1, unofficial].
## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
