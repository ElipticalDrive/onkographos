# Extraction of Surface Points (Sobierajski)

Also called: Visible boundary voxel extraction

## Overview
Extraction of Surface Points is a direct volume rendering method that accelerates surface visualization by extracting only the visible boundary voxels from the volume data [S1, unofficial]. It solves the problem of rendering overhead in large volumes by avoiding the processing of occluded or internal geometry. Unlike sibling methods that rely on bitmasks or hash sets for face culling, this approach operates on a list of extracted 3D points, projecting them onto the image plane while using viewing-direction-based visibility tests to prune back-facing surfaces [S1, unofficial].

## How it works
The method extracts a subset of boundary voxels that are potentially visible from a defined set of viewing directions, typically the six orthographic views [S1, unofficial]. These voxels are projected onto the image plane, shaded, and rendered. To prevent holes in the projected image, each voxel is projected onto multiple pixels depending on the viewing direction [S1, unofficial]. Back-facing voxels are culled based on the dot product of the surface normal and the viewing direction [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** Volume data (density values), set of viewing directions (e.g., 6 orthographic axes), cutting planes (optional).
*   **Output:** A list of visible boundary voxels with attributes (coordinates, normal vectors) and the final rendered image.

**Data Structures:**
*   **Voxel List:** A list of objects representing boundary voxels. Each object stores:
    *   `coordinates`: 3D integer coordinates $(x, y, z)$ of the voxel.
    *   `normal_vector`: 3D float vector $(n_x, n_y, n_z)$ representing the surface normal.
    *   `density`: Original density value (used for cutting planes).

**Algorithm Steps:**

1.  **Visibility Determination:**
    For each voxel in the volume, determine if it is a boundary voxel (i.e., it has at least one empty neighbor).
    For each boundary voxel, determine visibility from the set of viewing directions. A voxel is considered potentially visible if it is not occluded from any of the viewing directions. In the standard implementation, the six orthographic views are used [S1, unofficial].

2.  **2D Boundary Tracing:**
    Perform visibility calculations efficiently using a 2D boundary tracing algorithm on slices perpendicular to each coordinate axis. This identifies the boundary voxels for each viewing direction [S1, unofficial].

3.  **List Generation and Deduplication:**
    Generate a list of boundary voxels for all viewing directions. Remove duplicate elements from the list. Each voxel in the list must store its coordinates and normal vector [S1, unofficial].

4.  **Back-Face Culling:**
    For a specific viewing direction, reduce the voxel list size by culling back-facing voxels. A voxel is culled if the dot product of its surface normal $\mathbf{n}$ and the viewing direction vector $\mathbf{v}$ is positive:
    $$ \mathbf{n} \cdot \mathbf{v} > 0 $$
    If this condition is met, the voxel belongs to a back face and is not rendered [S1, unofficial].

5.  **Projection:**
    Project each remaining voxel onto the image plane. To avoid holes caused by adjacent voxels mapping to non-adjacent pixels, project each voxel onto several pixels depending on the viewing direction [S1, unofficial].

6.  **Shading and Rendering:**
    Shade the projected boundary surface points according to lighting conditions. If cutting planes are used, render voxels intersected by the cutting planes by projecting their original density values onto the image plane [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_surface_points(volume, viewing_directions):
    """
    Extracts visible boundary voxels from volume data.
    
    Args:
        volume: 3D array of density values.
        viewing_directions: List of 3D vectors representing viewing directions.
        
    Returns:
        visible_voxels: List of dicts with 'coords', 'normal', 'density'.
    """
    visible_voxels = []
    boundary_voxels = find_boundary_voxels(volume)
    
    for voxel in boundary_voxels:
        coords = voxel.coords
        normal = voxel.normal
        density = voxel.density
        
        is_visible = False
        for view_dir in viewing_directions:
            # Check if voxel is visible from this direction
            # Using 2D boundary tracing on slices perpendicular to axes
            if is_visible_from_direction(volume, coords, view_dir):
                is_visible = True
                break
        
        if is_visible:
            # Back-face culling for each viewing direction
            for view_dir in viewing_directions:
                if dot_product(normal, view_dir) <= 0:
                    visible_voxels.append({
                        'coords': coords,
                        'normal': normal,
                        'density': density,
                        'view_dir': view_dir
                    })
                    break # Voxel is visible from at least one direction without being back-facing
            
    # Remove duplicates
    unique_voxels = remove_duplicates(visible_voxels)
    return unique_voxels

def find_boundary_voxels(volume):
    """Identifies voxels that are on the surface of the volume."""
    boundary = []
    # Implementation depends on volume representation
    # [G] Iterate through all voxels, check if any neighbor is empty
    return boundary

def is_visible_from_direction(volume, coords, view_dir):
    """
    Determines if a voxel is visible from a specific direction.
    Uses 2D boundary tracing on slices.
    """
    # [G] Specific implementation of 2D boundary tracing is not detailed in sources
    # Assume it returns True if the voxel is not occluded
    return True

def dot_product(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def remove_duplicates(voxel_list):
    """Removes duplicate voxels from the list."""
    seen = set()
    unique = []
    for v in voxel_list:
        key = (v['coords'], v['normal'])
        if key not in seen:
            seen.add(key)
            unique.append(v)
    return unique
```

**Parameters:**
*   **Viewing Directions:** Typically six orthographic views are sufficient for an approximate set of visible voxels. Increasing the number of directions improves approximation but increases computation [S1, unofficial].
*   **Projection Size:** The number of pixels a voxel is projected onto depends on the viewing direction to avoid holes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Holes:** Adjacent voxels may map to non-adjacent pixels, causing holes. This is mitigated by projecting each voxel onto several pixels [S1, unofficial].
*   **Back-Facing Voxels:** Voxels with normals pointing away from the viewer must be culled to reduce rendering load [S1, unofficial].

## Subtopics

### [Trimmed Voxel Lists](11309-trimmed-voxel-lists.md)
Trimmed Voxel Lists is a surface extraction algorithm that represents an iso-surface using 3D points as geometric primitives, projecting exactly one point per visible surface cell to ensure image coverage without recursive subdivision. It differs from Dividing Cubes by avoiding subdivision; instead, it allows a single projected point to span up to three pixels on the image plane to guarantee coverage.

### Saito's Fast Previewing Algorithm
Saito's Fast Previewing Algorithm is a previewing method based on extracting a subset of boundary voxels according to a uniform distribution to increase rendering speed [S1, unofficial]. It extracts a subset of boundary voxels, converts them to geometrical primitives like crosslines perpendicular to the surface normal, and projects these primitives onto the image plane [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The visibility calculation using 2D boundary tracing on slices is efficient, but the complexity depends on the number of viewing directions and the volume size. Back-face culling reduces the number of voxels to be projected [S1, unofficial].
*   **Memory Complexity:** Requires storing a list of visible boundary voxels with their attributes (coordinates, normals, densities) [S1, unofficial].
*   **Strengths:** Fast display method for direct rendering of boundary surfaces. Reduces rendering speed dependency by culling back-facing voxels. Supports flexible manipulation of extracted voxels, such as cutting operations [S1, unofficial].
*   **Limitations:** May produce holes in the image if not projected onto multiple pixels. The approximation of visible voxels using only six orthographic views may miss some visible surfaces [S1, unofficial].
*   **When to Use:** Suitable for medical imaging systems where cutting operations and shaded isosurfaces are needed simultaneously. Useful for fast previewing when rendering speed is critical [S1, unofficial].

## Background
The method was proposed by Sobierajski as a fast display method for direct rendering of boundary surfaces [S1, unofficial]. It is based on the principle of extracting only those boundary voxels that are visible from a certain set of viewing directions. The approach is similar to back-face culling in polygon rendering [S1, unofficial]. Saito's Fast Previewing Algorithm is a variant that extracts a subset of boundary voxels for faster rendering [S1, unofficial].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
