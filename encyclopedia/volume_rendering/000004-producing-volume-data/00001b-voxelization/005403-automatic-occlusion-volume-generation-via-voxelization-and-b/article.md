# Automatic Occlusion Volume Generation via Voxelization and Box Expansion

Also called: HZB-OC Volume Generation, Voxel-based Occluder Generation

## Overview
This method automatically generates conservative, simplified occlusion volumes for Hierarchical Z-Buffer Occlusion Culling (HZB-OC) by converting a mesh into a voxel grid, identifying dense internal regions, and expanding axis-aligned boxes to cover them [S1, unofficial]. It differs from sibling methods like [VoxelGrid from Mesh (Fill Method)](1013-voxelgrid-from-mesh-fill-method.md) by not merely filling the volume, but by approximating the solid geometry with a minimal set of merged boxes to reduce overdraw and draw calls [S1, unofficial].

## How it works
**Inputs:** A watertight triangular mesh $M$, a target volume consumption percentage $P_{target}$ (typically 90%), and a voxel grid resolution defined by the mesh's bounding box [S1, unofficial].
**Outputs:** A single merged mesh representing the occlusion volume, generated via Constructive Solid Geometry (CSG) [S1, unofficial].

**Data Structures:**
1.  **Solid Voxel Set ($V_{solid}$):** A boolean grid or list of voxel coordinates $(x,y,z)$ representing voxels completely inside the mesh.
2.  **Surface Voxel Set ($V_{surf}$):** A boolean grid or list of voxel coordinates representing voxels intersected by the mesh surface.
3.  **Empty Voxel List ($L_{empty}$):** A list of voxel coordinates outside the mesh, used for distance optimization.
4.  **Box List ($B_{list}$):** A collection of Axis-Aligned Bounding Boxes (AABBs), each defined by minimum and maximum voxel coordinates $(min_x, min_y, min_z)$ and $(max_x, max_y, max_z)$.

**Steps:**

1.  **Voxelization:**
    Determine $V_{solid}$ and $V_{surf}$ using the method from [Schwarz 2010] [S1, unofficial].
    *   For each column of voxels aligned with the view ray (or axis, depending on implementation), cast a ray against $M$.
    *   Apply XOR logic: toggle a bit for each intersection. Voxels with an odd number of intersections are marked as potentially inside [S1, unofficial].
    *   A voxel is in $V_{solid}$ if its center is inside the mesh.
    *   Refine $V_{solid}$ by removing any voxel in the initial solid set that is also in $V_{surf}$ (i.e., the voxel is not *completely* inside) [S1, unofficial].
    *   Define $L_{empty}$ as all voxels not in $V_{solid}$.

2.  **Find Highest Density Voxel:**
    Identify the voxel $v_{dense}$ in $V_{solid}$ that is furthest from any voxel in $L_{empty}$, excluding any voxels already enclosed by boxes in $B_{list}$ [S1, unofficial].
    *   Optimization: Instead of calculating distance to all empty voxels, test against a pre-filtered list of empty voxels adjacent to the solid volume [S1, unofficial].

3.  **Box Expansion:**
    Initialize a box $B$ at $v_{dense}$ with a base size of 13 voxels (or the smallest possible box containing $v_{dense}$ if smaller) [S1, unofficial].
    *   Iteratively expand each of the 6 faces of $B$ in voxel space.
    *   Expansion stops for a specific face when the next voxel in that direction is in $L_{empty}$ or is already enclosed by an existing box in $B_{list}$ [S1, unofficial].
    *   Mark all newly enclosed voxels as "covered" to exclude them from future density calculations [S1, unofficial].

4.  **Repeat:**
    Repeat Steps 2 and 3 until a stopping condition is met:
    *   The total volume of boxes in $B_{list}$ exceeds $P_{target}$ of the total volume of $V_{solid}$ [S1, unofficial].
    *   The last box created consumed a negligible percentage of the total volume [S1, unofficial].
    *   (Optional) A maximum number of boxes is reached [S1, unofficial].

5.  **Filter Boxes (Optional):**
    Remove small or redundant boxes from $B_{list}$ to reduce the final triangle count [S1, unofficial].

