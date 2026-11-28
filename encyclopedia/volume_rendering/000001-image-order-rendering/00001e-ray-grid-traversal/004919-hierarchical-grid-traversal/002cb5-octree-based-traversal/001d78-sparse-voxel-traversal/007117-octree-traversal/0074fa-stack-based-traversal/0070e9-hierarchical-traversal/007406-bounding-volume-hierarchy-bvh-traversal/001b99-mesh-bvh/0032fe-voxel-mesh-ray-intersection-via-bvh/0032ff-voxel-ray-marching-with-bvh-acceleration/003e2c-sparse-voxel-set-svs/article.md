# Sparse Voxel Set (SVS)
## Overview
Sparse Voxel Set (SVS) is a ray-tracing acceleration method for Signed Distance Function (SDF) grids that constructs a Bounding Volume Hierarchy (BVH) from Axis-Aligned Bounding Boxes (AABBs) representing individual surface-intersecting voxels [S1, S2, unofficial]. It differs from software-based traversal methods (like Grid Sphere Tracing or Sparse Brick Set) by offloading the primary traversal logic to dedicated GPU ray-tracing hardware (DXR), thereby reducing the computational burden of the custom intersection shader [S2, unofficial].
## How it works
**Inputs:** A voxel grid containing Signed Distance Field (SDF) values; a threshold defining the surface boundary.
**Outputs:** A BVH structure suitable for DirectX Raytracing (DXR) hardware traversal.

**Data Structures:**
1.  **Voxel Data:** Each voxel in the sparse set stores a $4 \times 4 \times 4$ array of signed distance values [S1, unofficial]. This is larger than the standard $2 \times 2 \times 2$ neighborhood, providing easy access to neighboring voxel values required for normal evaluation [S1, unofficial].
2.  **AABB:** An Axis-Aligned Bounding Box is created for each voxel that intersects the surface (i.e., where the SDF crosses the threshold) [S1, unofficial].
3.  **BVH:** A hierarchy of these AABBs is built to accelerate ray intersection tests [S1, unofficial].

