# Multi-pass Projection Voxelization

Also called: Primitive Voxelization

## Overview
Multi-pass Projection Voxelization is the initial surface voxelization step that projects input surface primitives onto 2D planes and rasterizes them into a 3D volume, producing an "incorrect" surface voxelization that includes hidden primitives [S1]. It differs from its parent, Single 2D Projection Voxelization, by using multiple orthogonal projection directions and compositing the results to handle surfaces perpendicular to the projection plane, which would otherwise have zero area and fail to rasterize [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of surface primitives (e.g., triangles) after culling [S1].
*   **Output:** A 3D texture representing the initial voxelization, where voxels are marked if they intersect the projected primitives [S1].

**Data Structures:**
*   **3D Texture:** A single 3D texture with 32-bit RGBA format is used for volume encoding [S1]. For a resolution of $1024^3$, this requires 128 MB of video memory [S1].
*   **Multiple Render Targets (MRT):** The GPU renders to 8 color buffers simultaneously [S1].

**Parameters:**
*   **Volumetric Resolution ($n$):** The side length of the cubic volume [S1]. Typical value: $1024$ [S1].
*   **View Volume Enlargement:** The view volume is set slightly larger than the Minkowski sum bounding box to ensure outer voxels are connected [S1]. The enlargement percentage is calculated as:
    $$ \text{Enlargement} = \frac{2(1 + \epsilon)}{n - 2(1 + \epsilon)} $$
    where $n$ is the volumetric resolution and $\epsilon$ is an infinitesimal number [S1].

**Steps:**
1.  **Project:** Project each surface primitive onto a 2D plane using orthogonal projection [S1].
2.  **Rasterize:** Rasterize the projected surfaces to produce fragments containing depth information and 2D coordinates [S1].
3.  **Map:** Map each fragment to a corresponding voxel in the 3D volume using the 2D coordinates and depth [S1].
4.  **Encode:** Use a fragment shader to compute an RGBA color for each fragment based on its depth information, which is passed as a texture coordinate [S1].
5.  **Render Passes:** Render the vertex buffer object (VBO) multiple times using MRT to fill the 3D texture [S1]. For a $1024^3$ resolution, render 4 times ($1024 / 32 / 8$) [S1].
6.  **Handle Perpendicular Surfaces:** To address the failure of rasterizing surfaces perpendicular to the projection plane, project the model along three orthogonal directions (e.g., x, y, z) [S1].
7.  **Compose:** Use a fragment shader to sample the x and y 3D textures and write the result to the z texture, then delete the x and y textures to free memory [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_pass_projection_voxelization(primitives, resolution):
    # Initialize three 3D textures for x, y, z projection directions
    # Each texture has resolution^3 voxels with 32-bit RGBA format
    tex_x = [[[ [0.0, 0.0, 0.0, 0.0] for _ in range(resolution)] for _ in range(resolution)] for _ in range(resolution)]
    tex_y = [[[ [0.0, 0.0, 0.0, 0.0] for _ in range(resolution)] for _ in range(resolution)] for _ in range(resolution)]
    tex_z = [[[ [0.0, 0.0, 0.0, 0.0] for _ in range(resolution)] for _ in range(resolution)] for _ in range(resolution)]

    # Calculate view volume enlargement to ensure outer voxels are connected
    epsilon = 1e-9
    enlargement_factor = 2 * (1 + epsilon) / (resolution - 2 * (1 + epsilon))
    bb_min = [min(p.min_coord[d] for p in primitives) for d in range(3)]
    bb_max = [max(p.max_coord[d] for p in primitives) for d in range(3)]
    view_volume = expand_bounding_box(bb_min, bb_max, enlargement_factor)

    # Render along three orthogonal directions
    render_vbo(primitives, view_volume, 'x', tex_x)
    render_vbo(primitives, view_volume, 'y', tex_y)
    render_vbo(primitives, view_volume, 'z', tex_z)

    # Compose results: sample x and y textures, write to z texture
    for i in range(resolution):
        for j in range(resolution):
            for k in range(resolution):
                tex_z[i][j][k] = [max(tex_x[i][j][k][c], tex_y[i][j][k][c]) for c in range(4)]

    # Free memory by clearing x and y textures
    for i in range(resolution):
        for j in range(resolution):
            for k in range(resolution):
                tex_x[i][j][k] = [0.0, 0.0, 0.0, 0.0]
                tex_y[i][j][k] = [0.0, 0.0, 0.0, 0.0]

    return tex_z

def expand_bounding_box(bb_min, bb_max, factor):
    expanded_min = [bb_min[d] - (bb_max[d] - bb_min[d]) * factor / 2 for d in range(3)]
    expanded_max = [bb_max[d] + (bb_max[d] - bb_min[d]) * factor / 2 for d in range(3)]
    return expanded_min, expanded_max

def render_vbo(primitives, view_volume, direction, target_texture):
    # Projects primitives orthogonally along direction onto 2D plane
    # Rasterizes fragments with depth and 2D coordinates
    # Maps fragments to 3D volume slices using MRT (8 color buffers per pass)
    # Number of passes = resolution // 32 // 8
    passes = resolution // 32 // 8
    for p in range(passes):
        # Fragment shader computes RGBA color based on depth passed as texture coordinate
        # Writes to target_texture slice corresponding to pass p
        # Not given in the sources: exact GPU rasterization and MRT slice mapping logic
        pass
```

**Edge Cases and Pitfalls:**
*   **Perpendicular Surfaces:** Surfaces perpendicular to the projection plane have zero area and are not rasterized [S1]. This is mitigated by using three orthogonal projections [S1].
*   **Hidden Primitives:** The initial voxelization includes hidden primitives, which are later used as a barrier for flood fill [S1].

## Operating characteristics
*   **Time Complexity:** Not explicitly given, but the method takes approximately one second for a $1024^3$ resolution on a GPU [S1].
*   **Memory Complexity:** Requires 128 MB of video memory for a $1024^3$ resolution using a single 3D texture with 32-bit RGBA format [S1].
*   **Strengths:** Efficient use of GPU rasterization; reduces memory cost and number of render passes by using 3D textures and MRT [S1].
*   **Limitations:** Produces an "incorrect" surface voxelization that includes hidden primitives, requiring a subsequent flood fill step [S1].

## Background
Multi-pass Projection Voxelization, also known as Primitive Voxelization, was introduced in the paper "A GPU-based voxelization approach to 3D Minkowski sum computation" [S1]. It is used as the first step in a voxelization algorithm for computing 3D Minkowski sums, providing an initial surface voxelization that serves as a barrier for subsequent flood fill operations [S1].

## Sources
[S1] A GPU-based voxelization approach to 3D Minkowski sum computation, https://doi.org/10.1145/1839778.1839783, fetched 2026-11-05
