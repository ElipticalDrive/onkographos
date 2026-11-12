# Voxel Instancing System

Also called: VoxelInstancer

## Overview
The Voxel Instancing System is a method for procedurally spawning and managing large numbers of 3D objects (instances) on a voxel terrain surface. It solves the problem of rendering natural environments (grass, rocks, trees) efficiently by decoupling instance placement logic from the rendering pipeline, utilizing either high-performance `MultiMesh` buffers or flexible but slower standard scene instances. It distinguishes itself from general GPU instancing by integrating tightly with voxel terrain LOD systems and SDF-based surface snapping, allowing instances to adhere to complex terrain geometry without manual placement.

## How it works

### Inputs and Outputs
*   **Inputs**: A voxel terrain chunk (defined by a `VoxelStream` and `VoxelGenerator`), a `VoxelInstanceLibrary` resource defining spawnable items, and procedural generation parameters (density, noise, etc.).
*   **Outputs**: A set of rendered instances positioned on the terrain surface, potentially with collision shapes, managed within the scene graph or GPU buffers.

### Data Structures

**VoxelInstanceLibrary**
A resource assigned to the `library` property of the `VoxelInstancer` node. It contains a list of `VoxelInstanceLibraryItem` objects.

**VoxelInstanceLibraryItem**
Base class for all spawnable items.
*   `lod_index` (int): Determines the terrain LOD level at which this item spawns. Lower indices correspond to higher-resolution chunks (closer range); higher indices correspond to lower-resolution chunks (farther range).
*   `offset_along_normal` (float): Adjustment distance along the surface normal.
*   `snap_to_generator_sdf_enabled` (bool): Toggle for SDF-based surface snapping.
*   `persistent` (bool): If true, the instance's position is saved to the voxel stream and persists across edits/load.
*   `generator` (Object): The procedural generation logic defining where instances spawn.

**VoxelInstanceLibraryMultiMeshItem**
A specific item type for high-performance rendering.
*   `mesh` (Mesh): LOD0 mesh.
*   `lod1` (Mesh): LOD1 mesh.
*   `lod2` (Mesh): LOD2 mesh.
*   `lod3` (Mesh): LOD3 mesh.
*   `mesh_lod_settings` (Dictionary/Struct): Contains ratios (0.0–1.0) relative to the terrain LOD view distance to define switching thresholds.
*   `hide_beyond_max_distance` (bool): Hides instances beyond the last LOD's range (instances still generated, just not rendered).

**VoxelInstanceLibrarySceneItem**
A specific item type for complex objects.
*   `scene` (PackedScene): The scene resource to instance. The root must be `Node3D` or a derivative.
*   *Note*: Scene instances are created as child nodes of the `VoxelInstancer` in the scene graph.

### Steps

1.  **Library Setup**:
    *   Create or assign a `VoxelInstanceLibrary` to the `VoxelInstancer` node.
    *   Add items to the library. For `MultiMesh`, use "Add Multimesh item". For complex objects, use "Add Scene item".

2.  **Scene Conversion (Optional for MultiMesh)**:
    *   To create a `MultiMesh` from an existing scene, assign the `scene` property (runtime conversion, linked) or use the "Setup from scene" button (editor conversion, manual properties, not linked).
    *   The source scene structure must be:
        ```text
        - PhysicsBody (StaticBody, RigidBody...)
            - MeshInstance_LOD0
            - MeshInstance_LOD1
            - ...
            - CollisionShape1
            - ...
        ```
    *   Materials must be set via `material_override` on `MeshInstance` or directly on the mesh resource. Surface material properties on the node are not supported.

3.  **Configuration of Placement**:
    *   Set `lod_index` to control spawn range.
    *   Set `up_mode` in `VoxelInstancer`. For planetary terrains, use 'Sphere mode' to align instances to the local terrain origin.
    *   Configure `offset_along_normal` to adjust vertical placement.

4.  **Procedural Generation**:
    *   The system uses a built-in generator by default. Items spawn where procedural conditions (density, noise) are met.
    *   If `snap_to_generator_sdf_enabled` is true, the system queries the voxel generator for SDF data.
    *   *Requirement*: The generator must support series generation (querying arbitrary floating-point positions). At the time of writing, only `VoxelGeneratorGraph` supports this.
    *   *Snapping Logic*: The system samples the SDF at the instance's projected position. If at least 2 nearby samples are available, it interpolates to snap the instance closer to the surface. This is an approximation, not exact ray-casting.

