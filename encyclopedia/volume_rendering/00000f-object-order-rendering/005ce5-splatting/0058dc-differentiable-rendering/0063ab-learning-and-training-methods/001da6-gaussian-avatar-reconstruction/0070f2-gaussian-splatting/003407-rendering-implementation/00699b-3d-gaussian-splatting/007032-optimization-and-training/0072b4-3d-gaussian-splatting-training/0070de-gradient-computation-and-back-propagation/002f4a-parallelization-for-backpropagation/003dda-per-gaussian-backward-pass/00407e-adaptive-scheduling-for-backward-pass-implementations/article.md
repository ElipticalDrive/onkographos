# Adaptive Scheduling for Backward Pass Implementations
Also called: Rasterization backward with adaptive scheduling, Thompson sampling scheduler for backward pass

## Overview
This method minimizes atomic contention and maximizes thread utilization during the 3D Gaussian Splatting backward pass by dynamically switching between two distinct rasterization implementations [S1, unofficial]. It sets itself apart from static backward strategies by using a Thompson sampling scheduler to select the fastest implementation based on measured latency, addressing performance variance across different scenes and hardware configurations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: Projected Gaussians binned to tiles, tile pixel count $P$, Gaussian count $N$, subgroup size, and latency belief distributions [S1, unofficial].
*   Output: Accumulated gradients for Gaussian parameters [S1, unofficial].

**Data Structures:**
*   **Latency Beliefs:** Two Beta distributions storing the estimated latency performance for Implementation 1 and Implementation 2 [S1, unofficial].
*   **Shared Memory:** Used in Implementation 2 to store pre-computed transmittance and derivatives [S1, unofficial].

**Parameters:**
*   **Batch Size ($S$):** Chosen by rounding up $\sqrt{NP}$ to a multiple of subgroup size, capped at 128, where $N$ is the number of Gaussians and $P$ is the number of pixels in the tile [S1, unofficial].
*   **Gaussian Parameterization:** Follows [LDC* 25] to simplify opacity computation into matrix multiplication [S1, unofficial].

**Steps:**
1.  **Initialize Latency Beliefs:** Initialize Beta distributions for the latency of Implementation 1 and Implementation 2 [S1, unofficial].
2.  **Select Implementation:** Sample from both distributions and select the implementation with the highest sample value [S1, unofficial].
3.  **Execute Selected Implementation:**
    *   **If Implementation 1 is selected:**
        1.  Launch one thread block per tile [S1, unofficial].
        2.  Calculate dynamic Gaussian batch size $S = \min(128, \lceil \sqrt{NP} / \text{subgroup\_size} \rceil \times \text{subgroup\_size})$ [S1, unofficial].
        3.  Process Gaussians in batches of size $S$ for each tile [S1, unofficial].
    *   **If Implementation 2 is selected:**
        1.  For a Gaussian batch, run a forward pass parallelized over pixels [S1, unofficial].
        2.  Compute transmittance and its derivative with respect to Gaussian parameters for each Gaussian-pixel pair [S1, unofficial].
        3.  Store these values in shared memory [S1, unofficial].
        4.  Run a backward pass parallelized over Gaussians [S1, unofficial].
        5.  Fetch pre-computed transmittance and derivatives from shared memory [S1, unofficial].
        6.  Compute accumulated gradients [S1, unofficial].
