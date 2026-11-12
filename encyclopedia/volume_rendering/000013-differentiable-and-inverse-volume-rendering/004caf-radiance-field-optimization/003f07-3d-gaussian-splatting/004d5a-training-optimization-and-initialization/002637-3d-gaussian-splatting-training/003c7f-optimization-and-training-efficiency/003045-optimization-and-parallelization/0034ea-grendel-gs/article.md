# Grendel-GS
## Overview
Grendel-GS is a distributed optimization framework for 3D Gaussian Splatting that partitions the global set of Gaussian primitives across multiple GPUs to overcome single-device memory limits [S1, unofficial]. It enables the training of large scenes by allowing each GPU to render its local subset and combining results via inter-GPU communication at partition boundaries [S1, unofficial]. Unlike [TideGS](7350-tidegs.md), which streams data from disk on a single GPU, Grendel-GS distributes the parameter store across a multi-GPU cluster [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A global list of $N$ 3D Gaussians with parameters $\theta$ (position, covariance, opacity, color, SH coefficients) [G].
- Input: A set of training views (cameras) and ground truth images [G].
- Input: A partitioning strategy mapping Gaussians to $K$ GPUs [S1, unofficial].
- Output: Optimized Gaussian parameters distributed across $K$ GPUs [S1, unofficial].

**Data Structures:**
The global Gaussian list is partitioned into $K$ subsets $\mathcal{G}_0, \dots, \mathcal{G}_{K-1}$, where $\mathcal{G}_k$ resides on GPU $k$ [S1, unofficial]. The exact layout of each subset and the partitioning strategy (static vs. dynamic) are not specified in the sources [S1, unofficial].

**Steps:**
1. **Partitioning:** Divide the $N$ Gaussians into $K$ disjoint sets $\mathcal{G}_k$ assigned to GPU $k$ [S1, unofficial].
2. **Local Rendering:** Each GPU $k$ renders the scene using only Gaussians in $\mathcal{G}_k$ [S1, unofficial].
3. **Inter-GPU Communication:** GPUs exchange intermediate data at partition boundaries to combine rendering results [S1, unofficial].
4. **Global Composite:** Partial images are combined to form the final rendered image $\mathbf{I}$ [S1, unofficial].
5. **Loss Computation & Backpropagation:** Compute the loss $\mathcal{L}$ between $\mathbf{I}$ and the ground truth, then propagate gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ back to the respective GPUs [G].
6. **Parameter Update:** Each GPU updates its local subset of Gaussian parameters using the received gradients [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_pass(gaussians, cameras):
    # Project each Gaussian to 2D, compute 2D covariance, opacity, and color
    # Sort Gaussians by depth, then alpha-blend contributions into pixel colors
    # Returns: rendered_image, transmittance_map, gaussian_contributions
    pass # Standard differentiable rasterization forward pass [G]

def backward_pass(gaussians, cameras, grad_colors):
    # Backpropagate gradients through the alpha-blending chain
    # Compute gradients for position, covariance, opacity, color, and SH coefficients
    # Returns: gradients_for_gaussians
    pass # Standard differentiable rasterization backward pass [G]

def optimize(gaussians, cameras, ground_truth, num_gpus, num_iterations):
    # Partition Gaussians across GPUs
    partitions = [gaussians[i::num_gpus] for i in range(num_gpus)]
    
    for step in range(num_iterations):
        # Local rendering on each GPU
        local_outputs = [forward_pass(partitions[k], cameras) for k in range(num_gpus)]
        
        # Combine results via inter-GPU communication at boundaries
        # Exact communication protocol is not specified in the sources [S1, unofficial]
        global_image = combine_results(local_outputs)
        
        # Compute loss and gradients
        loss = compute_loss(global_image, ground_truth)
        grad_colors = compute_loss_gradient(loss, global_image)
        
        # Backward pass and parameter update on each GPU
        for k in range(num_gpus):
            grad_gaussians = backward_pass(partitions[k], cameras, grad_colors)
            partitions[k] = update_parameters(partitions[k], grad_gaussians)
            
    return partitions
```

**Edge Cases:**
- **Occlusion Across Partitions:** Gaussians on different GPUs may occlude each other, requiring careful handling of the alpha-blending order during the combination step [G].
- **Communication Overhead:** Inter-GPU communication latency can dominate computation time if not optimized [G].
- **Load Imbalance:** Uneven distribution of Gaussians or complex geometry on specific GPUs can lead to bottlenecks [G].

## Subtopics

### [TideGS](7350-tidegs.md)
A fully out-of-core optimization framework for training 3D Gaussian Splatting that enables training with over one billion primitives on a single GPU with limited VRAM. It solves the VRAM bottleneck by decoupling memory usage from the total scene size, keeping the full parameter table on SSD and materializing only the per-iteration working set in GPU memory. Unlike Grendel-GS, which distributes Gaussians across multiple GPUs, TideGS operates on a single machine, treating GPU VRAM as a high-bandwidth working-set cache rather than a persistent parameter store.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** $O(N/K)$ per GPU, enabling training of scenes with $N \gg \text{VRAM}_{\text{GPU}}$ [G].
- **Strengths:** Scales to large scenes by distributing memory and computation across multiple GPUs [S1, unofficial].
- **Limitations:** Communication overhead; complex implementation; requires high-bandwidth interconnects for optimal performance [G].
- **When to Use:** Large-scale scenes that exceed single-GPU memory capacity [S1, unofficial].
- **When Not to Use:** Small scenes or single-GPU setups where communication overhead outweighs benefits [G].

## Background
Grendel-GS was introduced as part of the effort to scale 3D Gaussian Splatting to massive scenes [S1, unofficial]. It addresses the memory bottleneck of standard 3DGS by distributing the Gaussian list across multiple GPUs [S1, unofficial]. It is related to other out-of-core methods like [CLM](link) and [TideGS](7350-tidegs.md) [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
