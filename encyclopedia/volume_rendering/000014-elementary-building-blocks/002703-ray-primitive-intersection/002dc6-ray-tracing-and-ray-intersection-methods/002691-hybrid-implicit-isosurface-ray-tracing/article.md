# Hybrid Implicit Isosurface Ray Tracing

Also called: Hybrid Isosurface Method

## Overview
Hybrid Implicit Isosurface Ray Tracing is a rendering technique that combines explicit voxel selection with implicit ray-primitive intersection to render high-quality, crack-free isosurfaces from rectilinear volumes [S1, unofficial]. It solves the problem of surface detail loss and artifacts common in sample-based ray casting, particularly in regions with high spatial refinement where fixed step sizes are too large to capture fine features [S1, unofficial]. Unlike pure implicit methods that traverse dense grids or explicit methods that generate polygonal meshes, this method builds a Bounding Volume Hierarchy (BVH) over a sparse list of "active voxels" and performs implicit ray-voxel intersections within those primitives [S1, unofficial].

## How it works
The method integrates into a ray tracing framework (specifically demonstrated in OSPRay) by treating the volume as a source of logical voxels. It avoids polygon extraction entirely, relying on implicit mathematical intersection within voxel bounds.

### Inputs and Outputs
*   **Input:** A rectilinear volume (regular grid, rectilinear grid, or BS-AMR) provided as a `VoxelSource`. The source must support querying voxel data and bounding boxes.
*   **Output:** A ray-traced image containing a high-quality, crack-free isosurface.

### Data Structures

**1. Voxel Abstraction**
Each voxel is treated as a logical cube with trilinearly interpolated scalar values at its vertices.
*   **Regular Grid Voxel:** Described by 12 values:
    *   3 values: 3D coordinates of the voxel origin.
    *   1 value: Width (assumed uniform or stored).
    *   8 values: Scalar values at the 8 vertices.
*   **Rectilinear Volume Voxel:** Requires 2 additional values to specify height and depth, totaling 14 values.
*   **Active Voxel Encoding:** Active voxels are encoded into a single 64-bit reference value. For example, in a BS-AMR context, this may consist of 21 bits for x, 21 bits for y, and 21 bits for z coordinates [S1, unofficial].

**2. Embree User Geometry**
The active voxels are exposed to the ray tracer as an Embree "user geometry."
*   **Primitives:** One primitive per active voxel.
*   **Bounds Callback:** Queries the `VoxelSource` for the bounding box of the voxel identified by the 64-bit reference.
*   **Intersection Callback:** An ISPC implementation of the Marmitt et al. ray-iso voxel intersection algorithm [S1, unofficial].

### Steps

1.  **Identify Active Voxels:**
    The `VoxelSource` generates a list of all active voxels. A voxel is active if its value range (min/max of its 8 vertices) overlaps with the user-specified isovalue(s) [S1, unofficial].
2.  **Encode References:**
    Store a 64-bit reference for each active voxel in a contiguous list [S1, unofficial].
3.  **Create User Geometry:**
    Initialize an Embree user geometry with the number of primitives equal to the number of active voxels [S1, unofficial].
4.  **Build BVH:**
    *   Register a `getBounds` callback for the user geometry.
    *   When Embree queries bounds for a primitive ID, decode the 64-bit reference, query the `VoxelSource` for the voxel's geometric bounding box, and return it.
    *   Embree constructs the BVH over these bounding boxes [S1, unofficial].
5.  **Ray Traversal and Intersection:**
    *   Traverse rays through the BVH.
    *   Upon hitting a primitive, invoke the intersection routine.
    *   Decode the 64-bit reference from the primitive ID.
    *   Query the full voxel data (vertex values and geometry) from the `VoxelSource` using the reference [S1, unofficial].
    *   Perform implicit ray-iso intersection using the Marmitt et al. algorithm [S1, unofficial].
    *   The intersection routine is implemented in ISPC to handle ray packets (widths of 4, 8, or 16) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the Hybrid Implicit Isosurface Ray Tracing workflow
# Assumes an Embree-like ray tracing framework and a VoxelSource interface

