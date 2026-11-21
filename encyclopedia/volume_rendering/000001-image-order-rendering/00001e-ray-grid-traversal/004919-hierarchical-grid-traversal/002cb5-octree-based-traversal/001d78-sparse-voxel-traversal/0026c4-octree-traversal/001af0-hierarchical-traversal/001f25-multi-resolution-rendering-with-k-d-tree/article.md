# Multi-Resolution Rendering with K-D Tree

Also called: Error-controlled approximation, Nodal region rendering

## Overview
Multi-resolution rendering with K-D Tree is a hierarchical volume rendering technique that accelerates image-order ray casting by storing approximations of the underlying data within K-D tree nodes [S1]. It solves the performance bottleneck of rendering large, high-resolution datasets by allowing the renderer to skip detailed traversal when the data within a spatial region can be represented by a simple model within a user-specified error threshold [S1]. Unlike wavelet-based methods which store only detail coefficients, this method stores a complete model for each region, enabling fast "preview" images and adaptive detail [S1]. It builds on [Hierarchical Traversal](896-hierarchical-traversal.md) [S1].

## How it works
**Inputs:** A K-D tree where each node represents a spatial subregion (nodal region) [S1]. A restrict box defining the region of interest [S1]. A user-defined acceptable error threshold $\epsilon$ [S1]. A user-defined polygon count cut-off $N_{cut}$ [S1]. Ray parameters (origin and direction) [S1].

**Outputs:** A set of subranges to be rendered, corresponding to K-D tree nodes that satisfy the error or leaf criteria [S1]. Rendered image pixels [S1].

**Data Structures:** Each K-D tree node contains:
1. **Spatial Bounds:** Defining the nodal region [S1].
2. **Error Term ($E$):** A scalar representing the deviation of the nodal model from the actual data [S1].
3. **Nodal Model Parameters:** Eight data values corresponding to the corners of the nodal region [S1].
4. **Polygon Subrange:** A contiguous range in a global polygon array ($Tid$) storing polygon IDs within the node's spatial region [S1].
5. **Children Pointers:** For non-leaf nodes [S1].

**Nodal Model Construction:** The nodal model is a trilinear function induced by eight data values stored at the corners of the nodal region [S1]. To compute the data value at a specific corner of the nodal region, the algorithm locates the grid cell in the original dataset that contains the corner point and interpolates within that grid cell to compute the data value at the corner [S1]. These eight values are stored directly in the tree node [S1]. The trilinear interpolation at a point $(x, y, z)$ within the nodal region is computed as:
$$V(x,y,z) = \sum_{i=0}^{1}\sum_{j=0}^{1}\sum_{k=0}^{1} w_i(x) w_j(y) w_k(z) V_{ijk}$$
where $V_{ijk}$ are the eight corner values and $w_i(t) = 1-t$ or $t$ depending on the corner index [G].

**Traversal Algorithm:** Traversal occurs in front-to-back visibility order [S1]. The algorithm determines which nodes to render in detail and which to approximate [S1].

1. **Initialize:** Start traversal at the root node [G].
2. **Check Restrict Box:** If the node is entirely outside the restrict box, return immediately [S1].
3. **Check Inclusion:** If the node is entirely inside the restrict box, add its polygon subrange ($Tid$) to the set of subranges to be rendered and return without traversing children [S1].
4. **Check Leaf/Cut-off:** If the node is a leaf, or if the number of polygons in its region is less than the user-specified cut-off $N_{cut}$, add its polygon subrange ($Tid$) to the set of subranges to be rendered and return [S1].
5. **Check Error:** If the node's stored error term $E$ is less than the acceptable error $\epsilon$, traversal stops at this node and the nodal region is drawn using the trilinear approximation model [S1].
6. **Recursive Step:** If none of the above exceptions apply (node is partially inside restrict box, not a leaf, polygon count $\ge N_{cut}$, and error $E \ge \epsilon$), add the polygon subrange ($Tid$) associated with the node itself (but not its descendants) to the set of subranges to be rendered, and recursively traverse the node's children [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trilinear(v000, v100, v010, v110, v001, v101, v011, v111, x, y, z):
    ix, iy, iz = 1.0 - x, 1.0 - y, 1.0 - z
    return (ix*iy*iz*v000 + x*iy*iz*v100 + ix*y*iz*v010 + x*y*iz*v110 +
            ix*iy*z*v001 + x*iy*z*v101 + ix*y*z*v011 + x*y*z*v111)

