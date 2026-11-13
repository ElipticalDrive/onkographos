# Frustum-Occlusion Culling for Octree Traversal

Also called: Frustum-octant traversal, Voxel LOD selection, Voxel LOD selection via Frustum Culling, Voxel Rasterization via Frustum Culling and Instancing

## Overview
This method identifies a set of potentially visible voxels by traversing an octree hierarchy against the camera's view frustum, automatically selecting the Level of Detail (LOD) based on screen projection size. It differs from sibling methods by combining visibility culling with implicit LOD termination during traversal, outputting a buffer of voxel attributes for subsequent hardware instancing rasterization.

## How it works
**Inputs and Outputs:**
- **Input:** Camera view frustum parameters, an octree data structure (typically breadth-first stored with an 8-bit occupancy mask per node), and the Axis-Aligned Bounding Box (AABB) enclosing the entire octree.
- **Output:** A buffer containing the position and size of each leaf voxel to be rendered.

**Data Structures:**
- **Octree Node:** Contains an 8-bit mask indicating the presence of children. In breadth-first storage, this is often a single byte per node [S1, unofficial].
- **Voxel Buffer:** A list of structures, each storing the 3D position (center) and size (edge length) of a voxel.

**Canonical Voxel Template:**
The geometric primitive for a voxel is a cube defined by extreme vertices $c_{min} = (-1, -1, -1)$ and $c_{max} = (+1, +1, +1)$ in local space [S1, unofficial].

**Algorithm Steps:**
1. **Initialize Traversal:** Start at the root octant of the octree.
2. **Frustum-Sphere Intersection:** For the current octant, compute a bounding sphere. The diameter of this sphere is the diagonal of the octant's AABB. Check if this sphere intersects the view frustum.
   - If the sphere does not intersect the frustum, prune this branch (do not traverse children).
   - If it intersects, proceed to step 3.
3. **LOD Check:** Calculate the projected area of the current octant on the screen.
   - If the projected area is less than the area of a single pixel, halt traversal for this branch.
   - If the octant has no more subdivisions (is a leaf in the current context) or the LOD condition is met, append the voxel's position and size to the buffer. The position and size are derived from the octant's AABB relative to the global octree AABB.
   - If the octant has children and the projected area is sufficient, traverse to the next level (step 2 for each child).
4. **Repeat:** Continue until all reachable branches are processed.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frustum_occlusion_cull_octree(root_node, frustum, global_aabb, voxel_buffer):
    """
    Traverses the octree to collect visible voxels.
    
    Args:
        root_node: The root node of the octree.
        frustum: Object containing frustum planes or intersection test function.
        global_aabb: The AABB enclosing the entire octree, used to compute voxel positions/sizes.
        voxel_buffer: List to append (position, size) tuples.
    """
    # Helper to check if a sphere intersects the frustum
    def sphere_intersects_frustum(center, radius):
        # Implementation depends on frustum representation (e.g., 6 planes)
        # Returns True if any part of the sphere is inside the frustum
        pass

    # Helper to compute projected area of an octant
    def get_projected_area(center, size, camera_params):
        # Project the 8 corners of the octant to screen space
        # Compute the bounding box of the projected points
        # Return width * height in pixels
        pass

    def traverse(node, current_aabb, depth):
        # Compute center and size of the current octant
        center = (current_aabb.min + current_aabb.max) / 2.0
        size = current_aabb.max - current_aabb.min
        
        # Compute bounding sphere radius (half diagonal)
        radius = length(current_aabb.max - current_aabb.min) / 2.0
        
        # Step 2: Frustum-Sphere Intersection
        if not sphere_intersects_frustum(center, radius):
            return

        # Step 3: LOD Check
        projected_area = get_projected_area(center, size, camera_params)
        pixel_area = 1.0 # Area of one pixel
        
        is_leaf = not node.has_children() # Check occupancy mask or explicit leaf flag
        
        if projected_area < pixel_area or is_leaf:
            # Halt traversal, append to buffer
            # Position is center, size is edge length
            voxel_buffer.append((center, size))
        else:
            # Traverse children
            if node.has_children():
                child_size = size / 2.0
                for i in range(8):
                    # Compute child AABB based on octant index i
                    child_aabb = compute_child_aabb(current_aabb, i, child_size)
                    # Check if child is occupied (via 8-bit mask)
                    if node.is_occupied(i):
                        traverse(node.children[i], child_aabb, depth + 1)

    # Start traversal from root
    traverse(root_node, global_aabb, 0)
```

**Parameters:**
- **LOD Threshold:** Projected area < 1 pixel. This ensures automatic LOD selection [S1, unofficial].
- **Bounding Sphere Approximation:** Using the octant diagonal as the sphere diameter is used for efficiency in frustum intersection tests [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Numeric Precision:** At extreme subdivision levels, numeric precision can become a problem [S1, unofficial].
- **Deep Octrees:** Too many nodes may be allocated for a low number of actual leaf octants. However, rendering rarely benefits from extreme subdivisions because LOD traversal halts at coarser levels based on screen-projection size [S1, unofficial].
- **Occupancy Mask:** The method relies on an 8-bit mask per node to determine occupancy and children presence [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- **Time:** Depends on the number of visible voxels and the depth of the octree. Traversal is pruned by frustum and LOD criteria.
- **Memory:** Octree storage is compact (1 byte per node in breadth-first order) [S1, unofficial]. The voxel buffer size is proportional to the number of visible leaf voxels.

**Measured Performance:**
- **Hardware:** Intel Core i5-2540M, NVIDIA Quadro 3000M, Windows 7, OpenGL/GLSL 4.2 [S1, unofficial].
- **Resolution:** 1920x1280 pixels, with 25% to 85% of pixels covered by shaded fragments [S1, unofficial].
- **Voxel Rendering Times (ms) for 'basketball' dataset at octree depth 10:**
  - Geometry Shader: 3.454 ms
  - Uniform Buffer: 2.549 ms
  - Hardware Texture Buffer: 2.773 ms
  - Instancing Vertex Buffer: 5.599 ms [S1, unofficial]

**Strengths and Limitations:**
- **Strengths:** Automatic LOD selection; compact octree storage; efficient frustum culling via sphere approximation.
- **Limitations:** Geometry Shader approach is slowest for voxels (unlike surfels) due to the complexity of the cube geometry [S1, unofficial]. Uniform buffers are the fastest buffering strategy despite increased draw call footprint [S1, unofficial].

**When to Use:**
- Suitable for voxel rasterization where LOD is important and hardware instancing is available.
- Not suitable if extreme subdivision levels are required without LOD culling, due to precision and memory issues [S1, unofficial].

## Background
**Origin:**
The method is described in the paper "Hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4d media" [S1, unofficial]. It was developed for rendering scanned 4D media, combining octree traversal with hardware instancing techniques.

**History and Development:**
The approach was tested on datasets like 'juggle' and 'soccer' [S1, unofficial]. It was also tested on older hardware (NVIDIA GeForce 8600M GT) but results were not detailed [S1, unofficial]. The paper highlights that multi-threaded execution is possible but not considered in the reported performance [S1, unofficial].

## Sources
[S1] Hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4d media, http://133.5.19.150/papers/2010_2013/1-1-36.pdf, fetched 2026-10-31 (unofficial)
