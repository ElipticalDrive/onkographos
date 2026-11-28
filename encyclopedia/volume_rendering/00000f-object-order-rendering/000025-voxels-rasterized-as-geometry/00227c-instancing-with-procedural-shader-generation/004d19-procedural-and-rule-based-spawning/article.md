# Procedural and Rule-Based Spawning
## Overview
Procedural and Rule-Based Spawning populates voxel terrains with large numbers of objects by evaluating spawn rules against terrain geometry rather than manual placement [S1]. It distinguishes itself from static instancing by dynamically generating instance positions and properties through procedural generators tied to the terrain system [S1].
## Shared principles
The family operates by evaluating spawn rules defined in a `VoxelInstanceLibrary` against eligible terrain chunks [S1]. The core mechanism involves:
1. **Chunk Eligibility**: The system iterates through loaded voxel chunks and matches them to items based on resolution level [S1].
2. **Candidate Generation**: The item's `generator` evaluates conditions (e.g., noise thresholds) at candidate positions within eligible chunks [S1].
3. **Surface Snapping (Optional)**: If enabled, the system queries the terrain generator for Signed Distance Field (SDF) values at candidate positions and nearby samples, interpolating to find the zero-crossing surface [S1]. This is an interpolation approximation, not exact ray-casting, and may fail if the terrain is edited after generation [S1].
4. **Normal Offset**: The instance position is displaced along the interpolated surface normal by `offset_along_normal` [S1].
5. **Alignment**: Instances are rotated to align with either the local terrain origin (sphere mode) or a fixed world up vector [S1].
6. **Instance Creation**: Instances are created as either `MultiMesh` (high-performance, static) or `Scene` (dynamic, scriptable) types [S1].
7. **Persistence**: If `persistent` is true and a `VoxelStream` is active, the instance's ID and position are saved; edited blocks prevent regeneration for these instances [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vector3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __add__(self, other): return Vector3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __mul__(self, scalar): return Vector3(self.x * scalar, self.y * scalar, self.z * scalar)
    def length(self): return (self.x**2 + self.y**2 + self.z**2)**0.5

class VoxelInstanceLibraryMultiMeshItem: pass
class VoxelInstanceLibrarySceneItem: pass

def align_to_sphere_normal(position):
    length = position.length()
    return Vector3(position.x / length, position.y / length, position.z / length)

def align_to_world_up():
    return Vector3(0, 1, 0)

def process_terrain_chunks(voxel_instancer, chunks):
    library = voxel_instancer.library
    generator = voxel_instancer.generator
    stream = voxel_instancer.stream
    up_mode = voxel_instancer.up_mode

    for chunk in chunks:
        for item in library.items:
            if chunk.lod_level != item.lod_index:
                continue

            candidates = item.generator.evaluate(chunk)

            for candidate in candidates:
                position = candidate.position

                if item.snap_to_generator_sdf_enabled:
                    # Query SDF at candidate and neighbors, interpolate to zero-crossing
                    # [Comment: Sources do not give exact interpolation routine. Standard: sample neighbors, linearly interpolate to find t where sdf(t) = 0.]
                    normal = generator.query_sdf_gradient(position)
                    position = position + normal * item.offset_along_normal
                else:
                    # [Comment: Sources do not give default fallback behavior. Standard: use chunk surface or center.]
                    position = position + Vector3(0, item.offset_along_normal, 0)

                if up_mode == 'Sphere mode':
                    rotation = align_to_sphere_normal(position)
                else:
                    rotation = align_to_world_up()

                if isinstance(item, VoxelInstanceLibraryMultiMeshItem):
                    mesh = select_lod_mesh(item, chunk, position)
                    # [Comment: Sources do not give exact batch addition routine. Standard: append transform and mesh index to batch buffer.]
                    pass
                elif isinstance(item, VoxelInstanceLibrarySceneItem):
                    scene_instance = item.scene.instantiate()
                    scene_instance.position = position
                    scene_instance.rotation = rotation
                    voxel_instancer.add_child(scene_instance)

                if item.persistent and stream:
                    stream.save_instance(item.id, position)

def select_lod_mesh(item, chunk, position):
    # [Comment: Sources do not give exact LOD selection logic. Standard: compare distance to camera against item.mesh_lod_settings ratios relative to chunk view distance.]
    return item.mesh
```

## Subtopics

### [Voxel Instancing System](12566-voxel-instancing-system.md)
The Voxel Instancing System is a procedural spawning mechanism for voxel terrains, designed primarily for natural elements like grass, rocks, and trees. It solves the problem of efficiently rendering thousands of distinct objects by evaluating spawn rules against terrain chunks and placing instances either as high-performance `MultiMesh` batches or flexible `Scene` nodes. It distinguishes itself from sibling methods by its tight integration with the voxel terrain's LOD system and its ability to snap instances to procedural surface approximations using SDF interpolation.

### [Block LOD Instancing](12572-block-lod-instancing.md)
Block LOD Instancing is a method for spawning voxel instances based on the resolution level (LOD) of the underlying terrain chunks. It solves the problem of efficiently populating large voxel worlds by assigning instance types to specific distance ranges via chunk resolution. It differs from standard instancing by tying spawn eligibility directly to the terrain's hierarchical LOD structure rather than a fixed global radius, allowing larger items to spawn on distant, low-resolution chunks while smaller items populate high-resolution areas.

#### Procedural Object Instancing
**Definition**: Items spawn based on procedural rules via a built-in generator, rather than manual painting. The generator is accessed via the `generator` property of `VoxelInstanceLibraryItem` [S1].
**Pitfalls**: Manual editing of instances in the editor is not supported [S1]. Custom instance generator API is recent and may change; it is not available to scripts at time of writing [S1]. `VoxelInstancer` registers to parent block events to know when to spawn, which is not available to the script API of `VoxelLodTerrain` [S1].

## Background
The Voxel Instancing System is part of the Voxel Tools suite for the Godot engine [S1]. It was designed to handle the specific constraints of voxel terrain rendering, where geometry changes dynamically and LODs vary by distance [G]. The persistent instance feature relies on `VoxelStreamSQLite` for data storage [S1].

## Sources
[S1] Instancing - Voxel Tools documentation, https://voxel-tools.readthedocs.io/en/latest/instancing/, fetched 2026-10-27
