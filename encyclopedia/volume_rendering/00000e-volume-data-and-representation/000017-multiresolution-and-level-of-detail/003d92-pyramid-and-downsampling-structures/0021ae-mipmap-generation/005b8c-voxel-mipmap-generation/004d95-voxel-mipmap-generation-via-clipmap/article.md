# Voxel Mipmap Generation via Clipmap

Also called: Clipmap Voxel Mipmap Generation

## Overview
This method generates a voxel mipmap pyramid from a clipmap-based volume representation using GPU compute shaders. It solves the problem of efficiently downsampling high-resolution voxel data into lower-resolution levels for rendering or lighting calculations, specifically tailored for clipmap architectures where data is organized in circular buffers or layered structures. It differs from standard texture downsampling by explicitly managing the transition from the clipmap's specific memory layout (involving `LayoutSize` and `ClipMapCount`) to a linear mipmap array, requiring intermediate textures to avoid read-write conflicts.

## How it works
**Inputs:**
- `ClipMaps`: The source clipmap volume data.
- `ClipMapResolution`: The resolution of the base clipmap layer.
- `ClipMapCount`: The number of clipmap layers.
- `LayoutSize`: A multiplier determining the Y-dimension scaling for the internal layout.
- `MipMaps`: The destination array for the final mipmap levels.

**Outputs:**
- `MipMaps`: An array of textures representing the mipmap pyramid.

**Data Structures:**
- `TempMipMaps`: An array of temporary textures used as intermediates. The size is `TempMipMaps.Length`.
- `resolution`: A 3D vector (`Int3`) tracking the current mipmap resolution dimensions.
- `threadGroupCounts`: A 3D vector defining the compute shader's thread group size.
- `threadNumbers`: A 3D vector defining the number of thread groups to dispatch.

**Parameters:**
- `LayoutSize`: Multiplier for Y-resolution during setup. [S1, unofficial]
- `ClipMapCount`: Number of clipmaps. [S1, unofficial]

**Steps:**

1.  **Initialize Thread Group Counts:**
    Set the initial thread group counts based on the largest clipmap resolution. The counts are derived by dividing the resolution dimensions by 4.
    $$ \text{threadGroupCounts} = \left\lfloor \frac{\text{ClipMapResolution}}{4} \right\rfloor $$
    Specifically:
    ```python
    threadGroupCounts = Int3(
        int(ClipMapResolution.X) // 4,
        int(ClipMapResolution.Y) // 4,
        int(ClipMapResolution.Z) // 4
    )
    ```
    [S1, unofficial]

2.  **Adjust Y-Resolution:**
    Scale the Y-resolution by `LayoutSize` to match the internal memory layout.
    ```python
    resolution = Int3(int(ClipMapResolution.X), int(ClipMapResolution.Y) * LayoutSize, int(ClipMapResolution.Z))
    ```
    [S1, unofficial]

3.  **Iterate Through Mipmaps:**
    Loop from mipmap index `i = 0` to `TempMipMaps.Length - 2`.

    a.  **Halve Resolution:**
        Divide the current resolution by 2 for each dimension.
        ```python
        resolution = resolution // 2
        ```
        [S1, unofficial]

    b.  **Clamp Resolution:**
        Ensure no dimension falls below 1.
        ```python
        if resolution.X < 1: resolution.X = 1
        if resolution.Y < 1: resolution.Y = 1
        if resolution.Z < 1: resolution.Z = 1
        ```
        [S1, unofficial]

    c.  **Adjust Thread Group Counts:**
        If any resolution dimension is smaller than the corresponding `threadGroupCounts` dimension, divide `threadGroupCounts` by 4 and clamp to a minimum of 1.
        ```python
        if resolution.X < threadGroupCounts.X or resolution.Y < threadGroupCounts.Y or resolution.Z < threadGroupCounts.Z:
            threadGroupCounts = Int3(
                max(1, threadGroupCounts.X // 4),
                max(1, threadGroupCounts.Y // 4),
                max(1, threadGroupCounts.Z // 4)
            )
        ```
        [S1, unofficial]

    d.  **Set Shader Configuration:**
        Assign the thread group counts to the mipmap shader.
        ```python
        mipmapShader.ThreadGroupCounts = threadGroupCounts
        ```
        [S1, unofficial]

    e.  **Calculate Dispatch Count:**
        Compute the number of thread groups to dispatch based on the current resolution and thread group counts.
        ```python
        threadNumbers = Int3(
            int(resolution.X) // threadGroupCounts.X,
            int(resolution.Y) // threadGroupCounts.Y,
            int(resolution.Z) // threadGroupCounts.Z
        )
        mipmapShader.ThreadNumbers = threadNumbers
        ```
        [S1, unofficial]

    f.  **Set Textures and Offsets:**
        - **For i = 0 (First Mipmap):**
          Set the read texture to `ClipMaps`.
          Calculate the read offset. The Y-offset accounts for the layout size and the position of the last clipmap layer.
          $$ \text{readOffset} = (0, \text{ClipMapResolution.Y} \times \text{LayoutSize} \times (\text{ClipMapCount} - 1), 0) $$
          Set the write texture to `TempMipMaps[0]`.
          [S1, unofficial]
        - **For i > 0 (Subsequent Mipmaps):**
          Set the read texture to the previous mipmap `TempMipMaps[i - 1]`.
          Set the read offset to `(0, 0, 0)`.
          Set the write texture to `TempMipMaps[i]`.
          [S1, unofficial]

    g.  **Execute Shader:**
        Dispatch the compute shader.
        ```python
        mipmapShader.Draw(drawContext)
        ```
        [S1, unofficial]

    h.  **Copy to Final Array:**
        Copy the generated region from the temporary texture to the final mipmap array to prevent read/write conflicts.
        ```python
        drawContext.CommandList.CopyRegion(TempMipMaps[i], 0, null, MipMaps, i)
        ```
        [S1, unofficial]

**Pitfalls:**
- **Read-Write Conflicts:** The GPU cannot read from and write to the same texture resource simultaneously, even if views point to different miplevels. This necessitates the use of `TempMipMaps` and the `CopyRegion` step. [S1, unofficial]

## Operating characteristics
Not given in the sources.

## Background
This method is implemented in the Xenko VoxelGI project, an unofficial extension or fork related to the Xenko game engine. It is designed for voxel-based global illumination (GI) systems that utilize clipmaps for efficient memory management of large voxel worlds.

## Sources
[S1] WhyPenguins/XenkoVoxelGI, https://github.com/WhyPenguins/XenkoVoxelGI, fetched 2026-10-26 (unofficial)
