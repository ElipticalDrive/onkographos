# Grendel-GS
Also called: Distributed 3DGS Optimization, Multi-GPU 3D Gaussian Splatting Training
## Overview
Grendel-GS is an out-of-core and distributed optimization method for training 3D Gaussian Splatting that partitions the Gaussian list across multiple GPUs [S1, unofficial]. It solves the memory bottleneck of large-scale scenes by allowing each GPU to render its own subset of Gaussians and combining results via inter-GPU communication at partition boundaries [S1, unofficial]. Unlike [TideGS](7350-tidegs.md), which operates on a single machine using disk as a cache, Grendel-GS distributes the parameter store across a GPU cluster [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A large set of 3D Gaussians $\mathcal{G}$, camera poses $\mathcal{C}$, and training images $\mathcal{I}$.
- Output: Optimized Gaussian parameters (position, covariance, opacity, spherical harmonics coefficients).

**Data Structures:**
- The global Gaussian list $\mathcal{G}$ is partitioned into $K$ subsets $\mathcal{G}_1, \dots, \mathcal{G}_K$, one for each GPU $k \in \{1, \dots, K\}$.
- Each GPU $k$ maintains a local buffer containing $\mathcal{G}_k$.

**Steps:**
1. **Partitioning:** Distribute the total Gaussian primitives across $K$ GPUs. The partitioning strategy is not specified in the sources [G].
2. **Local Rendering:** For each training image, each GPU $k$ renders the contribution of its local subset $\mathcal{G}_k$ using standard 3DGS rasterization. This produces a partial color and depth buffer for that GPU.
3. **Inter-GPU Communication:** At partition boundaries, partial results are communicated between GPUs. The exact protocol (e.g., all-reduce, specific boundary handling) is not detailed in the sources [G].
4. **Composite Loss:** The final rendered image is formed by combining the partial renders. The loss is computed against the ground truth image.
5. **Gradient Update:** Gradients are computed and applied to the local Gaussian parameters on each GPU. Updates to Gaussians that may have moved or require synchronization across partitions are handled via the inter-GPU communication mechanism [S1, unofficial].

**Parameters:**
- Number of GPUs $K$: Not given in the sources.
- Partitioning scheme: Not given in the sources.

**Edge Cases:**
- Boundary Gaussians: Gaussians that span multiple partitions require careful handling during rendering and gradient update to ensure consistency. The sources do not specify the exact mechanism [G].
- Communication Overhead: Inter-GPU communication latency can become a bottleneck. The sources do not provide performance metrics for this [G].

## Subtopics

### [TideGS](7350-tidegs.md)
TideGS is a fully out-of-core optimization framework for training 3D Gaussian Splatting (3DGS) that enables training with over one billion primitives on a single GPU with limited VRAM.

## Operating characteristics
- **Memory:** Scales with the number of GPUs, allowing training on scenes with billions of Gaussians that exceed single-GPU VRAM [S1, unofficial].
- **Performance:** Dependent on inter-GPU communication bandwidth and the efficiency of the partitioning strategy. Not given in the sources.
- **Strengths:** Enables training of extremely large-scale scenes.
- **Limitations:** Requires multi-GPU infrastructure and complex synchronization. Not given in the sources.

## Background
Grendel-GS is introduced as a method for out-of-core and distributed optimization of 3D Gaussian Splatting [S1, unofficial]. It is cited in the context of scaling 3DGS to very large scenes, alongside other methods like CLM and TideGS [S1, unofficial]. The specific authors and publication date are not provided in the sources [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
