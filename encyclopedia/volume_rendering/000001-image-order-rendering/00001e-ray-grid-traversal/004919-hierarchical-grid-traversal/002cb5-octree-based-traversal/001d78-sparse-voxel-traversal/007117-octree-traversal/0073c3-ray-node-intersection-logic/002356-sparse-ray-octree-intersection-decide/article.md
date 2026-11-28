# Sparse Ray-Octree Intersection (DECIDE)


## Overview
Sparse Ray-Octree Intersection (DECIDE) is a GPU-based algorithm for finding the first intersection of a ray with a sparse solid voxel octree, returning the hit position, normal, and node index [S2, unofficial][S3, unofficial]. It solves the problem of efficiently traversing hierarchical voxel structures to determine ray-volume intersections for rendering or collision detection. It sets itself apart from its siblings by using a fixed-size stack for traversal and descent, handling backtracking via explicit stack pops, and utilizing a specific bit-manipulation-based child selection and t-span update mechanism [S2, unofficial][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), octree root node index (typically 0), octree resolution, and a traversal stack.
*   **Output:** Hit position $\mathbf{p}$, hit normal $\mathbf{n}$, hit distance $t_{min}$, and the index of the node/voxel hit. If no intersection is found, an invalid hit is returned.

**Data Structures:**
*   **TraceContext:** A struct used for the traversal stack, aligned to 16 bytes. Fields:
    *   `o`: `glm::vec3` (origin of the node in world/object space).
    *   `r`: `float` (radius or half-extent of the node).
    *   `index`: `uint32_t` (node index in the octree array).
    *   `t`: `float` (intersection distance of the node's bounding box).
*   **Child Descriptor Masks:**
    *   `_node_m`: A constant array of 8 `uint32_t` masks corresponding to octree child indices 0-7. Bitmasks: $m_{000}=0b00000001u$, $m_{001}=0b00000010u$, $m_{011}=0b00000100u$, $m_{010}=0b00001000u$, $m_{110}=0b00010000u$, $m_{111}=0b00100000u$, $m_{101}=0b01000000u$, $m_{100}=0b10000000u$ [S3, unofficial].
    *   `_node_d`: A constant array of 8 `float3` vectors representing the offset direction for each octree child. Indices 0-7 map to binary coordinates $(x,y,z)$. Order: $d_{000}, d_{001}, d_{011}, d_{010}, d_{110}, d_{111}, d_{101}, d_{100}$ [S3, unofficial].
*   **Stack:** A fixed-size array of `TraceContext` structs. Maximum depth is typically 32 [S3, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Initialize `closest.t` to `infinity` and `closest.valid` to `false` [S3, unofficial].
    *   If the root node is empty, return an invalid hit [S3, unofficial].
    *   Push the root node to the stack using `add_node(0, vec3(0), resolution)` [S3, unofficial].
    *   Inside `add_node`, intersect the ray with the node's bounding box using `_intersect_box_akari`. If the hit is valid and $t < closest.t$, proceed [S3, unofficial].

2.  **Traversal Loop:**
    *   Loop while the stack is not empty [S3, unofficial]:
        *   Pop the top context from the stack: `ctx = stack.pop()` [S3, unofficial].
        *   **Early Culling:** If `ctx.t >= closest.t`, continue to the next iteration (prune this branch) [S3, unofficial].
        *   **Node Processing:**
            *   If the node is full (`node.full()`), update `closest = hit` and return [S3, unofficial].
            *   If the node is a leaf (`r == 2.0f`), iterate over its 8 children. For each child $i$ where the child mask bit is set, intersect the child box at $\mathbf{o} + \mathbf{d}[i]$ with radius $1.0f$. If the hit is valid and $t < closest.t$, update `closest` [S3, unofficial].
            *   If the node is an inner node, push the context `{o, r, index, hit.t}` to the stack [S3, unofficial].

3.  **Recursive/Iterative Descent (add_node implementation):**
    *   Calculate the node's bounding box $(boxMin, boxMax)$ and check ray-box intersection. If no intersection, return false [S3, unofficial].
    *   If the node is not at max depth and is not a leaf, iterate through its 8 children and recursively check intersection [S3, unofficial].
    *   For each child, calculate the child origin as `ctx.o + direction * half_r` and half radius as `half_r` [S3, unofficial].
    *   Call `add_node` recursively for the child with index `index + child_offset + i` [S3, unofficial].

4.  **Hit Computation:**
    *   Compute hit position: $\mathbf{hitPosition} = \mathbf{rayOrigin} + \mathbf{rayDir} \times t_{min}$ [S3, unofficial].
    *   Compute hit normal:
        *   $\mathbf{centerToHit} = \mathbf{hitPosition} - \mathbf{node->position}$ [S3, unofficial].
        *   $\mathbf{maxComp} = \max(|\frac{centerToHit.x}{halfSize.x}|, \max(|\frac{centerToHit.y}{halfSize.y}|, |\frac{centerToHit.z}{halfSize.z}|))$ [S3, unofficial].
        *   Determine normal based on which component equals `maxComp` [S3, unofficial].
    *   Set `hitAdd = !node->isActive` (indicates if a new voxel should be added next to the hit) [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import struct

# Constants
MAX_STACK_SIZE = 32
INF = float('inf')

# Node masks and directions as per [S3, unofficial]
NODE_M = [0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80]
NODE_D = [
    (0.0, 0.0, 0.0), (0.0, 0.0, 1.0), (0.0, 1.0, 1.0), (0.0, 1.0, 0.0),
    (1.0, 1.0, 0.0), (1.0, 1.0, 1.0), (1.0, 0.0, 1.0), (1.0, 0.0, 0.0)
]

class TraceContext:
    def __init__(self, o, r, index, t):
        self.o = o  # vec3
        self.r = r  # float
        self.index = index  # uint32
        self.t = t  # float

class Hit:
    def __init__(self):
        self.t = INF
        self.valid = False
        self.position = (0.0, 0.0, 0.0)
        self.normal = (0.0, 0.0, 0.0)
        self.node_index = 0

def intersect_box_akari(ray_o, ray_d, box_min, box_max):
    """
    Intersect ray with AABB. Returns t_min, t_max if hit, else None.
    Simplified implementation for demonstration.
    """
    t_min = -INF
    t_max = INF
    
    for i in range(3):
        if ray_d[i] == 0:
            if ray_o[i] < box_min[i] or ray_o[i] > box_max[i]:
                return None
        else:
            t1 = (box_min[i] - ray_o[i]) / ray_d[i]
            t2 = (box_max[i] - ray_o[i]) / ray_d[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
            
    if t_min > t_max:
        return None
    return (t_min, t_max)

def get_node_bounds(node_index, resolution):
    """
    Get bounding box for a node index in a solid voxel octree.
    Simplified for demonstration.
    """
    # Assuming a simple mapping for demonstration
    # In reality, this depends on the octree structure and node sizes
    size = resolution
    x = (node_index % 2) * size / 2
    y = ((node_index // 2) % 2) * size / 2
    z = ((node_index // 4) % 2) * size / 2
    
    box_min = (x, y, z)
    box_max = (x + size, y + size, z + size)
    return box_min, box_max

def sparse_ray_octree_intersection(ray_o, ray_d, octree, resolution):
    stack = []
    closest = Hit()
    
    # Check root node
    root_index = 0
    root_box_min, root_box_max = get_node_bounds(root_index, resolution)
    hit_result = intersect_box_akari(ray_o, ray_d, root_box_min, root_box_max)
    
    if hit_result is None:
        return closest
    
    t_min, t_max = hit_result
    if t_min >= closest.t:
        return closest
        
    # Push root to stack
    stack.append(TraceContext(ray_o, resolution, root_index, t_min))
    
    while stack:
        if len(stack) >= MAX_STACK_SIZE:
            print("Warning: Stack overflow")
            break
            
        ctx = stack.pop()
        
        # Early culling
        if ctx.t >= closest.t:
            continue
            
        # Get node bounds
        node_box_min, node_box_max = get_node_bounds(ctx.index, resolution)
        
        # Re-intersect to get accurate t for this node (simplified)
        # In practice, this might be passed from parent or calculated
        hit_result = intersect_box_akari(ray_o, ray_d, node_box_min, node_box_max)
        if hit_result is None:
            continue
            
        t_min, t_max = hit_result
        if t_min >= closest.t:
            continue
            
        # Check if node is full or leaf
        # Assuming octree.has_node and octree.is_full are available
        if octree.is_full(ctx.index):
            closest.t = t_min
            closest.valid = True
            closest.position = (ray_o[0] + ray_d[0] * t_min, 
                                ray_o[1] + ray_d[1] * t_min, 
                                ray_o[2] + ray_d[2] * t_min)
            # Compute normal
            center = ((node_box_min[0] + node_box_max[0])/2, 
                      (node_box_min[1] + node_box_max[1])/2, 
                      (node_box_min[2] + node_box_max[2])/2)
            half_size = (node_box_max[0] - node_box_min[0])/2
            center_to_hit = (closest.position[0] - center[0], 
                             closest.position[1] - center[1], 
                             closest.position[2] - center[2])
            max_comp = max(abs(center_to_hit[0]/half_size), 
                           abs(center_to_hit[1]/half_size), 
                           abs(center_to_hit[2]/half_size))
            
            if max_comp == abs(center_to_hit[0]/half_size):
                closest.normal = (1.0 if center_to_hit[0] > 0 else -1.0, 0.0, 0.0)
            elif max_comp == abs(center_to_hit[1]/half_size):
                closest.normal = (0.0, 1.0 if center_to_hit[1] > 0 else -1.0, 0.0)
            else:
                closest.normal = (0.0, 0.0, 1.0 if center_to_hit[2] > 0 else -1.0)
            closest.node_index = ctx.index
            return closest
            
        if ctx.r <= 1.0: # Leaf level
            # Iterate 8 children
            half_r = ctx.r / 2.0
            for i in range(8):
                if octree.has_child(ctx.index, i):
                    child_offset = NODE_D[i]
                    child_origin = (ctx.o[0] + child_offset[0] * half_r,
                                    ctx.o[1] + child_offset[1] * half_r,
                                    ctx.o[2] + child_offset[2] * half_r)
                    child_index = octree.get_child_index(ctx.index, i)
                    
                    child_box_min, child_box_max = get_node_bounds(child_index, half_r)
                    child_hit = intersect_box_akari(ray_o, ray_d, child_box_min, child_box_max)
                    
                    if child_hit is not None:
                        ct_min, ct_max = child_hit
                        if ct_min < closest.t:
                            # Update closest if it's a solid voxel
                            if octree.is_full(child_index):
                                closest.t = ct_min
                                closest.valid = True
                                closest.position = (ray_o[0] + ray_d[0] * ct_min, 
                                                    ray_o[1] + ray_d[1] * ct_min, 
                                                    ray_o[2] + ray_d[2] * ct_min)
                                center = ((child_box_min[0] + child_box_max[0])/2, 
                                          (child_box_min[1] + child_box_max[1])/2, 
                                          (child_box_min[2] + child_box_max[2])/2)
                                half_child_size = half_r / 2.0
                                center_to_hit = (closest.position[0] - center[0], 
                                                 closest.position[1] - center[1], 
                                                 closest.position[2] - center[2])
                                max_comp = max(abs(center_to_hit[0]/half_child_size), 
                                               abs(center_to_hit[1]/half_child_size), 
                                               abs(center_to_hit[2]/half_child_size))
                                
                                if max_comp == abs(center_to_hit[0]/half_child_size):
                                    closest.normal = (1.0 if center_to_hit[0] > 0 else -1.0, 0.0, 0.0)
                                elif max_comp == abs(center_to_hit[1]/half_child_size):
                                    closest.normal = (0.0, 1.0 if center_to_hit[1] > 0 else -1.0, 0.0)
                                else:
                                    closest.normal = (0.0, 0.0, 1.0 if center_to_hit[2] > 0 else -1.0)
                                closest.node_index = child_index
                                return closest
                            else:
                                # Push to stack
                                stack.append(TraceContext(child_origin, half_r, child_index, ct_min))
        else:
            # Inner node, push children to stack
            half_r = ctx.r / 2.0
            for i in range(8):
                if octree.has_child(ctx.index, i):
                    child_offset = NODE_D[i]
                    child_origin = (ctx.o[0] + child_offset[0] * half_r,
                                    ctx.o[1] + child_offset[1] * half_r,
                                    ctx.o[2] + child_offset[2] * half_r)
                    child_index = octree.get_child_index(ctx.index, i)
                    
                    child_box_min, child_box_max = get_node_bounds(child_index, half_r)
                    child_hit = intersect_box_akari(ray_o, ray_d, child_box_min, child_box_max)
                    
                    if child_hit is not None:
                        ct_min, ct_max = child_hit
                        if ct_min < closest.t:
                            stack.append(TraceContext(child_origin, half_r, child_index, ct_min))
                            
    return closest
```

**Parameters:**
*   `stack_size`: The maximum depth of the traversal stack, set to 32. If exceeded, a warning is printed [S3, unofficial].
*   `ubo.lod`: Level of Detail multiplier used for termination criteria [S2, unofficial].
*   `ubo.frameWidth`: Width of the framebuffer, used for debug coordinates [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Stack Overflow:** If the stack size (32) is exceeded during traversal, a warning is printed via `printf` [S3, unofficial].
*   **Early Culling:** If the intersection distance `t` of a node is greater than or equal to the current `closest.t`, the node and its children are skipped [S3, unofficial].
*   **Precision:** Floating-point precision issues may arise when calculating `t_min` and `t_max` for small nodes or rays nearly parallel to axes.
*   **Empty Root:** If the root node is empty, the algorithm returns an invalid hit immediately [S3, unofficial].

## Subtopics

### [Octree Voxel-Triangle Intersection Test](3316-octree-voxel-triangle-intersection-test.md)
A method to determine if a triangle intersects a voxel within a virtual uniform grid defined by an octree, used for culling or traversal in volume rendering. It implements the Separating Axis Theorem (SAT) to test intersection between a triangle and an axis-aligned bounding box (cube).

### [Woodcock Collision Estimation with Octree Acceleration](3419-woodcock-collision-estimation-with-octree-acceleration.md)
An optimized ray marching method that combines Woodcock collision estimation with an octree traversal to skip empty regions and nodes that do not intersect the ray's current segment, reducing the number of volume samples. It differs from standard ray marching by using the octree to cull non-intersecting nodes, and from its parent [Sparse Ray-Octree Intersection (DECIDE)](3006-sparse-ray-octree-intersection-decide.md) by employing stochastic collision estimation within leaf nodes rather than deterministic traversal.

### [LiNode Light Gathering with Octree Traversal](3957-linode-light-gathering-with-octree-traversal.md)
LiNode Light Gathering is a ray-surfel intersection method that traverses an octree of surfels to determine the closest hit color and distance for a given ray. It solves the problem of efficiently gathering lighting information from a sparse, hierarchical surfel representation. It differs from its siblings by operating on surfel primitives within octree leaves rather than triangles (Octree Voxel-Triangle Intersection Test), using deterministic traversal rather than stochastic estimation (Woodcock Collision Estimation with Octree Acceleration), or performing point queries (Octree Point Search for Closest Point).

### [Octree Point Search for Closest Point](4999-octree-point-search-for-closest-point.md)
This method uses an octree to efficiently find the closest point to a ray origin among all points intersected by the ray, filtering by a maximum range.

### [Leaf Consideration](5126-leaf-consideration.md)
Leaf Consideration is the termination step in sparse ray-octree traversal where an intersected leaf node is processed to determine occlusion and record hit data. It distinguishes between pure voxel leaves and virtual leaves (intermediate nodes acting as leaves) to provide the precise intersection point and normal required for shading. It sets itself apart from sibling methods like Octree Voxel-Triangle Intersection by focusing on axis-aligned bounding box (AABB) intersection and hit recording rather than geometric primitive intersection or light gathering.

## Operating characteristics
*   **Time Complexity:** $O(\log N)$ in the best case for sparse octrees, where $N$ is the number of voxels, as the traversal depth is logarithmic with respect to the resolution. In the worst case (dense octree or ray passing through many nodes), it can approach $O(N)$.
*   **Memory Complexity:** $O(D)$, where $D$ is the maximum depth of the octree, due to the fixed-size stack. The stack size is typically limited to 32 [S3, unofficial].
*   **Measured Performance:** Performance depends heavily on the sparsity of the octree and the coherence of the rays. GPU-based implementations benefit from parallel execution over threads [S2, unofficial].
*   **Scaling:** Scales well with resolution due to the hierarchical nature of the octree. However, deep octrees may lead to stack overflow if the stack size is not increased [S3, unofficial].
*   **Strengths:** Efficient for sparse scenes, reduces the number of nodes visited compared to uniform grid traversal, and provides precise intersection points and normals.
*   **Limitations:** Requires a pre-built octree structure, and stack overflow can occur for very deep trees. Performance may degrade for highly dense or complex scenes.
*   **When to Use:** Suitable for rendering or collision detection in scenes with sparse, hierarchical voxel data, especially on GPU architectures.
*   **When Not to Use:** Not ideal for very dense scenes where the overhead of octree traversal outweighs the benefits, or when stack space is severely limited.

## Background
DECIDE was introduced as a GPU-based algorithm for ray-octree intersection [S2, unofficial][S3, unofficial]. It is part of the broader field of volume rendering and ray tracing, where efficient traversal of hierarchical acceleration structures is crucial. The algorithm builds on the principles of ray-node intersection logic, specifically focusing on sparse voxel octrees [Parent](22847-ray-node-intersection-logic.md).

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
[S2] krre/origin, https://github.com/krre/origin, fetched 2026-10-26 (unofficial)
[S3] Mike-Leo-Smith/SolidVoxelization, https://github.com/Mike-Leo-Smith/SolidVoxelization, fetched 2026-10-27 (unofficial)
[S4] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
