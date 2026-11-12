# Progressive Downsampling


## Overview
Progressive downsampling generates a hierarchy of volume representations at decreasing spatial resolutions to accelerate rendering and enable interactive exploration. It solves the problem of high computational cost in dense volume rendering by allowing the renderer to start with a coarse approximation and gradually refine the image. It is distinguished from sibling methods like [Mipmap Generation](2602-mipmap-generation.md) by its explicit iterative refinement strategy, often using logarithmic decay to balance speed and accuracy during the rendering process.

## How it works
**Inputs:** Original volume dataset $V$ of size $N \times N \times N$; target resolution; iteration limit.
**Outputs:** A sequence of downsampled volumes $V_0, V_1, \dots, V_k$ where $V_0 = V$ and $V_i$ has lower resolution than $V_{i-1}$.

**Data Structures:**
The method typically employs a pyramid structure. For an original dataset of $N^3$ samples, the pyramid consists of $\log_2 N$ volumes [S2, unofficial].
- $V_0$: Original data.
- $V_i$: Created by averaging $2 \times 2 \times 2$ groups of samples from $V_{i-1}$ [S2, unofficial].
- Each level $i$ has linear resolution $N / 2^i$.

**Steps:**
1. **Initial Coarse Rendering:** Downsample the original image or volume by a factor of 8 initially to establish a baseline [S1].
2. **Iterative Refinement:** Gradually increase the resolution back to the original. The resolution increase follows a logarithmic decay schedule until a specified iteration count (e.g., 19,500) [S1].
3. **Pyramid Construction (General Case):**
   - For each level $i$ from 1 to $\log_2 N$:
     - For each voxel $(x, y, z)$ in $V_i$:
       - $V_i[x, y, z] = \frac{1}{8} \sum_{dx=0}^{1} \sum_{dy=0}^{1} \sum_{dz=0}^{1} V_{i-1}[2x+dx, 2y+dy, 2z+dz]$ [S2, unofficial].
4. **Rendering Loop:**
   - Render at current resolution.
   - If iteration count < limit, refine resolution according to decay schedule.
   - Repeat.

**Parameters:**
- Initial downsampling factor: 8x [S1].
- Logarithmic decay rate: Applied to resolution increase [S1].
- Iteration limit: 19,500 (example from specific implementation) [S1].

**Edge Cases:**
- **Boundary Conditions:** When $N$ is not a power of 2, padding or truncation is required before downsampling.
- **Precision:** Averaging operations may introduce floating-point errors; use double precision for intermediate calculations if necessary.

## Subtopics

### Bit-Packed Binary Rendering
An optimization for binary sample values where bits are packed into bytes to generate lower resolution images faster [S2, unofficial]. It processes data bit by bit for full resolution, but byte by byte for lower resolution [S2, unofficial]. If more than four bits of a byte are set, the byte represents an object element; otherwise, it is background [S2, unofficial]. This produces an image with one-half the linear resolution in approximately one-eighth the time [S2, unofficial].

### CorrBlockFast1D
Constructs a pyramid of correlation volumes between two feature maps for efficient 1D correlation sampling [S3, unofficial]. It initializes with `fmap1`, `fmap2`, `num_levels` (default 4), and `radius` (default 4) [S3, unofficial]. It computes initial correlation, reshapes it, and downsamples using `F.avg_pool2d` with stride 2 for each level [S3, unofficial]. The `__call__` method samples correlation using `CorrSampler.apply` with coordinates scaled by $2^i$ and concatenates results [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Initial rendering is fast due to coarse resolution. Refinement time increases with resolution. Bit-packed binary rendering offers significant speedup (1/8th time for 1/2 resolution) [S2, unofficial].
- **Memory Complexity:** Pyramid structure requires $O(N^3)$ total memory for all levels, but only one level is typically active in memory at a time if streamed.
- **Performance:** Effective for interactive visualization where immediate feedback is prioritized over final accuracy [S2, unofficial].
- **Limitations:** May introduce artifacts if the decay schedule is too aggressive or if the volume has high-frequency details not captured by coarse levels.

## Background
The concept of building a pyramid data structure for volume visualization was introduced in early volume rendering literature [S2, unofficial]. It allows for efficient splatting and hierarchical culling. The specific application of progressive downsampling with logarithmic decay for Gaussian Splatting was detailed in recent works on efficient rendering [S1].

## Sources
[S1] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S3] YaourtB/GPS_plus, https://github.com/YaourtB/GPS_plus, fetched 2026-10-27 (unofficial)
