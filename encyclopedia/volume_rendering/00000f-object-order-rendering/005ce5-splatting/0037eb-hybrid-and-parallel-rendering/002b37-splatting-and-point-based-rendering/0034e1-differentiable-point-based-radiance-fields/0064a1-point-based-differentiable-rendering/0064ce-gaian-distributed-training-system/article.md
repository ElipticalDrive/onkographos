# Gaian Distributed Training System

Also called: Gaian

## Overview
Gaian is a distributed training system for Point-based Differentiable Rendering (PBDR) that reduces communication overhead by optimizing data locality rather than randomly partitioning model state and compute [S1, unofficial]. It provides a unified API to support various PBDR algorithms while explicitly exposing data access patterns to enable locality optimization [S1, unofficial].

## How it works
Gaian partitions points and image rendering across GPUs using a two-stage optimization strategy that exploits spatial locality [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A PBDR algorithm definition (via user-defined functions), a dataset of images with camera poses, and a target number of GPUs.
*   **Output:** An optimized assignment of points to GPUs and a mapping of training batch images to GPUs, resulting in reduced communication volume and higher training throughput [S1, unofficial].

**Data Structures:**
*   **Point Cloud:** A collection of parameterized points (e.g., 3D Gaussians) with attributes such as position, covariance, opacity, and spherical harmonics coefficients [S1, unofficial].
*   **Access Pattern:** A mapping from each camera view (image) to the subset of points it renders (culls and splats) [S1, unofficial].
*   **GPU Assignment Map:** A mapping from each point ID to a specific GPU ID [S1, unofficial].
*   **Batch Assignment Map:** A mapping from each image in a training batch to a specific GPU ID [S1, unofficial].

**Parameters:**
*   **Number of GPUs:** The system supports scaling up to 128 GPUs [S1, unofficial].
*   **Batch Size:** The number of images processed per training step [S1, unofficial].
*   **Typical Values:** Not given in the sources.

**Steps:**

1.  **Define PBDR Algorithm:** The user implements the PBDR algorithm using Gaian’s API by providing three user-defined functions corresponding to the key PBDR steps: `cull`, `splat`, and `render` [S1, unofficial]. This interface exposes the data access pattern, i.e., which parts of the point cloud are accessed by each camera view [S1, unofficial].
2.  **Offline Optimization:** Optimize the placement of points across GPUs based on the access pattern of images to points over the entire dataset [S1, unofficial]. Points likely to be accessed together are assigned to the same GPU to minimize communication [S1, unofficial]. This step exploits the fact that locality among points remains stable over a relatively long time period [S1, unofficial].
3.  **Online Optimization:** Determine how to assign images in a training batch to GPUs based on the access pattern of images within the given batch [S1, unofficial]. This step exploits locality between cameras and their viewable points [S1, unofficial].
4.  **Online Optimizer Implementation:** Use an optimizer based on Linear Sum Assignment and iterative local search to solve the online optimization problem efficiently, as off-the-shelf optimizers are too slow for online optimization [S1, unofficial].
5.  **Distributed Training:** Execute the training loop where each GPU renders its assigned images using the locally stored points and fetches remote points as needed, guided by the optimized assignments [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Gaian Distributed Training System Pseudocode

def define_pbdR_algorithm(cull_func, splat_func, render_func):
    """
    User-defined functions that implement the PBDR algorithm.
    cull_func(camera, points): Returns subset of points visible to camera.
    splat_func(camera, points): Projects points to 2D and computes splats.
    render_func(camera, splats): Composites splats to form image.
    """
    pass

def offline_optimization(dataset, num_gpus):
    """
    Assigns points to GPUs based on access patterns over the entire dataset.
    """
    # Compute access pattern: for each image in dataset, identify accessed points
    access_pattern = {}
    for image in dataset:
        accessed_points = cull_func(image.camera, all_points)
        access_pattern[image.id] = accessed_points

    # Optimize point placement to group frequently co-accessed points
    # Details of the specific clustering/partitioning algorithm are not given in the sources.
    # [G] Assume a graph-based clustering or k-means variant on point co-access frequency.
    point_to_gpu_map = optimize_point_placement(access_pattern, num_gpus)
    return point_to_gpu_map

def online_optimization(batch, num_gpus, point_to_gpu_map):
    """
    Assigns images in a batch to GPUs based on current access patterns.
    """
    # Compute access pattern for the current batch
    batch_access_pattern = {}
    for image in batch:
        accessed_points = cull_func(image.camera, all_points)
        batch_access_pattern[image.id] = accessed_points

    # Use Linear Sum Assignment and iterative local search to assign images to GPUs
    # to minimize communication overhead given the point_to_gpu_map.
    # [G] The specific cost function and solver implementation details are not given.
    image_to_gpu_map = solve_assignment(batch_access_pattern, point_to_gpu_map, num_gpus)
    return image_to_gpu_map

def distributed_training_step(batch, point_to_gpu_map, image_to_gpu_map):
    """
    Executes one training step across GPUs.
    """
    for gpu in gpus:
        assigned_images = [img for img in batch if image_to_gpu_map[img.id] == gpu.id]
        local_points = [pt for pt in all_points if point_to_gpu_map[pt.id] == gpu.id]
        remote_points = fetch_remote_points(assigned_images, point_to_gpu_map, exclude=gpu.id)
        
        # Combine local and remote points for rendering
        all_accessible_points = local_points + remote_points
        
        for image in assigned_images:
            visible_points = cull_func(image.camera, all_accessible_points)
            splats = splat_func(image.camera, visible_points)
            rendered_image = render_func(image.camera, splats)
            compute_gradient(rendered_image, image.target)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.
*   **Degenerate Input:** Not given in the sources.

## Subtopics

### Gaian Locality Optimization
A sibling method that likely focuses on specific aspects of locality optimization within the Gaian framework. [Gaian Locality Optimization](19153-gaian-locality-optimization.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Gaian reduces communication by up to 91% and improves training throughput by 1.50–3.71× compared to existing systems (gsplat) across six datasets and up to 128 GPUs [S1, unofficial].
*   **Scaling:** Gaian scales to up to 128 GPUs [S1, unofficial].
*   **Strengths:** Reduces communication overhead significantly by optimizing data locality; provides a unified API for various PBDR algorithms; enables training of very large models (e.g., 500 million points) [S1, unofficial].
*   **Limitations:** Requires explicit definition of PBDR algorithm steps via API; optimization overhead for locality assignment [S1, unofficial].
*   **When to use:** When training PBDR models on large-scale datasets with many GPUs where communication overhead is a bottleneck [S1, unofficial].
*   **When not to use:** When the PBDR algorithm does not fit the cull-splat-render paradigm or when the overhead of locality optimization outweighs the communication savings for small-scale setups [S1, unofficial].

## Background
Gaian was proposed by Xiwen Min from New York University in a paper titled "Scaling Point-based Differentiable Rendering for Large-scale Reconstruction" [S1, unofficial]. The paper was published on arXiv in December 2025 [S1, unofficial]. Gaian addresses the challenge of distributed training for PBDR methods, which model 3D scenes as large collections of parameterized points and learn attributes via mini-batch gradient descent [S1, unofficial]. Existing systems randomly partition points and rendering tasks, leading to heavy communication as in-frustum points often reside on remote GPUs [S1, unofficial]. Gaian differs from existing systems by optimizing locality of access, both among points and between cameras and viewable points [S1, unofficial]. Gaian trained the largest 3D Gaussian Splatting model to date with 500 million points (29.5 billion parameters) on the MatrixCity dataset [S1, unofficial].

## Sources
[S1] Scaling Point-based Differentiable Rendering for Large-scale Reconstruction, https://doi.org/10.48550/arxiv.2512.20017, fetched 2026-10-26 (unofficial)
