# Hierarchical Ray Marching

Also called: March

## Overview
Hierarchical Ray Marching is a sparse voxel traversal technique that dynamically switches between octree node traversal and direct voxel sampling to optimize ray-marching performance [S1, unofficial]. It differs from fixed-leaf or constant-node approaches by adapting the traversal granularity (level) based on whether the ray is currently inside or outside occupied space, allowing it to skip empty regions rapidly at coarse levels and sample details precisely at fine levels [S1, unofficial].

## How it works
**Inputs:**
*   Ray origin ($P_0$) and direction ($D$).
*   Scene data: `octantSize` (size of level-0 octants), `octreeHighestLevel` (maximum octree depth index).
*   Voxel texture: 3D texture where each 32-bit unsigned integer stores 4 voxels (8-bit values each) packed in a specific layout.

**Outputs:**
*   Hit position, normal, accumulated iterations, status (miss, hit octree, hit voxel, out of bounds), and voxel data.

**Data Structures:**
*   **Voxel Packing:** Each texel in the 3D voxel texture is a `u32`. It contains 4 voxels. The packing order for a block at $(x, y, z)$ (where $x, z$ are even) is:
    *   Bit 0-7: Voxel at $(x, y, z)$
    *   Bit 8-15: Voxel at $(x+1, y, z)$
    *   Bit 16-23: Voxel at $(x, y, z+1)$
    *   Bit 24-31: Voxel at $(x+1, y, z+1)$
    *   *Note: The source code comment says "1,1 1,0 0,1 0,0" corresponding to bits 24-31, 16-23, 8-15, 0-7 respectively if reading left-to-right as x,z offsets. Specifically: shiftCount 0 is (0,0), 1 is (1,0), 2 is (0,1), 3 is (1,1).*

**Algorithm Steps:**
1.  Initialize `level = -1` (representing the finest voxel level). Initialize `totalIterations = 0`. Initialize `rayPos = P_0`.
2.  Loop for a maximum number of iterations (implementation-dependent, e.g., 1024 or more).
3.  Determine `unitSize`:
    *   If `level == -1`, `unitSize = 1` (voxel size).
    *   If `level >= 0`, `unitSize = octantSize << level`.
4.  Determine `maxIterations` for the current level:
    *   If `level == octreeHighestLevel`, `maxIterations = 1024`.
    *   Otherwise, `maxIterations = 4`.
5.  Perform traversal step:
    *   If `level == -1`: Call `MarchThroughVoxels(rayPos, D, maxIterations)`. This function samples the voxel texture directly.
    *   If `level >= 0`: Call `MarchThroughOctree(rayPos / unitSize, D, level, maxIterations)`. This function traverses the octree structure.
6.  Scale the result position: `rayHit.position *= unitSize`.
7.  Update state: `totalIterations += rayHit.iterations`. Update `rayPos = rayHit.position`.
8.  Adjust level based on `rayHit.status`:
    *   If `status == 0` (miss) and `level < octreeHighestLevel`: `level++` (move to coarser level). Continue loop.
    *   If `status == 1` (hit octree node) and `level > -1`: `level--` (move to finer level). Continue loop.
    *   If `status == 2` (hit voxel): Return result with position, normal, iterations, status, voxel data, and UV.
    *   If `status == 3` (out of bounds): Return result with position, status, and iterations.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_ray_marching(ray_origin, ray_dir, scene_data, voxel_texture):
    level = -1
    total_iterations = 0
    ray_pos = ray_origin
    octant_size = scene_data.octant_size
    highest_level = scene_data.octree_highest_level
    
    # Max iterations for the whole ray is not explicitly bounded in the loop condition 
    # other than by status returns or implicit engine limits. 
    # We assume a large enough loop or break on status 2/3.
    
    while True:
        # Calculate unit size for current level
        if level == -1:
            unit_size = 1
        else:
            unit_size = octant_size << level
            
        # Determine max iterations for this step
        if level == highest_level:
            max_iter = 1024
        else:
            max_iter = 4
            
        # Perform traversal
        if level == -1:
            hit = march_through_voxels(ray_pos, ray_dir, max_iter, voxel_texture)
        else:
            # Normalize position for octree traversal
            normalized_pos = ray_pos / unit_size
            hit = march_through_octree(normalized_pos, ray_dir, level, max_iter)
            
        # Scale position back to world space
        hit.position *= unit_size
        
        # Update state
        total_iterations += hit.iterations
        ray_pos = hit.position
        
        # Handle status
        if hit.status == 2: # Hit voxel
            return {
                'position': hit.position,
                'normal': hit.normal,
                'iterations': total_iterations,
                'status': 2,
                'voxel_data': hit.voxel_data,
                'uv': hit.uv
            }
        elif hit.status == 3: # Out of bounds
            return {
                'position': hit.position,
                'status': 3,
                'iterations': total_iterations
            }
        elif hit.status == 0: # Miss
            if level < highest_level:
                level += 1 # Go coarser
                continue
            else:
                # If at highest level and miss, likely end of ray or empty space
                # Implementation may return miss or continue? 
                # Source implies loop continues, but if level can't go higher, 
                # it might need to terminate. Assuming it terminates or returns miss.
                return {
                    'position': hit.position,
                    'status': 0,
                    'iterations': total_iterations
                }
        elif hit.status == 1: # Hit octree node
            if level > -1:
                level -= 1 # Go finer
                continue
            else:
                # If at voxel level and hit octree? This state is contradictory 
                # as level -1 uses voxels directly. 
                # This case might not occur or implies a logic error in source description.
                # Assuming we stay or handle as error.
                pass

def march_through_voxels(ray_pos, ray_dir, max_iter, voxel_texture):
    # Implementation details for voxel traversal are not fully provided in source.
    # It samples the 3D texture.
    # Returns hit with status 2 if voxel is non-zero, else 0.
    # [G] Standard ray-box intersection for voxel grid is assumed.
    pass

def march_through_octree(ray_pos, ray_dir, level, max_iter):
    # Implementation details for octree traversal are not fully provided in source.
    # It traverses the octree buffer.
    # Returns hit with status 1 if node is occupied, else 0.
    # [G] Standard octree traversal is assumed.
    pass
```

**Parameters:**
*   `sceneData.octantSize`: The size of octants at level 0. [S1, unofficial]
*   `sceneData.octreeHighestLevel`: The maximum octree level index. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Precision:** Scaling `rayPos` by `unitSize` and dividing by `unitSize` can introduce floating-point errors. [S1, unofficial]
*   **Boundary Conditions:** The source code checks for out-of-bounds using `any(octantPos >= vec3<u32>(dimAtLevel))`. [S1, unofficial]
*   **Voxel Packing:** Incorrect bit-shifting when extracting voxel values from the packed 32-bit integer will lead to incorrect sampling. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** Depends on the distribution of occupied space. Coarse levels allow skipping large empty volumes, while fine levels provide precision. [S1, unofficial]
*   **Memory Complexity:** Requires storage for the octree structure and the packed voxel texture. [S1, unofficial]
*   **Strengths:** Adaptive resolution allows for efficient rendering of sparse volumes. [S1, unofficial]
*   **Limitations:** Dynamic level switching adds overhead. [S1, unofficial]

## Background
Hierarchical Ray Marching is implemented in the WebGPU Voxel Raymarching project by James Kiernan. [S1, unofficial]

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
