# Neural Voxel Fields

Also called: VoxelNeRF

## Overview
Neural Voxel Fields replace the single global implicit function of standard NeRF with voxel-bounded implicit fields, assigning a trainable embedding to each vertex of a 3D grid to model RGB colors and volume density [S1]. This approach enables faster training than fully implicit methods by leveraging spatial locality, though it faces challenges when scaling to higher resolutions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D spatial position $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d}$.
*   **Output:** Volume density $\sigma$ and directional emitted color $\mathbf{c}$.

**Data Structures:**
The scene is represented by a grid of $n \times m$ voxels (in the source description, though typically 3D $N_x \times N_y \times N_z$ is implied for 3D space). Each voxel $i,j$ has a trainable embedding vector $\mathbf{v}_{i,j} \in \mathbb{R}^k$. In practice, a sparse voxel structure is used where nodes exist only in non-empty areas of 3D space [S1]. The model parameters are $\Theta$ (decoder weights) and $V$ (the set of voxel embeddings) [S1].

**Algorithm:**
1.  **Query:** For a given position $\mathbf{x}$, identify the six nearest voxels from the grid [S1].
2.  **Aggregate:** Aggregate the information from these six nearest voxel embeddings. The source states this aggregation is performed by a small implicit decoder $F_{\text{VoxelNeRF}}$ [S1].
3.  **Decode:** The decoder $F_{\text{VoxelNeRF}}(\mathbf{x}, \mathbf{d}; \Theta, V)$ maps the aggregated voxel information and the input coordinates to the output signal $(\mathbf{c}, \sigma)$ [S1].
4.  **Render:** Use the standard NeRF volume rendering procedure (ray marching) to synthesize images from the predicted $\sigma$ and $\mathbf{c}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_neural_field_query(x, d, grid_embeddings, decoder_weights):
    """
    Query the Neural Voxel Field for density and color.
    
    Args:
        x: 3D position vector (R^3)
        d: Viewing direction vector
        grid_embeddings: Sparse grid of voxel embeddings v_{i,j} in R^k
        decoder_weights: Parameters Theta for the small implicit decoder
        
    Returns:
        sigma: Volume density
        c: Color vector
    """
    # 1. Identify the six nearest voxels to position x in the grid
    # Note: The source specifies "six nearest voxels" [S1].
    # Implementation detail: This typically involves trilinear interpolation 
    # or aggregation from the 8 corner voxels of the containing cell, 
    # or a specific neighborhood search. The source does not specify 
    # the exact aggregation math, only that information is aggregated [S1].
    nearest_voxels = find_six_nearest_voxels(x, grid_embeddings)
    
    # 2. Aggregate information from the nearest voxels
    # The source states a small implicit decoder aggregates this info [S1].
    aggregated_features = aggregate_voxel_features(nearest_voxels)
    
    # 3. Decode to density and color
    # F_VoxelNeRF(x, d; Theta, V) = (c, sigma) [S1]
    sigma, c = decoder_forward(aggregated_features, x, d, decoder_weights)
    
    return sigma, c
```

**Edge Cases and Pitfalls:**
*   **Scaling:** Scaling the solution to higher resolutions is problematic for voxel representations due to memory and computational constraints [S1].
*   **Sparsity:** The method relies on sparse voxel structures for non-empty areas; handling empty space requires careful initialization or masking [S1].

## Subtopics

### DVGO Explicit Volumetric Representation
DVGO represents geometry with explicit density voxel grids $V^{(\text{density})} \in \mathbb{R}^{1 \times N_x \times N_y \times N_z}$ [S2, unofficial]. It uses a hybrid architecture for color prediction comprising a shallow MLP parameterized by $\Theta$ and a feature voxel grid $V^{(\text{feat})} \in \mathbb{R}^{C \times N_x \times N_y \times N_z}$ [S2, unofficial]. Given a 3D position $\mathbf{p}$ and viewing direction $\mathbf{v}$, volume density $\sigma$ and color $\mathbf{c}$ are estimated as:
$$ \sigma = \text{interp}(\mathbf{p}, V^{(\text{density})}) $$
$$ \mathbf{c} = \text{MLP}_\Theta(\text{interp}(\mathbf{p}, V^{(\text{feat})}), \mathbf{p}, \mathbf{v}) $$
Positional encoding is applied to both $\mathbf{p}$ and $\mathbf{v}$ as in NeRF [S2, unofficial].

## Operating characteristics
*   **Performance:** Voxel representations allow for faster training compared to vanilla NeRF [S1].
*   **Scaling:** Scaling to higher resolutions is problematic [S1].
*   **Strengths:** Faster training than fully implicit methods; supports sparse structures for non-empty areas [S1].
*   **Limitations:** Resolution scaling issues; memory usage grows with grid resolution [S1].

## Background
Neural Voxel Fields were introduced as an alternative to the fully implicit architecture of NeRF, which must be trained on each object separately and suffers from slow optical ray marching for high-resolution imagery [S1]. The method assigns voxel embeddings at each vertex and uses a small implicit decoder to aggregate information, aiming to balance the sharpness of NeRF with the efficiency of voxel-based methods [S1].

## Sources
[S1] MultiPlaneNeRF: Neural radiance field with non-trainable representation, https://doi.org/10.1016/j.eswa.2025.127350, fetched 2026-10-29
[S2] Voxurf: Voxel-based Efficient and Accurate Neural Surface Reconstruction, https://doi.org/10.48550/arxiv.2208.12697, fetched 2026-10-26 (unofficial)