class HybridIsosurfaceGeometry:
    def __init__(self, voxel_source, isovalue):
        self.voxel_source = voxel_source
        self.isovalue = isovalue
        self.active_voxels = []
        self.voxel_references = [] # List of 64-bit integers

    def build(self):
        # Step 1: Extract active voxels
        # The VoxelSource determines which voxels contain the isovalue
        active_list = self.voxel_source.get_active_voxels(self.isovalue)
        
        # Step 2: Encode references
        for voxel in active_list:
            ref = self.encode_voxel_reference(voxel)
            self.voxel_references.append(ref)
            
        # Step 3: Create Embree User Geometry
        # 'num_primitives' equals the number of active voxels
        self.embree_geometry = create_user_geometry(len(self.voxel_references))
        
        # Register callbacks
        self.embree_geometry.set_bounds_callback(self.get_bounds)
        self.embree_geometry.set_intersection_callback(self.intersect)
        
        # Step 4: Build BVH (handled by Embree API)
        self.embree_geometry.build_bvh()

    def encode_voxel_reference(self, voxel):
        # Encode voxel coordinates into a 64-bit integer
        # Example: 21 bits for x, 21 bits for y, 21 bits for z
        # [G] Implementation details depend on specific coordinate bit-packing scheme
        x = voxel.x
        y = voxel.y
        z = voxel.z
        return (x << 42) | (y << 21) | z

    def decode_voxel_reference(self, ref):
        z = ref & 0x1FFFFF
        y = (ref >> 21) & 0x1FFFFF
        x = (ref >> 42) & 0x1FFFFF
        return {'x': x, 'y': y, 'z': z}

    def get_bounds(self, primitive_id):
        # Step 4 (Callback): Query bounding box from VoxelSource
        ref = self.voxel_references[primitive_id]
        voxel_coords = self.decode_voxel_reference(ref)
        
        # Query the geometric bounding box of this voxel from the source
        bbox = self.voxel_source.get_voxel_bbox(voxel_coords)
        return bbox

    def intersect(self, ray, primitive_id):
        # Step 5 (Callback): Implicit Ray-Voxel Intersection
        ref = self.voxel_references[primitive_id]
        voxel_coords = self.decode_voxel_reference(ref)
        
        # Query full voxel data (vertex values) from VoxelSource
        voxel_data = self.voxel_source.get_voxel_data(voxel_coords)
        
        # Perform implicit intersection using Marmitt et al. algorithm
        # ray: origin o, direction d
        # voxel_data: 8 vertex scalar values, 8 vertex positions
        hit = marmitt_ray_iso_intersect(ray, voxel_data)
        
        return hit

def marmitt_ray_iso_intersect(ray, voxel_data):
    """
    Implicit ray-iso voxel intersection.
    Uses trilinear interpolation within the voxel to find the isosurface.
    Returns hit distance or null if no intersection.
    [G] Specific implementation of Marmitt et al. [27] is complex;
    this represents the logical call.
    """
    # 1. Intersect ray with voxel bounding box to get entry/exit t
    t_min, t_max = ray_box_intersect(ray, voxel_data.bbox)
    if t_max < 0:
        return None
        
    # 2. Sample the scalar field along the ray segment [t_min, t_max]
    # 3. Find zero-crossing of (scalar(t) - isovalue)
    # 4. Return closest valid hit
    # [G] Detailed numerical method omitted as per source constraints
    return find_isosurface_hit(ray, t_min, t_max, voxel_data)
```

### Parameters and Edge Cases
*   **Ray Packet Size:** The intersection routine processes rays in packets of 4, 8, or 16. While the source notes that active rays per voxel are often small (wasteful for large packets), this is dictated by the ISPC/Embree callback mechanism [S1, unofficial].
*   **Precision:** The method relies on the `VoxelSource` providing accurate bounding boxes and vertex values. Degenerate voxels (zero width/height/depth) must be handled by the source or filtered out during active voxel selection.
*   **Crack-Free Guarantee:** The method yields crack-free surfaces at boundaries, unlike sample-based methods which may miss the surface in refined regions due to fixed step sizes [S1, unofficial].

## Subtopics

### [Active Cell Filtering](9505-active-cell-filtering.md)
A pre-processing optimization that prunes input cells whose value ranges do not overlap with the isovalue, reducing the number of primitives before BVH construction.

### [Dual Cell Optimization for Isosurface Extraction](9508-dual-cell-optimization-for-isosurface-extraction.md)
An optimization that merges eight octants into a single dual cell in non-boundary regions to reduce primitive count, achieving ~4× speedup in extraction time.

## Operating characteristics
*   **Performance:** The method yields better framerates than OSPRay's built-in sample-based isosurface method while providing higher quality [S1, unofficial].
*   **Quality:** Produces high-quality, crack-free isosurfaces with more detail in refined regions. Sample-based methods frequently miss the surface and lose key features in these regions due to fixed step sizes being too large [S1, unofficial].
*   **Memory:** Memory overhead is reduced by filtering active voxels and optimizing BVH construction (e.g., via active cell filtering or dual cell optimization) [S1, unofficial].
*   **Hardware:** Evaluated on multi-socket Xeon workstations (e.g., 72 cores, 24 cores) [S1, unofficial].
*   **Limitations:** The reliance on ray packets (4/8/16-wide) in the intersection callback may be inefficient if the number of active rays per voxel is consistently low, as the overhead of packet processing outweighs the benefit [S1, unofficial].

## Background
The method was introduced by Feng et al. in 2018 as part of the OSPRay framework [S1, unofficial]. It extends previous work on AMR volume rendering (Wald et al.) by integrating a hybrid geometry type that supports interactive visualization of gigascale BS-AMR data [S1, unofficial]. The core intersection algorithm is based on Marmitt et al. [27].

## Sources
[S1] Feng et al., "Isosurface Ray Tracing of Adaptive Mesh Refinement Data," https://www.sci.utah.edu/~feng/assets/papers/Fen-AMR-VIS18.pdf, fetched 2026-10-26 (unofficial)
