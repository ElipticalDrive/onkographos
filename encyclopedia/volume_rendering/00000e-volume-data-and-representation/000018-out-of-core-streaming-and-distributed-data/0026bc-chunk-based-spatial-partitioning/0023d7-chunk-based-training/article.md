# Chunk-based Training

Also called: Parallel chunk-based training

## Overview
Chunk-based training is a strategy for optimizing neural volume representations on large-scale datasets by partitioning the global volume into fixed-size chunks, allowing independent optimization across multiple GPUs [S1, unofficial]. It differs from single-GPU training by enabling parallel processing of spatially distinct regions, and from distributed MPI-based partitioning by relying on independent chunk optimization rather than synchronized global gradient aggregation [1093-mpi-based-volume-partitioning.md].

## How it works
**Inputs and Outputs:**
- Input: A large-scale volumetric dataset (e.g., histology, microscopy) and a target number of chunks $n_{chunk}$.
- Output: A set of trained neural volume models, one per chunk, representing the local density and color fields.

**Data Structures:**
- **Chunk Grid:** A 3D grid of dimensions $N_x \times N_y \times N_z$ such that $N_x N_y N_z = n_{chunk}$.
- **Local Coordinate Space:** Each chunk $c$ is mapped to a normalized local coordinate space. The global volume is normalized to $[-1, 1]^3$. For a chunk at grid index $(i, j, k)$, the local space bounds are scaled by $1/\sqrt[3]{n_{chunk}}$ (assuming cubic chunks) or specific scaling factors if non-cubic.
    - Example: For 8 chunks ($2 \times 2 \times 2$), the space is $[-0.5, 0.5]^3$.
    - Example: For 64 chunks ($4 \times 4 \times 4$), the space is $[-0.25, 0.25]^3$.
    - Example: For 125 chunks ($5 \times 5 \times 5$), the space is $[-0.2, 0.2]^3$ [S1, unofficial].

**Steps:**
1. **Partitioning:** Divide the global volume into $n_{chunk}$ uniform chunks.
2. **Coordinate Mapping:** For each chunk, define a local coordinate system scaled to fit within the global normalized space. A global point $\mathbf{p}$ maps to a local point $\mathbf{p}_{local}$ for chunk $(i,j,k)$ via:
   $$ \mathbf{p}_{local} = (\mathbf{p} - \mathbf{offset}_{chunk}) \times \text{scale} $$
   where $\mathbf{offset}_{chunk}$ is the chunk's center in global space and $\text{scale}$ is the inverse of the chunk's physical extent in normalized units [S1, unofficial].
3. **Subvolume Configuration (Optional):** If using subvolume configuration, partition the training unit (chunk or full volume) into voxelized cubic subvolumes. The subvolume side length is set to approximately half the longest axis of the training unit. For flat volumes (e.g., Neuron), the smallest axis is used instead [S1, unofficial].
4. **Optimization:** Train the neural representation for each chunk independently.
   - For small datasets (e.g., BigBrain, Woodbranch), training may be sequential on a single GPU [S1, unofficial].
   - For large datasets (e.g., Hemibrain), training is parallelized across multiple GPUs, one per chunk or group of chunks [S1, unofficial].
5. **Assembly:** The final volume is represented by the collection of chunk-specific models. Rendering queries map the ray to the appropriate chunk's local space.

**Parameters:**
- $n_{chunk}$: Number of chunks. Typical values: 8 (BigBrain, Woodbranch), 64 (Hemibrain vDeltaA), 125 (Hemibrain lLN2P) [S1, unofficial].
- Subvolume side length: $\approx 0.5 \times \max(\text{axis lengths})$ of the training unit [S1, unofficial].

**Edge Cases:**
- **Flat Volumes:** For anisotropic datasets (e.g., Neuron), the subvolume configuration adapts to the smallest axis to maintain aspect ratio balance [S1, unofficial].
- **Boundary Discontinuities:** As with other chunk-based methods, naive training may lead to discontinuities at chunk boundaries. Overlap regions or boundary handling techniques (e.g., replicated voxels) may be required during rendering, though the training itself is independent [3862-chunk-based-spatial-partitioning.md].

## Subtopics

### Subvolume Configuration
A variant where the training unit (full volume or individual chunk) is further partitioned into voxelized cubic subvolumes during optimization. The subvolume side length is set to approximately half the longest axis of the training unit. For flat volumes, the smallest axis is used. This allows for more granular control over the optimization process and potentially better handling of anisotropic data [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(n_{chunk} \times T_{chunk})$, where $T_{chunk}$ is the time to train one chunk. With $G$ GPUs, time is approximately $O(\frac{n_{chunk}}{G} \times T_{chunk})$ if perfectly parallelized.
- **Memory Complexity:** $O(V_{chunk})$ per GPU, where $V_{chunk}$ is the memory required for one chunk's model and data. Total memory is $O(V_{total})$, but distributed across GPUs.
- **Performance:** On NVIDIA RTX A6000 GPUs, BigBrain and Woodbranch (smaller datasets) were trained sequentially on a single GPU. Hemibrain (vDeltaA) used 4 GPUs, and Hemibrain (lLN2P) used 5 GPUs [S1, unofficial].
- **Strengths:** Enables training on datasets that do not fit in a single GPU's memory. Parallelizes the training process.
- **Limitations:** Potential discontinuities at chunk boundaries. Requires careful coordinate mapping. Not suitable for methods requiring global context during training.

## Background
Chunk-based training was introduced in the context of 3D Ellipsoid-based Sparse Volume Rendering (ESVR) to handle large-scale volumetric datasets like Hemibrain [S1, unofficial]. It builds on the principles of chunk-based spatial partitioning, leveraging data locality and independent processing [3862-chunk-based-spatial-partitioning.md].

## Sources
[S1] ESVR: 3D Ellipsoid-based Sparse Volume Rendering via Structure-aware Primitive Learning and Per-primitive Ray Sampling, https://arxiv.org/abs/2608.05564v1, fetched 2026-10-26 (unofficial)
