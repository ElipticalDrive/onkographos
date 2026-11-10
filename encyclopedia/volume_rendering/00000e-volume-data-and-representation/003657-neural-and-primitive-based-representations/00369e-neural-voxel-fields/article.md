# Neural Voxel Fields

Also called: VoxelNeRF

## Overview
Neural Voxel Fields replace the single implicit function of standard MLP-based radiance fields with a voxel-bounded implicit representation, where trainable embeddings are assigned to grid vertices and aggregated by a small decoder to predict color and density [S1]. This approach decouples spatial indexing from the neural query, enabling faster training than vanilla NeRF while avoiding the memory explosion of dense 3D feature grids [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D spatial coordinate $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d}$.
*   **Output:** Volume density $\sigma$ and emitted color $\mathbf{c}$.

**Data Structures:**
The scene is represented by a voxel grid $V$.
*   **Structure:** A grid of dimensions $n \times m$ (or $N_x \times N_y \times N_z$ in 3D).
*   **Content:** Each vertex (or voxel center) $i,j$ holds a trainable embedding vector $\mathbf{v}_{i,j} \in \mathbb{R}^k$.
*   **Sparsity:** In practice, a sparse voxel structure is often used, where nodes exist only in non-empty areas of 3D space to mitigate memory costs [S1].

**Algorithm:**
1.  **Query:** For a given position $\mathbf{x}$, identify the six nearest voxels from the grid surrounding $\mathbf{x}$ [S1].
2.  **Aggregation:** Aggregate the information from these six nearest voxel embeddings. The specific interpolation method (e.g., trilinear) is implied by "aggregate information" but not explicitly defined in the source text beyond the nearest-neighbor selection [S1].
3.  **Decoding:** Pass the aggregated features and the viewing direction $\mathbf{d}$ into a small implicit decoder network $F_{\text{VoxelNeRF}}$ parameterized by weights $\Theta$ and the voxel representation $V$:
    $$ F_{\text{VoxelNeRF}}(\mathbf{x}, \mathbf{d}; \Theta, V) = (\mathbf{c}, \sigma) $$
4.  **Rendering:** Use the predicted $\sigma$ and $\mathbf{c}$ in a standard volume rendering pipeline (ray marching) to synthesize images, using the same loss function as vanilla NeRF [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_neural_voxel_field(x, d, voxel_grid, decoder_weights):
    """
    Query a Neural Voxel Field for density and color.
    
    Args:
        x: 3D position vector (R^3).
        d: Viewing direction vector.
        voxel_grid: Sparse or dense grid of embeddings V.
        decoder_weights: Parameters Theta for the small implicit decoder.
        
    Returns:
        sigma: Volume density.
        c: Color vector.
    """
    # 1. Identify the six nearest voxels surrounding position x
    # Note: The source [S1] specifies "six nearest voxels" but does not 
    # define the exact interpolation kernel (e.g., trilinear vs. nearest).
    # We assume a standard aggregation over the local neighborhood.
    neighbors = get_six_nearest_neighbors(x, voxel_grid)
    
    # 2. Aggregate information from the neighbors
    # [G] Standard implementation would use trilinear interpolation 
    # based on x's position relative to the voxel corners.
    aggregated_features = aggregate_embeddings(neighbors)
    
    # 3. Decode using the small implicit network
    # F_VoxelNeRF(x, d; Theta, V) = (c, sigma)
    sigma, c = decoder_network(aggregated_features, d, decoder_weights)
    
    return sigma, c
```

**Parameters:**
*   $k$: Dimension of the voxel embedding vector. Not given in the sources.
*   Grid resolution ($n \times m$ or $N_x \times N_y \times N_z$): Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Scaling:** Scaling the solution to higher resolutions is problematic for voxel representations due to memory constraints [S1].
*   **Sparsity:** While sparse structures help, the definition of "non-empty areas" requires a prior or initialization strategy [G].

## Subtopics

### DVGO Explicit Volumetric Representation
DVGO (Dynamic Voxel Grid Optimization) represents geometry with an explicit density voxel grid $V^{(\text{density})} \in \mathbb{R}^{1 \times N_x \times N_y \times N_z}$ [S2, unofficial]. It uses a hybrid architecture for color prediction comprising a shallow MLP parameterized by $\Theta$ and a feature voxel grid $V^{(\text{feat})} \in \mathbb{R}^{C \times N_x \times N_y \times N_z}$ [S2, unofficial]. Color $\mathbf{c}$ is estimated by interpolating features from $V^{(\text{feat})}$ at position $\mathbf{p}$, passing them through the MLP along with the position and viewing direction, and applying positional encoding [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Training is faster than vanilla NeRF [S1]. DVGO variants report approximately a 20x speedup compared to fully implicit methods like NeuS [S2, unofficial].
*   **Memory Complexity:** Scales with the number of voxels. Dense grids suffer from high memory usage at high resolutions [S1]. Sparse implementations mitigate this but add indexing overhead.
*   **Strengths:** Faster training than MLP-only fields; sharper renders than some implicit baselines; efficient for static scenes [S1].
*   **Limitations:** Poor scalability to high resolutions due to voxel memory footprint [S1].
*   **When to use:** When training speed is critical and the scene geometry is relatively coarse or can be represented sparsely.
*   **When not to use:** When high-resolution detail is required without significant memory optimization, or for dynamic scenes requiring frequent grid updates.

## Background
Neural Voxel Fields were introduced as an alternative to fully implicit neural radiance fields (NeRF) to address limitations in training speed and capacity [S1]. The concept builds on earlier work using voxel-bounded implicit fields [11, 17, 22, 28] cited in [S1]. The approach was notably discussed in the context of MultiPlaneNeRF and related works exploring non-trainable or hybrid representations [S1]. DVGO, a specific explicit variant, was proposed by Sun et al. (2022a) [S2, unofficial].

## Sources
[S1] MultiPlaneNeRF: Neural radiance field with non-trainable representation, https://doi.org/10.1016/j.eswa.2025.127350, fetched 2026-10-29
[S2] Voxurf: Voxel-based Efficient and Accurate Neural Surface Reconstruction, https://doi.org/10.48550/arxiv.2208.12697, fetched 2026-10-26 (unofficial)
