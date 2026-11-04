# Cuberille Method

Also called: Cuberille Model, Herman-Liu method, Meagher's method, Octree Cuberille

## Overview
The Cuberille Method represents a 3D volume as a discrete set of voxels (parallelepipeds) where each voxel is either fully occupied or empty, creating a "hollow polygonal shell" approximation of the object's surface [S1, unofficial]. It solves the problem of visualizing segmented volumetric data by projecting these discrete voxel faces to the image plane using standard rasterization techniques [S1, unofficial]. Unlike smooth surface extraction methods (e.g., [Marching Cubes](link)), it preserves the blocky, discrete nature of the input data, making it suitable for applications where voxel integrity is paramount or where the data is inherently binary [S1, unofficial]. It differs from [Greedy Meshing](1776-greedy-meshing.md) by not merging adjacent coplanar faces into larger quads, resulting in a higher polygon count but simpler generation logic [S1, S2, unofficial].

## How it works
**Inputs:** A 3D binary array (voxel grid) where $1$ indicates an occupied voxel and $0$ indicates empty space. The array is typically sorted in memory (e.g., z-order or row-major).
**Outputs:** A rendered image of the object's surface, or a mesh of exposed voxel faces.

**Data Structures:**
1.  **Binary Array:** A 3D grid of bits or bytes. Let $V(x,y,z)$ be the value at coordinates $(x,y,z)$.
2.  **Octree (Optional):** A hierarchical structure where nodes represent $2 \times 2 \times 2$ blocks of voxels. A node is marked "empty" if all 8 children are 0, and "full" if all are 1. This is used to accelerate traversal for sparse data [S2, unofficial].

**Steps:**
1.  **Segmentation/Thresholding:**
    *   If the input is not binary, apply a threshold or connectivity algorithm (e.g., Lorenson et al. [S1, unofficial]) to isolate the object of interest.
    *   Generate a binary array $V$ where $V_{ijk} = 1$ if the voxel belongs to the object, else $0$ [S1, S2, unofficial].

2.  **Face Identification:**
    *   For each occupied voxel $V_{ijk}=1$, determine which of its 6 faces are exposed.
    *   A face is exposed if the adjacent voxel in that direction is empty ($0$) or out of bounds.
    *   The 6 faces correspond to normals: $\pm \hat{x}, \pm \hat{y}, \pm \hat{z}$ [S1, unofficial].

3.  **Rendering (Rasterization):**
    *   **Z-Buffer Approach:** Render each exposed face as a quad (or two triangles) into a depth buffer. Since voxels are axis-aligned, face culling can be applied: only 3 of the 6 faces are visible from any single viewpoint [S1, unofficial].
    *   **Painter's Algorithm Approach:** If the data is stored in a back-to-front sorted order (e.g., by z-coordinate), faces can be rendered in that order without a z-buffer, leveraging the "Painter's algorithm" [S2, unofficial]. This avoids depth testing overhead but requires pre-sorting or a specific memory layout [S2, unofficial].

