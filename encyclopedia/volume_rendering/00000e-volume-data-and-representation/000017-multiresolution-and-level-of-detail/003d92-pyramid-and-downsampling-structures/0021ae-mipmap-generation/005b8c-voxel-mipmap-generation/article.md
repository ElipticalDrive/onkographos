# Voxel Mipmap Generation

Also called: Multi-Resolution Voxshot Generation, Voxel Cone Tracing Mipmap Generation and Clamping, Voxel Mipmap Pass, Voxel Mipmapping

## Overview
Voxel Mipmap Generation creates a hierarchy of downsampled voxel representations from a primary volume buffer to enable rendering at varying scales or distances [S1, unofficial]. It differs from [Atlas Mipmapping](2639-atlas-mipmapping.md) by operating on a contiguous 3D voxel grid rather than 2D texture atlases, and from [GPU 3D Mip Chain Generation](5067-gpu-3d-mip-chain-generation.md) by often employing specific compute shader passes or CPU-side accumulation loops for the downsampling step [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A source voxel volume buffer (Level 0) containing visible cylinder voxels or a full volume [S1, unofficial].
*   **Output:** A series of downsampled voxel buffers (Levels 1 to $N$) [S1, unofficial].

**Data Structures:**
*   **Voxel Buffer:** A 3D array of color values. In the CPU implementation, voxels are stored in a linearized format where a voxel at $(x, y, z)$ is accessed via an index calculated as $((x \cdot 2 + i) \cdot \text{count}[2] \cdot 2) + ((y \cdot 2 + j) \cdot \text{count}[0] \cdot \text{count}[2] \cdot 4) + (z \cdot 2 + k)$ [S1, unofficial].
*   **Mip Chain:** A sequence of buffers where Level $m$ has dimensions $\lfloor \text{Level}_{m-1} / 2 \rfloor$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Check if continuous voxelization is enabled; if not, skip generation [S3, unofficial]. Bind the source voxel volume for reading [S3, unofficial]. Set the initial destination resolution (e.g., $256 \gg 1 = 128$) [S3, unofficial]. Copy visible voxels from the display buffer to Level 0 [S1, unofficial].
2.  **Downsampling Loop (for $m = 1$ to $N$):** Reduce the dimensions of the current level by half: $\text{count}[0] \leftarrow \text{count}[0]/2$, $\text{count}[1] \leftarrow \text{count}[1]/2$, $\text{count}[2] \leftarrow \text{count}[2]/2$ [S1, unofficial]. For each voxel $(x, y, z)$ at Level $m$, sample the $2 \times 2 \times 2$ block from Level $m-1$ centered at $(x \cdot 2, y \cdot 2, z \cdot 2)$ [S1, unofficial]. Sum the Red, Green, and Blue components across the 8 samples [S1, unofficial]. Average by dividing by 3 per channel (the source code explicitly uses `/3`, which is likely a typo for `/8` or specific to a 3-channel normalization, but is implemented as `/3`) [S1, unofficial]. Clamp each channel to 255 [S1, unofficial]. Write to Level $m$ [S1, unofficial].
3.  **GPU Implementation Details:** Dispatch a compute shader (`VOXEL_MIPMAP_PASS_COMPUTE_SHADER`) [S3, unofficial]. Pass uniforms for the source mip level index and destination mip resolution [S3, unofficial]. Calculate work groups by dividing `dstMipRes` by 4.0 and taking the ceiling for X, Y, and Z dimensions [S3, unofficial]. Insert memory barriers before and after the dispatch to ensure texture fetch and image write coherence [S3, unofficial]. Halve the destination resolution for the next iteration [S3, unofficial].

**Formulas:**
$$ \text{rgb}_c = \sum_{i=0}^{1} \sum_{j=0}^{1} \sum_{k=0}^{1} C_{m-1}(x \cdot 2 + i, y \cdot 2 + j, z \cdot 2 + k)_c $$
where $c \in \{R, G, B\}$ and $C_{m-1}$ is the color at Level $m-1$ [S1, unofficial].

**Parameters:**
*   `MIPMAP_LEVELS`: Total number of levels. In the GPU implementation, the loop runs up to `MIPMAP_LEVELS - 2` [S3, unofficial].
*   `dstMipRes`: Initial destination resolution, often half the source (e.g., 128 if source is 256) [S3, unofficial].
*   `ContinousVoxelizationPass`: Boolean flag to enable/disable the pass [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Division Error:** The source code divides by 3 instead of 8 (the number of samples in a $2 \times 2 \times 2$ block) [S1, unofficial]. This results in incorrect color averaging unless compensated by other pipeline stages [S1, unofficial].
*   **Boundary Conditions:** When dimensions are odd, halving may result in truncation. The sampling logic assumes even alignment or specific padding [S1, unofficial].
*   **Precision:** Accumulating 8 uint8 values into a sum can exceed 255, requiring intermediate integer types before division [S1, unofficial].
*   **Hardcap on vlevel:** If there is no hardcap on the volume level (`vlevel`), `textureLod` calls accessing any mipmap level above 1 return a `#000000` color [S2, unofficial].
*   **Mipmap Generation Quality:** If the 3D texture MIPmaps are not generated correctly, wider cone angles produce strange wavy results instead of smooth indirect lighting [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_mipmaps(source_volume, max_levels, continuous_voxelization_enabled=True):
    """
    Generates mip levels for a voxel volume.
    source_volume: 3D list of voxels (x, y, z) -> [r, g, b]
    max_levels: integer, total number of mip levels to generate
    continuous_voxelization_enabled: boolean flag from source [S3, unofficial]
    """
    mip_chain = [source_volume]
    if not continuous_voxelization_enabled:
        return mip_chain

    current_level = source_volume
    current_res = [len(current_level), len(current_level[0]), len(current_level[0][0])]

    for m in range(1, max_levels):
        new_res = [d // 2 for d in current_res]
        next_level = [[[0.0, 0.0, 0.0] for _ in range(new_res[2])] for _ in range(new_res[1])]

        for x in range(new_res[0]):
            for y in range(new_res[1]):
                for z in range(new_res[2]):
                    rgb_sum = [0, 0, 0]
                    for i in range(2):
                        for j in range(2):
                            for k in range(2):
                                src_x = x * 2 + i
                                src_y = y * 2 + j
                                src_z = z * 2 + k
                                color = current_level[src_x][src_y][src_z]
                                rgb_sum[0] += color[0]
                                rgb_sum[1] += color[1]
                                rgb_sum[2] += color[2]
                    
                    # Source code divides by 3 per channel [S1, unofficial]
                    r = min(rgb_sum[0] / 3, 255.0)
                    g = min(rgb_sum[1] / 3, 255.0)
                    b = min(rgb_sum[2] / 3, 255.0)
                    
                    next_level[x][y][z] = [r, g, b]

        mip_chain.append(next_level)
        current_level = next_level
        current_res = new_res
        # Memory barrier inserted here in GPU implementations [S3, unofficial]

    return mip_chain
```

## Subtopics

### [Voxel Mipmap Generation via Clipmap](13505-voxel-mipmap-generation-via-clipmap.md)
Generates a voxel mipmap pyramid from a clipmap-based volume representation using GPU compute shaders. It solves the problem of efficiently downsampling high-resolution voxel data into lower-resolution levels for rendering or lighting calculations, specifically tailored for clipmap architectures where data is organized in circular buffers or layered structures. It differs from standard texture downsampling by explicitly managing the transition from the clipmap's specific memory layout (involving `LayoutSize` and `ClipMapCount`) to a linear mipmap array, requiring intermediate textures to avoid read-write conflicts. Choose this when memory bandwidth is a bottleneck or for large, sparse volumes.

### [Adaptive Rendering](18776-adaptive-rendering.md)
A volume visualization technique that improves time efficiency by performing ray tracing on coarse-resolution data first, then incrementally refining the result by incorporating information from higher-resolution levels. It functions as an intelligent masking mechanism that accelerates rendering by skipping empty or non-useful voxels identified in the low-resolution domain before calculating new voxels from higher resolutions for the integration equation. Unlike octree-based methods, it avoids the extra memory cost of auxiliary tree structures by leveraging the compressed nature of the multiresolution pyramid itself. Choose this when rendering performance is critical and empty space dominates the volume.

### [JPEG-Clear for 3D voxmaps](20067-jpeg-clear-for-3d-voxmaps.md)
A storage and retrieval method for volumetric data that replaces the original full-resolution dataset with a set of difference (diff) files derived from a multi-resolution pyramid, allowing visualization tools to access approximations without loading the full-resolution data. Choose this when storage footprint and data transfer bandwidth are primary constraints.

## Operating characteristics
*   **Time Complexity:** $O(V)$ where $V$ is the total number of voxels in the pyramid [G]. Each level processes half the voxels of the previous one, forming a geometric series [G].
*   **Memory Complexity:** $O(V_0)$ where $V_0$ is the size of the base level [G]. The total memory for the chain is $V_0 + V_0/8 + V_0/64 + \dots \approx 1.14 V_0$ [G].
*   **Performance:** The GPU implementation uses compute shaders with work groups of size 4, dispatching $\lceil \text{Res}/4 \rceil^3$ groups [S3, unofficial]. This allows for parallel processing of voxels [G].
*   **Strengths:** Enables LOD rendering for voxel volumes and reduces fill rate requirements for distant objects [S1, unofficial].
*   **Limitations:** The specific averaging bug (div by 3) may require post-processing correction [S1, unofficial]. Memory bandwidth can be high for large volumes [G]. Incorrect MIPmap generation causes visual artifacts like wavy lighting [S2, unofficial].

## Background
The method originates from implementations in volumetric display systems and voxel cone tracing engines. The CPU-based "voxshot" approach is documented in the Multivox project [S1, unofficial], while the GPU compute shader variant is found in the Voxel-Cone-Tracing repository [S3, unofficial].

## Sources
[S1] Multivox: Volumetric Display, https://github.com/AncientJames/multivox, fetched 2026-10-26 (unofficial)
[S2] Strange Voxel Cone Tracing Results, https://stackoverflow.com/questions/51688914/strange-voxel-cone-tracing-results, fetched 2026-10-26 (unofficial)
[S3] maritim/Voxel-Cone-Tracing, https://github.com/maritim/Voxel-Cone-Tracing, fetched 2026-10-26 (unofficial)
