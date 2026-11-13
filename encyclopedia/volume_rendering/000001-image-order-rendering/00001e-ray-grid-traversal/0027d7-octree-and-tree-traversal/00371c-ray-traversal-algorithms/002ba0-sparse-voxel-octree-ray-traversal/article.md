# Sparse Voxel Octree Ray Traversal

Also called: OctreeSimple Shader, SVO Traversal, Sparse Voxel Octree (SVO) Traversal, Sparse Voxel Octree Ray Casting, Sparse Voxel Octree Tracing

## Overview
Sparse Voxel Octree (SVO) Ray Traversal is a hierarchical raycasting technique that renders volumetric data stored in a 3D sparse voxel octree by traversing the tree structure along ray paths to find occupied cells and their bounding boxes [S2, unofficial][S4, unofficial]. It solves the problem of efficiently rendering large, sparse 3D voxel datasets by skipping empty space, providing an implicit Level-of-Detail (LOD) mechanism based on viewer distance [S3, unofficial][S8]. Unlike uniform grid ray marching, which samples every voxel, SVO traversal uses hierarchical node pointers to jump over empty regions, and unlike BVH-based approaches, it operates directly on the octree structure without an intermediate bounding volume hierarchy [S1, unofficial].

## How it works
The method traverses a Sparse Voxel Octree to find the first occupied leaf node (voxel with color data) intersected by a ray [S1, unofficial][S7, unofficial]. The traversal is typically implemented as an iterative process using an explicit stack to handle backtracking, as recursion is often unsupported in GPU shader languages like HLSL or GLSL [S3, unofficial][S4, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Ray: Origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).
    *   SVO Data: A buffer or texture containing the octree nodes.
    *   Camera Parameters: Viewport size, FOV, projection matrices (for ray generation).
*   **Outputs:**
    *   Hit position, surface normal, and color/opacity of the first intersected voxel.
    *   Background color if no hit is found.

### Data Structures
The SVO is stored as a flat array of nodes. Common layouts include:
1.  **Pointer/Flag Structure:** Each node is a 32-bit integer. Bit 31 indicates the type (0 = Pointer to children, 1 = Leaf/Voxel) [S4, unofficial]. Bits 0-30 store the pointer to the first child node or the head of an attribute list [S4, unofficial].
2.  **Mask Structure:** Some implementations use a parent element containing `masks` (uint) and `childPointer` (uint). Masks track valid and leaf status of children using specific bit ranges (e.g., bits 24+ for valid, 16+ for leaf) [S1, unofficial].
3.  **Node Pool:** A struct `SVONode` containing two u32 values: `node[0]` with flags (bit 31: empty, bit 30: leaf, bits 0-29: child index) and `node[1]` with color data [S7, unofficial].
4.  **Stack:** A fixed-size array (e.g., depth 16-32) storing traversal state. For GPU implementations, this is often a parallel array or a struct containing `node_address`, `frontal_mask`, `t_min`, and `t_corner` [S9, unofficial].

### Algorithm Steps

1.  **Ray Initialization:**
    *   Generate ray origin and direction from camera parameters [S5, unofficial][S9, unofficial].
    *   Transform ray origin to octree space if necessary (e.g., shift by +1.0 or +1.5 to handle coordinate system differences) [S4, unofficial][S7, unofficial].
    *   Clamp small ray direction components to a small epsilon (e.g., $10^{-4}$ or $3.55 \times 10^{-15}$) to prevent division by zero during slab intersection calculations [S7, unofficial][S9, unofficial].
    *   Precompute ray coefficients for slab intersection: $t_{coef} = 1.0 / |\mathbf{d}|$ and $t_{bias} = t_{coef} \cdot \mathbf{o}$ [S7, unofficial].

2.  **Root Intersection:**
    *   Compute the intersection of the ray with the root bounding box (AABB) [S1, unofficial][S9, unofficial].
    *   Calculate $t_{min}$ and $t_{max}$ for each axis using the slab method:
        $$ t_{min} = \max_i \left( \frac{box_{min,i} - o_i}{d_i} \right), \quad t_{max} = \min_i \left( \frac{box_{max,i} - o_i}{d_i} \right) $$
    *   If $t_{min} > t_{max}$ or $t_{max} < 0$, the ray misses the octree; return background color [S1, unofficial][S9, unofficial].
    *   Set initial traversal state: `node_address` = root index, `depth` = 0, `scale` = root radius, `t_min` = intersection entry distance [S1, unofficial][S9, unofficial].

