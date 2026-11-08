# Grendel-GS
Also called: Distributed Out-of-Core Optimization for 3DGS
## Overview
Grendel-GS is a distributed optimization framework for 3D Gaussian Splatting that partitions the global set of Gaussian primitives across multiple GPUs to overcome single-device memory limits [S1, unofficial]. It enables the training of scenes with billions of Gaussians by allowing each GPU to render its local subset and combining results via inter-GPU communication at partition boundaries [S1, unofficial]. Unlike [TideGS](7350-tidegs.md), which streams data from disk on a single GPU, Grendel-GS distributes the parameter store across a multi-GPU cluster [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A global list of $N$ 3D Gaussians, each defined by position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$, opacity $\alpha_i$, spherical harmonic coefficients $\mathbf{c}_i$, and scaling $\mathbf{s}_i$.
- Input: A set of training views (cameras).
- Input: A partitioning strategy mapping Gaussians to $K$ GPUs.
- Output: Optimized Gaussian parameters stored distributedly across $K$ GPUs.

**Data Structures:**
The global Gaussian list is partitioned into $K$ subsets $\mathcal{G}_0, \dots, \mathcal{G}_{K-1}$, where $\mathcal{G}_k$ resides on GPU $k$. Each subset contains a contiguous block of Gaussian attributes. The partitioning is static or dynamic based on scene geometry or load balancing.

**Steps:**
1. **Partitioning:** Divide the $N$ Gaussians into $K$ disjoint sets $\mathcal{G}_k$ assigned to GPU $k$.
2. **Local Rendering:** Each GPU $k$ renders the scene using only Gaussians in $\mathcal{G}_k$. This produces a partial color image $\mathbf{I}_k$ and partial transmittance/depth information.
3. **Inter-GPU Communication:** At partition boundaries, GPUs exchange necessary intermediate data (e.g., transmittance $T$ or accumulated color contributions) to ensure correct alpha-blending across the global scene. This step resolves dependencies where a Gaussian on GPU $j$ occludes or is occluded by a Gaussian on GPU $k$.
4. **Global Composite:** The partial images are combined to form the final rendered image $\mathbf{I}$ for each training view.
5. **Loss Computation:** Compute the loss $\mathcal{L}$ between $\mathbf{I}$ and the ground truth image.
6. **Gradient Distribution:** Gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ are computed and distributed back to the respective GPUs based on which Gaussian contributed to the loss.
7. **Parameter Update:** Each GPU updates its local subset of Gaussian parameters using the received gradients.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume K GPUs, each with rank k
# Global Gaussian list G is partitioned into G_k on GPU k

def forward_pass(k, G_k, cameras):
    # Render local subset
    I_k, T_k, depth_k = render(G_k, cameras)
    return I_k, T_k, depth_k

def backward_pass(k, G_k, grad_I, cameras):
    # Compute local gradients
    grad_G_k = compute_gradients(G_k, grad_I, cameras)
    return grad_G_k

def optimize(G, cameras, K):
    # Partition G across K GPUs
    G_partitions = partition(G, K)
    
    for iteration in range(num_iterations):
        # Forward pass on each GPU
        results = [forward_pass(k, G_partitions[k], cameras) for k in range(K)]
        
        # Combine results (simplified: assumes no cross-Gaussian occlusion dependency 
        # or that communication handles it)
        # In practice, this requires exchanging T or color contributions at boundaries
        I_global = combine(results)
        
        # Compute loss
        L = loss(I_global, ground_truth)
        
        # Backward pass
        grad_I = compute_loss_gradient(L, I_global)
        grad_G_partitions = [backward_pass(k, G_partitions[k], grad_I, cameras) for k in range(K)]
        
        # Update parameters
        for k in range(K):
            G_partitions[k] = update(G_partitions[k], grad_G_partitions[k])
            
    return G_partitions
```

**Edge Cases:**
- **Occlusion Across Partitions:** If a Gaussian on GPU $j$ occludes a Gaussian on GPU $k$, the alpha-blending order must be respected. This requires communication of transmittance or depth information between GPUs [S1, unofficial].
- **Load Imbalance:** Uneven distribution of Gaussians or complex geometry on specific GPUs can lead to bottlenecks. Dynamic partitioning may be required.
- **Communication Overhead:** Inter-GPU communication latency can dominate computation time if not optimized.

## Subtopics
## Operating characteristics
- **Time Complexity:** $O(N \cdot V)$ per iteration, where $N$ is the number of Gaussians and $V$ is the number of views, distributed across $K$ GPUs. Communication overhead adds $O(K \cdot B)$, where $B$ is the boundary size.
- **Memory Complexity:** $O(N/K)$ per GPU, enabling training of scenes with $N \gg \text{VRAM}_{\text{GPU}}$.
- **Strengths:** Scales to billions of Gaussians; utilizes multiple GPUs efficiently.
- **Limitations:** Communication overhead; complex implementation; requires high-bandwidth interconnects (e.g., NVLink) for optimal performance.
- **When to Use:** Large-scale scenes (e.g., urban environments, large indoor spaces) that exceed single-GPU memory.
- **When Not to Use:** Small scenes; single-GPU setups; scenarios where communication latency is high.

## Background
Grendel-GS was introduced as part of the effort to scale 3D Gaussian Splatting to massive scenes. It addresses the memory bottleneck of standard 3DGS by distributing the Gaussian list across multiple GPUs [S1, unofficial]. It is related to other out-of-core methods like [CLM](link) and [TideGS](7350-tidegs.md) [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
