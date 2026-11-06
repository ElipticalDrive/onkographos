# Extraction of Surface Points (Sobierajski)
Also called: Sobierajski's method, Visible boundary voxel extraction

## Overview
A fast display method for direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It sets itself apart from its siblings by using a 2D boundary tracing algorithm on axis-aligned slices to determine visibility from a fixed set of viewing directions, rather than block-based culling or mask projections. It builds on [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md).

## How it works
**Inputs and Outputs**:
- Input: 3D volume data array $V$ with occupancy states.
- Output: A list of visible boundary voxels, each storing coordinates and normal vectors.

**Data Structures**:
- `VoxelList`: A dynamic list of objects. Each object contains:
  - `coords`: $(x, y, z)$ integer indices.
  - `normal`: $(n_x, n_y, n_z)$ unit vector aligned with principal axes.
  - `density`: Scalar value (for cutting plane support).

**Steps**:
1. **Visibility Determination**: Define a set of viewing directions. Typically, six orthographic views along $\pm \hat{x}, \pm \hat{y}, \pm \hat{z}$ are used [S1, unofficial].
2. **2D Boundary Tracing**: For each viewing direction, perform a 2D boundary tracing algorithm on slices perpendicular to that axis. Identify boundary voxels that are visible from that direction [S1, unofficial].
3. **Deduplication**: Merge the lists of visible voxels from all viewing directions and remove duplicate entries. The resulting list contains each visible boundary voxel exactly once [S1, unofficial].
4. **Back-Face Culling**: For each voxel in the list, calculate the dot product between its surface normal $\mathbf{n}$ and the current viewing direction $\mathbf{v}$. If $\mathbf{n} \cdot \mathbf{v} > 0$, the voxel is a back face and is removed from the list [S1, unofficial].
5. **Projection**: Project each remaining voxel onto the image plane. To avoid holes caused by adjacent voxels mapping to non-adjacent pixels, project each voxel onto several pixels depending on the viewing direction [S1, unofficial].
6. **Rendering**: Shade the projected boundary surface points according to lighting conditions. If cutting planes are used, voxels intersected by the plane are rendered by projecting their original density values onto the image plane [S1, unofficial].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_visible_boundary_voxels(volume, viewing_directions):
    """
    Extracts visible boundary voxels using Sobierajski's method.
    
    Args:
        volume: 3D array of occupancy states.
        viewing_directions: List of 3D vectors representing viewing directions.
        
    Returns:
        List of visible boundary voxels with attributes.
    """
    visible_voxels = set()
    
    # Step 2: 2D Boundary Tracing for each direction
    for v_dir in viewing_directions:
        # Determine axis perpendicular to viewing direction
        # For orthographic views, this is straightforward
        axis = get_perpendicular_axis(v_dir)
        
        # Perform 2D boundary tracing on slices perpendicular to axis
        # This identifies boundary voxels visible from v_dir
        slice_voxels = trace_boundary_2d(volume, axis, v_dir)
        
        # Add to set to handle duplicates implicitly
        for voxel in slice_voxels:
            visible_voxels.add(voxel)
            
    # Step 4: Back-Face Culling
    final_voxels = []
    for voxel in visible_voxels:
        # Calculate dot product of normal and viewing direction
        # Note: viewing direction should be consistent (e.g., from camera to object)
        dot_product = dot(voxel.normal, v_dir)
        
        # If dot product is positive, it's a back face
        if dot_product <= 0:
            final_voxels.append(voxel)
            
    return final_voxels

def trace_boundary_2d(volume, axis, view_dir):
    """
    Performs 2D boundary tracing on slices perpendicular to axis.
    Returns list of visible boundary voxels.
    """
    # Implementation detail: Iterate through slices perpendicular to axis
    # For each slice, identify boundary voxels based on occupancy
    # and visibility from view_dir
    boundary_voxels = []
    
    # Iterate over all slices perpendicular to the given axis
    for slice_idx in range(volume.size[axis]):
        slice_data = get_slice(volume, axis, slice_idx)
        
        # Identify boundary voxels in this slice
        # A voxel is a boundary if it is occupied and adjacent to empty space
        # in the direction of the view
        for voxel in slice_data:
            if is_boundary_voxel(voxel, slice_data, view_dir):
                boundary_voxels.append(voxel)
                
    return boundary_voxels

def is_boundary_voxel(voxel, slice_data, view_dir):
    """
    Checks if a voxel is a boundary voxel visible from view_dir.
    """
    # Check if voxel is occupied
    if not voxel.occupied:
        return False
        
    # Check if adjacent voxel in view direction is empty
    # This determines visibility from the current viewing direction
    adjacent_voxel = get_adjacent_voxel(voxel, view_dir)
    
    if adjacent_voxel is None or not adjacent_voxel.occupied:
        return True
        
    return False
```

## Subtopics

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It extracts a subset of boundary voxels, converts them to geometrical primitives like crosslines perpendicular to the surface normal, and projects them onto the image plane [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on the number of viewing directions and the size of the volume. 2D boundary tracing is efficient for orthographic views.
- **Memory Complexity**: Proportional to the number of visible boundary voxels.
- **Strengths**: Fast extraction of visible surfaces; supports cutting planes; simple implementation.
- **Limitations**: May produce holes if voxel projection is not handled carefully; limited to axis-aligned normals; visibility approximation depends on the number of viewing directions.
- **When to use**: Fast previewing or direct rendering of boundary surfaces where high precision is not critical.
- **When not to use**: When smooth surfaces or arbitrary viewing angles are required.

## Background
Proposed by Sobierajski [S1, unofficial] as a fast display method for direct rendering of boundary surfaces. It is part of the broader field of volume rendering techniques, specifically focusing on surface extraction and visibility optimization.

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