5.  **Persistence and Saving**:
    *   If `persistent` is true, the instance ID and position are saved to the voxel stream.
    *   *Constraint*: Only `VoxelStreamSQLite` supports saving persistent instances.
    *   If the terrain is edited, persistent instances in the modified block are saved rather than regenerated. Non-persistent instances are regenerated.

6.  **Rendering**:
    *   `MultiMesh` items are rendered using GPU instancing.
    *   `Scene` items are rendered as standard scene nodes.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelInstancer:
    def __init__(self, library: VoxelInstanceLibrary, stream: VoxelStream, generator: VoxelGenerator):
        self.library = library
        self.stream = stream
        self.generator = generator
        self.up_mode = "Flat" # or "Sphere"

    def process_chunk(self, chunk: VoxelChunk):
        """
        Generates and updates instances for a specific voxel chunk.
        """
        # 1. Determine active items based on LOD
        active_items = []
        for item in self.library.items:
            if item.lod_index == chunk.lod_level:
                active_items.append(item)

        # 2. Generate candidate positions
        candidates = []
        for item in active_items:
            # Use the item's generator to find valid positions
            # This typically involves noise sampling and density checks
            positions = item.generator.generate_positions(chunk.bounds)
            candidates.extend(positions)

        # 3. Snap to surface if enabled
        if self.generator.supports_series_generation():
            for candidate in candidates:
                if candidate.item.snap_to_generator_sdf_enabled:
                    # Query SDF at candidate position and neighbors
                    sdf_val = self.generator.query(candidate.position)
                    # Interpolate using nearby samples to find surface intersection
                    snapped_pos = self._interpolate_surface(candidate.position, sdf_val)
                    candidate.position = snapped_pos
                    candidate.offset_along_normal += self._calculate_offset(snapped_pos, candidate.position)

        # 4. Update Rendering
        for item in active_items:
            if isinstance(item, VoxelInstanceLibraryMultiMeshItem):
                self._update_multimesh(item, candidates_for_item(item))
            elif isinstance(item, VoxelInstanceLibrarySceneItem):
                self._update_scene_instances(item, candidates_for_item(item))

    def _update_multimesh(self, item: VoxelInstanceLibraryMultiMeshItem, instances: List[InstanceData]):
        """
        Updates the GPU MultiMesh buffer.
        """
        # Select LOD based on distance to camera
        lod_mesh = self._select_lod(item, instances)
        
        # Update buffer
        # Note: Changing MultiMesh.mesh to a different LOD causes performance issues
        # until specific PRs are merged.
        item.multimesh.mesh = lod_mesh
        item.multimesh.set_instance_count(len(instances))
        
        for i, inst in enumerate(instances):
            item.multimesh.set_instance_transform(i, inst.transform)
            if item.has_collision:
                # Collision shapes are supported but not settable in inspector
                # Must be handled via code or scene conversion
                pass

    def _update_scene_instances(self, item: VoxelInstanceLibrarySceneItem, instances: List[InstanceData]):
        """
        Creates or updates scene nodes.
        """
        # This is slow and can cause freezes if too many instances are created
        # Ensure item.persistent is checked to avoid regenerating saved instances
        for inst in instances:
            node = self._get_or_create_node(item.scene)
            node.transform = inst.transform
            self.add_child(node)

    def _select_lod(self, item: VoxelInstanceLibraryMultiMeshItem, instances: List[InstanceData]) -> Mesh:
        """
        Selects the appropriate LOD mesh based on distance ratios.
        """
        # Mesh LOD settings are ratios relative to the terrain LOD view distance
        # e.g., ratio 0.5 means switch at 50% of the chunk's view distance
        for i, ratio in enumerate(item.mesh_lod_settings.ratios):
            if item.lod_view_distance * ratio > distance_to_camera:
                return item.get_lod_mesh(i)
        return item.lod_mesh # Default to highest detail if not specified

    def _interpolate_surface(self, pos: Vector3, sdf: float) -> Vector3:
        """
        Approximate surface snapping using SDF gradients.
        """
        # Requires at least 2 nearby samples
        # This is a simplified representation of the interpolation logic
        gradient = self.generator.query_gradient(pos)
        if gradient.length() > 0:
            return pos - (sdf / gradient.length()) * gradient.normalized()
        return pos

    def _calculate_offset(self, snapped: Vector3, original: Vector3) -> float:
        """
        Calculates the distance moved along the normal.
        """
        return (snapped - original).dot(self._get_normal_at(snapped))
