# Chunk-based Training

Also called: Parallel chunk-based training

## Overview
Chunk-based Training is a strategy for optimizing large-scale volumetric datasets by partitioning the global domain into spatial chunks, enabling parallel optimization across multiple GPUs [S1, unofficial]. It distinguishes itself from sibling methods like [MPI-based Volume Partitioning](1093-mpi-based-volume-partitioning.md) by focusing on the training phase's data distribution and coordinate scaling rather than just inference-time rendering or streaming. It addresses the memory and computational limits of training implicit neural representations on massive volumes (e.g., BigBrain, Hemibrain) by breaking the problem into manageable, independent sub-problems [S1, unofficial].

## How it works
**Inputs:**
- Global volume domain $\Omega$.
- Number of chunks $n_{chunk}$ along each axis (assuming cubic chunks for simplicity, or distinct counts per axis).
- Hardware configuration (number of GPUs).

**Outputs:**
- Optimized neural network parameters for each chunk.

**Data Structures:**
- The global volume is conceptually divided into a grid of chunks. For a regular grid, the chunk index $\mathbf{k} = (k_x, k_y, k_z)$ for a voxel at global coordinate $\mathbf{p} = (x, y, z)$ is calculated as:
  $$
  k_x = \lfloor x / W_x \rfloor, \quad k_y = \lfloor y / W_y \rfloor, \quad k_z = \lfloor z / W_z \rfloor
  $$
  where $W_x, W_y, W_z$ are the chunk dimensions in voxels [Parent].
- Each chunk is treated as an independent training unit.

**Steps:**
1. **Coordinate Normalization:** The full volume is optimized in a normalized $[-1, 1]^3$ coordinate space [S1, unofficial].
2. **Coordinate Scaling:** For large volumes, the training space is scaled according to the number of chunks. If the volume is divided into $N$ chunks along an axis, the coordinate range for that axis is scaled to $[-0.5, 0.5]$ if $N=8$, $[-0.25, 0.25]$ if $N=64$, and $[-0.2, 0.2]$ if $N=125$ [S1, unofficial]. This ensures that the neural network focuses on local features within each chunk's bounds.
3. **Subvolume Partitioning (Optional Variant):** During optimization, each training unit (full volume or individual chunk) is partitioned into voxelized cubic subvolumes [S1, unofficial].
   - **Subvolume Side Length:** Set to approximately half the longest axis of the training unit. For flat volumes (e.g., Neuron), the smallest axis is used instead [S1, unofficial].
4. **Parallel Optimization:** Each chunk is assigned to a GPU. Training proceeds independently for each chunk using the scaled coordinates and subvolume sampling [S1, unofficial].

**Parameters:**
- $n_{chunk}$: Number of chunks. Typical values: 8 for BigBrain and Woodbranch, 64 for Hemibrain (vDeltaA), and 125 for Hemibrain (lLN2P) [S1, unofficial].
- Subvolume side length: Approximately half the longest axis of the training unit [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Flat Volumes:** For non-cubic volumes (e.g., Neuron), the subvolume configuration must adapt to the smallest axis to maintain appropriate sampling density [S1, unofficial].
- **Boundary Artifacts:** While the parent method [Chunk-based spatial partitioning](3862-chunk-based-spatial-partitioning.md) discusses overlap regions for rendering, training chunks are typically optimized independently. Discontinuities at chunk boundaries may require post-processing or specific boundary conditions during inference [Parent].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def chunk_based_training(volume_data, n_chunk, num_gpus):
    """
    Perform chunk-based training for a large volume.
    
    Args:
        volume_data: The full volumetric dataset.
        n_chunk: Number of chunks along each axis (assumed cubic grid).
        num_gpus: Number of available GPUs.
        
    Returns:
        A dictionary mapping chunk indices to trained model parameters.
    """
    # Calculate chunk dimensions
    # Assuming volume dimensions are V_x, V_y, V_z
    # W_x = V_x / n_chunk, etc.
    # This calculation depends on the specific volume dimensions not provided in sources.
    # [G] Placeholder for actual dimension calculation based on volume_data.shape
    
    # Determine coordinate scaling factor
    # Based on n_chunk: 8 -> 0.5, 64 -> 0.25, 125 -> 0.2
    if n_chunk == 8:
        scale = 0.5
    elif n_chunk == 64:
        scale = 0.25
    elif n_chunk == 125:
        scale = 0.2
    else:
        # Not given in the sources for other n_chunk values.
        # [G] Default to 1.0 / n_chunk^(1/3) or similar scaling if needed.
        scale = 1.0 / (n_chunk ** (1/3))
        
    chunk_models = {}
    
    # Iterate over all chunks
    for k_x in range(n_chunk):
        for k_y in range(n_chunk):
            for k_z in range(n_chunk):
                chunk_index = (k_x, k_y, k_z)
                
                # Extract chunk data
                # [G] Implementation detail: slice volume_data based on chunk indices
                chunk_data = extract_chunk(volume_data, k_x, k_y, k_z)
                
                # Initialize model for this chunk
                model = initialize_model()
                
                # Define coordinate transformation
                def transform_coords(coords):
                    # Normalize to [-1, 1] then scale
                    normalized = (coords - 0.5) * 2.0  # [-1, 1]
                    scaled = normalized * scale        # Scaled range
                    return scaled
                
                # Train model on chunk data
                # [G] Standard training loop: sample rays, compute loss, backpropagate
                train_model(model, chunk_data, transform_coords)
                
                chunk_models[chunk_index] = model.parameters()
                
    return chunk_models
```

## Subtopics

### Subvolume Configuration
A variant where each training unit (full volume or individual chunk) is further partitioned into voxelized cubic subvolumes during optimization [S1, unofficial]. The subvolume side length is set to approximately half the longest axis of the training unit. For flat volumes such as Neuron, the smallest axis is used instead [S1, unofficial]. This allows for more granular sampling and potentially better handling of anisotropic data.

## Operating characteristics
**Time and Memory Complexity:**
- Memory usage is reduced per GPU by a factor of $n_{chunk}^3$ compared to training the full volume on a single GPU [S1, unofficial].
- Training time is reduced by parallelizing across GPUs, assuming linear scaling with the number of GPUs [S1, unofficial].

**Measured Performance:**
- Hardware: All training performed on NVIDIA RTX A6000 GPUs [S1, unofficial].
- BigBrain and Woodbranch were trained sequentially on a single A6000 (8 chunks) [S1, unofficial].
- Hemibrain (vDeltaA) used 4 GPUs (64 chunks) [S1, unofficial].
- Hemibrain (lLN2P) used 5 GPUs (125 chunks) [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Enables training on extremely large volumes that would not fit in GPU memory. Parallelizes the optimization process.
- **Limitations:** Potential discontinuities at chunk boundaries. Requires careful coordinate scaling to ensure the neural network learns local features correctly.

**When to Use:**
- Use when the volume is too large for a single GPU.
- Use when parallel training resources (multiple GPUs) are available.

**When Not to Use:**
- Use when the volume fits in a single GPU's memory and parallelization overhead is not justified.
- Use when boundary continuity is critical and post-processing is not feasible.

## Background
Chunk-based Training was introduced in the context of 3D Ellipsoid-based Sparse Volume Rendering (ESVR) to handle large-scale datasets like BigBrain and Hemibrain [S1, unofficial]. It builds on the principles of chunk-based spatial partitioning [Parent] to facilitate efficient training of implicit neural representations.

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
