# Static Mesh Caching
Also called: Mesh Caching, Voxel Mesh Caching

## Overview
Static Mesh Caching reduces draw calls and GPU data upload frequency by aggregating all voxel geometry into a single `Mesh` object [S1, unofficial]. It differs from its parent [Voxel Block Placement with Face Culling](923-voxel-block-placement-with-face-culling.md) by ignoring shared face optimization; instead of toggling visibility, it rebuilds the full geometry of every visible voxel into one buffer [S1, unofficial]. It differs from [Dynamic Block Mesh Management](3574-dynamic-block-mesh-management.md) by maintaining a single global mesh rather than managing multiple independent mesh objects [S1, unofficial].

## How it works
**Inputs:** A world data structure containing a `HashSet` of active voxel positions (`IVec3`) and an optional cached `Mesh` field [S1, unofficial].
**Outputs:** A single `Mesh` object containing vertices for all active voxels [S1, unofficial].

**Data Structures:**
The world state is represented by a `World` struct containing:
- `voxels`: A `HashSet` of 3D integer coordinates (`IVec3`) representing active voxel positions.
- `mesh`: An optional `Mesh` object. If `none`, the mesh is invalid or uninitialized [S1, unofficial].

**Steps:**
1. **Check Validity:** During the render pass, check if `world.mesh` is not `none` [S1, unofficial].
2. **Render Cached:** If the mesh exists, upload it once and draw it using a single draw call [S1, unofficial].
3. **Rebuild if Invalid:** If `world.mesh` is `none`:
   a. Initialize an empty vertex list.
   b. Iterate through every position in `world.voxels`.
   c. For each position, generate the 36 vertices (6 faces × 6 vertices) of a unit cube centered at that position [G].
   d. Append these vertices to the list.
   e. Construct a new `Mesh` from the accumulated vertices and assign it to `world.mesh` [S1, unofficial].
   f. Draw the newly created mesh [S1, unofficial].
4. **Invalidate on Change:** Whenever a voxel is added or removed from `world.voxels`, set `world.mesh` to `none` to force a rebuild on the next render frame [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class World:
    voxels: set
    mesh: Optional[Mesh]

    def render(self):
        if self.mesh is not None:
            draw_mesh(self.mesh)
        else:
            self.mesh = self._create_mesh()
            draw_mesh(self.mesh)

    def _create_mesh(self) -> Mesh:
        vertices = []
        for voxel_pos in self.voxels:
            # Generate 36 vertices for a unit cube at voxel_pos [G]
            vertices.extend(draw_cube(voxel_pos))
        return Mesh.from_vertices(vertices)

    def flip_voxel(self, position: IVec3):
        self.mesh = None
        if position in self.voxels:
            self.voxels.remove(position)
        else:
            self.voxels.add(position)
```

**Edge Cases and Pitfalls:**
- **Visual Discrepancies:** Modifying `world.voxels` without setting `world.mesh` to `none` causes the renderer to display stale geometry [S1, unofficial].
- **Performance Cost:** Rebuilding the entire mesh is computationally expensive for large worlds. For example, a 2048x2048 world contains over 4 million cubes; rebuilding this geometry for a single block change is prohibitive [S1, unofficial].

## Subtopics

### [Chunked World Management](1405-chunked-world-management.md)
Splits the voxel world into independent chunks to allow localized mesh rebuilding, solving the performance bottleneck of global invalidation inherent in Static Mesh Caching. It differs from its parent by partitioning the single mesh object into multiple chunk-specific meshes, enabling independent updates per chunk rather than global invalidation.

## Operating characteristics
- **Time Complexity:** $O(V)$ for mesh creation, where $V$ is the number of voxels. $O(1)$ for rendering if the mesh is valid. $O(1)$ for invalidation, but $O(V)$ for the subsequent rebuild [G].
- **Memory Complexity:** $O(V)$ to store vertex data for all voxels [G].
- **Performance:** Significantly reduces draw calls from $V$ (or number of chunks) to 1. However, it incurs high CPU cost during updates [S1, unofficial].
- **Strengths:** Minimal draw call overhead; simple implementation.
- **Limitations:** Cannot handle dynamic worlds efficiently. Rebuilding the entire mesh for a single change is too slow for large resolutions (e.g., > 32x32) [S1, unofficial].
- **When to use:** Small, static, or rarely changing voxel worlds.
- **When not to use:** Large, dynamic voxel worlds where frequent block placement/removal occurs.

## Background
Originated in tutorials for building voxel engines from scratch, specifically highlighted in "Make yourself a Voxel Engine THIS weekend" [S1, unofficial]. It serves as a foundational optimization step before introducing more complex techniques like face culling or chunking [S1, unofficial].

## Sources
[S1] Make yourself a Voxel Engine THIS weekend, https://daymare.net/blogs/voxel-engine-in-a-weekend/, fetched 2026-10-26 (unofficial)
