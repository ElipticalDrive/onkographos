# Texture Packing Strategy

Also called: Separate Scalar Textures for Bandwidth Optimization

## Overview
Texture Packing Strategy maps a single underlying volume dataset to multiple distinct GPU textures, each scaled by a specific scalar multiplier to reduce GPU memory consumption [S2]. It solves excessive memory usage when rendering high-resolution volumes with multiple attribute channels by avoiding raw voxel data duplication [S2]. Unlike [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md), which combines attributes into a single RGBA texture for simultaneous access, this strategy separates attributes into individual scalar textures to minimize memory bandwidth when attributes are accessed independently [S1, unofficial].

## How it works
The method relies on GPU texture mapping capabilities to treat a single volume as multiple logical textures with different scaling factors [S2].

**Inputs and Outputs:**
*   **Input:** A single base volume dataset $V$ (e.g., emission volume $V_{em}$) [S2].
*   **Input:** Scalar multipliers $k_{em}$, $k_{ab}$, and $k_{ref}$ for emission, absorption, and reflection respectively [S2].
*   **Output:** Multiple texture objects ($T_{em}, T_{ab}, T_{ref}$) that return $k \cdot V$ upon sampling [S2].

**Data Structures:**
*   **Base Volume:** Stored once in GPU memory [S2].
*   **Texture Objects:** Each texture object references the same underlying memory buffer but applies a different scalar multiplier during the sampling operation [S2].
*   **Scalar Multipliers:** Floating-point values defined by the renderer configuration [S2].

**Steps:**
1.  **Volume Mapping:** Map the single base volume to multiple texture objects. For example, map $V_{em}$ to $T_{ab}$ and $T_{ref}$ [S2].
2.  **Scalar Configuration:** Set the scalar multipliers for each texture object: $k_{em}$ for emission, $k_{ab}$ for absorption, and $k_{ref}$ for reflection [S2].
3.  **Sampling:** Perform texture lookups using trilinear interpolation. The value returned by a texture lookup $T(x,y,z)$ is effectively $k \cdot V(x,y,z)$, where $k$ is the scalar multiplier associated with that texture object [S2].
4.  **Gradient Computation (Optional):** To further save memory, gradients ($\nabla_{x,y,z}$) can be computed on the fly during rendering rather than stored in a separate texture [S2].
5.  **Data Transfer:** Copy volumes to GPU memory only if they have been modified since the last rendering process [S2].

**Formulas:**
The sampled value $S$ for a texture $T$ with scalar $k$ is:
$$ S = k \cdot V(x, y, z) $$
Where:
*   $V(x, y, z)$ is the value from the base volume at coordinates $(x, y, z)$ [G].
*   $k$ is the scalar multiplier ($k_{em}, k_{ab},$ or $k_{ref}$) [S2].

**Parameters:**
*   **Interpolation:** Trilinear interpolation is used for all texture lookups [S2].
*   **Scalar Multipliers:** $k_{em}, k_{ab}, k_{ref}$ are configurable per texture object [S2].

**Edge Cases and Pitfalls:**
*   **Computational Cost:** Computing gradients on the fly is computationally expensive, particularly for movie sequences [S2].
*   **Memory vs. Compute Trade-off:** While mapping one volume to multiple textures saves GPU memory, it may increase computational load if gradients are not pre-computed [S2].
*   **Bandwidth Degradation:** Packing frequently accessed attributes (like inside-outside information) and unused attributes (like velocity) into a single RGBA texture increases memory transfer by 4x, which is inefficient if the unused data is not accessed during the current pass [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear_interpolate(volume, x, y, z):
    """Standard 3D trilinear interpolation over a 3D array."""
    ix, iy, iz = int(x), int(y), int(z)
    fx, fy, fz = x - ix, y - iy, z - iz
    # Fetch 8 neighboring voxels
    c000 = volume[ix, iy, iz]
    c100 = volume[ix+1, iy, iz]
    c010 = volume[ix, iy+1, iz]
    c110 = volume[ix+1, iy+1, iz]
    c001 = volume[ix, iy, iz+1]
    c101 = volume[ix+1, iy, iz+1]
    c011 = volume[ix, iy+1, iz+1]
    c111 = volume[ix+1, iy+1, iz+1]
    # Interpolate along x
    l1 = c000 * (1 - fx) + c100 * fx
    l2 = c010 * (1 - fx) + c110 * fx
    l3 = c001 * (1 - fx) + c101 * fx
    l4 = c011 * (1 - fx) + c111 * fx
    # Interpolate along y
    l5 = l1 * (1 - fy) + l2 * fy
    l6 = l3 * (1 - fy) + l4 * fy
    # Interpolate along z
    return l5 * (1 - fz) + l6 * fz

def setup_textures(base_volume, scalars):
    """Maps a single base volume to multiple texture objects with scalar multipliers."""
    textures = {}
    for name, k in scalars.items():
        textures[name] = {'volume': base_volume, 'scalar': k}
    return textures

def sample_texture(texture, x, y, z):
    """Samples a texture using trilinear interpolation and applies the scalar multiplier."""
    base_val = trilinear_interpolate(texture['volume'], x, y, z)
    return texture['scalar'] * base_val

def update_gpu_if_modified(volumes, device_memory):
    """Copies volumes to GPU memory only if they have been modified since the last rendering process."""
    if volumes.modified:
        device_memory = volumes.data
        volumes.modified = False
    return device_memory
```

## Subtopics

### Separate Scalar Textures for Bandwidth Optimization
A specific application where frequently accessed attributes (like inside-outside information for pressure solves) are kept in their own scalar textures to minimize bandwidth [S1, unofficial]. This is chosen when attributes are accessed independently or when packing them into a single RGBA texture would cause unnecessary memory transfer, such as transferring velocity data when only inside-outside is needed [S1, unofficial]. It is distinct from multi-channel packing, which is optimal for simultaneous access to all channels [S1, unofficial].

## Operating characteristics
*   **Memory Complexity:** Reduces GPU memory consumption by storing the base volume only once, regardless of the number of scalar multipliers [S2].
*   **Bandwidth:** Optimizes bandwidth by avoiding the transfer of unused data, preventing 4x memory transfer overhead when packing unrelated attributes [S1, unofficial].
*   **Performance:** Increases rendering performance for scenes with multiple images when volumes are copied only upon modification [S2]. Computing gradients on the fly increases computational expense [S2].
*   **Strengths:** Significant memory savings for high-resolution volumes, flexible scalar multiplication, reduced host-to-GPU traffic.
*   **Limitations:** Increased computational cost if gradients are computed on the fly, requires GPU texture mapping support for scalar multiplication.

## Background
The technique was introduced in a GPU-enabled volume renderer paper to save GPU memory by mapping one volume to multiple textures with scalar multipliers [S2]. It is also discussed in real-time fluid simulation literature, where avoiding the packing of multiple cell attributes into a single texture is recommended to prevent unnecessary memory bandwidth degradation [S1, unofficial].

## Sources
[S1] Real-Time Simulation and Rendering of 3D Fluids, https://wo1fsea.github.io/files/About_NavMesh/P5_GPU_Gems_3_Real-Time_Simulation_and_Rendering_of_3D_Fluids.pdf, fetched 2026-10-27 (unofficial)
[S2] GPU-Enabled Volume Renderer for Use with MATLAB, https://doi.org/10.3390/digital4040049, fetched 2026-10-29
