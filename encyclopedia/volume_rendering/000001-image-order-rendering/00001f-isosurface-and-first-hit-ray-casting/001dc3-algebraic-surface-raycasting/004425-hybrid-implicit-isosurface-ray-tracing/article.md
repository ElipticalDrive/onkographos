# Hybrid Implicit Isosurface Ray Tracing

Also called: Hybrid Isosurface Method, Implicit isosurface ray tracing, Wang et al. hybrid method

## Overview
Hybrid Implicit Isosurface Ray Tracing is a rendering technique that extracts a list of active vertex-centered cubic primitives (octants or dual cells) from cell-centered TB-AMR data, builds a BVH over them, and performs implicit ray-isosurface intersections within each primitive, avoiding the memory cost of explicit triangle extraction [S2]. It combines explicit extraction (selecting active voxels) with implicit tracing (intersecting rays against the trilinear field within those voxels) to produce high-quality, crack-free isosurfaces [S1, unofficial]. Unlike sample-based isosurface methods that use fixed step sizes and may miss features in refined regions, this method adapts to the local resolution, providing more detail and better framerates for AMR data [S1, unofficial].

## How it works
The method operates on rectilinear volumes (regular grids, rectilinear grids, BS-AMR) by treating them as a source of logical voxels. Each voxel is a cube with trilinearly interpolated scalar values at its vertices [S1, unofficial].

### Inputs and Outputs
*   **Input:** A rectilinear volume (e.g., TB-AMR data) and a user-specified isovalue.
*   **Output:** A rendered image of the isosurface.

### Data Structures
**Voxel Abstraction:**
Each logical voxel is described by 12 values:
1.  3 values for 3D coordinates (position).
2.  1 value for width.
3.  8 values for scalar values at the vertices.
*Note:* For general rectilinear volumes, 2 additional values are required to specify height and depth [S1, unofficial].

**Active Voxel Encoding:**
Active voxels are encoded into a single 64-bit value (e.g., 21:21:21 bit coordinates) to allow efficient storage and lookup [S1, unofficial].

**Geometry Type:**
The method uses "user geometry" primitives compatible with Embree. The primitives are either:
*   **Octants:** Vertex-centered cubes where the center is the input cell center [S2].
*   **Dual Cells:** Vertex-centered cubes where the center is a vertex of the original grid [S2].

### Steps
1.  **Voxel Source Initialization:** Create a `VoxelSource` entity that can quickly generate a list of active voxels from the input volume [S1, unofficial].
2.  **Primitive Extraction:**
    *   Iterate through all input cells $C_i$.
    *   For each cell, generate up to 8 octants $O_{ij}$ or dual cells $D_{ij}$ [S2].
    *   Compute the scalar values at the vertices of each primitive using the GTI (Grid Trilinear Interpolation) method to ensure surface continuity [S2].
    *   *Optimization:* If a dual cell can be constructed (vertices lie at the same AMR level), emit the dual cell instead of the octant to avoid overlap and reduce primitive count [S2].
3.  **Active Cell Filtering:**
    *   A primitive is active if its value range (min/max of its 8 vertex values) overlaps with the user-specified isovalue [S1, unofficial][S2].
    *   Discard inactive primitives [S1, unofficial].
4.  **BVH Construction:**
    *   Create an Embree "user geometry" with one primitive per active voxel [S1, unofficial].
    *   Implement a `getBounds` callback: Query the `VoxelSource` for the bounding box of the respective voxel using its 64-bit reference [S1, unofficial].
    *   Build an Embree BVH over these primitives [S1, unofficial][S2].