**Algorithm Steps:**
1.  **Voxel Selection:** Identify all voxels in the grid that intersect the implicit surface defined by the SDF threshold [S1, unofficial].
2.  **AABB Construction:** For each selected voxel, compute the AABB that encloses the voxel's spatial extent [S1, unofficial].
3.  **BVH Build:** Construct a Bounding Volume Hierarchy from the set of AABBs [S1, unofficial].
4.  **Ray Tracing Setup:** Configure the DXR pipeline to use the constructed BVH [S1, unofficial].
5.  **Traversal:** For each ray, perform BVH traversal using the GPU's RTX ray tracing hardware [S1, unofficial].
6.  **Intersection Shader:** When a leaf node (voxel) is reached, invoke a custom intersection shader [S1, unofficial].
7.  **SDF Evaluation:** Inside the intersection shader, sample the $4 \times 4 \times 4$ SDF data associated with the voxel to determine the precise intersection point and normal [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for SVS setup and traversal logic
# Note: Actual DXR API calls are abstracted; this represents the logical flow.

class SparseVoxelSet:
    def __init__(self, grid_resolution, sdf_values, threshold):
        self.threshold = threshold
        self.voxels = []
        self.bvh = None
        
        # Step 1 & 2: Identify surface voxels and create AABBs
        for x in range(grid_resolution.x):
            for y in range(grid_resolution.y):
                for z in range(grid_resolution.z):
                    # Check if voxel intersects surface
                    # A voxel intersects if the SDF values at its corners span the threshold
                    if self._voxel_intersects_surface(x, y, z, sdf_values):
                        # Store 4x4x4 SDF values for normal evaluation [S1, unofficial]
                        sdf_data = self._get_4x4x4_sdf(x, y, z, sdf_values)
                        aabb = self._compute_aabb(x, y, z)
                        self.voxels.append({
                            'aabb': aabb,
                            'sdf_data': sdf_data
                        })
        
        # Step 3: Build BVH [S1, unofficial]
        self.bvh = BVHBuilder(self.voxels)

    def _voxel_intersects_surface(self, x, y, z, sdf_values):
        # Determine if the voxel contains the surface
        # This typically involves checking the 8 corners of the voxel
        # against the threshold. If signs differ, it intersects.
        # [G] Implementation detail: Check min/max SDF in voxel vs threshold
        return True # Placeholder for specific intersection logic

    def _get_4x4x4_sdf(self, x, y, z, sdf_values):
        # Retrieve 4x4x4 signed distance values around the voxel [S1, unofficial]
        # This allows easy access to neighboring values for normal evaluation
        return sdf_values[x-1:x+4, y-1:y+4, z-1:z+4]

    def _compute_aabb(self, x, y, z):
        # Compute AABB for the voxel
        return AABB(min=(x, y, z), max=(x+1, y+1, z+1))

    def trace_ray(self, ray_origin, ray_direction):
        # Step 5: Perform BVH traversal using RTX hardware [S1, unofficial]
        # The hardware handles the traversal tree walk
        hit = self.bvh.intersect(ray_origin, ray_direction)
        
        if hit:
            # Step 6: Invoke custom intersection shader [S1, unofficial]
            # Step 7: Evaluate SDF for precise hit
            voxel_data = hit.voxel_data
            sdf_val = self._evaluate_sdf(hit.position, voxel_data)
            normal = self._compute_normal(hit.position, voxel_data)
            return sdf_val, normal
        return None

    def _evaluate_sdf(self, position, sdf_data):
        # Sample the 4x4x4 grid at the hit position
        # [G] Trilinear interpolation of the 4x4x4 data
        pass

    def _compute_normal(self, position, sdf_data):
        # Compute normal using neighboring SDF values [S1, unofficial]
        # [G] Central difference or gradient estimation using the 4x4x4 data
        pass
```

**Parameters:**
*   **SDF Neighborhood Size:** $4 \times 4 \times 4$ values per voxel [S1, unofficial].
*   **Threshold:** The SDF value defining the surface (typically 0) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Overhead:** SVS has a "much more significant memory overhead" compared to other methods due to the storage of $4 \times 4 \times 4$ SDF values per voxel and the BVH structure itself [S2, unofficial].
*   **Complexity:** While simpler to construct than Sparse Voxel Octrees, the memory cost is a primary limitation [S2, unofficial].

## Operating characteristics
*   **Performance:** SVS outperformed other methods (grid sphere tracing, sparse brick set, sparse voxel octree) in general scenarios [S2, unofficial].
*   **Hardware Dependency:** Performance relies heavily on the efficiency of the GPU's RTX ray-tracing hardware [S1, S2, unofficial].
*   **Shader Workload:** Requires less work in the custom software intersection shader compared to software-based traversal methods because the BVH traversal is handled by hardware [S2, unofficial].
*   **Memory:** High memory usage due to the $4 \times 4 \times 4$ SDF storage per voxel and BVH overhead [S2, unofficial].
*   **Construction:** Simpler and faster to construct than Sparse Voxel Octrees [S2, unofficial].
*   **Scene Suitability:** Effective in general scenes; however, cache coherency in complex scenes might favor other methods like Sparse Brick Set if hardware acceleration is not fully utilized or if memory bandwidth is a bottleneck [S2, unofficial].

## Background
*   **Origin:** The method is described in the context of real-time rendering of Signed Distance Fields, leveraging DirectX Raytracing (DXR) hardware [S1, S2, unofficial].
*   **Development:** It is presented as an alternative to software-based SDF traversal techniques such as Grid Sphere Tracing (similar to Claybook [Aaltonen 2018]) and Sparse Brick Set (SBS) [S1, S2, unofficial].
*   **Usage:** Used for real-time rendering of SDF-based geometry, particularly where high performance is required and GPU ray-tracing hardware is available [S1, S2, unofficial].

## Sources
[S1] Ray tracing of signed distance function grids, https://www.jcgt.org/published/0011/03/06/paper-lowres.pdf, fetched 2026-10-25 (unofficial)
[S2] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
