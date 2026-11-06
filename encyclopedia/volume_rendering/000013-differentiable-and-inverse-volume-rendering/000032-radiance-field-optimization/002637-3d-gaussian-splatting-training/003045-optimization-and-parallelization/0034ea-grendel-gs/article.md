# Grendel-GS

Also called: Distributed 3D Gaussian Splatting Optimization

## Overview
Grendel-GS is an out-of-core and distributed optimization method for 3D Gaussian Splatting that partitions the Gaussian list across multiple GPUs to handle scenes exceeding single-GPU memory capacity [S1, unofficial]. It differs from standard single-GPU training by allowing each GPU to render its own subset of Gaussians and combining results via inter-GPU communication at partition boundaries [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A large set of 3D Gaussians $\mathcal{G}$, a set of training views $\mathcal{V}$, and a distributed system of $N$ GPUs.
- Output: Optimized Gaussian parameters (position, covariance, opacity, color, SH coefficients) distributed across the GPU cluster.

**Data Structures:**
- The global Gaussian list $\mathcal{G}$ is partitioned into $N$ subsets $\mathcal{G}_0, \mathcal{G}_1, \dots, \mathcal{G}_{N-1}$, where each subset $\mathcal{G}_i$ is assigned to GPU $i$.
- Each Gaussian retains its standard parameter layout (position $\mathbf{p}$, rotation $\mathbf{q}$, scale $\mathbf{s}$, opacity $\alpha$, SH coefficients $\mathbf{c}$).

**Steps:**
1. **Partitioning:** Distribute the initial Gaussian cloud $\mathcal{G}$ across the available GPUs. The partitioning strategy is not specified in the sources [G].
2. **Local Rendering:** For each training view, each GPU $i$ renders the image contribution from its local subset $\mathcal{G}_i$ using the standard differentiable rasterizer.
3. **Inter-GPU Communication:** At partition boundaries, GPUs exchange necessary data (e.g., transmittance or partial color contributions) to ensure correct alpha-blending across the global set of Gaussians. The exact communication protocol is not detailed [G].
4. **Gradient Aggregation:** Gradients computed for each Gaussian are aggregated back to their respective GPUs.
5. **Update:** Each GPU updates the parameters of its local Gaussian subset using the aggregated gradients.

**Parameters:**
- Number of GPUs: $N$.
- Partitioning scheme: Not given in the sources.

**Edge Cases and Pitfalls:**
- Communication overhead between GPUs can become a bottleneck.
- Load balancing across GPUs depends on the spatial distribution of Gaussians and view angles.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Grendel-GS distributed optimization step
# Note: Specific communication details are not provided in sources [G]

def grendel_gs_step(gaussians_global, views, num_gpus):
    # Partition Gaussians across GPUs
    gaussians_local = partition(gaussians_global, num_gpus)
    
    # Initialize gradients
    gradients_local = [zeros_like(g) for g in gaussians_local]
    
    for view in views:
        # Local rendering on each GPU
        local_colors = []
        for i in range(num_gpus):
            # Render subset of Gaussians
            color_i = render(gaussians_local[i], view)
            local_colors.append(color_i)
        
        # Combine results via inter-GPU communication
        # Exact method of combining partial renders is not specified [G]
        global_color = combine(local_colors)
        
        # Compute loss and gradients
        loss = compute_loss(global_color, view.target)
        local_grads = compute_gradients(loss, gaussians_local, view)
        
        # Aggregate gradients
        for i in range(num_gpus):
            gradients_local[i] += local_grads[i]
    
    # Update Gaussians
    for i in range(num_gpus):
        gaussians_local[i] = update(gaussians_local[i], gradients_local[i])
    
    return gaussians_local
```

## Subtopics

### TideGS
TideGS is a fully out-of-core optimization method that pushes the frontier to over one billion primitives by streaming Gaussians from disk in a view-dependent order, training on a single machine with only a fraction of the data in GPU memory [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Reduces per-GPU memory usage by partitioning the Gaussian list, enabling training on scenes larger than single-GPU memory.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the number of GPUs, but communication overhead may limit efficiency.
- **Strengths:** Enables training on very large scenes that do not fit in a single GPU's memory.
- **Limitations:** Requires multiple GPUs and inter-GPU communication infrastructure.
- **When to use:** When the scene size exceeds the memory capacity of a single GPU.
- **When not to use:** When resources are limited to a single GPU or when communication overhead is prohibitive.

## Background
Grendel-GS was introduced as part of the development of out-of-core and distributed optimization methods for 3D Gaussian Splatting [S1, unofficial]. It addresses the memory limitations of standard 3DGS training by distributing the Gaussian list across multiple GPUs.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