3.  **Traversal Loop:**
    *   While `depth` < `MAX_DEPTH`:
        *   **Fetch Node:** Load the current node value from the SVO buffer at `node_address` [S9, unofficial].
        *   **Check Node Type:**
            *   **Leaf Node:** If the node indicates a leaf (e.g., bit 31 is 1 or value > 0 and MSB not set), the ray has hit a voxel. Break the loop and sample the voxel color/attributes [S9, unofficial].
            *   **Empty Node:** If the node indicates empty space (e.g., bit 31 is 0 and value is 0, or specific empty flag), skip this subtree. Advance the ray to the next potential intersection using the "Empty Space Skipping" logic (see below) [S9, unofficial].
            *   **Internal Node:** If the node is an internal node (e.g., MSB set), push the current state (`node_address`, `frontal_mask`, `t_min`, `t_corner`) onto the stack [S9, unofficial].
        *   **Descend:**
            *   Halve the `scale` and update `step` ($scale \times 0.5$) [S9, unofficial].
            *   Determine the child index to visit. This is typically done by comparing the ray's position relative to the center of the current node's bounding box [S1, unofficial].
            *   Calculate the center of the current node: $\mathbf{c} = (\mathbf{min} + \mathbf{max}) / 2$ [S9, unofficial].
            *   Compute $t_{center} = (\mathbf{c} - \mathbf{o}) / \mathbf{d}$ [S9, unofficial].
            *   Determine the `frontal_mask` based on whether the ray enters the front or back half of the node along each axis (comparing $t_{center}$ components to $t_{min}$) [S9, unofficial].
            *   Update `node_address` to the child's address (extracted from the current node's value) [S9, unofficial].
            *   Update `t_corner` and `frontal_mask` for the new child node [S9, unofficial].

4.  **Empty Space Skipping (Advance):**
    *   When an empty node is encountered, the ray must jump to the next occupied node or exit the current bounding box.
    *   Calculate $t_{corner\_max} = \min(t_{corner.x}, t_{corner.y}, t_{corner.z})$ [S9, unofficial].
    *   Update $t_{min} = t_{corner\_max}$ [S9, unofficial].
    *   Determine `step_mask` for axes where the ray exits the current child's slab (where $t_{corner}$ component $\le t_{corner\_max}$) [S9, unofficial].
    *   Advance the ray position in those axes by adding $\text{sign}(\mathbf{d}) \cdot \text{step} / \mathbf{d}$ to the corner coordinates [S9, unofficial].
    *   XOR `step_mask` into `frontal_mask` to reflect the new position relative to the parent's center [S9, unofficial].
    *   Repeat until a non-empty child is found or the ray exits the root [S9, unofficial].

5.  **Backtracking:**
    *   If the ray exits a subtree (e.g., $t_{min} > t_{max}$ for the current child) or reaches the end of a branch, pop the state from the stack [S9, unofficial].
    *   Restore `node_address`, `frontal_mask`, `t_min`, and `t_corner` from the stack [S9, unofficial].
    *   Continue traversal from the parent node, selecting the next sibling child [S9, unofficial].

