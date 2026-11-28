# Gaian Distributed Training System

Also called: Gaian, Gaian PBDR Interface, Gaian's programming abstractions, PBDR_program interface

## Overview
Gaian is a distributed training system for Point-based Differentiable Rendering (PBDR) that reduces communication overhead by optimizing data locality rather than randomly partitioning model state and compute [S1, unofficial]. It provides a unified API to support various PBDR algorithms while explicitly exposing data access patterns to enable locality optimization [S1, unofficial].

## How it works
Gaian partitions points and image rendering across GPUs using a two-stage optimization strategy that exploits spatial locality [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A PBDR algorithm definition via three user-defined functions, a dataset of images with camera poses, and a target number of GPUs [S1, unofficial].
*   **Output:** An optimized assignment of points to GPUs and a mapping of training batch images to GPUs, resulting in reduced communication volume and higher training throughput [S1, unofficial].

**Data Structures:**
*   **Point Cloud ($PC$):** A dictionary of tensors storing point attributes, where each tensor has shape $S \times l$ ($S$ is total points, $l$ is floats per attribute) [S1, unofficial].
*   **Splats ($SP$):** A dictionary of tensors representing view-dependent splats, where each tensor's first dimension equals the number of in-frustum points [S1, unofficial].
*   **Access Pattern:** A mapping from each camera view to the subset of point IDs it renders [S1, unofficial].
*   **GPU Assignment Maps:** Mappings from point IDs to GPU IDs and from image IDs to GPU IDs [S1, unofficial].

**Numbered Steps:**
1.  **Define PBDR Algorithm:** The user implements the algorithm using Gaian’s API by providing three user-defined functions corresponding to the key PBDR steps: `pts_culling`, `pts_splatting`, and `image_render` [S1, unofficial]. This interface exposes the data access pattern, i.e., which parts of the point cloud are accessed by each camera view [S1, unofficial].
2.  **Offline Optimization:** Optimize the placement of points across GPUs based on the access pattern of images to points over the entire dataset [S1, unofficial]. Points likely to be accessed together are assigned to the same GPU to minimize communication [S1, unofficial]. This step exploits the fact that locality among points remains stable over a relatively long time period [S1, unofficial].
3.  **Online Optimization:** Determine how to assign images in a training batch to GPUs based on the access pattern of images within the given batch [S1, unofficial]. This step exploits locality between cameras and their viewable points [S1, unofficial].
4.  **Online Optimizer Implementation:** Use an optimizer based on Linear Sum Assignment and iterative local search to solve the online optimization problem efficiently, as off-the-shelf optimizers are too slow for online optimization [S1, unofficial].
5.  **Distributed Training:** Execute the training loop where each GPU renders its assigned images using the locally stored points and fetches remote points as needed, guided by the optimized assignments [S1, unofficial].

**Parameters:**
*   **Number of GPUs:** The system supports scaling up to 128 GPUs [S1, unofficial].
*   **Typical Values:** Not given in the sources. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Differentiability:** `pts_splatting` and `image_render` must be differentiable for the backward pass [S1, unofficial].
*   **Precision:** Not given in the sources. [S1, unofficial]
*   **Boundaries:** Not given in the sources. [S1, unofficial]
*   **Degenerate Input:** Not given in the sources. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Gaian Distributed Training System Pseudocode
# User-defined PBDR functions (provided by algorithm implementer)
def pts_culling(view, PC):
    """Compute point IDs within the given view's frustum. Returns 1D tensor of indices."""
    # [G] Standard frustum culling: project points to view space, filter by bounds
    pass

def pts_splatting(view, PC, infrustum_ids):
    """Transform only infrustum points to view-dependent splats. Returns SP dict of tensors."""
    # [G] Standard differentiable projection: map 3D points to 2D splats
    pass

def image_render(view, SP):
    """Render image from view-dependent splats. Returns rendered image tensor."""
    # [G] Standard differentiable compositing: alpha-blend splats
    pass

def offline_optimization(dataset, num_gpus, PC):
    """Assign points to GPUs based on access patterns over the entire dataset."""
    access_pattern = {}
    for image in dataset:
        access_pattern[image.id] = pts_culling(image.view, PC)
    # Group points frequently co-accessed onto the same GPU
    # [G] Graph-based clustering or k-means on point co-access frequency
    point_to_gpu_map = {} # Result of clustering
    return point_to_gpu_map

def online_optimization(batch, num_gpus, point_to_gpu_map, PC):
    """Assign images in a batch to GPUs based on batch access patterns."""
    batch_access_pattern = {}
    for image in batch:
        batch_access_pattern[image.id] = pts_culling(image.view, PC)
    # Minimize communication using Linear Sum Assignment and iterative local search
    # [G] Construct cost matrix of remote fetch volume, solve assignment
    image_to_gpu_map = {} # Result of assignment
    return image_to_gpu_map

def distributed_training_step(batch, point_to_gpu_map, image_to_gpu_map, PC):
    """Execute one training step across GPUs."""
    for gpu_id in range(num_gpus):
        assigned_images = [img for img in batch if image_to_gpu_map[img.id] == gpu_id]
        local_point_ids = [pid for pid in PC if point_to_gpu_map[pid] == gpu_id]
        # Fetch remote points required by assigned images
        remote_point_ids = set()
        for img in assigned_images:
            visible_ids = pts_culling(img.view, PC)
            for pid in visible_ids:
                if point_to_gpu_map[pid] != gpu_id:
                    remote_point_ids.add(pid)
        all_point_ids = local_point_ids + list(remote_point_ids)
        for img in assigned_images:
            visible_ids = pts_culling(img.view, PC)
            splats = pts_splatting(img.view, PC, visible_ids)
            rendered_image = image_render(img.view, splats)
            # [G] Standard autograd backward pass to compute gradients and update PC
```

## Subtopics

### Gaian Locality Optimization
A sibling method that focuses on specific aspects of locality optimization within the Gaian framework. [Gaian Locality Optimization](19153-gaian-locality-optimization.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1, unofficial]
*   **Memory Complexity:** Not given in the sources. [S1, unofficial]
*   **Measured Performance:** Gaian reduces communication by up to 91% and improves training throughput by 1.50–3.71× compared to existing systems (gsplat) across six datasets and up to 128 GPUs [S1, unofficial].
*   **Scaling:** Gaian scales to up to 128 GPUs [S1, unofficial].
*   **Strengths:** Reduces communication overhead significantly by optimizing data locality; provides a unified API for various PBDR algorithms; enables training of very large models (e.g., 500 million points) [S1, unofficial].
*   **Limitations:** Requires explicit definition of PBDR algorithm steps via API; optimization overhead for locality assignment [S1, unofficial].
*   **When to use:** When training PBDR models on large-scale datasets with many GPUs where communication overhead is a bottleneck [S1, unofficial].

## Background
Gaian was proposed by Xiwen Min from New York University in a paper titled "Scaling Point-based Differentiable Rendering for Large-scale Reconstruction" [S1, unofficial]. The paper was published on arXiv in December 2025 [S1, unofficial]. Gaian addresses the challenge of distributed training for PBDR methods, which model 3D scenes as large collections of parameterized points and learn attributes via mini-batch gradient descent [S1, unofficial]. Existing systems randomly partition points and rendering tasks, leading to heavy communication as in-frustum points often reside on remote GPUs [S1, unofficial]. Gaian differs from existing systems by optimizing locality of access, both among points and between cameras and viewable points [S1, unofficial]. Gaian trained the largest 3D Gaussian Splatting model to date with 500 million points (29.5 billion parameters) on the MatrixCity dataset [S1, unofficial].

## Sources
[S1] Scaling Point-based Differentiable Rendering for Large-scale Reconstruction, https://doi.org/10.48550/arxiv.2512.20017, fetched 2026-10-26 (unofficial)
