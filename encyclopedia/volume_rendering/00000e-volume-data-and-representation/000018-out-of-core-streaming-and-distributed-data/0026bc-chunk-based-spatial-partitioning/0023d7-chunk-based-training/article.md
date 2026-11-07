# Chunk-based Training

Also called: Parallel chunk-based training

## Overview
Chunk-based Training is a strategy for optimizing large-scale volumetric datasets by partitioning the global volume into discrete chunks, enabling parallel optimization across multiple GPUs [S1, unofficial]. It differs from single-GPU training by scaling the coordinate space and training units according to the number of chunks, allowing efficient handling of massive datasets like Hemibrain [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Large-scale volumetric dataset (e.g., BigBrain, Woodbranch, Hemibrain).
- Output: Optimized neural volume representation (e.g., ellipsoid primitives).

**Data Structures:**
- **Chunk Grid:** A logical grid of $n_{chunk}$ chunks. The global volume is mapped to a normalized coordinate space.
- **Subvolumes:** Each training unit (full volume or individual chunk) is further partitioned into voxelized cubic subvolumes for optimization [S1, unofficial].

**Steps:**
1. **Coordinate Space Normalization:**
   - For small datasets, the full volume is optimized in a normalized $[-1, 1]^3$ coordinate space [S1, unofficial].
   - For large datasets, the training space is scaled according to the number of chunks $n_{chunk}$. The range becomes $[-\frac{1}{2\sqrt[3]{n_{chunk}}}, \frac{1}{2\sqrt[3]{n_{chunk}}}]^3$.
     - Example: For $n_{chunk}=8$, range is $[-0.5, 0.5]^3$.
     - Example: For $n_{chunk}=64$, range is $[-0.25, 0.25]^3$.
     - Example: For $n_{chunk}=125$, range is $[-0.2, 0.2]^3$ [S1, unofficial].

2. **Chunk Assignment:**
   - The global volume is decomposed into $n_{chunk}$ chunks. Each chunk is assigned to a specific GPU [S1, unofficial].

3. **Subvolume Partitioning:**
   - During optimization, the training unit (full volume or individual chunk) is partitioned into voxelized cubic subvolumes [S1, unofficial].
   - **Subvolume Side Length:** Set to approximately half the longest axis of the training unit. For flat volumes (e.g., Neuron), the smallest axis is used instead [S1, unofficial].

4. **Parallel Optimization:**
   - Each GPU optimizes its assigned chunk(s) using the subvolumes. The optimization process involves structure-aware primitive learning and per-primitive ray sampling [S1, unofficial].

**Parameters:**
- $n_{chunk}$: Number of chunks.
  - BigBrain: 8 chunks [S1, unofficial].
  - Woodbranch: 8 chunks [S1, unofficial].
  - Hemibrain (vDeltaA): 64 chunks [S1, unofficial].
  - Hemibrain (lLN2P): 125 chunks [S1, unofficial].
- Subvolume side length: ~0.5 * max(axis lengths) of training unit [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Flat Volumes:** For non-cubic volumes (e.g., Neuron), the subvolume side length is determined by the smallest axis to maintain cubic subvolumes [S1, unofficial].
- **Precision:** The scaling of the coordinate space must be precise to avoid boundary artifacts between chunks [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chunk_based_training(dataset, n_chunk, num_gpus):
    # Step 1: Determine coordinate space scaling
    if n_chunk == 1:
        coord_range = [-1.0, 1.0]
    else:
        scale = 1.0 / (2.0 * (n_chunk ** (1.0/3.0)))
        coord_range = [-scale, scale]
    
    # Step 2: Partition volume into chunks
    chunks = partition_volume(dataset, n_chunk)
    
    # Step 3: Assign chunks to GPUs
    gpu_assignments = assign_to_gpus(chunks, num_gpus)
    
    # Step 4: Parallel optimization
    results = []
    for gpu_id, gpu_chunks in gpu_assignments.items():
        # Determine subvolume size for this chunk
        training_unit = gpu_chunks
        max_axis = max(training_unit.dimensions)
        subvolume_side = max_axis / 2.0
        
        # For flat volumes, use smallest axis
        min_axis = min(training_unit.dimensions)
        if is_flat_volume(training_unit):
            subvolume_side = min_axis / 2.0
            
        subvolumes = partition_into_subvolumes(training_unit, subvolume_side)
        
        # Optimize chunk using subvolumes
        optimized_chunk = optimize_chunk(subvolumes, coord_range)
        results.append(optimized_chunk)
        
    return merge_results(results)
```

## Subtopics

### Subvolume Configuration
A variant where the training unit (full volume or individual chunk) is partitioned into voxelized cubic subvolumes during optimization. The subvolume side length is set to approximately half the longest axis of the training unit. For flat volumes, the smallest axis is used instead [S1, unofficial]. This configuration helps manage memory and computational load during the optimization process.

## Operating characteristics
- **Time Complexity:** Scales with the number of chunks and GPUs. Parallel optimization reduces total training time compared to single-GPU training [S1, unofficial].
- **Memory Complexity:** Each GPU only needs to store the data for its assigned chunks and subvolumes, reducing peak memory usage per GPU [S1, unofficial].
- **Performance:**
  - BigBrain and Woodbranch: Trained sequentially on a single NVIDIA RTX A6000 GPU [S1, unofficial].
  - Hemibrain (vDeltaA): Trained on 4 NVIDIA RTX A6000 GPUs [S1, unofficial].
  - Hemibrain (lLN2P): Trained on 5 NVIDIA RTX A6000 GPUs [S1, unofficial].
- **Strengths:** Enables training of massive datasets that do not fit into a single GPU's memory. Parallel processing accelerates optimization [S1, unofficial].
- **Limitations:** Requires careful coordination of chunk boundaries and coordinate space scaling. Performance may be limited by inter-GPU communication if chunks are not independent [S1, unofficial].

## Background
Chunk-based Training was introduced in the ESVR (3D Ellipsoid-based Sparse Volume Rendering) paper, which proposed structure-aware primitive learning and per-primitive ray sampling for large-scale volumetric data [S1, unofficial]. The method was developed to handle datasets like BigBrain, Woodbranch, and Hemibrain, which are too large for traditional single-GPU training approaches [S1, unofficial].

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
