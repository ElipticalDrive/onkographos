# Voxel Data Initialization and Binding

Also called: Grid Hierarchy Setup, Voxel World Data Initialization

## Overview
This method initializes the voxel world data by loading voxel structures from files or generating empty grids, creating the corresponding GPU buffers and textures, and setting up the bind groups required for the trace shader. It differs from its sibling [Voxel Grid Construction](3085-voxel-grid-construction.md) by focusing on the memory allocation and shader binding phase rather than the algorithmic population of voxel data, and from [Voxel Grid Population & Raymarching Setup](21267-voxel-grid-population-raymarching-setup.md) by handling the initial buffer creation before the compute shader dispatch.

## How it works
**Inputs:** Voxel world data source (file path or empty generation request).
**Outputs:** Initialized GPU buffers (Grid Hierarchy, Voxel World), textures, sampler, and a Bind Group ready for the trace shader.

**Data Structures:**
1.  **Grid Hierarchy (GH):** A structure representing the voxel grid levels.
2.  **Levels Uniform Array:** An array of `UVec4` vectors derived from `gh.levels`.
3.  **Grid Hierarchy Buffer:** A storage buffer containing the serialized GH data.
4.  **Bind Group Layout:** Defines 5 bindings for the trace shader:
    *   Binding 0: Uniform Buffer (`TraceUniforms`)
    *   Binding 1: TextureView (`Voxel World`)
    *   Binding 2: Storage Buffer (`Grid Hierarchy`)
    *   Binding 3: TextureView (`Mip Texture`)
    *   Binding 4: Sampler (`Texture Sampler`)

**Steps:**
1.  **Determine Source:** Check if the voxel data is to be loaded from a file or generated as an empty grid.
2.  **Parse/Generate GH Structure:**
    *   If empty: Create an empty `GH` structure.
    *   If from file: Read the file and parse it into a `GH` structure using `GH::from_vox`.
3.  **Construct Levels Uniform Array:**
    *   Iterate through `gh.levels`.
    *   For each level `i`, create a `UVec4` by copying `gh.levels[i]` into the `x` component and setting `y`, `z`, and `w` components to zero.
4.  **Create Grid Hierarchy Buffer:**
    *   Calculate buffer size using `gh.get_buffer_size()`.
    *   Create a buffer with usage flags `STORAGE` and `COPY_DST`.
    *   Copy the GH data into this buffer.
5.  **Setup Bind Group:**
    *   Bind the `TraceUniforms` uniform buffer to binding 0.
    *   Bind the `Voxel World` texture view to binding 1.
    *   Bind the `Grid Hierarchy` storage buffer to binding 2.
    *   Bind the `Mip Texture` view to binding 3.
    *   Bind the `Texture Sampler` to binding 4.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def initialize_voxel_data(source_type, file_path=None):
    # Step 1 & 2: Parse/Generate GH Structure
    if source_type == "empty":
        gh = GH() # Create empty GH structure
    elif source_type == "file":
        raw_data = read_file(file_path)
        gh = GH.from_vox(raw_data)
    else:
        raise ValueError("Unknown source type")

    # Step 3: Construct Levels Uniform Array
    levels_uniform_array = []
    for level_data in gh.levels:
        # level_data is assumed to be a scalar or vec1 compatible with UVec4.x
        uvec4 = UVec4(x=level_data, y=0, z=0, w=0)
        levels_uniform_array.append(uvec4)

    # Step 4: Create Grid Hierarchy Buffer
    buffer_size = gh.get_buffer_size()
    grid_hierarchy_buffer = create_buffer(
        size=buffer_size,
        usage=BufferUsage.STORAGE | BufferUsage.COPY_DST
    )
    # Copy GH data to buffer (implementation detail of GH serialization)
    grid_hierarchy_buffer.write_data(gh.serialize())

    # Step 5: Setup Bind Group
    bind_group_layout = get_bind_group_layout() # Pre-defined layout with 5 bindings
    
    bind_group = create_bind_group(
        layout=bind_group_layout,
        entries=[
            BindEntry(binding=0, resource=TraceUniforms.buffer),
            BindEntry(binding=1, resource=VoxelWorld.texture_view),
            BindEntry(binding=2, resource=grid_hierarchy_buffer),
            BindEntry(binding=3, resource=MipTexture.texture_view),
            BindEntry(binding=4, resource=TextureSampler)
        ]
    )
    
    return bind_group, grid_hierarchy_buffer
```

**Edge Cases and Pitfalls:**
*   **File Parsing:** If the file format is incorrect, `GH::from_vox` may fail. Error handling should be implemented.
*   **Buffer Size:** `gh.get_buffer_size()` must accurately reflect the serialized size of the GH structure to avoid buffer overflows or under-allocation.
*   **Binding Order:** The bind group layout is fixed. Binding indices must match the shader's expectations exactly.

## Subtopics

### [Voxel Grid Population & Raymarching Setup](21267-voxel-grid-population-raymarching-setup.md)
Populates the dense 3D voxel grid with distance field data and configures the GPU compute pipeline for volumetric raymarching. It bridges the gap between abstract voxel objects and the shader-executable texture buffers, solving the problem of translating spatially scattered voxel data into a contiguous memory layout suitable for efficient 3D texture sampling. It differs from its parent by handling the specific algorithmic mapping of voxel coordinates to grid indices and the explicit dispatch of the rendering compute shader.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the size of the voxel data for parsing and buffer creation.
*   **Memory Complexity:** Proportional to the size of the voxel grid and hierarchy data.
*   **Strengths:** Provides a standardized way to load and bind voxel data for rendering.
*   **Limitations:** Dependent on the efficiency of `GH::from_vox` and buffer creation.
*   **When to Use:** At the start of a scene or when loading a new voxel world.
*   **When Not to Use:** When modifying existing voxel data in-place without reloading.

## Background
This method is part of the voxel engine implementation in the Bevy game engine, specifically in the `bevy-voxel-engine` project. It handles the low-level GPU resource management for voxel data.

## Sources
[S1] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
