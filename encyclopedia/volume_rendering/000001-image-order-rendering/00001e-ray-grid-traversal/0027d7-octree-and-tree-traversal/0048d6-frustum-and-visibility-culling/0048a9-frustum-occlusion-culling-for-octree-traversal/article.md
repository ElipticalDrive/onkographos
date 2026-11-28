# Frustum-Occlusion Culling for Octree Traversal

Also called: Frustum-octant traversal, Voxel LOD selection, Voxel LOD selection via Frustum Culling, Voxel Rasterization via Frustum Culling and Instancing

## Overview
This method identifies a set of visible voxels by traversing an octree hierarchy and culling nodes that do not intersect the camera's view frustum or whose projected screen area is smaller than a single pixel [S1, unofficial]. It serves as a visibility pre-pass for voxel rasterization, automatically determining the Level of Detail (LOD) for rendering [S1, unofficial]. Unlike standard ray-marching which samples volume density, this approach outputs a list of 3D bounding boxes to be rasterized as geometry using hardware instancing [S1, unofficial].

## How it works
**Inputs:** Camera view frustum, octree hierarchy with axis-aligned bounding boxes (AABBs), global AABB, screen resolution $(W, H)$ [S1, unofficial].
**Outputs:** Buffer of visible voxels, each containing position and size [S1, unofficial].
**Data Structures:** Octree node with AABB and 8-bit occupancy mask [S1, unofficial]. Voxel buffer storing `{position: vec3, size: float}` [S1, unofficial].

**Algorithm:**
1. **Frustum Intersection Test:** Approximate octant-AABB intersection as a frustum-sphere intersection for efficiency [S1, unofficial]. The sphere diameter equals the octant's AABB diagonal [S1, unofficial]. If the sphere does not intersect the frustum, cull the branch [G].
2. **Projected Area Test (LOD Selection):** Calculate the octant's projected area on the screen [G]. If the area is less than one pixel, halt traversal for this branch [S1, unofficial].
3. **Termination:** Halt traversal if the node has no further subdivisions (leaf) [S1, unofficial].
4. **Buffer Population:** When halted (LOD cutoff, leaf, or frustum exit), append the voxel's position and size to the buffer [S1, unofficial]. These attributes are derived implicitly from the node's AABB within the global coordinate space [S1, unofficial].
5. **Rasterization:** After traversal, pass the buffer to a hardware instancing pipeline to rasterize the voxels [S1, unofficial].

**Formulas:**
* Sphere center: $\mathbf{c} = \frac{\mathbf{aabb}_{min} + \mathbf{aabb}_{max}}{2}$
* Sphere radius: $r = \frac{\|\mathbf{aabb}_{max} - \mathbf{aabb}_{min}\|}{2}$
* LOD condition: $A_{screen} < 1.0$ pixel [S1, unofficial]

**Parameters:** Canonical voxel template: cube with $c_{min} = (-1, -1, -1)$ and $c_{max} = (+1, +1, +1)$ [S1, unofficial]. LOD threshold: projected area $< 1$ pixel [S1, unofficial].

**Edge Cases and Pitfalls:** Numeric precision degrades at extreme subdivision depths [S1, unofficial]. Over-subdivision wastes memory when LOD would naturally halt at coarser levels [S1, unofficial]. Sphere approximation may cause false positives (culling fewer nodes than strict AABB test) but never false negatives [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_octree(node, frustum, screen_width, screen_height, voxel_buffer):
    # 1. Frustum Culling via Sphere Approximation
    center = (node.aabb.min + node.aabb.max) / 2.0
    radius = length(node.aabb.max - node.aabb.min) / 2.0
    if not frustum.intersects_sphere(center, radius):
        return

    # 2. Projected Area / LOD Check
    projected_area = calculate_projected_area(node.aabb, frustum.view_matrix, frustum.proj_matrix, screen_width, screen_height)
    if projected_area < 1.0 or node.is_leaf():
        voxel_buffer.append({'position': center, 'size': length(node.aabb.max - node.aabb.min)})
        return

    # 3. Recurse into children
    for child in node.children:
        if child is not None:
            traverse_octree(child, frustum, screen_width, screen_height, voxel_buffer)

def calculate_projected_area(aabb, view_matrix, proj_matrix, w, h):
    # Transform 8 AABB corners to clip space [G]
    corners = []
    for x in [aabb.min.x, aabb.max.x]:
        for y in [aabb.min.y, aabb.max.y]:
            for z in [aabb.min.z, aabb.max.z]:
                corners.append(vec4(x, y, z, 1.0))
    clip_corners = [proj_matrix * (view_matrix * c) for c in corners]
    
    # Convert to NDC [-1, 1] and find screen-space bounding box [G]
    ndc_x = [(c.x / c.w) for c in clip_corners]
    ndc_y = [(c.y / c.w) for c in clip_corners]
    min_x, max_x = min(ndc_x), max(ndc_x)
    min_y, max_y = min(ndc_y), max(ndc_y)
    
    # Map NDC to screen pixels
    screen_w = max(0.0, (max_x - min_x) * 0.5 * w)
    screen_h = max(0.0, (max_y - min_y) * 0.5 * h)
    return screen_w * screen_h
```

## Operating characteristics
**Time Complexity:** Worst-case $O(N)$ for a full octree [G]. Average case visits only visible and LOD-relevant nodes [S1, unofficial].
**Memory Complexity:** Octree stored breadth-first with 1 byte per node mask [S1, unofficial]. Buffer size scales with visible voxel count [S1, unofficial].
**Performance:** Profiled at 1920x1280 resolution with 25%–85% pixel coverage on Intel Core i5-2540M / NVIDIA Quadro 3000M / Windows 7 / OpenGL 4.2 [S1, unofficial]. Voxel rasterization times (ms) for 'basketball' dataset at depth 10: Geometry Shader 3.454, Uniform Buffer 2.549, Hardware Texture Buffer 2.773, Instancing Vertex Buffer 5.599 [S1, unofficial].
**Strengths and Limitations:** Automatic LOD selection and efficient frustum culling [S1, unofficial]. Geometry Shader is slowest for voxels due to hardware optimization for quads; Uniform Buffer is fastest [S1, unofficial].

## Background
Described in the context of hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4D media [S1, unofficial]. Highlights trade-offs between GPU instancing techniques for voxel data [S1, unofficial].

## Sources
[S1] Hardware-accelerated geometry instancing for surfel and voxel rendering of scanned 4d media, http://133.5.19.150/papers/2010_2013/1-1-36.pdf, fetched 2026-10-31 (unofficial)
