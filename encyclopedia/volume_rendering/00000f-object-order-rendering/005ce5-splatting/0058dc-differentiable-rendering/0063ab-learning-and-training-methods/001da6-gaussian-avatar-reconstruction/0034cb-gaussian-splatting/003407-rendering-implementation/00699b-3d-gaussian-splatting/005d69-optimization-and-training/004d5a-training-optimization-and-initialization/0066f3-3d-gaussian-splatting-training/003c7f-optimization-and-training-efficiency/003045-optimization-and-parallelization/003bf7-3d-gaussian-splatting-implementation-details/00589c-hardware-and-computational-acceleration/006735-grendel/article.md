# Grendel

Also called: distributed 3DGS training system

## Overview
Grendel is a distributed training system for 3D Gaussian Splatting (3DGS) that partitions parameters and parallelizes computation across multiple GPUs to overcome single-GPU memory constraints in high-resolution and large-scale reconstruction [S1]. It distinguishes itself from single-GPU baselines and divide-and-conquer approaches like CityGS by supporting batched training with multiple views and exploiting mixed parallelism (Gaussian-wise and pixel-wise) with sparse all-to-all communication to handle dynamic workload imbalances [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of training images (batched), initial 3D Gaussian parameters (position, covariance, opacity, spherical harmonics coefficients), and optimizer states.
*   **Outputs:** Optimized 3D Gaussian parameters and rendered images.

**Data Structures:**
*   **Gaussian Parameters:** Distributed uniformly across $G$ GPUs. Each GPU stores a subset of Gaussians, including their parameters and optimizer states (Adam moments) [S1].
*   **Pixel Blocks:** Images are partitioned into $16 \times 16$-pixel blocks [S1].
*   **Runtime Estimates:** A history of rendering times per pixel block is maintained to estimate computational load [S1].

**Numbered Steps:**

1.  **Gaussian-wise Distribution (Forward):**
    *   Partition the 3D Gaussians (parameters and optimizer states) uniformly across $G$ GPUs [S1].
    *   Each GPU independently computes the Gaussian transformation for its assigned subset of Gaussians [S1].
    *   *Note:* Computation load per Gaussian is assumed uniform, so uniform distribution is sufficient for this step [S1].

2.  **Sparse All-to-All Communication (Forward):**
    *   Each GPU determines the set of Gaussians that intersect with its assigned pixel partition [S1].
    *   Perform sparse all-to-all communication to transfer only the intersecting Gaussians to the GPUs responsible for rendering those pixels [S1].
    *   *Optimization:* Assign contiguous image areas to GPUs during pixel-wise distribution to exploit spatial locality and minimize the number of Gaussians transferred [S1].

3.  **Pixel-wise Distribution (Rendering & Loss):**
    *   Partition each image in the batch into $16 \times 16$-pixel blocks [S1].
    *   Distribute consecutive subsequences of blocks to different GPUs using an adaptive strategy based on estimated runtime [S1].
    *   Each GPU renders its assigned blocks and computes the loss [S1].
    *   *Note:* A GPU may be assigned blocks from different images in the batch [S1].

4.  **Sparse All-to-All Communication (Backward):**
    *   Perform a reversed all-to-all communication to transfer gradients back to the GPUs holding the corresponding Gaussian parameters [S1].

5.  **Optimizer Update:**
    *   Each GPU updates the parameters and optimizer states for the Gaussians it owns [S1].

6.  **Dynamic Load Balancing (Pixel-wise):**
    *   After the first few epochs, record the rendering time (including loss and backward computation) for each pixel block [S1].
    *   Compute the average per-pixel computation time for each GPU based on its assigned blocks [S1].
    *   Re-split images in subsequent iterations so that the sum of computation times for pixels assigned to all GPUs is equal, using $16 \times 16$ blocks as the split granularity [S1].
    *   Use **Algorithm 1** to calculate division points for rebalancing.

7.  **Dynamic Load Balancing (Gaussian-wise):**
    *   Redistribute 3D Gaussians uniformly after every few densification steps (cloning and splitting) to restore uniformity, as new Gaussians are added at varying rates depending on local scene details [S1].

**Pseudocode: Algorithm 1 Calculation of Division Points**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_division_points(estimated_runtimes: list, num_gpus: int) -> list:
    """
    Calculate division points to split pixel blocks among GPUs for load balancing.
    
    Args:
        estimated_runtimes: List of estimated runtimes for each pixel block (ET).
        num_gpus: Number of GPUs (G).
        
    Returns:
        DP: List of division points (indices) to split the blocks.
    """
    import torch
    
    # 1: CT <- TORCH.CUMSUM(ET)
    cumulative_times = torch.cumsum(torch.tensor(estimated_runtimes), dim=0)
    
    # 2: ETgpu <- CT[B-1]/G
    total_time = cumulative_times[-1]
    target_time_per_gpu = total_time / num_gpus
    
    # 3: TH <- TORCH.ARANGE(0, G) * ETgpu
    thresholds = torch.arange(0, num_gpus) * target_time_per_gpu
    
    # 4: DP <- TORCH.SEARCHSORTED(CT, TH)
    # Searchsorted returns indices where thresholds would be inserted to maintain order.
    # These indices represent the division points between GPU assignments.
    division_points = torch.searchsorted(cumulative_times, thresholds)
    
    # 5: return DP
    return division_points.tolist()
```

**Formulas:**
*   **Learning Rate Scaling:**
    $$ \lambda' = \lambda \times \sqrt{\text{batch\_size}} $$
    Where $\lambda$ is the original learning rate and $\lambda'$ is the adjusted learning rate [S1].
*   **Momentum Scaling (Adam):**
    $$ \beta_1' = \beta_1^{\text{batch\_size}}, \quad \beta_2' = \beta_2^{\text{batch\_size}} $$
    Where $\beta_1, \beta_2$ are the original first and second moment estimates, and $\beta_1', \beta_2'$ are the adjusted parameters [S1].

**Parameters:**
*   **Block Size:** $16 \times 16$ pixels [S1].
*   **Maximum Tested Batch Size:** 32 [S1].
*   **Learning Rate Scaling Rule:** $\sqrt{\text{batch\_size}}$ [S1].

**Edge Cases and Pitfalls:**
*   **Hyperparameter Stability:** Increasing batch size without adjusting hyperparameters leads to unstable and inefficient training [S1]. The Independent Gradients Hypothesis assumes gradients from different views are independent, but cameras with similar poses offer similar gradients, causing correlation and sparsity changes as batch size increases [S1].
*   **Workload Imbalance:** Static workload partitioning is suboptimal due to dynamic and unbalanced workloads caused by varying Gaussian densities and scene complexity (e.g., sky vs. detailed foreground) [S1].

## Subtopics

### DeSplat
A method for differentiable rendering that complements distributed training by optimizing the splatting process itself. [DeSplat](12925-desplat.md)

### 360 Gaussian Splatting
Focuses on spherical scene representation, which may interact with distributed partitioning strategies. [360 Gaussian Splatting](12929-360-gaussian-splatting.md)

### Data Compression and Storage Optimization
Reduces memory footprint, potentially easing the burden on distributed GPU memory. [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)

## Operating characteristics
*   **Time Complexity:** Training time scales with the number of Gaussians and images. Grendel achieves near-linear speedup in rendering speed with added GPUs.
*   **Memory Complexity:** Memory usage per GPU is reduced by distributing Gaussians and pixel blocks.
*   **Measured Performance:**
    *   **Rubble (3436x4591, 40.4M Gaussians):** 21.7 img/s on 4 GPUs vs. 8.8 img/s on 1 GPU. Training time on 4 A100s: 51 minutes vs. CityGS Official's 2 hours 3 minutes [S1]. Test PSNR: 27.28 (16 GPUs) vs. 26.28 (1 GPU, 11.2M Gaussians) [S1].
    *   **Matrixcity Block-all (1080x1920, 24M Gaussians):** 76.8 img/s on 4 GPUs vs. 29.2 img/s on 1 GPU. Training time on 4 A100s: 54 minutes vs. CityGS Official's 3 hours 31 minutes [S1].
    *   **Bicycle (1275x1920):** 80.3 img/s on 4 GPUs vs. 42.6 img/s on 1 GPU. Training time on 4 A100s: 1 hour 13 minutes vs. CityGS Official's 1 hour 20 minutes [S1].
*   **Strengths:** Handles high-resolution and large-scale scenes; supports batched training for better GPU utilization; dynamic load balancing handles scene complexity variations.
*   **Limitations:** Communication overhead from sparse all-to-all operations; requires careful hyperparameter scaling for large batches.
*   **When to Use:** Large-scale, high-resolution 3D reconstruction tasks that exceed single-GPU memory limits.
*   **When Not to Use:** Small-scale scenes where single-GPU training is sufficient and simpler to implement.

## Background
Grendel was introduced in the paper "On Scaling Up 3D Gaussian Splatting Training" (ICLR 2025) by researchers from NYU Systems [S1]. It addresses the limitation of 3DGS training being confined to a single GPU, which restricts the resolution and scale of reconstructible scenes [S1]. The system is open-source and available at https://github.com/nyu-systems/Grendel-GS [S1].

## Sources
[S1] On Scaling Up 3D Gaussian Splatting Training, https://doi.org/10.1007/978-3-031-91989-3_2, fetched 2026-10-26