5.  **Ray Traversal and Intersection:**
    *   Traverse rays through the BVH [S1, unofficial][S2].
    *   For each hit primitive, perform an implicit ray-isosurface intersection [S1, unofficial][S2].
    *   The intersection routine queries the full voxel data from the `VoxelSource` using the 64-bit reference [S1, unofficial].
    *   The intersection algorithm used is an ISPC version of Marmitt et al.'s ray-iso voxel intersection [S1, unofficial].
    *   Rays are processed in packets of 4, 8, or 16 [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
class VoxelSource:
    def get_bounds(self, voxel_ref_64bit):
        # Returns bounding box of the voxel
        pass

    def get_data(self, voxel_ref_64bit):
        # Returns 12 values: [x, y, z, width, v0...v7]
        # For rectilinear: [x, y, z, width, height, depth, v0...v7]
        pass

class HybridIsosurfaceRenderer:
    def __init__(self, volume_data, isovalue):
        self.volume_data = volume_data
        self.isovalue = isovalue
        self.active_primitives = []
        self.bvh = None

    def extract_primitives(self):
        """
        Extract active vertex-centered primitives (octants or dual cells)
        from cell-centered TB-AMR data.
        """
        primitives = []
        
        # Iterate through all input cells
        for cell in self.volume_data.cells:
            # Generate up to 8 octants for the cell
            for j in range(8):
                octant = Octant(cell, j)
                
                # Compute vertex values using GTI to ensure continuity
                octant.vertex_values = compute_gti_values(octant, self.volume_data)
                
                # Check if dual cell can be used instead (optimization)
                # Dual cell is used if all its vertices are at the same AMR level
                if can_form_dual_cell(octant, self.volume_data):
                    dual_cell = DualCell(octant)
                    dual_cell.vertex_values = octant.vertex_values
                    primitives.append(dual_cell)
                else:
                    primitives.append(octant)
        
        # Filter active primitives
        active = []
        for prim in primitives:
            min_val = min(prim.vertex_values)
            max_val = max(prim.vertex_values)
            if min_val <= self.isovalue <= max_val:
                active.append(prim)
        
        self.active_primitives = active
        return active

    def build_bvh(self):
        """
        Build an Embree BVH over the active primitives.
        """
        # Create user geometry
        user_geom = EmbreeUserGeometry(num_primitives=len(self.active_primitives))
        
        # Define getBounds callback
        def get_bounds_callback(primitive_index):
            prim = self.active_primitives[primitive_index]
            # Query voxel data to get bounding box
            # Note: In implementation, voxel_ref_64bit is stored with the primitive
            voxel_data = self.volume_data.get_voxel_data(prim.ref)
            return voxel_data.bounding_box
        
        user_geom.set_bounds_callback(get_bounds_callback)
        
        # Build BVH
        self.bvh = EmbreeBVH(user_geom)
        return self.bvh

    def render(self, rays):
        """
        Traverse rays and perform implicit intersection.
        """
        # Traverse rays through BVH
        hits = self.bvh.intersect(rays)
        
        results = []
        for ray, hit in hits:
            prim_index = hit.primitive_index
            prim = self.active_primitives[prim_index]
            
            # Query full voxel data
            voxel_data = self.volume_data.get_voxel_data(prim.ref)
            
            # Perform implicit ray-isosurface intersection
            # Using Marmitt et al. algorithm
            t_hit = marmitt_ray_iso_intersection(ray, voxel_data)
            
            if t_hit is not None:
                results.append((ray, t_hit, prim))
        
        return results

# Helper functions (conceptual)
def compute_gti_values(octant, volume_data):
    # Interpolate scalar values at the 8 vertices of the octant
    # from the underlying cell-centered AMR data
    pass

def can_form_dual_cell(octant, volume_data):
    # Check if all 8 vertices of the dual cell (centered at grid vertex)
    # lie at the same AMR level
    pass

def marmitt_ray_iso_intersection(ray, voxel_data):
    # Ray-iso voxel intersection for trilinearly interpolated field
    # Returns t parameter along ray where isosurface is hit
    pass
```

### Parameters and Edge Cases
*   **Isosvalue:** User-specified scalar value defining the surface [S1, unofficial][S2].
*   **Ray Packet Size:** 4, 8, or 16 rays [S1, unofficial].
*   **Precision:** The method relies on trilinear interpolation. Degenerate cases where the isovalue equals the min/max vertex value may require careful handling to avoid missed hits or double hits, though the source notes that the method yields a "crack-free" surface [S1, unofficial].
*   **Boundary Conditions:** The method handles AMR boundaries by using dual cells in non-boundary regions and octants elsewhere, ensuring consistency [S2].

## Subtopics

### [Active Cell Filtering](9505-active-cell-filtering.md)
A pre-processing optimization that reduces the number of geometric primitives generated for rendering by discarding input cells whose value ranges do not overlap with the user-specified isovalue.

### [Dual Cell Optimization for Isosurface Extraction](9508-dual-cell-optimization-for-isosurface-extraction.md)
An optimization that replaces octants with dual cells in non-boundary (same-level) regions to reduce the number of primitives traversed, achieving a ~4x speedup in extraction time.

## Operating characteristics
*   **Time Complexity:** The main performance bottleneck lies in the first process where resampling is required to extract primitives [S2]. BVH construction is efficient (~110 million primitives per second with Embree) [S2].
*   **Memory Complexity:** Avoids the memory cost of explicit triangle extraction (e.g., Marching Cubes) by not generating a large number of triangles [S2].
*   **Performance:** Yields better framerates than sample-based isosurface methods for AMR data [S1, unofficial].
*   **Quality:** Produces high-quality, crack-free isosurfaces with more detail in refined regions compared to sample-based methods which may miss the surface due to fixed step sizes [S1, unofficial].
*   **Hardware:** Evaluated on multi-core CPUs (Xeon E7-8890 v3, Xeon Skylake Gold 6136) [S1, unofficial].
*   **When to use:** When rendering high-quality isosurfaces from TB-AMR data is required, and memory efficiency is important.
*   **When not to use:** If explicit mesh generation is required for further processing (e.g., physics simulation), as this method only renders the surface implicitly.

## Background
The method was introduced by Wang et al. in 2019 [WWW*19] and integrated into the OSPRay ray tracing framework [S1, unofficial]. It builds on the principles of implicit isosurface ray tracing [MKW*04] and explicit extraction [S2]. The implementation in OSPRay extends the existing AMR volume structure [S1, unofficial].

## Sources
[S1] CPU Isosurface Ray Tracing of Adaptive Mesh Refinement Da, https://www.sci.utah.edu/~feng/assets/papers/Fen-AMR-VIS18.pdf, fetched 2026-10-26 (unofficial)
[S2] CPU Ray Tracing of Tree‐Based Adaptive Mesh Refinement Data, https://publications.sci.utah.edu/publications/Wan2020a/cgf.13958.pdf, fetched 2026-10-27