```

### Parameters and Typical Values
*   `up_mode`: "Flat" (default) or "Sphere" (for planets).
*   `lod_index`: Integer, typically 0 for close-range high-detail items, higher for distant low-detail items.
*   `mesh_lod_settings.ratios`: Floats between 0.0 and 1.0, defining switch distances relative to terrain LOD view distance.
*   `offset_along_normal`: Float, typically small values (e.g., 0.0 to 0.5 units) to prevent sinking or floating.

### Edge Cases and Pitfalls
*   **Cyclic Reference Crash**: If a scene is added to the library and the library is loaded from that same scene, Godot crashes due to cyclic reference [S1].
*   **Performance Issue**: Changing `MultiMesh.mesh` to a different LOD causes a performance issue [S1].
*   **SDF Snapping Limitations**:
    *   Requires a generator that supports series generation (e.g., `VoxelGeneratorGraph`) [S1].
    *   Does not work if the terrain is modified; instances in edited areas will not regenerate correctly unless set to `persistent` [S1].
    *   Tends to "bury" or "float" instances far away when low-resolution meshes are active, as they are moved closer to the high-resolution surface [S1].
*   **Scene Instance Limits**: Adding many complex scene instances can cause freezes or crashes. Default density is lower for scene items [S1].
*   **Persistence**: Only `VoxelStreamSQLite` supports saving persistent instances [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Memory**: `MultiMesh` instances are highly memory efficient, storing only transform data and a single mesh reference per instance. `Scene` instances consume significantly more memory due to the duplication of node trees and resources in the scene graph.
*   **Time**: `MultiMesh` updates are fast, dominated by GPU buffer updates. `Scene` instance updates are slow, involving CPU-side node creation and scene graph traversal. SDF snapping adds overhead proportional to the number of samples required for interpolation.

### Measured Performance
*   `MultiMesh` instances can be "extremely numerous" [S1].
*   `Scene` instances are "much slower" and should be "tuned to low numbers" [S1].

### Scaling
*   Scales well with `MultiMesh` instances, limited primarily by GPU buffer sizes and draw call overhead (which is minimal due to instancing).
*   Scales poorly with `Scene` instances, limited by CPU performance and scene graph complexity.

### Strengths and Limitations
*   **Strengths**:
    *   High performance for natural spawning (grass, rocks) via `MultiMesh` [S1].
    *   Integration with voxel terrain LOD and SDF for accurate surface placement [S1].
    *   Support for persistent instances in editable worlds [S1].
*   **Limitations**:
    *   Not suited for complex man-made structures [S1].
    *   `MultiMesh` has limitations on behavior (no scripts, animations) [S1].
    *   SDF snapping is approximate and fails on modified terrain without persistence [S1].
    *   Only `VoxelStreamSQLite` supports persistence [S1].

### When to Use
*   Use for natural spawning (grass, rocks, trees, semi-random foliage) [S1].
*   Use `MultiMesh` for high-density, static, or simple animated objects.
*   Use `Scene` instances for low-density, interactive, or complex animated objects.

### When Not to Use
*   Do not use for complex man-made structures like houses or villages [S1].
*   Do not use `Scene` instances for large-scale terrain decoration due to performance risks [S1].

## Background

The Voxel Instancing System is part of the Voxel Tools plugin for the Godot Engine. It was developed to address the lack of efficient procedural instancing in earlier versions of Godot (specifically Godot 3, which lacked automatic mesh LOD) [S1]. Godot 4 introduced automatic mesh LOD, but the system provides finer control over mesh and material selection per LOD level, as well as integration with voxel terrain systems [S1]. It is primarily used in voxel-based games for terrain decoration.

## Sources

[S1] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
