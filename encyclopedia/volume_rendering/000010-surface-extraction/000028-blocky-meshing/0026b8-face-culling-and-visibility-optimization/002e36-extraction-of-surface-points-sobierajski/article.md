# Extraction of Surface Points (Sobierajski)

Also called: Sobierajski's method, Visible boundary voxel extraction

## Overview
Extraction of Surface Points (Sobierajski) is a fast display method for the direct rendering of boundary surfaces by extracting only visible boundary voxels from volume data [S1, unofficial]. It solves the problem of efficient surface visualization by pre-filtering the voxel set based on viewing directions, reducing the number of primitives passed to the renderer [S1, unofficial]. Unlike [Face Culling](8735-face-culling.md) which operates on mesh geometry after construction, this method operates on the voxel grid directly to identify potentially visible surfaces before rendering [S1, unofficial].

## How it works
**Inputs:** 3D volume data (density/opacity field); set of viewing directions (typically six orthographic views along coordinate axes) [S1, unofficial].
**Outputs:** A list of unique boundary voxels with attributes (coordinates, normal vectors) suitable for rendering [S1, unofficial].

**Data Structures:**
*   **Boundary Voxel List:** A dynamic array or linked list storing extracted boundary voxels.
*   **Voxel Attribute:** For each voxel, stores 3D coordinates $(x, y, z)$ and surface normal vector $\mathbf{n}$ [S1, unofficial].
*   **Visibility Mask:** Implicitly derived from the 2D boundary tracing results; no explicit large mask structure is required if processing is done slice-by-slice [S1, unofficial].

**Steps:**

1.  **Visibility Determination via Orthographic Views:**
    Determine the visibility of boundary voxels using a set of viewing directions. For efficiency, six orthographic views (along $\pm x, \pm y, \pm z$ axes) are typically sufficient to approximate the set of all potentially visible boundary voxels [S1, unofficial]. Increasing the number of directions improves approximation accuracy at the cost of performance [S1, unofficial].

2.  **2D Boundary Tracing:**
    For each viewing direction, perform visibility calculations using a 2D boundary tracing algorithm on slices perpendicular to the corresponding coordinate axis [S1, unofficial]. This identifies which voxels form the visible boundary from that specific viewpoint [S1, unofficial].

3.  **Extraction and Deduplication:**
    Collect all voxels identified as visible in any of the viewing directions. Remove duplicate elements from the resulting set to ensure each boundary voxel is listed only once [S1, unofficial].

4.  **Attribute Assignment:**
    For each unique boundary voxel in the list, compute and store necessary rendering attributes, specifically the voxel's 3D coordinates and its surface normal vector $\mathbf{n}$ [S1, unofficial].

5.  **Voxel Culling (Backface Culling):**
    Reduce the size of the voxel list to improve rendering speed. For a specific viewing direction $\mathbf{v}$, calculate the dot product between the surface normal $\mathbf{n}$ and the viewing direction $\mathbf{v}$ [S1, unofficial].
    *   If $\mathbf{n} \cdot \mathbf{v} > 0$, the voxel is a back face and is not rendered [S1, unofficial].
    *   If $\mathbf{n} \cdot \mathbf{v} \le 0$, the voxel is retained for rendering [S1, unofficial].
    This step is analogous to backface culling in polygon rendering [S1, unofficial].

6.  **Projection and Hole Avoidance:**
    Project each voxel onto the image plane. Since adjacent voxels may map to non-adjacent pixels, holes can appear in the image [S1, unofficial]. To avoid this, project each voxel onto several pixels depending on the viewing direction [S1, unofficial].

