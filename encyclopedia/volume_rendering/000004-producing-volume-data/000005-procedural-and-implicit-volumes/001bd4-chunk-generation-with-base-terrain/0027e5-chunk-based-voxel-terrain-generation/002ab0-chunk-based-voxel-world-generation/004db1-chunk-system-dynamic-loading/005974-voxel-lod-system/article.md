# Voxel LOD System

Also called: On-the-fly generation system, Voxel Plugin LOD

## Overview
The Voxel LOD System manages the rendering of massive procedural voxel worlds (up to $2^{31}-1$ units per axis) by dynamically adjusting chunk resolution based on viewer proximity, ensuring that only necessary data is queried from world generators [S1]. It distinguishes itself from static LOD approaches by combining on-the-fly procedural generation with a transition mesh system to eliminate visual seams between chunks of differing resolutions [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera/player position, world generator functions, `World Size` configuration, `Max LOD` limit, `Invoker Distance Threshold`.
*   **Outputs:** Rendered voxel meshes with varying resolutions, collision data, and seamless transitions between LOD levels.

**Data Structures:**
*   **Voxel Invoker Component:** A component attached to entities (players/NPCs) that signals the voxel world to generate meshes and collisions for its vicinity [S1].
    *   **Base Class:** `UVoxelInvokerComponentBase` [S1].
    *   **Required Methods:**
        *   `GetInvokerVoxelPosition()`: Returns the world position of the invoker [S1].
        *   `GetInvokerSettings()`: Returns configuration settings for the invoker [S1].
        *   `IsLocalInvoker()` (Optional): Returns boolean indicating if the invoker is local [S1].
*   **LOD Octree:** A hierarchical structure tracking which chunks require updates based on invoker positions [S1].
*   **Transition Meshes:** Intermediate geometry used to bridge gaps between chunks of different resolutions [S1].

**Parameters:**
*   **World Size:** The configurable dimension of the voxel world. Default example: 1,048,576. Recommended tight fit: 2,048 [S1].
*   **Max LOD:** A property limiting the minimum resolution. Setting it too low causes a rendering error [S1].
*   **Invoker Distance Threshold:** The distance an invoker must move to trigger a rebuild of the LOD Octree [S1].
*   **Constant LOD:** A boolean option. If enabled, skips LOD updates when the world is rendered at the highest resolution everywhere, saving CPU time [S1].

**Steps:**
1.  **Initialize Invokers:** The system checks for existing `Voxel Invoker Components` in the scene [S1].
2.  **Spawn Default Invoker:** If no invoker components are present, spawn a default invoker component that follows the active camera [S1].
3.  **Determine LODs:** Use the invoker components to signal the voxel world which sections require high-resolution meshes and collisions [S1].
4.  **Calculate Chunk Resolution:** Determine the resolution for each chunk based on its LOD level. The resolution decreases by powers of two:
    *   LOD 0: $32 \times 32 \times 32$
    *   LOD 1: $64 \times 64 \times 64$
    *   LOD 2: $128 \times 128 \times 128$
    *   General formula: Resolution $= 32 \times 2^{LOD}$ [S1].
5.  **Query Data:** For chunks with LOD > 0, query only the necessary data from world generators, as low-resolution chunks require less data querying [S1].
6.  **Generate Meshes:** Create voxel meshes for the calculated resolutions.
7.  **Apply Transitions:** Insert transition meshes between adjacent chunks of different LODs to prevent visual seams [S1].
8.  **Update Octree:** Monitor invoker movement. If an invoker moves more than the `Invoker Distance Threshold`, rebuild the LOD Octree [S1].
9.  **Runtime Updates:** If LOD settings are changed at runtime, call `Apply LOD Settings` to reflect changes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelInvokerComponentBase:
    def GetInvokerVoxelPosition(self):
        # Returns the world position of the invoker
        pass

    def GetInvokerSettings(self):
        # Returns configuration settings for the invoker
        pass

    def IsLocalInvoker(self):
        # Returns boolean indicating if the invoker is local
        return False

class VoxelWorld:
    def __init__(self, world_size, max_lod, invoker_distance_threshold):
        self.world_size = world_size
        self.max_lod = max_lod
        self.invoker_distance_threshold = invoker_distance_threshold
        self.invokers = []
        self.lod_octree = None
        self.constant_lod = False

    def AddInvoker(self, invoker: VoxelInvokerComponentBase):
        self.invokers.append(invoker)

    def UpdateLODs(self):
        # Determine which LODs to render based on invoker positions
        # If no invokers, use default camera-following invoker
        if not self.invokers:
            self.spawn_default_invoker()

        # Check if invoker moved beyond threshold
        for invoker in self.invokers:
            if self._invoker_moved_beyond_threshold(invoker):
                self.RebuildLODOctree()
                break

        # If constant LOD is enabled and world is highest res everywhere, skip updates
        if self.constant_lod and self._is_full_resolution():
            return

        # Calculate chunk resolutions
        for chunk in self.active_chunks:
            lod_level = self._calculate_lod_level(chunk, self.invokers)
            # Ensure LOD does not exceed Max LOD
            if lod_level > self.max_lod:
                lod_level = self.max_lod
            
            # Resolution = 32 * 2^lod_level
            resolution = 32 * (2 ** lod_level)
            
            # Query data from world generators
            # Only query necessary data for LOD > 0
            data = self._query_world_generators(chunk, resolution)
            
            # Generate mesh
            mesh = self._generate_mesh(data, resolution)
            
            # Apply transition meshes if adjacent to different LOD chunks
            self._apply_transitions(chunk, mesh)

    def _invoker_moved_beyond_threshold(self, invoker):
        # Check if invoker moved more than Invoker Distance Threshold
        # [G] Implementation detail: Compare current position with last recorded position
        pass

    def RebuildLODOctree(self):
        # Rebuild the LOD Octree based on current invoker positions
        pass

    def _calculate_lod_level(self, chunk, invokers):
        # Calculate LOD level based on distance to nearest invoker
        # [G] Implementation detail: Distance-based LOD calculation
        pass

    def _query_world_generators(self, chunk, resolution):
        # Query world generators for chunk data at specified resolution
        pass

    def _generate_mesh(self, data, resolution):
        # Generate voxel mesh from data
        pass

    def _apply_transitions(self, chunk, mesh):
        # Insert transition meshes between chunks of different LODs
        pass

    def spawn_default_invoker(self):
        # Spawn a default invoker component that follows the active camera
        pass

    def _is_full_resolution(self):
        # Check if world is rendered at highest resolution everywhere
        pass

    def ApplyLODSettings(self):
        # Apply runtime changes to LOD settings
        pass
```

**Edge Cases and Pitfalls:**
*   **Max LOD Error:** Setting `Max LOD` too low causes an error and prevents the voxel world from rendering. Ensure `World Size` is as small as possible to allow low `Max LOD` values [S1].
*   **Constant LOD:** If the world has the highest resolution LOD everywhere, LODs never change. Enable the `Constant LOD` option to save CPU time in this case [S1].
*   **World Size:** A large `World Size` (e.g., default Planet size of 1,048,576) may prevent the use of low `Max LOD` values. Tighten `World Size` (e.g., to 2,048) to allow lower resolutions [S1].

## Operating characteristics
*   **Time Complexity:** Chunk resolution decreases by powers of two, making low-resolution chunks relatively cheap to compute even for large areas [S1]. The LOD Octree is rebuilt only when an invoker moves beyond the `Invoker Distance Threshold`, reducing update frequency [S1].
*   **Memory Complexity:** On-the-fly generation ensures that world generators are only queried when needed, reducing memory overhead for unvisited areas [S1].
*   **Strengths:** Supports extremely large worlds ($2^{31}-1$ units per axis) with dynamic LODs. Transition meshes eliminate visual seams between LOD levels.
*   **Limitations:** Requires careful tuning of `World Size` and `Max LOD` to avoid rendering errors. High-resolution LODs everywhere can consume significant CPU time unless `Constant LOD` is enabled.
*   **When to Use:** Ideal for procedural voxel worlds where memory and performance are critical, and seamless visual transitions are required.
*   **When Not to Use:** Not suitable for static worlds with fixed geometry where LODs do not need to change dynamically.

## Background
The Voxel LOD System is part of the Voxel Plugin, which supports worlds up to $2^{31}-1$ voxels per axis [S1]. It was designed to enable on-the-fly generation of massive voxel worlds by querying world generators only when needed and adjusting chunk resolutions based on viewer proximity [S1]. The system uses Voxel Invoker Components to signal the voxel world about player/NPC presence and requires transition meshes to prevent seams between chunks of different LODs [S1].

## Sources
[S1] World Size and Level Of Details, https://docs.voxelplugin.com/1.2/core-systems/voxelworld/world-size-and-level-of-details, fetched 2026-10-27