4.  **Shading:**
    *   Assign normals to faces based on their orientation (only 6 possible normal vectors: $\pm \hat{x}, \pm \hat{y}, \pm \hat{z}$) [S1, unofficial].
    *   Apply lighting models. Note that naive polyhedral shading causes aliasing/banding due to the discrete normals [S1, unofficial].
    *   **Mitigation:** Apply low-pass filtering to the final image to reduce banding artifacts, at the cost of resolution [S1, unofficial]. Alternatively, use context-based normal estimation from neighboring voxels for smoother shading [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_cuberille(volume, width, height, camera):
    """
    volume: 3D array of shape (X, Y, Z) with values 0 or 1.
    Returns: Image buffer.
    """
    # Initialize image and z-buffer
    image = zeros((height, width, 3))
    z_buffer = zeros((height, width))
    
    # Collect all exposed faces
    faces = []
    
    # Iterate over all voxels
    for z in range(volume.shape[2]):
        for y in range(volume.shape[1]):
            for x in range(volume.shape[0]):
                if volume[x, y, z] == 1:
                    # Check 6 neighbors
                    neighbors = [
                        (x+1, y, z, 1, 0, 0),   # +X
                        (x-1, y, z, -1, 0, 0),  # -X
                        (x, y+1, z, 0, 1, 0),   # +Y
                        (x, y-1, z, 0, -1, 0),  # -Y
                        (x, y, z+1, 0, 0, 1),   # +Z
                        (x, y, z-1, 0, 0, -1)   # -Z
                    ]
                    
                    for nx, ny, nz, nx_norm, ny_norm, nz_norm in neighbors:
                        # Check if neighbor is empty or out of bounds
                        is_empty = True
                        if 0 <= nx < volume.shape[0] and \
                           0 <= ny < volume.shape[1] and \
                           0 <= nz < volume.shape[2]:
                            if volume[nx, ny, nz] == 1:
                                is_empty = False
                        
                        if is_empty:
                            # Face is exposed. Add to list.
                            # Face center is at (x+0.5, y+0.5, z+0.5)
                            # Face vertices depend on normal.
                            faces.append({
                                'center': (x + 0.5, y + 0.5, z + 0.5),
                                'normal': (nx_norm, ny_norm, nz_norm),
                                'voxel_pos': (x, y, z)
                            })
    
    # Sort faces back-to-front relative to camera for Painter's algorithm
    # Or use Z-buffer if sorting is not feasible
    faces.sort(key=lambda f: dot(f['center'], camera.direction), reverse=True)
    
    for face in faces:
        # Project face vertices to screen space
        screen_coords = project(face['center'], camera)
        
        # Determine if face is front-facing (optional culling)
        if dot(face['normal'], camera.direction) > 0:
            continue # Backface culling
            
        # Render face quad
        # Use simple shading based on normal
        color = apply_lighting(face['normal'])
        
        # Draw to image/z-buffer
        draw_quad(screen_coords, color, z_buffer)
        
    return image
```

**Parameters:**
*   **Threshold:** Value separating object from background. Not given in the sources.
*   **Filtering Strength:** For low-pass filtering to reduce banding. Not given in the sources.

**Edge Cases:**
*   **Degenerate Input:** Empty volume results in no output.
*   **Precision:** Floating point errors in projection may cause z-fighting if using z-buffer with identical depths.
*   **Banding:** Discrete normals cause visible aliasing in shading. Mitigated by filtering [S1, unofficial].

## Subtopics

### Slab Extrusion (Staircase Representation)
A variant where 2D slices are thresholded and extruded along the axial direction by the slice-to-slice distance, then stacked [S1, unofficial]. It produces a staircase-like representation and is viewed using simple geometric routines. It is less accurate than full 3D cuberille but simpler to implement for slice-based data [S1, unofficial].

## Operating characteristics
**Time Complexity:** $O(N)$ where $N$ is the number of voxels, as each voxel is checked for 6 neighbors. With an octree, traversal is faster for sparse data, but construction overhead exists [S2, unofficial].
**Memory Complexity:** $O(N)$ for the binary array. Octree storage depends on sparsity; it can be more compact for sparse data but has overhead for dense data [S2, unofficial].
**Performance:** Fast for small volumes. Rendering speed depends on the number of exposed faces. Z-buffer approach is robust but requires depth testing. Painter's algorithm avoids depth testing but requires sorting [S2, unofficial].
**Strengths:** Simple implementation; preserves voxel structure; fast for sparse data with octrees [S2, unofficial].
**Limitations:** High polygon count (no face merging); aliasing/banding artifacts from discrete normals; binary decision loses sub-voxel information [S1, unofficial].
**When to use:** When voxel-level accuracy is required, or for binary segmentation visualization. Not suitable for smooth surface visualization [S1, unofficial].

## Background
The Cuberille Model was first defined by Herman et al. [S1, unofficial]. The method was proposed by Herman and Liu in 1979 [S2, unofficial]. Meagher (1984) proposed storing the binary array as an octree to improve traversal speed for sparse data [S2, unofficial]. Lorenson and Cline (1987) introduced Marching Cubes, which is often compared to Cuberille as a smoother alternative [S2, unofficial]. The method is used in medical imaging for stereotactic neurosurgery planning and volume visualization [S1, unofficial].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S2] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