7.  **Rendering:**
    Pass the filtered set of surface points to the rendering engine [S1, unofficial].
    *   **Shading:** Shade the projected boundary surface points according to lighting conditions [S1, unofficial].
    *   **Cutting Planes:** If cutting planes are used, render voxels intersected by the cutting plane by projecting their original density values onto the image plane [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sobierajski_extraction(volume_data, viewing_directions):
    """
    Extract visible boundary voxels from volume data.
    
    Args:
        volume_data: 3D array of density/opacity values.
        viewing_directions: List of 3D vectors representing viewing directions.
        
    Returns:
        list_of_voxels: List of dicts with 'coords' and 'normal'.
    """
    boundary_voxels = set()
    
    # Step 1 & 2: Determine visibility using 2D boundary tracing on slices
    for direction in viewing_directions:
        # Identify axis perpendicular to viewing direction
        axis = get_perpendicular_axis(direction)
        
        # Perform 2D boundary tracing on slices perpendicular to the axis
        # This returns a set of voxel coordinates visible from this direction
        visible_from_view = trace_2d_boundary(volume_data, axis)
        boundary_voxels.update(visible_from_view)
        
    # Step 3: Deduplication is handled by using a set data structure
        
    # Step 4: Compute attributes
    voxel_list = []
    for voxel_coords in boundary_voxels:
        normal = compute_surface_normal(volume_data, voxel_coords)
        voxel_list.append({
            'coords': voxel_coords,
            'normal': normal
        })
        
    # Step 5: Voxel Culling (Backface Culling)
    # Note: The source implies culling is done per viewing direction during 
    # projection or as a pre-filter. Here we filter based on the primary 
    # viewing direction used for the final render or the set of directions.
    # For a single final view 'final_view':
    final_view = viewing_directions[0] # Example: use first direction as primary
    filtered_voxels = []
    for voxel in voxel_list:
        n = voxel['normal']
        v = final_view
        if dot_product(n, v) <= 0:
            filtered_voxels.append(voxel)
            
    # Step 6 & 7: Projection and Rendering
    # The actual projection and shading are handled by the rendering engine.
    # The output is the filtered voxel list.
    return filtered_voxels

def trace_2d_boundary(volume_data, axis):
    """
    Perform 2D boundary tracing on slices perpendicular to the given axis.
    Returns a set of voxel coordinates.
    """
    visible_voxels = set()
    # Iterate over slices perpendicular to the axis
    for slice_index in range(volume_data.shape[axis]):
        slice_data = get_slice(volume_data, axis, slice_index)
        # Trace boundaries in the 2D slice
        # A voxel is a boundary voxel if it is occupied and has at least one 
        # empty neighbor in the slice plane or along the axis
        boundary_in_slice = find_2d_boundaries(slice_data)
        for (x, y) in boundary_in_slice:
            voxel_coords = reconstruct_3d_coords(x, y, slice_index, axis)
            visible_voxels.add(voxel_coords)
    return visible_voxels

def compute_surface_normal(volume_data, coords):
    """
    Compute the surface normal for a boundary voxel.
    """
    # Normal is typically the direction from the occupied voxel to the empty neighbor
    # or the gradient of the density field.
    # For simplicity, assume it's the direction to the nearest empty neighbor.
    neighbors = get_6_neighbors(coords)
    for n in neighbors:
        if not is_occupied(volume_data, n):
            return normalize(vector_subtract(n, coords))
    return [0, 0, 0] # Fallback, though boundary voxels should have at least one empty neighbor

def dot_product(v1, v2):
    return sum(a * b for a, b in zip(v1, v2))

def normalize(v):
    mag = (sum(x**2 for x in v)) ** 0.5
    return [x / mag for x in v]

def vector_subtract(v1, v2):
    return [a - b for a, b in zip(v1, v2)]

def get_slice(volume_data, axis, index):
    # Extract 2D slice from 3D volume
    # Implementation depends on array layout
    pass

def find_2d_boundaries(slice_data):
    # Find boundary voxels in 2D slice
    # A voxel is a boundary if it is occupied and has an empty neighbor in the 2D plane
    pass

def get_6_neighbors(coords):
    x, y, z = coords
    return [
        (x+1, y, z), (x-1, y, z),
        (x, y+1, z), (x, y-1, z),
        (x, y, z+1), (x, y, z-1)
    ]

def is_occupied(volume_data, coords):
    # Check if voxel is within bounds and has non-zero density
    pass

def reconstruct_3d_coords(x, y, slice_index, axis):
    # Map 2D slice coordinates back to 3D
    pass

def get_perpendicular_axis(direction):
    # Return the axis index (0, 1, or 2) perpendicular to the direction
    # For orthographic views along axes, this is straightforward
    pass
```

**Parameters:**
*   **Number of Viewing Directions:** Typically 6 (orthographic views along $\pm x, \pm y, \pm z$) [S1, unofficial]. Increasing this number improves the approximation of visible surfaces [S1, unofficial].
*   **Cutting Planes:** Optional. If used, voxels intersected by planes are rendered using original density values [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Holes in Image:** Adjacent voxels may map to non-adjacent pixels, causing visual holes. This is mitigated by projecting each voxel onto several pixels [S1, unofficial].
*   **Backface Visibility:** Voxels with $\mathbf{n} \cdot \mathbf{v} > 0$ are back faces and should be culled to improve performance [S1, unofficial].
*   **Duplicate Voxels:** The same voxel may be visible from multiple viewing directions. Deduplication is required to avoid redundant rendering [S1, unofficial].

## Subtopics

### Saito's Fast Previewing Algorithm
A previewing algorithm based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It extracts a subset of boundary voxels, converts them to geometrical primitives (e.g., crosslines perpendicular to the surface normal), and projects these primitives onto the image plane [S1, unofficial]. This method prioritizes speed over completeness, making it suitable for interactive previews [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The visibility calculation is performed using 2D boundary tracing on slices, which is efficient. The complexity is proportional to the number of voxels and the number of viewing directions [S1, unofficial].
*   **Memory Complexity:** Requires storage for the volume data and the list of boundary voxels. The list size is reduced by culling, but duplicates must be managed [S1, unofficial].
*   **Performance:** Rendering speed is directly related to the length of the voxel list. Voxel culling reduces this length, improving performance [S1, unofficial].
*   **Strengths:** Fast display method; supports flexible manipulation of extracted voxels; allows simultaneous rendering of shaded isosurfaces and cross-sectional slices [S1, unofficial].
*   **Limitations:** Approximate visibility set (unless many viewing directions are used); potential for holes if not projected onto multiple pixels [S1, unofficial].
*   **When to Use:** When fast previewing or direct rendering of boundary surfaces is required, especially in medical imaging systems where cutting operations are important [S1, unofficial].
*   **When Not to Use:** When high accuracy of the visible surface set is critical and many viewing directions are too expensive; when hole-free rendering is required without multi-pixel projection overhead.

## Background
Sobierajski proposed this fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is cited in the context of medical volume rendering techniques, where flexible manipulation of extracted voxels, such as cutting operations, is important [S1, unofficial]. The method is compared with other surface extraction and rendering techniques, highlighting its efficiency in pre-filtering visible voxels [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
