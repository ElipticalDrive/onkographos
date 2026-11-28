# MPI-hybrid Raycasting

Also called: hybrid MPI-raycasting

## Overview
MPI-hybrid raycasting is a parallel volume rendering method that combines distributed-memory parallelism (MPI) with shared-memory threading to optimize ray integration performance. It solves the problem of suboptimal scaling in pure MPI implementations by aggregating distributed data blocks into larger rectangular geometries and processing them with multiple threads per MPI process [S1, S2]. Unlike pure MPI approaches where each process handles a single raycasting task, this method multi-threads the raycasting step within each task using image-parallel decomposition, significantly improving performance at lower concurrencies and reducing memory footprint [S1, S2].

## How it works
**Inputs:**
- Distributed volume dataset decomposed into blocks.
- Camera parameters (origin, direction, field of view).
- Transfer function for mapping scalar values to color and opacity.
- Interpolation method (trilinear).
- Rendering target (image buffer).

**Outputs:**
- Final composited image.
- Raycasting time metrics.

**Data Structures:**
- **Block Geometry:** The volume is decomposed into rectangular blocks formed by aggregating $1 \times 2 \times 3$ cubic blocks. These rectangular blocks are longest in the z-direction [S1, S2].
- **Thread Pool:** Each MPI Process Element (PE) launches six raycasting threads [S1, S2].
- **Ray Integration State:** For each ray, state includes current position, accumulated color, accumulated opacity, and step size [S2].

**Steps:**
1. **Decomposition:** The volume is partitioned into rectangular blocks using the $1 \times 2 \times 3$ aggregation scheme [S1, S2].
2. **Data Distribution:** Each MPI PE obtains its assigned data block. Ghost data exchange is minimized by this geometry [S2].
3. **Thread Launch:** Each MPI PE launches six raycasting threads [S1, S2].
4. **Ray Tracing (Per Thread):**
   - Each thread independently processes a subset of rays [S2].
   - For each ray, test for intersection with the data block's bounding box [S2].
   - If a hit occurs, integrate the ray by sampling data values at a fixed interval along the ray [S2].
   - Apply trilinear interpolation for data sampling [S2].
   - Apply the transfer function to sampled values to determine color and opacity [S2].
   - Aggregate color and opacity using front-to-back compositing for the ray's image position [S2].
5. **Synchronization:** The final raycasting time is determined by the thread that takes the most integration steps [S2].
6. **Compositing:** Results from all threads and PEs are composited to form the final image [S2].

**Formulas:**
- **Ray Sampling:** $P(t) = O + t \cdot D$, where $O$ is ray origin, $D$ is direction, $t$ is distance [S2].
- **Data Sampling:** $V = \text{TrilinearInterpolation}(P(t))$ [S2].
- **Transfer Function:** $(C, \alpha) = \text{TF}(V)$ [S2].
- **Front-to-Back Compositing:**
  $$ C_{out} = C_{in} + (1 - \alpha_{in}) \cdot C \cdot \alpha $$
  $$ \alpha_{out} = \alpha_{in} + (1 - \alpha_{in}) \cdot \alpha $$
  where $(C_{in}, \alpha_{in})$ is the accumulated color and opacity, and $(C, \alpha)$ is the current sample's color and opacity [S2].

**Parameters:**
- **Aggregation Factor:** $1 \times 2 \times 3$ cubic blocks [S1, S2].
- **Threads per PE:** 6 [S1, S2].
- **Interpolation:** Trilinear [S2].
- **Shader:** Phong-style [S2].

**Edge Cases and Pitfalls:**
- **Performance Scaling Limit:** Scaling is limited by the maximum number of ray integration steps, which is determined by the interaction of decomposition geometry and camera direction [S1, S2].
- **Concurrency Dependency:** At lower concurrencies, MPI-hybrid outperforms MPI-only by up to 11%. At higher concurrencies, the trend flips, and MPI-only outperforms MPI-hybrid by 10% [S1, S2].
- **Ideal Scaling:** Identical scaling to MPI-only is expected if an eight-core system with a $2 \times 2 \times 2$ aggregation factor were used [S1, S2].

## Subtopics

### [Hybrid Parallel Volume Rendering](21181-hybrid-parallel-volume-rendering.md)
A parallel programming model that combines distributed-memory parallelism (MPI) for inter-node communication with shared-memory parallelism (threads/GPU) for intra-node computation. It solves the problem of diminishing returns and high communication costs associated with extreme concurrency in pure MPI implementations by reducing the number of MPI participants. Unlike pure MPI approaches where raycasting runs serially on each task, this method multi-threads the raycasting step within each task using image-parallel decomposition, significantly improving performance, reducing memory footprint, and lowering communication bandwidth requirements.

## Operating characteristics
- **Time Complexity:** Raycasting phase scales nearly linearly because it involves no inter-processor communication [S2].
- **Memory Complexity:** Requires less exchange and storage of ghost data by roughly 40% across all concurrencies compared to perfectly cubic distributed blocks [S2].
- **Measured Performance:**
  - Linear scaling up to 216,000 cores for the raycasting phase with MPI-only [S1, S2].
  - At lower concurrencies, MPI-hybrid outperforms MPI-only by as much as 11% [S1, S2].
  - At higher concurrencies, MPI-only outperforms MPI-hybrid by 10% [S1, S2].
  - At 216,000 cores, raycasting is less than 20% of the total runtime [S2].
  - MPI-hybrid is over 50% faster than MPI-only at 216,000 cores due to gains in the compositing phase [S2].
- **Strengths:** Reduced ghost data storage/exchange, better performance at lower concurrencies, lower memory footprint.
- **Limitations:** Performance scaling limited by ray integration steps (geometry/camera interaction), potential underperformance at high concurrencies compared to pure MPI.
- **When to Use:** When running on systems with moderate concurrency where communication overhead is significant, or when memory bandwidth for ghost data is a bottleneck.
- **When Not to Use:** At very high concurrencies where pure MPI may outperform, or when the camera direction interacts poorly with the $1 \times 2 \times 3$ geometry.

## Background
- **Origin:** Hybrid Parallelism for Volume Rendering on Large, Multi-core Systems [S1, S2].
- **History:** Developed to address the limitations of pure MPI volume rendering, specifically the high communication costs and diminishing returns at extreme concurrency. The method leverages the hybrid memory decomposition strategy to optimize block geometry for raycasting efficiency.
- **Usage:** Used in large-scale scientific visualization on multi-core systems, particularly where distributed memory parallelism is combined with shared-memory threading for improved performance.

## Sources
[S1] Hybrid Parallelism for Volume Rendering on Large, Multi-core Systems, https://digital.library.unt.edu/ark:/67531/metadc1014417/?q=scientific volume visualization, fetched 2026-10-28
[S2] Hybrid Parallelism for Volume Rendering on Large, Multi-core Systems, https://digital.library.unt.edu/ark:/67531/metadc1014428/?q=scientific volume visualization, fetched 2026-10-28
