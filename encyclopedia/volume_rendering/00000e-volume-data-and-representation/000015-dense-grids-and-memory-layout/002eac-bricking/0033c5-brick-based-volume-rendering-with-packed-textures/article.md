# Brick-based volume rendering with packed textures

Also called: Index-Texture brick mapping, Packed-Texture volume rendering

## Overview
This method renders volumetric data larger than GPU memory by partitioning the volume into fixed-size bricks, storing only non-transparent bricks in a compact 1D or 2D "Packed-Texture," and using a separate "Index-Texture" to map 3D spatial coordinates to brick indices. It differs from sibling methods by avoiding dynamic rebinding or complex memory pools, instead relying on a static index lookup to access a dense, filtered-free texture region.

## How it works
**Inputs:**
- A 3D volume dataset $V$ of size $N_x \times N_y \times N_z$.
- A transfer function $T$ mapping scalar values to opacity/color.
- A brick dimension $B$ (e.g., $8 \times 8 \times 8$ voxels).

**Outputs:**
- A Packed-Texture containing only non-transparent bricks.
- An Index-Texture mapping 3D coordinates to brick indices.

**Data Structures:**
1.  **Index-Texture:** A 3D texture (or 2D texture with 3D coordinate mapping) where each texel stores an integer index. The resolution matches the number of bricks in each dimension: $N_{bx} = \lceil N_x / B \rceil$, $N_{by} = \lceil N_y / B \rceil$, $N_{bz} = \lceil N_z / B \rceil$.
2.  **Packed-Texture:** A 1D or 2D texture containing the voxel data of non-transparent bricks concatenated in a specific order (e.g., row-major).

**Steps:**
1.  **Brick Partitioning:** Divide the volume cube into bricks of size $B \times B \times B$. Calculate the total number of bricks $N_{bricks} = N_{bx} \times N_{by} \times N_{bz}$.
2.  **Transparency Culling:** For each brick, determine if it contains any non-transparent voxels using the transfer function. If a brick is fully transparent, it is excluded from the Packed-Texture.
3.  **Packed-Texture Construction:** Assign a unique compact index to each non-transparent brick. Store the voxel data of these bricks sequentially in the Packed-Texture.
4.  **Index-Texture Construction:** For each brick position $(i, j, k)$ in the global grid, store its compact index in the Packed-Texture into the corresponding location in the Index-Texture. If a brick is transparent, store a sentinel value (e.g., -1) or handle it as a special case.
5.  **Ray Casting Sampling:**
    -   For a sample point at global fractional coordinate $\mathbf{p} = (x, y, z) \in [0, 1]^3$:
    -   Calculate the scaled position: $\mathbf{p}_{scaled} = \mathbf{p} \times N_{blocks}$, where $N_{blocks} = (N_{bx}, N_{by}, N_{bz})$.
    -   Calculate the brick index: $\mathbf{idx} = \lfloor \mathbf{p}_{scaled} \rfloor$.
    -   Calculate the local fractional position within the brick: $\mathbf{p}_{local} = \mathbf{p}_{scaled} - \mathbf{idx}$.
    -   Access the Index-Texture at $\mathbf{idx}$ to retrieve the compact brick index $I$.
    -   If $I$ is valid, sample the Packed-Texture at the appropriate offset and $\mathbf{p}_{local}$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing
def build_index_and_packed_texture(volume, transfer_function, brick_size):
    N = volume.shape
    brick_dims = (N[0] // brick_size[0], N[1] // brick_size[1], N[2] // brick_size[2])
    
    packed_bricks = []
    index_texture_data = np.full(brick_dims, -1, dtype=np.int32)
    
    for i in range(brick_dims[0]):
        for j in range(brick_dims[1]):
            for k in range(brick_dims[2]):
                # Extract brick region
                x0, y0, z0 = i * brick_size[0], j * brick_size[1], k * brick_size[2]
                x1, y1, z1 = x0 + brick_size[0], y0 + brick_size[1], z0 + brick_size[2]
                brick = volume[x0:x1, y0:y1, z0:z1]
                
                # Check transparency
                if not is_fully_transparent(brick, transfer_function):
                    compact_idx = len(packed_bricks)
                    packed_bricks.append(brick)
                    index_texture_data[i, j, k] = compact_idx
    
    return index_texture_data, packed_bricks

# Rendering Step (Shader-like)
def sample_volume(p_frac, index_texture, packed_bricks, brick_size, brick_dims):
    # p_frac is in [0, 1]
    p_scaled = p_frac * brick_dims
    idx = np.floor(p_scaled).astype(int)
    p_local = p_scaled - idx
    
    # Clamp idx to valid range
    idx = np.clip(idx, 0, brick_dims - 1)
    
    compact_idx = index_texture[idx[0], idx[1], idx[2]]
    
    if compact_idx == -1:
        return 0.0, 0.0, 0.0, 0.0 # Transparent
    
    # Retrieve brick data
    brick = packed_bricks[compact_idx]
    
    # Sample brick data at p_local (0 to 1)
    # Note: Filtering must be handled manually or with nearest-neighbor if disabled
    return sample_brick(brick, p_local)
```

**Parameters:**
-   `brick_size`: Typical values are $8^3$ or $16^3$ voxels. Smaller bricks increase index texture size but improve granularity. Larger bricks reduce index texture overhead but may waste memory on partially transparent bricks.

**Edge Cases and Pitfalls:**
-   **Filtering:** When storing blocks into a regular, packed 3D texture, filtering must be disabled to avoid interpolation across brick boundaries. All filtering/interpolation must be done manually in the shader using the local fractional position [S1, unofficial].
-   **Fencepost Problem:** Care must be taken at brick boundaries to ensure samples do not incorrectly interpolate between adjacent bricks. The local position $\mathbf{p}_{local}$ must be clamped to $[0, 1]$ or handled explicitly to avoid accessing out-of-bounds data in the packed texture [S1, unofficial].
-   **Transparent Bricks:** Fully transparent bricks are skipped in the Packed-Texture, but their indices are still mapped in the Index-Texture to maintain spatial consistency.

## Subtopics

## Operating characteristics
-   **Time Complexity:** $O(1)$ per sample for index lookup and brick retrieval.
-   **Memory Complexity:** $O(N_{bricks})$ for the Index-Texture plus $O(N_{non-transparent})$ for the Packed-Texture.
-   **Strengths:** Simple implementation; efficient memory usage for sparse volumes; no dynamic texture binding overhead.
-   **Limitations:** Manual filtering required; fencepost issues at brick boundaries; static brick layout may lead to memory waste if many bricks are partially transparent.

## Background
This method originates from community discussions on GPU-based volume rendering optimization, specifically addressing the challenge of rendering large datasets that exceed GPU memory. It was proposed as a practical solution using standard OpenGL textures without requiring advanced features like bindless textures or virtual memory systems [S1, unofficial].

## Sources
[S1] volume rendering of large data by packing texture, https://stackoverflow.com/questions/13045663/volume-rendering-of-large-data-by-packing-texture, fetched 2026-10-26 (unofficial)