6.  **Merge with CSG:**
    Combine all boxes in $B_{list}$ into a single mesh using Constructive Solid Geometry (CSG) union operations to eliminate internal faces and reduce overdraw [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_occlusion_volume(mesh, target_percentage=0.9):
    # Step 1: Voxelization
    # Requires Schwarz 2010 solid voxelization
    solid_voxels = set()
    surface_voxels = set()
    
    # Implementation of Schwarz 2010 logic:
    # For each voxel column, raycast mesh. XOR intersections.
    # If center is inside (odd intersections) AND not intersecting surface, add to solid.
    solid_voxels = voxelize_solid(mesh)
    surface_voxels = voxelize_surface(mesh)
    
    # Remove voxels that are not completely inside
    solid_voxels = solid_voxels - surface_voxels
    
    # Define empty voxels as complement within bounding box
    bounding_box = get_bounding_box(mesh)
    all_voxels = get_voxel_grid(bounding_box)
    empty_voxels = all_voxels - solid_voxels
    
    boxes = []
    covered_voxels = set()
    total_solid_volume = len(solid_voxels)
    covered_volume = 0
    
    while covered_volume < total_solid_volume * target_percentage:
        # Step 2: Find densest voxel
        # Exclude voxels already covered by existing boxes
        available_voxels = solid_voxels - covered_voxels
        
        if not available_voxels:
            break
            
        densest_voxel = find_furthest_from_empty(available_voxels, empty_voxels)
        
        # Step 3: Box Expansion
        # Start with a base box, e.g., 13x13x13 or minimal containing the voxel
        box = create_initial_box(densest_voxel, size=13)
        
        # Expand in all 6 directions until hitting empty or covered voxels
        expand_box(box, empty_voxels, covered_voxels)
        
        # Mark new voxels as covered
        new_voxels = get_voxels_in_box(box)
        covered_voxels.update(new_voxels)
        covered_volume += len(new_voxels)
        
        boxes.append(box)
        
        # Check stopping condition for small last box
        if len(new_voxels) < threshold:
            break
            
    # Step 6: Merge with CSG
    final_mesh = csg_union(boxes)
    return final_mesh

def find_furthest_from_empty(voxels, empty_voxels):
    # Optimization: test against list of empty voxels rather than all
    # Return voxel with max min-distance to any empty voxel
    max_dist = -1
    best_voxel = None
    for v in voxels:
        dist = min_distance_to_set(v, empty_voxels)
        if dist > max_dist:
            max_dist = dist
            best_voxel = v
    return best_voxel

def expand_box(box, empty_voxels, covered_voxels):
    directions = [
        (1, 0, 0), (-1, 0, 0),
        (0, 1, 0), (0, -1, 0),
        (0, 0, 1), (0, 0, -1)
    ]
    for dx, dy, dz in directions:
        while True:
            next_voxel = box.get_next_voxel_in_direction(dx, dy, dz)
            if next_voxel in empty_voxels or next_voxel in covered_voxels:
                break
            box.expand(dx, dy, dz)
```

## Operating characteristics
*   **Time Complexity:** Dominated by voxelization and the iterative box expansion. CPU implementation took ~20 seconds for ~10,000 triangles and 503 voxels; GPU (OpenCL) took ~6 seconds on an Nvidia 540M [S1, unofficial].
*   **Memory Complexity:** Proportional to the voxel grid resolution and the number of boxes generated.
*   **Strengths:** Generates conservative occluders (no overestimation of volume); simple geometry (boxes) reduces rendering cost; automated pipeline [S1, unofficial].
*   **Limitations:** Requires watertight meshes (holes break the XOR logic); Axis-Aligned Bounding Boxes (AABBs) are inefficient for models at non-right angles, leading to overestimation of volume compared to Oriented Bounding Boxes (OBBs) [S1, unofficial].
*   **When to use:** When automatic generation of HZB-OC occluders is needed and meshes are watertight.
*   **When not to use:** For non-watertight meshes without pre-processing; when tight-fitting occluders are critical for non-axis-aligned objects [S1, unofficial].

## Background
The method was described by Nick Darnell in a blog post detailing the generation of occlusion volumes for Hierarchical Z-Buffer Occlusion Culling (HZB-OC) [S1, unofficial]. It builds on the voxelization techniques from [Schwarz 2010] and uses CSG algorithms similar to those described by Sander van Rossen and Matthew Baranowski in *Game Development Tools* [S1, unofficial]. The author noted that while AABBs are simple, OBBs might offer better fitting for angled models [S1, unofficial].

## Sources
[S1] Hierarchical Z-Buffer Occlusion Culling - Generating Occlusion Volumes - Nick Darnell’s Blog, https://www.nickdarnell.com/hierarchical-z-buffer-occlusion-culling-generating-occlusion-volumes/, fetched 2026-10-29 (unofficial)