def ray_node_intersection(node_bounds, ray_origin, ray_direction):
    t_min, t_max = 0.0, float('inf')
    for axis in range(3):
        if ray_direction[axis] == 0:
            if not (node_bounds.min[axis] <= ray_origin[axis] <= node_bounds.max[axis]):
                return None
        else:
            t1 = (node_bounds.min[axis] - ray_origin[axis]) / ray_direction[axis]
            t2 = (node_bounds.max[axis] - ray_origin[axis]) / ray_direction[axis]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    return (t_min, t_max) if t_min <= t_max else None

def draw_nodal_region(node, ray_origin, ray_direction):
    bounds = ray_node_intersection(node.bounds, ray_origin, ray_direction)
    if bounds is None:
        return
    t_start, t_end = bounds
    dx = node.bounds.max.x - node.bounds.min.x
    dy = node.bounds.max.y - node.bounds.min.y
    dz = node.bounds.max.z - node.bounds.min.z
    steps = max(1, int((t_end - t_start) / max(dx, dy, dz)))
    for i in range(steps + 1):
        t = t_start + (t_end - t_start) * i / steps
        px, py, pz = ray_origin + ray_direction * t
        x = (px - node.bounds.min.x) / dx
        y = (py - node.bounds.min.y) / dy
        z = (pz - node.bounds.min.z) / dz
        value = trilinear(*node.corners, x, y, z)
        # Composite value into framebuffer [G]
        pass

def traverse_kd_node(node, restrict_box, epsilon, n_cut, tid_array, render_set, ray_origin, ray_direction):
    if (node.bounds.max.x < restrict_box.min.x or node.bounds.min.x > restrict_box.max.x or
        node.bounds.max.y < restrict_box.min.y or node.bounds.min.y > restrict_box.max.y or
        node.bounds.max.z < restrict_box.min.z or node.bounds.min.z > restrict_box.max.z):
        return
    if (node.bounds.min.x >= restrict_box.min.x and node.bounds.max.x <= restrict_box.max.x and
        node.bounds.min.y >= restrict_box.min.y and node.bounds.max.y <= restrict_box.max.y and
        node.bounds.min.z >= restrict_box.min.z and node.bounds.max.z <= restrict_box.max.z):
        render_set.add(tid_array[node.tid_start : node.tid_end])
        return
    if node.is_leaf or node.polygon_count < n_cut:
        render_set.add(tid_array[node.tid_start : node.tid_end])
        return
    if node.error < epsilon:
        draw_nodal_region(node, ray_origin, ray_direction)
        return
    render_set.add(tid_array[node.tid_start : node.tid_end])
    for child in node.children:
        traverse_kd_node(child, restrict_box, epsilon, n_cut, tid_array, render_set, ray_origin, ray_direction)
```

**Parameters:** Acceptable error $\epsilon$ (user-specified) [S1]. Polygon count cut-off $N_{cut}$ (user-specified) [S1].

**Edge Cases and Pitfalls:** Technical problems exist to avoid visual discontinuities on the boundaries between nodal regions that use different rendering methods (approximation vs. detailed polygon rendering) [S1]. If standard cell projection methods are used exclusively for nodal regions, discontinuities are avoided and graphics hardware acceleration is possible [S1].

## Operating characteristics
* **Time Complexity:** Not given in the sources [S1].
* **Memory Complexity:** Not given in the sources [S1].
* **Measured Performance:** The technique constructs very fast "preview" images to help orient the user in a very large data set [S1].
* **Strengths:** It enables adaptive detail based on user-defined error and supports hierarchical and parallelizable direct volume rendering for irregular and multiple grids [S1].
* **Limitations:** It requires careful handling of boundaries to avoid discontinuities between approximated and detailed regions [S1].
* **When to use:** It is suitable for large datasets where interactive preview or adaptive detail is required, provided visual continuity across approximation boundaries can be managed [S1].
* **When not to use:** It should be avoided in scenes requiring strict visual continuity across all spatial boundaries without specialized boundary handling [S1].

## Background
The technique was proposed for hierarchical and parallelizable direct volume rendering of irregular and multiple grids [S1]. It departs from wavelet-based approaches by storing a complete model for each region rather than just detail information [S1]. Not given in the sources [G].

## Sources
[S1] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
