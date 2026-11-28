# Grendel

Also called: distributed 3DGS training system

## Overview
Grendel is a distributed training system for 3D Gaussian Splatting (3DGS) that partitions parameters and parallelizes computation across multiple GPUs to overcome single-GPU memory constraints in high-resolution and large-scale reconstruction [S1]. It distinguishes itself by exploiting mixed parallelism (Gaussian-wise and pixel-wise) and spatial locality to handle dynamic, unbalanced workloads, while supporting batched training with multiple camera views to improve GPU utilization [S1]. Unlike systems that rely on dense weight sharding or divide-and-conquer cell partitioning, Grendel uses sparse all-to-all communication to transfer only relevant Gaussians and employs dynamic load balancing to maintain efficiency [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians (parameters and optimizer states), a batch of $B$ training images (camera views), and the number of GPUs $G$.
*   **Outputs:** Optimized 3D Gaussian parameters and rendered images matching the input batch.

**Data Structures:**
*   **Gaussian Set:** Distributed uniformly across $G$ GPUs. Each GPU stores a subset of Gaussians, including their mean, covariance, opacity, and spherical harmonics coefficients, along with their Adam optimizer states (first and second moments) [S1].
*   **Pixel Blocks:** Each image is partitioned into non-overlapping $16 \times 16$ pixel blocks [S1]. These blocks are serialized into a list and distributed to GPUs [S1].

**Algorithm Steps:**

1.  **Gaussian-wise Distribution (Forward):**
    *   Partition the 3D Gaussians uniformly across the $G$ GPUs [S1].
    *   Each GPU independently computes the Gaussian transformation (projection to screen space) for its assigned subset of Gaussians [S1].

2.  **Sparse All-to-All Communication (Forward):**
    *   Each GPU determines the set of Gaussians from its local subset that intersect with the pixels in its assigned pixel partition [S1].
    *   Perform a sparse all-to-all communication where GPUs exchange the necessary Gaussians. A GPU sends Gaussians to other GPUs if they intersect with pixels assigned to those remote GPUs [S1].
    *   This step exploits spatial locality: each pixel partition only requires a small subset of all Gaussians, minimizing data transfer compared to dense sharding [S1].

3.  **Pixel-wise Rendering and Loss Computation:**
    *   Each GPU renders the pixels in its assigned partition using the received Gaussians and computes the loss [S1].
    *   Note: A GPU may be assigned blocks from different images in the batch [S1].

4.  **Sparse All-to-All Communication (Backward):**
    *   Perform a reversed all-to-all communication to distribute gradient updates back to the GPUs holding the original Gaussian parameters [S1].

5.  **Gaussian-wise Distribution (Backward/Update):**
    *   Each GPU updates the parameters and optimizer states for its locally stored Gaussians using the received gradients [S1].

6.  **Dynamic Load Balancing (Pixel-wise):**
    *   **Measurement:** After the first few epochs, record the rendering time (including loss and backward computation) for each $16 \times 16$ pixel block [S1].
    *   **Estimation:** Compute the average per-pixel computation time for each GPU based on its assigned blocks [S1].
    *   **Rebalancing:** In subsequent iterations, re-split the images into subsequences of blocks such that the sum of estimated computation times for pixels assigned to all GPUs is equal [S1].
    *   **Division Point Calculation:** Use Algorithm 1 to determine split points based on cumulative estimated runtimes [S1].

7.  **Dynamic Load Balancing (Gaussian-wise):**
    *   Monitor the densification process (cloning and splitting of Gaussians) [S1].
    *   Redistribute the 3D Gaussians uniformly across GPUs after every few densification steps to restore balance, as new Gaussians are added at varying rates depending on local scene detail [S1].

**Hyperparameter Scaling for Batched Training:**
Grendel supports batch sizes greater than 1. To maintain stability and efficiency, hyperparameters are scaled based on the batch size $b$ using the Independent Gradients Hypothesis [S1]:
*   **Learning Rate:** $\lambda' = \lambda \times \sqrt{b}$, where $\lambda$ is the original learning rate [S1].
*   **Momentum:** $\beta_1' = \beta_1^b$ and $\beta_2' = \beta_2^b$, where $\beta_1, \beta_2$ are the original Adam momentum parameters [S1].

**Pseudocode: Pixel-wise Rebalancing (Algorithm 1)**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def calculate_division_points(estimated_runtimes: list, num_gpus: int) -> list:
    """
    Calculates division points to split pixel blocks among GPUs for load balancing.
    
    Args:
        estimated_runtimes: List of estimated runtimes for each pixel block (ET).
        num_gpus: Number of GPUs (G).
        
    Returns:
        DP: List of division points (indices) to split the blocks.
    """
    # 1. Compute cumulative sum of estimated runtimes
    # CT[i] is the total estimated time for blocks 0 to i
    ct = torch.cumsum(torch.tensor(estimated_runtimes), dim=0)
    
    # 2. Calculate the target runtime per GPU
    # Total time divided by number of GPUs
    total_time = ct[-1]
    target_time_per_gpu = total_time / num_gpus
    
    # 3. Create thresholds for division points
    # Thresholds are multiples of target_time_per_gpu: [0, T, 2T, ..., (G-1)T]
    thresholds = torch.arange(num_gpus) * target_time_per_gpu
    
    # 4. Find the indices in the cumulative sum array that correspond to these thresholds
    # searchsorted finds the indices where thresholds would be inserted to maintain order
    # This effectively finds the block indices that split the cumulative time evenly
    division_points = torch.searchsorted(ct, thresholds)
    
    return division_points.tolist()
```

**Edge Cases and Pitfalls:**
*   **Gradient Correlation:** Increasing batch size without adjusting hyperparameters leads to unstable training. The Independent Gradients Hypothesis assumes gradients from different views are independent, but cameras with similar poses produce correlated gradients, causing sparsity changes [S1].
*   **Workload Imbalance:** Static partitioning fails due to varying Gaussian densities and scene complexity (e.g., sky vs. detailed foreground) [S1]. Dynamic rebalancing is required [S1].
*   **Densification Imbalance:** New Gaussians added via cloning/splitting disrupt uniform distribution. Regular redistribution is necessary [S1].

## Operating characteristics
*   **Time Complexity:** The system achieves near-linear scaling in rendering speed with the number of GPUs. For example, on the Rubble scene, rendering speed increases from 8.8 img/s on 1 GPU to 21.7 img/s on 4 GPUs [S1].
*   **Memory Complexity:** By distributing Gaussians and optimizer states, Grendel overcomes single-GPU memory limits, allowing for significantly larger scene representations (e.g., 40.4 million Gaussians on 16 GPUs vs. 11.2 million on 1 GPU) [S1].
*   **Communication Overhead:** Sparse all-to-all communication minimizes data transfer by only moving relevant Gaussians, leveraging spatial locality [S1].
*   **Strengths:**
    *   Supports large batch sizes (tested up to 32) for better GPU utilization [S1].
    *   Enables high-resolution reconstruction (4K) with superior visual quality (PSNR 27.28 vs 26.28) [S1].
    *   Simplifies multi-GPU usage compared to divide-and-conquer methods; users can simply allocate more GPUs [S1].
*   **Limitations:**
    *   Requires dynamic load balancing logic, adding complexity to the training loop [S1].
    *   Performance depends on the effectiveness of the sparse communication; highly irregular Gaussian distributions may reduce efficiency [S1].
*   **When to Use:** Use Grendel for large-scale, high-resolution 3DGS training where single-GPU memory is insufficient or when faster training times are desired via batched parallelism [S1].
*   **When Not to Use:** For small scenes that fit comfortably on a single GPU, the overhead of distributed communication and rebalancing may not be justified [S1].

## Background
Grendel was introduced in the paper "On Scaling Up 3D Gaussian Splatting Training" [S1]. It addresses the limitation of standard 3DGS training, which is confined to single GPUs, thereby restricting the scale and resolution of reconstructible scenes [S1]. The system was developed to enable distributed training by exploiting the mixed parallelism inherent in 3DGS rendering [S1]. It is available as an open-source project at https://github.com/nyu-systems/Grendel-GS [S1].

## Sources
[S1] On Scaling Up 3D Gaussian Splatting Training, https://doi.org/10.1007/978-3-031-91989-3_2, fetched 2026-10-26