6.  **Leaf Handling:**
    *   Upon reaching a leaf node, sample the voxel data (color, normal, etc.) [S4, unofficial].
    *   Calculate the surface normal based on the ray exit face (using the dominant axis of the vector from the node center to the hit point) [S1, unofficial].
    *   Apply lighting and shading models if required [S4, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_DEPTH = 32
EPS = 1e-4

# Data Structures
struct StackEntry:
    node_address: uint
    frontal_mask: uint
    t_min: float
    t_corner: vec3

def traverse_svo(ray_origin, ray_direction, svo_buffer, root_min, root_max):
    # 1. Ray Initialization
    # Clamp zero direction components
    d = vec3(
        EPS if abs(ray_direction.x) < EPS else ray_direction.x,
        EPS if abs(ray_direction.y) < EPS else ray_direction.y,
        EPS if abs(ray_direction.z) < EPS else ray_direction.z
    )
    
    # Precompute coefficients
    t_coef = 1.0 / abs(d)
    t_bias = t_coef * ray_origin
    
    # 2. Root Intersection
    t_min_root = max(
        (root_min.x - ray_origin.x) / d.x,
        (root_min.y - ray_origin.y) / d.y,
        (root_min.z - ray_origin.z) / d.z
    )
    t_max_root = min(
        (root_max.x - ray_origin.x) / d.x,
        (root_max.y - ray_origin.y) / d.y,
        (root_max.z - ray_origin.z) / d.z
    )
    
    if t_min_root > t_max_root or t_max_root < 0:
        return background_color
    
    t_min = max(t_min_root, 0.0)
    
    # 3. Initial State
    node_address = 0 # Root index
    depth = 0
    scale = (root_max.x - root_min.x) / 2.0 # Assuming cubic root
    step = scale
    center = (root_min + root_max) / 2.0
    
    # Determine initial frontal mask based on ray entry relative to center
    t_center = (center - ray_origin) / d
    frontal_mask = 0
    if t_center.x < t_min: frontal_mask |= 1
    if t_center.y < t_min: frontal_mask |= 2
    if t_center.z < t_min: frontal_mask |= 4
    
    # Initial corner
    corner = center + step * vec3(
        -1.0 if (frontal_mask & 1) else 1.0,
        -1.0 if (frontal_mask & 2) else 1.0,
        -1.0 if (frontal_mask & 4) else 1.0
    )
    t_corner = (corner - ray_origin) / d
    
    stack = [StackEntry() for _ in range(MAX_DEPTH)]
    stack_depth = 0
    
    # 4. Traversal Loop
    while depth < MAX_DEPTH:
        # Fetch Node
        value = svo_buffer[node_address]
        
        # Check Node Type (Assuming MSB 0x80000000 indicates internal node)
        if value & 0x80000000:
            # Internal Node: Push state and descend
            stack[stack_depth] = StackEntry(node_address, frontal_mask, t_min, t_corner)
            stack_depth += 1
            
            # Halve scale and step
            scale *= 0.5
            step *= 0.5
            
            # Update node address to child
            node_address = value & 0x7FFFFFFF
            
            # Update center and corner for new child
            # The new center is the current corner
            center = corner
            
            # Recalculate t_center for new child
            t_center = (center - ray_origin) / d
            
            # Reset frontal mask for new child relative to its new center
            # Note: The frontal mask logic depends on the specific implementation's coordinate system
            # Here we assume the mask bits correspond to x, y, z axes relative to the new center
            new_frontal_mask = 0
            if t_center.x < t_min: new_frontal_mask |= 1
            if t_center.y < t_min: new_frontal_mask |= 2
            if t_center.z < t_min: new_frontal_mask |= 4
            frontal_mask = new_frontal_mask
            
            # Update corner for the new child
            corner = center + step * vec3(
                -1.0 if (frontal_mask & 1) else 1.0,
                -1.0 if (frontal_mask & 2) else 1.0,
                -1.0 if (frontal_mask & 4) else 1.0
            )
            t_corner = (corner - ray_origin) / d
            
            depth += 1
            continue
            
        elif value > 0:
            # Leaf Node: Hit found
            # Sample color from svo_buffer or attribute list
            return sample_leaf_color(value, ray_origin, ray_direction, t_min)
        else:
            # Empty Node: Skip
            # Advance ray to next potential intersection
            t_corner_max = min(t_corner.x, t_corner.y, t_corner.z)
            t_min = t_corner_max
            
            # Determine step mask
            step_mask = 0
            if t_corner.x <= t_corner_max: step_mask |= 1
            if t_corner.y <= t_corner_max: step_mask |= 2
            if t_corner.z <= t_corner_max: step_mask |= 4
            
            # Advance corner
            corner += sign(d) * step / d * vec3(
                1.0 if (step_mask & 1) else 0.0,
                1.0 if (step_mask & 2) else 0.0,
                1.0 if (step_mask & 4) else 0.0
            )
            t_corner = (corner - ray_origin) / d
            
            # Update frontal mask
            frontal_mask ^= step_mask
            
            # Check if we have stepped out of the current node's bounds
            # If frontal_mask is 0, we might need to backtrack or exit
            if (frontal_mask & step_mask) == 0:
                # Backtrack
                if stack_depth > 0:
                    stack_depth -= 1
                    entry = stack[stack_depth]
                    node_address = entry.node_address
                    frontal_mask = entry.frontal_mask
                    t_min = entry.t_min
                    t_corner = entry.t_corner
                    depth -= 1
                else:
                    # Exited root
                    return background_color
            else:
                # Continue with current node (empty space skipping loop)
                continue
                
    return background_color
```

### Parameters and Typical Values
*   **MAX_DEPTH:** 16-32, depending on the resolution of the voxel grid [S2, unofficial][S9, unofficial].
*   **EPS:** $10^{-4}$ to $10^{-15}$, used to prevent division by zero [S7, unofficial][S9, unofficial].
*   **FOV:** 45-50 degrees [S6, unofficial].
*   **Root Radius:** Depends on the scene size, often normalized to 1.0 or 2.0 [S1, unofficial].

### Edge Cases and Pitfalls
*   **Boundary Precision:** If the ray position lies exactly on the boundary defined by the node center, the child index selection can be ambiguous. A small epsilon ($0.1 \times \text{dir}$) is added to the position to break ties [S1, unofficial].
*   **Zero Direction Components:** Ray directions with zero components cause division by zero. Components are clamped to $\pm \text{EPS}$ based on their sign [S9, unofficial].
*   **Stack Overflow:** The traversal uses a fixed-size stack. If the octree depth exceeds the stack size, backtracking may fail. The stack size should be at least the maximum depth of the octree [S9, unofficial].
*   **Texture Repeat Artifacts:** In texture-based SVO implementations, if the lookup point is outside the unit cube, texture wrapping can occur. The ray is intersected with the unit box, and the lookup point is advanced slightly beyond the intersection [S2, unofficial].
*   **Z-Fighting/Gaps:** Using exact intersection points can cause visual artifacts. Stepping slightly past the boundary (e.g., multiplying by 1.0001) is recommended [S2, unofficial].

## Subtopics

### [Sparse Voxel DAG Construction with Subtree Caching](1289-sparse-voxel-dag-construction-with-subtree-caching.md)
Replaces the explicit octree with a shared graph structure to reduce memory usage by merging identical subtrees.

### [Sparse Voxel Octree Ray Tracing](2989-sparse-voxel-octree-ray-tracing.md)
Adds shading and hit-computation logic for final image generation, using a single draw call for a fullscreen triangle.

### [GigaVoxels](3057-gigavoxels.md)
Introduces a hierarchical mipmap-like structure and a centralized cache manager for on-demand data streaming.

### [SVO Node Location](3249-svo-node-location.md)
Determines the specific octree node intersected by a ray at a given position, enabling payload retrieval.

### [Sparse Voxel Octree Ray Traversal (State Machine)](3299-sparse-voxel-octree-ray-traversal-state-machine.md)
Implements traversal as a deterministic finite state machine using fixed-point arithmetic, suitable for hardware synthesis.

### [Animated Sparse Voxel Octrees (ASVO)](3326-animated-sparse-voxel-octrees-asvo.md)
Decouples traversal into a parallel job queue system to handle skeletal animation of voxel models.

### [SVO Ray Tracing](3451-svo-ray-tracing.md)
Converts the SVO into a BVH-like structure for efficient ray intersection and volume compositing.

### [Hierarchical Voxel Traversal with Portal Teleportation](6596-hierarchical-voxel-traversal-with-portal-teleportation.md)
Supports teleportation through portals defined in the voxel data, maintaining transformation matrices for seamless visual continuity.

### Ray Casting with Octrees
Uses an octree data structure to minimize the number of transparent voxels during accumulation, as a group of transparent voxels may be represented as a single node in the octree [S10, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(\log N)$ on average for sparse data, where $N$ is the number of voxels, due to the hierarchical skipping of empty space. In the worst case (dense data), it approaches $O(N)$ [S3, unofficial].
*   **Memory Complexity:** $O(N)$ for the SVO structure, but significantly reduced compared to uniform grids due to sparsity. DAG variants can further reduce this by merging identical subtrees [S8].
*   **Measured Performance:** Can render scenes with >1 billion voxels at 20 FPS at 720p on modern GPUs (e.g., AMD 5700) [S7, unofficial].
*   **Strengths:** Efficient handling of large, sparse datasets; implicit LOD; no need for polygonal geometry [S1, unofficial][S8].
*   **Limitations:** Dynamic updates to voxel data can be inefficient, potentially requiring full tree regeneration [S3, unofficial]. Memory constraints at very high resolutions [S8].
*   **When to Use:** Real-time rendering of large voxel-based scenes, especially where sparsity is high [S7, unofficial].
*   **When Not to Use:** Highly dense volumetric data where uniform grid ray marching might be simpler and faster due to cache locality [S3, unofficial].

## Background
Sparse Voxel Octrees were introduced as a hierarchical data structure for raycasting, allowing for efficient rendering of large 3D voxel datasets by storing only non-empty nodes [S3, unofficial]. The technique builds on the concept of octrees, where space is recursively subdivided into eight octants [S1, unofficial]. The baseline SVO ray traversal method provides an implicit LOD mechanism and is often used as a comparison point for more advanced variants like DAGs [S8]. It is widely used in real-time voxel rendering pipelines, such as GigaVoxels, and in research repositories for rendering complex voxel scenes [S7, unofficial][S10, unofficial].

## Sources
[S1] Onlinerocker/SVO, https://github.com/Onlinerocker/SVO, fetched 2026-10-24 (unofficial)
[S2] RvanderLaan/lwjgl-voxel-raycasting, https://github.com/RvanderLaan/lwjgl-voxel-raycasting, fetched 2026-10-24 (unofficial)
[S3] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S4] xandergos/unity-sparse-voxel-octrees, https://github.com/xandergos/unity-sparse-voxel-octrees, fetched 2026-10-26 (unofficial)
[S5] gabrielmuller/zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
[S6] krre/origin, https://github.com/krre/origin, fetched 2026-10-26 (unofficial)
[S7] mxbrt/svo, https://github.com/mxbrt/svo, fetched 2026-10-26 (unofficial)
[S8] (PDF) High resolution sparse voxel DAGs - ResearchGate, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
[S9] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S10] Introduction to volume rendering, https://slidetodoc.com/introduction-to-volume-rendering-presented-by-zvi-devir/, fetched 2026-10-29 (unofficial)
