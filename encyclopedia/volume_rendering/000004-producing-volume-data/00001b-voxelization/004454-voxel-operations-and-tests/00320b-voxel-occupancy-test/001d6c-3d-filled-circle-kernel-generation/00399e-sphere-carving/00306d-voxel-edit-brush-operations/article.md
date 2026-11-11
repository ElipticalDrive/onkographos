# Voxel Edit Brush Operations

Also called: Place/Carve/Fill/Paint, Real-time Voxel Editing, VoxelEdit

## Overview
Voxel Edit Brush Operations provide a runtime pipeline for modifying voxel occupancy and material properties within a world-space spherical radius [S1, unofficial][S2]. It solves the problem of interactive, real-time terrain or volume manipulation (such as carving or filling) by applying discrete edits to an unbounded voxel grid [S1, unofficial][S2]. Unlike parent methods that may rely on sparse structures or offline processing, this approach ensures consistent behavior across chunk boundaries by addressing voxels using unbounded world-grid coordinates rather than chunk-local indices [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A world-space ray (origin, direction), a brush radius, and an operation type (Place, Carve, Fill, Paint) [S1, unofficial][S2].
*   **Output:** Modified voxel occupancy and material data in the underlying grid, potentially triggering mesh rebuilds for affected chunks [S1, unofficial].

**Data Structures:**
*   **VoxelGrid:** An unbounded grid accessed via world-space coordinates. The grid is logically divided into chunks (e.g., 32-cell boundaries), but logical addressing is global [S1, unofficial].
*   **VoxelEditCommand:** A command object pushed to the edit queue upon stroke release [S1, unofficial].
*   **Stroke Buffer:** Accumulates edits across frames during mouse interaction [S1, unofficial].

**Steps:**
1.  **Ray Traversal:** Transform the input mouse ray into the voxel terrain's local space if the terrain is moved or scaled [S1, unofficial]. Perform a DDA voxel raycast to determine the hit cell, entered face normal, distance, and world point [S1, unofficial].
2.  **Brush Application:** Define a spherical area of influence centered on the interaction point (or hit point) with the specified brush radius [S1, unofficial][S2].
3.  **Operation Execution:** Apply the selected operation (Place, Carve, Fill, Paint) to all voxels within the spherical radius [S1, unofficial][S2].
    *   *Place:* Set voxels to occupied.
    *   *Carve:* Set voxels to empty.
    *   *Fill:* Fill empty spaces within the radius.
    *   *Paint:* Modify material properties [S1, unofficial][S2].
4.  **Chunk Management:** Identify the chunks affected by the edit. Mark the owning chunk and any existing neighbor across a shared face as "dirty" to trigger a rebuild of the mesh ring around the edit [S1, unofficial].
5.  **Stroke Accumulation:** During active interaction, accumulate edits into a stroke buffer. Snapshots of whole chunks (not just cell deltas) are taken to ensure lossless undo/redo on sparse volumes [S1, unofficial].
6.  **Command Submission:** On stroke release, push a single `VoxelEditCommand` containing the accumulated changes [S1, unofficial].
7.  **Mesh Rebuild:** The voxel mesh builder processes dirty chunks, pruning meshes for chunks that no longer exist and rebuilding those that have changed [S1, unofficial].

**Parameters:**
*   **Brush Radius:** A world-space radius defining the spherical area of influence [S1, unofficial].
*   **Operation Type:** Enum specifying Place, Carve, Fill, or Paint [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Chunk Boundaries:** Addressing must use unbounded world-grid coordinates to prevent behavior changes at chunk boundaries (e.g., 32-cell boundaries) [S1, unofficial].
*   **Undo/Redo:** Undoing an edit that created a chunk must remove the chunk allocation entirely to avoid leaving empty allocations behind [S1, unofficial].
*   **Ray Start Inside Solid:** If the ray starts inside solid, the entered face normal is zero [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_voxel_edit(ray_origin, ray_direction, brush_radius, operation_type):
    # Transform ray to local space if needed
    local_ray = transform_to_local_space(ray_origin, ray_direction)
    
    # Perform DDA raycast to find hit
    hit = dda_voxel_raycast(local_ray)
    
    # Determine center for brush (hit point or ray origin)
    center = hit.world_point if hit.hit else ray_origin
    
    # Identify affected chunks
    affected_chunks = get_chunks_in_sphere(center, brush_radius)
    
    # Snapshot chunks for undo support
    snapshot_chunks = [chunk.clone() for chunk in affected_chunks]
    
    # Apply operation to voxels in sphere
    for chunk in affected_chunks:
        for voxel in chunk.get_voxels_in_sphere(center, brush_radius):
            if operation_type == "Carve":
                voxel.occupancy = 0
            elif operation_type == "Place":
                voxel.occupancy = 1
            elif operation_type == "Fill":
                if voxel.occupancy == 0:
                    voxel.occupancy = 1
            elif operation_type == "Paint":
                voxel.material = apply_paint_material(voxel, center, brush_radius)
            
            # Mark chunk as dirty
            chunk.mark_dirty()
            # Mark neighbors as dirty if shared face
            for neighbor in chunk.get_shared_face_neighbors():
                neighbor.mark_dirty()
    
    # Create command
    command = VoxelEditCommand(
        operation=operation_type,
        affected_chunks=affected_chunks,
        snapshots=snapshot_chunks,
        brush_radius=brush_radius
    )
    
    # Push command to queue
    edit_queue.push(command)
    
    # Trigger mesh rebuild for dirty chunks
    rebuild_meshes(affected_chunks)
```

## Subtopics

### [Voxel Edit Brush Operations — Place/Carve/Fill/Paint, Real-time Voxel Editing, VoxelEdit](12596-voxel-edit-brush-operations-place-carve-fill-paint-real-time.md)
Implements real-time voxel carving in a browser-based 3D Gaussian Splatting renderer using a discrete voxel occupancy mask injected into the fragment shader. It solves the problem of crisp, axis-aligned removal of splats using a `discard` instruction, avoiding the performance penalty of per-fragment analytic SDF evaluations or per-splat geometric tests. Unlike parent methods that may rely on sparse structures or offline processing, this approach uses a dense 3D texture sized to the voxel grid, enabling O(1) per-fragment lookup regardless of the number of carved cells.

## Operating characteristics
*   **Time Complexity:** O(1) per-fragment lookup for the dense texture variant [S1, unofficial]. Chunk-based edits are proportional to the number of affected cells and chunks [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of changed chunks for sparse volumes [S1, unofficial]. Dense texture variant requires memory for the 3D texture [S1, unofficial].
*   **Performance:** Real-time editing is supported [S2]. Byte-equality validation against reference implementations (voxlap C's `setspans`) ensures correctness [S2].
*   **Strengths:** Consistent behavior across chunk boundaries, lossless undo/redo on sparse volumes, real-time interaction [S1, unofficial][S2].
*   **Limitations:** Requires careful management of chunk boundaries and dirty regions [S1, unofficial]. Dense texture variant may have higher memory usage [S1, unofficial].

## Background
Originating from implementations like OloEngineBase and roxlap-cli, this method enables interactive voxel editing in real-time applications [S1, unofficial][S2]. It builds on foundational voxelization and occupancy testing techniques [Parent].

## Sources
[S1] feat(terrain): DDA voxel raycast + editing brushes with undo/redo, https://github.com/drsnuggles8/OloEngineBase/pull/973, fetched 2026-10-25 (unofficial)
[S2] roxlap-cli 0.32.0, https://docs.rs/crate/roxlap-cli/latest, fetched 2026-10-29
