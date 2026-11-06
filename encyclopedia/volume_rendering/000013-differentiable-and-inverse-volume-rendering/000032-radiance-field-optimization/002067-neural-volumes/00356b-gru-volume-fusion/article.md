# GRU Volume Fusion

Also called: Fusing Volumes for Global Reconstruction

## Overview
GRU Volume Fusion is a recurrent module that incrementally aggregates per-frame local feature volumes into a single global radiance field representation [S1]. It solves the problem of maintaining a consistent, memory-efficient global scene model while processing a stream of sparse local reconstructions, distinguishing itself from static fusion methods by using Gated Recurrent Units (GRUs) with sparse 3D convolutions to adaptively update only the relevant voxels [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The local sparse feature volume from the current frame $t$, denoted $\mathcal{V}_t$, and the global feature volume from the previous frame $t-1$, denoted $\mathcal{V}_{t-1}^g$ [S1].
*   **Output:** The updated global feature volume $\mathcal{V}_t^g$ [S1].

**Data Structures:**
*   $\mathcal{V}_t$ and $\mathcal{V}_{t-1}^g$ are sparse 3D voxel grids. The exact bit-packing or sparse storage format (e.g., hash table, octree) is not specified in the sources, but the operations are defined element-wise on the voxel coordinates covered by $\mathcal{V}_t$ [S1].
*   The networks $M_z, M_r, M_t$ are deep neural networks composed of sparse 3D convolution layers [S1].

**Parameters:**
*   Activation functions: Sigmoid for $M_z$ and $M_r$; Tanh for $M_t$ [S1].
*   Typical values for network depth or kernel sizes are not given in the sources [S1].

**Step-by-Step Method:**
1.  **Concatenation:** For every voxel coordinate present in the local volume $\mathcal{V}_t$, concatenate the feature vector from the previous global volume $\mathcal{V}_{t-1}^g$ and the current local volume $\mathcal{V}_t$. Let $[\cdot, \cdot]$ denote this concatenation operation [S1].
2.  **Compute Update Gate ($z_t$):** Apply network $M_z$ to the concatenated features. The output is the update gate $z_t$, passed through a sigmoid activation function [S1].
    $$z_t = M_z([\mathcal{V}_{t-1}^g, \mathcal{V}_t])$$
3.  **Compute Reset Gate ($r_t$):** Apply network $M_r$ to the concatenated features. The output is the reset gate $r_t$, passed through a sigmoid activation function [S1].
    $$r_t = M_r([\mathcal{V}_{t-1}^g, \mathcal{V}_t])$$
4.  **Compute Candidate Volume ($\tilde{\mathcal{V}}_t^g$):** Element-wise multiply the previous global volume $\mathcal{V}_{t-1}^g$ by the reset gate $r_t$. Concatenate this result with the current local volume $\mathcal{V}_t$. Apply network $M_t$ to this concatenation. The output is the candidate global volume $\tilde{\mathcal{V}}_t^g$, passed through a tanh activation function [S1].
    $$\tilde{\mathcal{V}}_t^g = M_t([r_t * \mathcal{V}_{t-1}^g, \mathcal{V}_t])$$
5.  **Update Global Volume ($\mathcal{V}_t^g$):** Compute the new global volume using the standard GRU update rule: blend the previous global volume and the candidate volume using the update gate $z_t$.
    $$\mathcal{V}_t^g = (1 - z_t) * \mathcal{V}_{t-1}^g + z_t * \tilde{\mathcal{V}}_t^g$$
6.  **Sparse Application:** Steps 1–5 are applied **only** to the voxels covered by the local volume $\mathcal{V}_t$. All other voxels in the global volume $\mathcal{V}_{t-1}^g$ that are not covered by $\mathcal{V}_t$ remain unchanged in $\mathcal{V}_t^g$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gru_volume_fusion(V_prev_global, V_local, M_z, M_r, M_t):
    """
    Incrementally fuses local feature volume into global volume using GRU.
    
    Args:
        V_prev_global: Sparse 3D volume (dict or sparse tensor) representing V_{t-1}^g.
        V_local: Sparse 3D volume (dict or sparse tensor) representing V_t.
        M_z: Sparse 3D CNN network for update gate.
        M_r: Sparse 3D CNN network for reset gate.
        M_t: Sparse 3D CNN network for candidate volume.
        
    Returns:
        V_curr_global: Updated sparse 3D volume V_t^g.
    """
    # Initialize current global volume as a copy of previous
    # Note: In a sparse implementation, this is a shallow copy or reference
    # to unchanged voxels. We will explicitly update only covered voxels.
    V_curr_global = V_prev_global.copy()
    
    # Identify voxel coordinates covered by the local volume
    covered_coords = V_local.keys()
    
    for coord in covered_coords:
        # Get feature vectors for this coordinate
        # If a coordinate is missing in a sparse volume, it is treated as zero
        v_prev = V_prev_global.get_feature(coord) # Returns vector or zero
        v_local = V_local.get_feature(coord)      # Returns vector or zero
        
        # Step 1: Concatenate
        concat_input = concatenate(v_prev, v_local)
        
        # Step 2: Compute Update Gate z_t
        # M_z outputs sigmoid activation
        z_t = M_z.forward(concat_input)
        
        # Step 3: Compute Reset Gate r_t
        # M_r outputs sigmoid activation
        r_t = M_r.forward(concat_input)
        
        # Step 4: Compute Candidate Volume
        # Element-wise multiplication of reset gate and previous global feature
        reset_prev = elementwise_multiply(r_t, v_prev)
        
        # Concatenate reset-modified previous feature with local feature
        candidate_input = concatenate(reset_prev, v_local)
        
        # M_t outputs tanh activation
        v_candidate = M_t.forward(candidate_input)
        
        # Step 5: Update Global Volume
        # Element-wise operations
        # (1 - z_t) * v_prev + z_t * v_candidate
        v_new = elementwise_multiply(elementwise_multiply(subtract(1, z_t), v_prev), 1) + \
                elementwise_multiply(z_t, v_candidate)
        
        # Step 6: Write back to global volume
        V_curr_global.set_feature(coord, v_new)
        
    return V_curr_global
```

**Edge Cases and Pitfalls:**
*   **Uncovered Voxels:** The method explicitly states that voxels not covered by $\mathcal{V}_t$ are kept unchanged. Implementers must ensure that the sparse data structure supports efficient iteration over only the active keys and does not inadvertently zero out or delete inactive voxels [S1].
*   **Precision:** The use of sigmoid and tanh activations requires careful handling of gradient flow during backpropagation, particularly if the sparse convolution layers are deep [S1].
*   **Coordinate Alignment:** The local volume $\mathcal{V}_t$ and global volume $\mathcal{V}_{t-1}^g$ must share the same coordinate system and voxel resolution for the element-wise operations to be valid [S1].

## Subtopics

### Voxel Grid Decoders
Decodes the global feature volume into RGB and alpha values for rendering. GRU Volume Fusion provides the feature volume input to this decoder. [Voxel Grid Decoders](2219-voxel-grid-decoders.md)

### RenderNet Architecture
The specific neural network architecture used for the sparse 3D convolutions ($M_z, M_r, M_t$) and the final decoding. GRU Volume Fusion relies on these architectures for feature extraction. [RenderNet Architecture](6691-rendernet-architecture.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active voxels in the local volume $\mathcal{V}_t$, as the networks are applied only to covered voxels [S1].
*   **Memory Complexity:** Proportional to the size of the global volume $\mathcal{V}^g$, which grows as new areas are scanned, but remains sparse [S1].
*   **Strengths:** Adaptive improvement of global reconstruction by filling holes and refining features while maintaining consistency; efficient due to sparse application [S1].
*   **Limitations:** Requires sequential processing of frames; performance depends on the accuracy of the local volume reconstruction $\mathcal{V}_t$ [S1].
*   **When to use:** Large-scale scene reconstruction where incremental updates are needed and global consistency is critical [S1].
*   **When not to use:** Real-time rendering of static scenes where a single global volume can be computed offline without incremental updates [S1].

## Background
GRU Volume Fusion was introduced in the paper "NeRFusion: Fusing Radiance Fields for Large-Scale Scene Reconstruction" by [S1]. It builds on the Neural Volumes framework, which optimizes a deep 3D convolutional network to predict a discretized RGB$\alpha$ voxel grid and a 3D warp grid [Parent]. The method leverages Gated Recurrent Units (GRUs) [10] to handle the temporal aspect of multi-view reconstruction, allowing the network to learn to recurrently fuse per-frame local reconstructions into a high-quality global radiance field [S1].

## Sources
[S1] NeRFusion: Fusing Radiance Fields for Large-Scale Scene Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00537, fetched 2026-10-26
