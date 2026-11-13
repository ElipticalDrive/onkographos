# VoxelBlockyModelFluid

Also called: Fluid model

## Overview
`VoxelBlockyModelFluid` is a procedural model for rendering flowing water with multiple levels, designed to mimic the visual style of games like Minecraft [S1]. It solves the problem of efficiently rendering dynamic fluid surfaces without the memory overhead of precomputing variations for every possible fluid height combination [S1]. Unlike static terrain models, it calculates vertex displacements on the fly during mesh generation to raise top corners to match the highest neighbor fluid of the same type [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel grid containing fluid blocks, a `VoxelBlockyFluid` resource, and one or more `VoxelBlockyModelFluid` instances.
*   **Output:** A mesh where fluid vertices are displaced based on neighbor heights, with UV channels encoding flow direction and state [S1].

**Data Structures:**
1.  **`VoxelBlockyFluid` Resource:** A file (`.tres`) shared across all levels of a specific fluid type. It defines the material and identity of the fluid [S1].
2.  **`VoxelBlockyModelFluid` Instance:** Created for each fluid level. It contains:
    *   `fluid`: Reference to the shared `VoxelBlockyFluid` resource.
    *   `level`: Integer ID identifying the specific height level of this fluid instance [S1].
    *   `transparency_index`: Must be set higher than solid blocks to prevent side culling issues [S1].
    *   `material`: A `ShaderMaterial` containing the special shader for flow animation [S1].

**Numbered Steps:**
1.  **Library Setup:** Create a `VoxelBlockyFluid` resource and save it as a `.tres` file [S1].
2.  **Model Creation:** In the voxel library, create a `VoxelBlockyModelFluid` for every distinct fluid level. Assign the same `fluid` resource to the `fluid` property of all instances so the mesher recognizes them as part of the same fluid family. Consecutive IDs for levels are convenient but not required [S1].
3.  **Mesh Generation (Displacement):** For each fluid voxel, the mesher raises the top corners to match the height of the highest neighbor fluid of the same type [S1].
4.  **UV Encoding:** The mesher populates the UV channels with flow information:
    *   `UV.x`: Projection axis for texture sampling. $0 = X, 1 = Y, 2 = Z$ [S1].
    *   `UV.y`: Flowing state integer (0–8) [S1].
5.  **Shader Execution:** The shader uses `UV.x` to select the projection axis and `UV.y` to determine the flow direction. It samples the texture using local vertex coordinates (triplanar-like mapping) to animate the water [S1].

**Formulas and Parameters:**
*   **Flow State (`UV.y`) Mapping:** The integer value is proportional to an angle representing flow direction [S1].

| Value | Direction |
| :--- | :--- |
| 0 | Straight +X |
| 1 | Diagonal +X -Z |
| 2 | Straight -Z |
| 3 | Diagonal -X -Z |
| 4 | Straight -X |
| 5 | Diagonal -X +Z |
| 6 | Straight +Z |
| 7 | Diagonal +X +Z |
| 8 | Idle |

*   **Transparency Index:** Must be $> $ transparency index of solid blocks [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Setup Phase (Editor/Initialization)
fluid_resource = create_VoxelBlockyFluid_resource()
fluid_resource.save("fluid.tres")

# Create models for each level (e.g., level 1 and level 2)
model_level_1 = create_VoxelBlockyModelFluid()
model_level_1.fluid = fluid_resource
model_level_1.level = 1
model_level_1.transparency_index = 10 # Higher than solid blocks

model_level_2 = create_VoxelBlockyModelFluid()
model_level_2.fluid = fluid_resource
model_level_2.level = 2
model_level_2.transparency_index = 10

# Mesh Generation Phase (Per Block Update)
def generate_fluid_mesh(block_data):
    for each voxel in block_data:
        if voxel.is_fluid():
            # Find highest neighbor fluid of the same type
            max_neighbor_height = get_highest_neighbor_fluid_height(voxel)
            
            # Displace top corners to match max_neighbor_height
            voxel.top_corners.y = max_neighbor_height
            
            # Determine flow direction based on neighbor liquid levels
            # This logic is typically part of the user's simulation step
            flow_state = calculate_flow_state(voxel)
            
            # Encode in UVs
            # UV.x: Projection axis (0=X, 1=Y, 2=Z)
            # UV.y: Flow state (0-8)
            voxel.uv.x = get_projection_axis(voxel.normal)
            voxel.uv.y = flow_state

# Shader Phase (GPU)
def render_fluid_shader(vertex, uv):
    # uv.y is the flow state
    flow_state = uv.y
    
    # uv.x is the projection axis
    axis = uv.x
    
    # Sample texture based on local coordinates and flow state
    # Implementation depends on specific shader language (GLSL/HLSL)
    color = sample_water_texture(vertex.local_coords, flow_state, axis)
    return color
```

**Edge Cases and Pitfalls:**
*   **Normals:** Normals of the top side remain flat; only corner positions are displaced. Shading does not change with slope [S1].
*   **Backfaces:** No backfaces are generated. Unlike Minecraft, which culls backfaces differently in edge cases, this engine does not differentiate. Disabling backface culling is a workaround but may cause other issues [S1].
*   **Simulation:** Fluids do not simulate behavior on their own; the user must implement the flow logic [S1].

## Subtopics

### Water Flowing Algorithm
An algorithm to simulate liquid flow and dispersion for water blocks by updating their liquid amount and transferring it to adjacent or underlying blocks [S2, unofficial]. It operates on `BlockInstance` objects, each holding an `amount` of liquid between `MIN_LIQUID_AMOUNT` and `MAX_LIQUID_AMOUNT` [S2, unofficial].

**Steps:**
1.  For each liquid block instance, read its current `amount` [S2, unofficial].
2.  If `amount < MIN_LIQUID_AMOUNT`, call `disperseWater()` to remove the block [S2, unofficial].
3.  Else if the block directly underneath is air, reduce the instance's height by 1 (make it fall) [S2, unofficial].
4.  Else if the block underneath is liquid, transfer as much liquid as possible from the current instance to the instance below [S2, unofficial].
5.  If the current instance still has liquid remaining, transfer as much liquid as possible to the four horizontal neighbors: $(x+1, z), (x, z+1), (x-1, z), (x, z-1)$ [S2, unofficial].

**Parameters:**
*   `MIN_LIQUID_AMOUNT`: Threshold below which the block is removed [S2, unofficial].
*   `MAX_LIQUID_AMOUNT`: Maximum liquid capacity for a water block instance [S2, unofficial].
*   **Update Frequency:** The algorithm is called once every 4 frames to balance realism and performance, as it requires rebuilding the terrain mesh [S2, unofficial].

**Pitfalls:**
*   Rebuilding the terrain mesh on every update is computationally expensive. Throttling to every 4 frames mitigates this cost [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** High during mesh generation due to per-block neighbor lookups and displacement calculations. The fluid simulation algorithm is called every 4 frames, adding significant overhead if the terrain is large [S1][S2, unofficial].
*   **Memory Complexity:** Low precomputed memory usage compared to static models, as variations are calculated at runtime [S1]. However, it requires storing `BlockInstance` data for each liquid block to track liquid amounts [S2, unofficial].
*   **Strengths:** Efficient handling of multiple fluid levels without massive precomputed asset variations; simple procedural displacement [S1].
*   **Limitations:** Flat normals on top surfaces reduce shading realism; no backface generation limits visual fidelity in edge cases; simulation is not built-in and requires custom implementation [S1][S2, unofficial].
*   **When to Use:** When a Minecraft-like fluid appearance with multiple levels is needed and performance allows for periodic mesh rebuilds [S1][S2, unofficial].
*   **When Not to Use:** When high-fidelity slope shading or complex backface culling is required, or when real-time fluid simulation without mesh rebuilds is necessary [S1][S2, unofficial].

## Background
`VoxelBlockyModelFluid` is part of the Voxel Tools library for Godot Engine, designed for blocky voxel terrains [S1]. The water flowing algorithm described in the notes originates from the CubeEngine project by rpereira-dev, which implements a simple liquid simulation for blocky games [S2, unofficial]. The approach of using UV channels to encode flow direction for shader-based animation is a common technique in voxel games to avoid complex vertex shader logic [S1].

## Sources
[S1] Blocky terrains - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/blocky_terrain/, fetched 2026-10-27
[S2] rpereira-dev/CubeEngine, https://github.com/rpereira-dev/CubeEngine, fetched 2026-10-27 (unofficial)
