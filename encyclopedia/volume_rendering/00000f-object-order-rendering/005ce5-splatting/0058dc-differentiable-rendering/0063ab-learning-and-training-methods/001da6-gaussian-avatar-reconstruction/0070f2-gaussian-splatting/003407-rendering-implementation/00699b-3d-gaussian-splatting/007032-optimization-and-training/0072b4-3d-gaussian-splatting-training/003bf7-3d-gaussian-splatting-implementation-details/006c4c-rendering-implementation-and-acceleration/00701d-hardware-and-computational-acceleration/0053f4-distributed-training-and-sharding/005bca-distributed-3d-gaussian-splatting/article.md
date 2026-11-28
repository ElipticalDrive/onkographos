# Distributed 3D Gaussian Splatting

Also called: Distributed 3D Gaussian Splatting for Scientific Visualization, Distributed 3D-GS, Multi-GPU 3D-GS

## Overview
Distributed 3D Gaussian Splatting is a multi-GPU extension of the 3D Gaussian Splatting (3D-GS) pipeline designed for scientific visualization, enabling the scalable processing of large datasets that exceed the memory capacity of a single GPU [S1]. It differs from single-GPU baselines by distributing the global point cloud and Gaussian parameters across multiple workers, allowing for the reconstruction of scenes with tens of millions of primitives [S1]. Unlike [BlitzGS](15088-blitzgs.md) which focuses on inference acceleration, this method addresses the fundamental memory bottleneck during the optimization phase [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Scientific volume data (e.g., VTK, HDF5), number of GPUs ($N_{GPU}$), image resolution ($R \times R$), number of synthetic views ($N_{views}$).
*   **Output:** Optimized set of 3D Gaussian primitives (position, covariance, opacity, spherical harmonics coefficients) distributed across the GPU cluster.

**Data Structures:**
The method relies on the standard 3D Gaussian primitive definition provided by [Rendering Implementation and Acceleration](21069-rendering-implementation-and-acceleration.md). The global state consists of:
1.  **Point Cloud:** A set of 3D points extracted from the isosurface.
2.  **Gaussian Parameters:** For each Gaussian $i$: mean $\mu_i$, covariance matrix $\Sigma_i$ (or rotation $R_i$ and scale $S_i$), opacity $\alpha_i$, and spherical harmonics coefficients $c_{i,lm}$.

**Sharding Strategy:**
The global point cloud and associated Gaussian parameters are partitioned into shards. Each GPU $k$ holds a subset of the Gaussians $\mathcal{G}_k$ such that $\bigcup_k \mathcal{G}_k$ constitutes the full scene [S1]. The training data (synthetic views) is also distributed across workers [S1].

**Algorithm Steps:**
1.  **Initialization:** Extract isosurface point clouds directly from scientific volume data using ParaView [S1]. Initialize Gaussian primitives at these locations. This bypasses image-based reconstruction methods [S1].
2.  **View Generation:** Generate a set of synthetic camera views in a structured orbit. The number of views is set to 448 to support data complexity, an increase from the 250 views used in prior work [S1].
3.  **Distribution:** Distribute the shards of the global point cloud and Gaussian parameters to each GPU [S1]. Distribute the corresponding training views to the workers [S1].
4.  **Forward Pass (Per GPU):** Each GPU renders its assigned shard of Gaussians for its assigned views. This involves projecting 3D Gaussians to 2D screen space and performing alpha blending, utilizing the tiling, sorting, and culling mechanisms described in [Rendering Implementation and Acceleration](21069-rendering-implementation-and-acceleration.md) [S1].
5.  **Loss Computation:** Compute the reconstruction loss (e.g., L1 loss, D-SSIM) between the rendered images and the ground truth synthetic views for the assigned views [S1].
6.  **Gradient Synchronization:** Synchronize gradients across all GPUs via a fused all-reduce scheme [S1]. This ensures that the optimization updates are consistent across the distributed shards.
7.  **Parameter Update:** Update the Gaussian parameters (position, scale, rotation, opacity, SH coefficients) using an optimizer (typically Adam) based on the synchronized gradients [S1].
8.  **Iteration:** Repeat steps 4-7 until convergence.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Distributed 3D Gaussian Splatting Training Loop
# Assumes a distributed environment with N GPUs
# Each GPU executes this code with its local rank 'rank'

def distributed_3dgs_training(volume_data, num_gpus, resolution, num_views=448):
    # 1. Initialization
    # Extract isosurface point cloud using ParaView (external tool)
    global_point_cloud = extract_isosurface_point_cloud(volume_data)
    
    # Initialize Gaussian parameters for all points
    # Positions, Scales, Rotations, Opacities, SH Coefficients
    global_gaussians = initialize_gaussians(global_point_cloud)
    
    # 2. Distribution
    # Shard the global data across GPUs
    # Each GPU gets a subset of Gaussians and a subset of views
    local_gaussians = shard_data(global_gaussians, rank, num_gpus)
    local_views = generate_orbit_views(num_views, rank, num_gpus)
    
    # 3. Training Loop
    for iteration in range(max_iterations):
        # Forward Pass: Render local shard
        rendered_images = []
        for view in local_views:
            # Project local Gaussians to 2D
            # Sort by depth, cull, and alpha blend
            # This uses the acceleration primitives from parent article
            image = render_gaussians(local_gaussians, view.camera_params)
            rendered_images.append(image)
        
        # Compute Loss
        # Compare rendered images with ground truth synthetic views
        loss = compute_loss(rendered_images, local_views.ground_truth)
        
        # Backward Pass
        gradients = compute_gradients(loss, local_gaussians)
        
        # Synchronize Gradients via All-Reduce
        # Fused all-reduce scheme to aggregate gradients from all GPUs
        synced_gradients = all_reduce(gradients, op='sum')
        
        # Update Parameters
        # Apply optimizer step (e.g., Adam)
        local_gaussians = update_parameters(local_gaussians, synced_gradients)
        
    return local_gaussians
```

**Parameters:**
*   **Number of synthetic camera views:** 448 [S1].
*   **Image resolutions:** 512x512, 1024x1024, 2048x2048 [S1].
*   **Hardware:** NVIDIA A100 GPUs [S1].

**Edge Cases and Pitfalls:**
*   **Memory Limits:** A single A100 GPU supports up to approximately 11.2M Gaussians. Datasets larger than this (e.g., Miranda with ~18M Gaussians) are infeasible on a single GPU due to memory exhaustion [S1].
*   **Data Complexity:** Insufficient number of views (e.g., 250) may fail to capture the complexity of scientific data, necessitating an increase to 448 views [S1].

## Operating characteristics
*   **Time Complexity:** Training time scales with the number of Gaussians, views, and resolution. The method demonstrates near-linear speedup with the number of GPUs. For the Kingsnake dataset (4M Gaussians) at 2048x2048 resolution, 4 GPUs provide a 5.6x speedup compared to 1 GPU (8.50 min vs 48.00 min) [S1].
*   **Memory Complexity:** Memory usage is distributed. The total memory required is proportional to the number of Gaussians divided by the number of GPUs. This allows training on datasets exceeding single-GPU VRAM (e.g., 18M Gaussians on 4 A100s) [S1].
*   **Strengths:** Enables high-resolution reconstruction of large-scale scientific data that cannot fit on a single GPU. Improves training throughput significantly [S1].
*   **Limitations:** Requires a multi-GPU setup and network bandwidth for gradient synchronization. Performance is limited by the slowest GPU and communication overhead [S1].
*   **When to use:** When the number of Gaussian primitives exceeds the memory capacity of a single GPU (e.g., >11.2M on A100) or when high-resolution training on large datasets is required [S1].
*   **When not to use:** For small datasets that fit comfortably on a single GPU, where the overhead of distributed training may not be justified.

## Background
*   **Origin:** The method was presented by Mateevitsi, Papka, Petruzza, and Sizzi in 2025 [S1].
*   **History:** It builds on previous work by Sewell et al. [4] which demonstrated high-fidelity isosurface reconstruction using Gaussian primitives from scientific volume data. It adapts the distributed training design from Grendel-GS [6] to the 3D-GS pipeline.
*   **Usage:** Designed for HPC-based scientific workflows, enabling real-time post hoc and in situ visualization of complex simulations [S1].

## Sources
[S1] Toward Distributed 3D Gaussian Splatting for High-Resolution Isosurface Visualization, https://doi.org/10.1109/escience65000.2025.00056, fetched 2026-10-29
