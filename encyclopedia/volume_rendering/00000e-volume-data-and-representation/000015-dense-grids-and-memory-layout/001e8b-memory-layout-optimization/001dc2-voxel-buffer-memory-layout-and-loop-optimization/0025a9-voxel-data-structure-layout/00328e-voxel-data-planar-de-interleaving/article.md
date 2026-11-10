# Voxel Data Planar De-interleaving

Also called: Data Reduction with Reuse, DeInterleave, ReInterleave, Voxel Data Planar Re-interleaving

## Overview
Voxel Data Planar De-interleaving converts an Array of Structures (AoS) of voxel data into a Structure of Arrays (SoA) planar format to optimize memory access patterns and enable data reduction techniques such as slice reuse [S1, unofficial][S2]. It serves as the inverse operation of planar re-interleaving, restoring the AoS layout from planar buffers when required by rendering or simulation pipelines [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Input (De-interleave):** `TArray<FVoxelData> Data` containing $N$ voxels. Each `FVoxelData` structure contains four fields: `MaterialID`, `Density`, `BiomeID`, and `Metadata` [S1, unofficial].
*   **Output (De-interleave):** `TArray<uint8> OutPlanes` of size $N \times 4$ bytes [S1, unofficial].
*   **Input (Re-interleave):** `uint8* Planes` consisting of four contiguous blocks of $N$ bytes each: Material, Density, Biome, and Metadata [S1, unofficial].
*   **Output (Re-interleave):** `TArray<FVoxelData> OutData` containing $N$ voxels [S1, unofficial].

### Data Structures
The input AoS layout (`FVoxelData`) packs four 1-byte fields sequentially:
1.  `MaterialID` (1 byte)
2.  `Density` (1 byte)
3.  `BiomeID` (1 byte)
4.  `Metadata` (1 byte)

The output SoA layout (`OutPlanes`) separates these into four contiguous planes:
1.  Bytes $0$ to $N-1$: MaterialID plane
2.  Bytes $N$ to $2N-1$: Density plane
3.  Bytes $2N$ to $3N-1$: BiomeID plane
4.  Bytes $3N$ to $4N-1$: Metadata plane

### Steps

#### De-interleaving (AoS to SoA)
1.  Let $N$ be the number of voxels (`Data.Num()`).
2.  Allocate `OutPlanes` of size $N \times 4$ bytes.
3.  Reinterpret `Data` as a `uint8*` source pointer `Src`.
4.  Compute pointers for the four output planes:
    *   `Mat` points to `OutPlanes[0]`
    *   `Den` points to `OutPlanes[N]`
    *   `Bio` points to `OutPlanes[2N]`
    *   `Meta` points to `OutPlanes[3N]`
5.  Loop $i$ from $0$ to $N-1$:
    *   `Mat[i] = Src[i * 4 + 0]`
    *   `Den[i] = Src[i * 4 + 1]`
    *   `Bio[i] = Src[i * 4 + 2]`
    *   `Meta[i] = Src[i * 4 + 3]`

#### Re-interleaving (SoA to AoS)
1.  Let $N$ be the number of voxels.
2.  Allocate `OutData` of size $N$.
3.  Reinterpret `Planes` as a `uint8*` source pointer `Src`.
4.  Compute pointers for the four input planes:
    *   `Mat` points to `Src[0]`
    *   `Den` points to `Src[N]`
    *   `Bio` points to `Src[2N]`
    *   `Meta` points to `Src[3N]`
5.  Reinterpret `OutData` as a `uint8*` destination pointer `Dst`.
6.  Loop $i$ from $0$ to $N-1$:
    *   `Dst[i * 4 + 0] = Mat[i]`
    *   `Dst[i * 4 + 1] = Den[i]`
    *   `Dst[i * 4 + 2] = Bio[i]`
    *   `Dst[i * 4 + 3] = Meta[i]`

### Data Reduction and Reuse (Contextual Optimization)
While the core de-interleaving operation transforms layout, the planar format enables specific data reduction strategies described in swept-volume display systems [S2]:
1.  **Activation Filtering:** Only activated voxels are stored in the planar buffers, reducing memory footprint [S2].
2.  **Slice Reuse:** To reduce data for swept-volumetric display, slice data from indices $(N_{slice}/2 + 1)$ to $N_{slice}$ is assigned the same data as slices $0$ to $(N_{slice}/2 - 1)$ [S2].
3.  **Conditional Color Assignment:** When querying memory based on panel coordinates, if a matching condition is satisfied at the termination of the query, color data is assigned; otherwise, it is set to 0 [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def de_interleave(data: list[FVoxelData]) -> list[uint8]:
    """
    Converts AoS FVoxelData to SoA planar uint8 layout.
    FVoxelData has 4 bytes: [MaterialID, Density, BiomeID, Metadata]
    """
    n = len(data)
    out_planes = [0] * (n * 4)
    
    # Pointers to the start of each plane
    mat_plane = out_planes[0:n]
    den_plane = out_planes[n:2*n]
    bio_plane = out_planes[2*n:3*n]
    meta_plane = out_planes[3*n:4*n]
    
    for i in range(n):
        voxel = data[i]
        # Assuming FVoxelData is packed as [Mat, Den, Bio, Meta]
        mat_plane[i] = voxel.material_id
        den_plane[i] = voxel.density
        bio_plane[i] = voxel.biome_id
        meta_plane[i] = voxel.metadata
        
    return out_planes

def re_interleave(planes: list[uint8], n: int) -> list[FVoxelData]:
    """
    Converts SoA planar uint8 layout back to AoS FVoxelData.
    """
    out_data = []
    
    mat_plane = planes[0:n]
    den_plane = planes[n:2*n]
    bio_plane = planes[2*n:3*n]
    meta_plane = planes[3*n:4*n]
    
    for i in range(n):
        voxel = FVoxelData()
        voxel.material_id = mat_plane[i]
        voxel.density = den_plane[i]
        voxel.biome_id = bio_plane[i]
        voxel.metadata = meta_plane[i]
        out_data.append(voxel)
        
    return out_data
```

### Edge Cases and Pitfalls
*   **Precision:** The method assumes 1-byte fields. If `FVoxelData` uses larger types, the byte offsets ($i*4+0$, etc.) and total size ($N*4$) must be adjusted accordingly [S1, unofficial].
*   **Boundaries:** The loop bounds are strictly $0$ to $N-1$. Accessing outside these bounds results in undefined behavior [S1, unofficial].
*   **Degenerate Input:** If $N=0$, the output arrays should be empty. The sources do not specify behavior for invalid pointers [S1, unofficial].

## Subtopics

### Voxel Data Structure
A generic term for the organization of voxel data. De-interleaving is a specific transformation applied to this structure to optimize memory access [S1, unofficial].

### Voxel Data Layout
Refers to the physical arrangement of voxel data in memory. De-interleaving changes the layout from AoS to SoA [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for both de-interleaving and re-interleaving, where $N$ is the number of voxels. Each voxel is processed exactly once with constant-time memory accesses [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for the output buffer. The de-interleaved buffer requires $4N$ bytes for 1-byte fields [S1, unofficial].
*   **Strengths:** Improves cache locality for operations that access a single field across many voxels (e.g., updating all densities). Enables data reduction techniques like slice reuse in swept-volume displays [S2].
*   **Limitations:** Requires an additional buffer for the planar format. Re-interleaving is necessary if the downstream system expects AoS [S1, unofficial].
*   **When to use:** Use when rendering or simulation algorithms benefit from SoA access patterns or when implementing data reduction via slice reuse [S1, unofficial][S2]. Avoid if the system strictly requires AoS and memory bandwidth is not a bottleneck.

## Background
The concept of de-interleaving (AoS to SoA) is a standard memory layout optimization in graphics and scientific computing [S1, unofficial]. The specific application to voxel data in the context of "Data Reduction with Reuse" for swept-volume volumetric displays was described in 2014 [S2]. The implementation details for `FVoxelData` are found in the VoxelWorlds project [S1, unofficial].

## Sources
[S1] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S2] An Improved Design of 3D Swept-Volume Volumetric Display, https://jcomputers.us/vol9/jcp0901-34.pdf, fetched 2026-10-28