4.  **Update Latency Beliefs:** Measure the actual latency of the selected implementation [S1, unofficial]. Update the corresponding Beta distribution parameters based on the measured latency [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random
import time

def select_implementation(latency_beliefs):
    """Select implementation using Thompson sampling."""
    samples = [random.betavariate(alpha, beta) for alpha, beta in latency_beliefs]
    return samples.index(max(samples))

def update_belief(latency_beliefs, selected_impl, latency):
    """Update latency belief based on measured latency."""
    # [G] Standard Bayesian update for minimization: lower latency increases confidence (alpha)
    alpha, beta = latency_beliefs[selected_impl]
    latency_beliefs[selected_impl] = [alpha + 1.0 / latency, beta]

def implementation_1(tile_gaussians, P, subgroup_size, max_batch_size):
    """Implementation 1: One thread block per tile, dynamic batch size."""
    N = len(tile_gaussians)
    if N == 0:
        return
    sqrt_np = math.sqrt(N * P)
    S = math.ceil(sqrt_np / subgroup_size) * subgroup_size
    S = min(S, max_batch_size)
    for i in range(0, N, S):
        batch = tile_gaussians[i:i+S]
        # [G] Process batch: compute gradients and atomically accumulate to global memory
        pass

def implementation_2(tile_gaussians, P, subgroup_size, max_batch_size):
    """Implementation 2: Forward pass over pixels, backward pass over Gaussians, shared memory."""
    N = len(tile_gaussians)
    if N == 0:
        return
    sqrt_np = math.sqrt(N * P)
    S = math.ceil(sqrt_np / subgroup_size) * subgroup_size
    S = min(S, max_batch_size)
    shared_memory = []
    for i in range(0, N, S):
        batch = tile_gaussians[i:i+S]
        # [G] Forward pass parallelized over pixels: compute transmittance and derivatives per Gaussian-pixel pair, store in shared_memory
        pass
    # [G] Backward pass parallelized over Gaussians: fetch from shared_memory, compute accumulated gradients
    pass

def run_adaptive_backward(tile_gaussians, P, subgroup_size, max_batch_size, latency_beliefs):
    """Main function to run adaptive backward pass."""
    selected_impl = select_implementation(latency_beliefs)
    start_time = time.perf_counter()
    if selected_impl == 0:
        implementation_1(tile_gaussians, P, subgroup_size, max_batch_size)
    else:
        implementation_2(tile_gaussians, P, subgroup_size, max_batch_size)
    end_time = time.perf_counter()
    latency = end_time - start_time
    update_belief(latency_beliefs, selected_impl, latency)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Tile boundaries are handled via branchless ellipse range functions during projection, not during scheduling [S1, unofficial].
*   **Degenerate Input:** If a tile has no Gaussians, both implementations return immediately [S1, unofficial].

## Subtopics

### [Adaptive View and Pixel Sampling Scheduling](15689-adaptive-view-and-pixel-sampling-scheduling.md)
A scheduling mechanism that jointly adjusts the tile-wise pixel sampling rate ($r_{tile}$) and the number of sampled views ($K$) over training iterations to allocate more computation during difficult stages and reduce it as optimization converges. It sets itself apart from static sampling strategies by dynamically scaling the view budget and pixel density based on reconstruction error and training progress, achieving a trade-off between training speed and reconstruction quality. When to choose it: Not given in the sources.

### [Cooperative Scheduling and Synchronization](18788-cooperative-scheduling-and-synchronization.md)
Cooperative Scheduling and Synchronization addresses idle resources and gradient contention caused by uneven tile lists, variable pixel traversal, and backward propagation accumulation. It sets itself apart from static mapping strategies by dynamically redistributing work and organizing computation to minimize global atomic updates, replacing them with local reductions or cache-cooperative operations. When to choose it: Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Implementation 1 latency is approximately proportional to $\frac{P+S}{P} \times \frac{N+S}{S}$ [S1, unofficial]. Implementation 2 complexity is not explicitly given but targets improved thread utilization and reduced divergence [S1, unofficial].
*   **Memory Complexity:** Implementation 2 requires shared memory for transmittance and derivatives, constraining batch or tile size based on hardware limits [S1, unofficial].
*   **Measured Performance:** On NVIDIA RTX 3090, Implementation 2 is 20%-30% faster on the Mip-NeRF 360 bicycle scene, while Implementation 1 is faster on the garden scene [S1, unofficial].
*   **Scaling:** Automatically adapts to different scenes and hardware via the Thompson sampling scheduler [S1, unofficial].
*   **Strengths:** Reduces atomic contention, maximizes thread utilization, and automatically selects the optimal implementation per context [S1, unofficial].
*   **Limitations:** Thompson sampling adds minimal overhead; gains depend on scene-specific performance variance [S1, unofficial].
*   **When to Use:** Training 3D Gaussian Splatting models where backward pass performance varies across scenes or hardware [S1, unofficial].
*   **When Not to Use:** When a single implementation is consistently faster across all scenarios, or scheduling overhead outweighs gains [S1, unofficial].

## Background
This method was introduced in "VkSplat: High-Performance 3DGS Training in Vulkan Compute" [S1, unofficial]. It addresses high atomic contention and subgroup reduction in existing 3DGS training implementations [S1, unofficial]. Implementation 1 is inspired by [MGK* 24], while the overall approach improves upon [KKLD23] and [YLK* 25] by dynamically selecting the best backward strategy [S1, unofficial].

## Sources
[S1] VkSplat: High-Performance 3DGS Training in Vulkan Compute, https://doi.org/10.2312/egs.20261024, fetched 2026-10-29 (unofficial)
