# Domain Decomposition Training and Inference

Also called: Domain Decomposition Training, Parallel SRN training

## Overview
A model-parallel strategy for fitting large-scale volumes by partitioning the data into a grid of bricks, where one SRN is trained per brick in parallel on multiple GPUs [S1]. It avoids out-of-core I/O overhead by ensuring each brick fits in GPU memory and accelerates inference using spatial hashing to map query points to the correct model [S1]. It is distinguished from single-model approaches by enabling the reconstruction of volumes (e.g., 450 GB) in under 7 minutes on 8 GPUs while increasing accuracy over a single model with the same total parameter count [S1].

## How it works
**Inputs:** Large-scale volume data, available GPUs, brick grid dimensions ($I \times J \times K$) [S1].
**Outputs:** A set of trained SRN models (specifically APMGSRN), each associated with a spatial extent [S1].

**Data Structures:**
*   **Brick Grid:** The volume is divided into a 3D grid of bricks [S1].
*   **Job Queue:** A list of training jobs assigned to available GPUs [S1].
*   **Spatial Hash Table:** A hash function mapping 3D spatial coordinates to the index of the brick (and thus the model) responsible for that region [S1].

**Steps:**
1.  **Pre-calculation:** Calculate the spatial extents of each network within the volume's total extents, including ghost cells if applicable, and generate a list of jobs assigned to available GPUs [S1].
2.  **Parallel Training Loop:** Assign a job to an available GPU. Initialize an APMGSRN model on that GPU. Load only the data within the assigned brick's extents into GPU memory, leveraging parallel file access formats like NetCDF to minimize I/O overhead [S1].
3.  **Early Stopping Training:** Train the model by monitoring reconstruction loss $L_{rec}$. If $L_{rec}$ has not decreased by 0.01% over 500 iterations, reduce the learning rate by a factor of 10. Repeat this reduction up to 3 times, then terminate training for that model [S1].
4.  **GPU Recycling:** When training finishes, return the GPU to the available list and assign the next job [S1].
5.  **Inference Setup:** Load all trained models into GPU memory on the inference machine [S1].
6.  **Inference Query:** For a query point $\mathbf{p}$, compute the hash to determine the corresponding brick index, retrieve the associated model, and query it for volume properties [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def calculate_brick_extent(brick_id, grid_shape, volume_extent):
    """Returns (min_coord, max_coord) for a brick in a uniform 3D grid."""
    min_c, max_c = volume_extent
    size = max_c - min_c
    brick_min = min_c + (brick_id[0] * size / grid_shape[0], 
                         brick_id[1] * size / grid_shape[1], 
                         brick_id[2] * size / grid_shape[2])
    brick_max = min_c + ((brick_id[0] + 1) * size / grid_shape[0], 
                         (brick_id[1] + 1) * size / grid_shape[1], 
                         (brick_id[2] + 1) * size / grid_shape[2])
    return brick_min, brick_max

def sample_points(data_subset, num_samples=1024):
    """Uniformly samples points and targets from the brick's data subset."""
    points = [random.uniform(0, 1) for _ in range(num_samples * 3)]
    targets = [random.random() for _ in range(num_samples)] # Placeholder for actual density/value
    return points, targets

def calculate_loss(predictions, targets):
    """Computes mean squared error."""
    return sum((p - t) ** 2 for p, t in zip(predictions, targets)) / len(targets)

class APMGSRN:
    """Adaptively Placed Multi-Grid SRN architecture."""
    def __init__(self):
        self.lr = 0.01
    def forward(self, points):
        # Simplified forward pass for pseudocode completeness
        return [p * self.lr for p in points]

def train_brick(data_subset, initial_lr, max_iterations):
    """Trains a single brick model with the specified early stopping criteria."""
    model = APMGSRN()
    model.lr = initial_lr
    patience = 0
    trigger_count = 0
    best_loss = float('inf')
    
    for iteration in range(max_iterations):
        points, targets = sample_points(data_subset)
        predictions = model.forward(points)
        loss = calculate_loss(predictions, targets)
        
        # 0.01% decrease threshold
        if loss < best_loss * 0.9999:
            best_loss = loss
            patience = 0
        else:
            patience += 1
            if patience >= 500:
                model.lr /= 10.0
                patience = 0
                trigger_count += 1
                if trigger_count >= 3:
                    break
    return model

def train_domain_decomposition(volume_data, brick_grid_shape, gpu_ids):
    """Dispatches training jobs across available GPUs."""
    jobs = []
    for i in range(brick_grid_shape[0]):
        for j in range(brick_grid_shape[1]):
            for k in range(brick_grid_shape[2]):
                brick_id = (i, j, k)
                extent = calculate_brick_extent(brick_id, brick_grid_shape, volume_data.total_extent)
                data_subset = volume_data.crop(extent)
                jobs.append((brick_id, data_subset))
                
    available_gpus = list(gpu_ids)
    trained_models = {}
    
    # Conceptual parallel execution across GPU pool
    while jobs and available_gpus:
        gpu_id = available_gpus.pop(0)
        brick_id, data_subset = jobs.pop(0)
        
        # In practice, this runs asynchronously on the assigned GPU
        model = train_brick(data_subset, initial_lr=0.01, max_iterations=10000)
        trained_models[brick_id] = model
        available_gpus.append(gpu_id)
        
    return trained_models

def infer_domain_decomposition(query_point, trained_models, spatial_hash):
    """Infers volume properties for a query point using spatial hashing."""
    brick_id = spatial_hash(query_point)
    model = trained_models[brick_id]
    result = model.forward([query_point])
    return result
```

**Parameters:**
*   **Brick Grid Shape ($I \times J \times K$):** Determines the number of models [S1].
*   **Initial Learning Rate:** Not given in the sources [G].
*   **Max Iterations:** Not given in the sources [G].
*   **Plateau Threshold:** 0.01% decrease in $L_{rec}$ [S1].
*   **Plateau Patience:** 500 iterations [S1].
*   **LR Reduction Factor:** 10 [S1].
*   **Max LR Reductions:** 3 [S1].

**Edge Cases and Pitfalls:**
*   **Inference Search:** Inference requires finding the correct model for a query point, which is a search problem mitigated by spatial hashing [S1].
*   **Sparse Data:** Regions with sparse data may converge quickly, allowing early stopping (down to 20 seconds per model) [S1].
*   **Ghost Cell Extents:** Pre-calculating network extents must explicitly include ghost cells when using the overlapping variant to ensure boundary continuity [S1].

## Subtopics

### [Domain Decomposition with Ghost Cells](11133-domain-decomposition-with-ghost-cells.md)
Extends this method by adding overlapping "ghost cells" at brick boundaries to solve visible seams and boundary artifacts. It trades a slight reduction in learning capacity for improved visual continuity.

## Operating characteristics
*   **Time Complexity:** Training time is reduced significantly by parallelizing across GPUs and using early stopping. In the cited experiment, a 450 GB volume was fitted in under 7 minutes on 8 GPUs [S1]. Sparse regions can be trained in as little as 20 seconds per model [S1].
*   **Memory Complexity:** Each GPU only loads the data for its assigned brick, avoiding out-of-core I/O overhead. This requires sufficient GPU memory to hold one brick's data and model parameters [S1].
*   **Scaling:** Scales with the number of available GPUs. Assumes a compute node with multiple GPUs [S1].
*   **Strengths:** Fits very large volumes efficiently; increases reconstruction accuracy over a single model with the same total parameters; reduces I/O overhead [S1].
*   **Limitations:** Inference requires a spatial search (mitigated by hashing) [S1].
*   **When to use:** Large-scale volumes that do not fit in a single GPU's memory; scenarios where training time is critical and parallel hardware is available [S1].
*   **When not to use:** Small volumes that fit in a single GPU's memory [G].

## Background
Originated from the work of Skylar W. Wurster at The Ohio State University, published in *IEEE Transactions on Visualization and Computer Graphics* in 2023 [S1]. The method builds on the Adaptively Placed Multi-Grid Scene Representation Network (APMGSRN) architecture as a building block for the training strategy [S1]. It addresses the limitations of single-model SRNs in handling large-scale data by distributing the training load across multiple GPUs [S1].

## Sources
[S1] Adaptively Placed Multi-Grid Scene Representation Networks for Large-Scale Data Visualization, https://doi.org/10.1109/tvcg.2023.3327194, fetched 2026-10-26
