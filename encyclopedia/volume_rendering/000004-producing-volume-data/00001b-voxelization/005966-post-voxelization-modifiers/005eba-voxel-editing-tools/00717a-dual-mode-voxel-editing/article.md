# Dual-Mode Voxel Editing

Also called: View/Edit Modes

## Overview
Dual-Mode Voxel Editing is a workflow strategy that separates voxel data representation into two distinct states—**View** and **Edit**—to balance high-performance mesh-based visualization with flexible voxel-level manipulation [S1, unofficial]. It solves the performance bottleneck of dynamically grouping voxels into objects for rendering by isolating the heavy voxel-grid computation to a single active object in Edit mode, while rendering all other objects as pre-computed triangulated meshes in View mode [S1, unofficial]. This approach avoids the cost of scanning all chunks to find voxels for every object update, which breaks the performance of chunk-based systems [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scene containing multiple voxel-based objects. Each object is defined by a 3D grid of voxels.
*   **Output:** A rendered scene where objects are displayed as triangulated meshes, and one selected object is editable via voxel manipulation.

**Data Structures:**
1.  **Chunk:** A fixed 3D grid of bytes with dimensions $32 \times 32 \times 32$. Each byte represents a Voxel ID mapped to a color via a 1D texture palette.
2.  **World Grid (Object Grid):** A fixed 3D grid of Chunks with dimensions $8 \times 8 \times 8$.
    *   Total resolution per object: $256 \times 256 \times 256$ voxels ($32 \times 8 = 256$).
    *   Total chunks per object: $8 \times 8 \times 8 = 512$ chunks.
3.  **Voxel Data:** Each voxel is described by 1 byte, representing a Material ID in the range $[1, 255]$.
4.  **Mesh:** A triangulated mesh generated from a Chunk or a combination of Chunks using Greedy meshing.

**Parameters:**
*   **Voxel Grid Resolution:** $256^3$ voxels per object [S1, unofficial].
*   **Voxel Data Type:** 1 byte per voxel [S1, unofficial].
*   **Maximum Memory Usage per Object (Edit Mode):** $256^3$ bytes = 16 MB [S1, unofficial].
*   **Maximum Number of Meshes per Object (Edit Mode):** 512 (one per chunk) [S1, unofficial].
*   **Editing Constraint:** Only a single object can be edited at a time [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** For each object in the scene, allocate a $256^3$ voxel grid (1 byte per voxel) and a corresponding World Grid of $8 \times 8 \times 8$ Chunks.
2.  **View Mode Rendering:**
    *   Represent each object as a single triangulated mesh (1 Mesh per object).
    *   Render the scene using these meshes.
    *   *Note:* If objects are moved, seamless meshes larger than $256^3$ can be created [S1, unofficial].
3.  **Transition to Edit Mode:**
    *   Select one object to edit.
    *   Load the 3D grid of chunks for the selected object from disk into memory [S1, unofficial].
    *   Switch the rendering representation for this object from a single mesh to a group of up to 512 triangulated chunks (one mesh per chunk) [S1, unofficial].
    *   Render the scene with the selected object as chunks and all other objects as single meshes.
4.  **Editing:**
    *   Perform voxel-level manipulation (e.g., painting, deletion, boolean operations) on the active object's voxel grid [S1, unofficial].
    *   Update the corresponding chunks and their meshes as needed.
5.  **Transition to View Mode:**
    *   Save the 3D grid of chunks for the edited object to disk [S1, unofficial].
    *   Combine all chunks of the edited object into a single triangulated mesh using Greedy meshing [S1, unofficial].
    *   Switch the rendering representation for this object back to a single mesh.
    *   Render the scene with all objects as single meshes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Chunk:
    def __init__(self):
        # 32x32x32 grid of bytes
        self.data = [0] * (32 * 32 * 32)
        self.mesh = None # Triangulated mesh generated from this chunk

class Object:
    def __init__(self):
        # 8x8x8 grid of Chunks
        self.chunks = [[Chunk() for _ in range(8)] for _ in range(8)]
        self.voxel_grid = [0] * (256 * 256 * 256) # 1 byte per voxel
        self.full_mesh = None # Single mesh for View mode
        self.chunk_meshes = [] # List of 512 meshes for Edit mode
        self.is_edited = False

    def get_chunk_index(self, x, y, z):
        # Map global voxel coordinates to chunk and local coordinates
        cx = x // 32
        cy = y // 32
        cz = z // 32
        lx = x % 32
        ly = y % 32
        lz = z % 32
        return (cx, cy, cz), (lx, ly, lz)

    def set_voxel(self, x, y, z, material_id):
        # material_id in [1, 255]
        chunk_coords, local_coords = self.get_chunk_index(x, y, z)
        cx, cy, cz = chunk_coords
        lx, ly, lz = local_coords
        # Update voxel grid
        idx = x + y * 256 + z * 256 * 256
        self.voxel_grid[idx] = material_id
        # Update chunk data
        chunk_idx = lx + ly * 32 + lz * 32 * 32
        self.chunks[cx][cy][cz].data[chunk_idx] = material_id
        # Mark chunk for mesh rebuild
        self.chunks[cx][cy][cz].mesh = None

    def rebuild_chunk_mesh(self, chunk):
        # Generate triangulated mesh from chunk data using Greedy meshing
        # Implementation details of Greedy meshing are not specified in sources
        chunk.mesh = generate_greedy_mesh(chunk.data)

    def rebuild_full_mesh(self):
        # Combine all 512 chunk meshes into a single mesh
        # This step causes a performance freeze
        self.full_mesh = combine_meshes([chunk.mesh for row in self.chunks for chunk in row])

    def enter_edit_mode(self):
        # Load from disk
        self.load_from_disk()
        # Rebuild individual chunk meshes
        for row in self.chunks:
            for chunk in row:
                if chunk.mesh is None:
                    self.rebuild_chunk_mesh(chunk)
        self.chunk_meshes = [chunk.mesh for row in self.chunks for chunk in row]

    def enter_view_mode(self):
        # Save to disk
        self.save_to_disk()
        # Combine chunks into single mesh
        self.rebuild_full_mesh()

class Scene:
    def __init__(self):
        self.objects = []
        self.edit_mode_object = None

    def set_edit_mode(self, obj_index):
        if self.edit_mode_object is not None:
            self.edit_mode_object.enter_view_mode()
        self.edit_mode_object = self.objects[obj_index]
        self.edit_mode_object.enter_edit_mode()

    def render(self):
        for obj in self.objects:
            if obj == self.edit_mode_object:
                # Render as group of chunks (up to 512 meshes)
                for mesh in obj.chunk_meshes:
                    draw_mesh(mesh)
            else:
                # Render as single mesh
                draw_mesh(obj.full_mesh)
```

**Edge Cases and Pitfalls:**
*   **Performance Freeze:** Saving/loading chunks to/from disk during mode changes causes a freeze [S1, unofficial].
*   **Mesh Combination Cost:** Combining all 512 chunks into a single mesh in Edit-to-View transition causes a freeze [S1, unofficial].
*   **Memory Limit:** Each object in Edit mode consumes 16 MB of RAM [S1, unofficial].
*   **Single Object Edit:** Only one object can be edited at a time [S1, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   **View Mode:** $O(1)$ per object for rendering (1 mesh per object).
    *   **Edit Mode:** $O(N)$ per object for rendering, where $N$ is the number of chunks (up to 512).
    *   **Mode Transition:** High cost due to disk I/O and mesh combination [S1, unofficial].
*   **Memory Complexity:**
    *   **Edit Mode:** 16 MB per object for the voxel grid [S1, unofficial].
    *   **View Mode:** Memory usage equivalent to polygonal meshes (normals, etc.) for non-active objects [S1, unofficial].
*   **Performance:**
    *   Fast voxel updates in Edit mode (1 chunk per mouse click) [S1, unofficial].
    *   Avoids the performance break of scanning all chunks for object updates [S1, unofficial].
*   **Strengths:**
    *   Good memory usage for non-active objects [S1, unofficial].
    *   Supports object grouping, export, deletion, and editing [S1, unofficial].
    *   Objects can be moved to create seamless meshes larger than $256^3$ voxels [S1, unofficial].
    *   Supports boolean operations (subtraction, addition) between objects using their internal 3D grids [S1, unofficial].
*   **Limitations:**
    *   Too many meshes per edited object (up to 512) [S1, unofficial].
    *   Performance freeze during mode changes [S1, unofficial].
    *   Requires saving each object separately to disk [S1, unofficial].

## Background
Dual-Mode Voxel Editing was developed as a solution for voxel editing software to support object grouping and efficient rendering. It addresses the limitations of chunk-based systems where looking up all chunks to find voxels for an object breaks performance [S1, unofficial]. The method was implemented to allow features like object visibility changes and boolean operations while maintaining interactive performance [S1, unofficial]. It is comparable to MagicaVoxel's group system (Outline), though the internal implementation details of MagicaVoxel are not publicly documented [S1, unofficial].

## Sources
[S1] Group voxels into object?, https://gamedev.stackexchange.com/questions/205011/group-voxels-into-object, fetched 2026-10-26 (unofficial)
