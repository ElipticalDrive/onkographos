# SVO Walk (Traversal) — fast_walk

Also called: BVH Traversal with SVO Leaves, BVH-SVO Hybrid Traversal

## Overview
This method is a hybrid ray traversal algorithm that combines a Bounding Volume Hierarchy (BVH) for coarse culling with Sparse Voxel Octree (SVO) structures for fine-grained density evaluation [S1, unofficial]. It efficiently renders large, sparse volumetric datasets by delegating leaf-node processing to an SVO ray marcher while using BVH ray-sphere intersection to skip empty regions [S1, unofficial]. It reduces visited nodes compared to pure SVO traversal and maintains SVO memory efficiency at the leaves compared to pure BVH traversal [S1, unofficial].

## How it works
**Inputs:** Ray origin $O$ and direction $D$; BVH node buffer `BVHNodes`; BVH leaf buffer `BVHLeafs`; SVO data structures accessed via leaf `address`; output accumulators `min_t`, `o_t`, `o_color`, `o_normal` [S1, unofficial].
**Data Structures:** `BVHNode` contains `center` (`vec4`), `radius` (`float`), `left_child` (`uint`), `right_child` (`uint`); stored in `BVHNodes` buffer [S1, unofficial]. `BVHLeaf` contains `inv_model` (`mat4`), `address` (`uint`); stored in `BVHLeafs` buffer [S1, unofficial].
**Parameters:** `BVH_STACK_SIZE` is 64 [S1, unofficial]. `MAX_T` is 10000.0 [S1, unofficial].
**Algorithm Steps:**
1. Initialize stack `S` with root index 0. Set `min_t = ∞`, `o_t = 0`, `o_color = (0,0,0,0)`, `o_normal = (0,0,0)` [S1, unofficial].
2. Loop:
   a. Pop `idx` from `S`. If empty, break [S1, unofficial].
   b. Load `node = BVHNodes[idx]` [S1, unofficial].
   c. Check leaf: `is_leaf = (node.left_child & 0x80000000) != 0` [S1, unofficial].
   d. If leaf: call leaf tracer for left (`node.left_child ^ 0x80000000`) and right (`node.right_child ^ 0x80000000`). Update accumulators on hit [S1, unofficial].
   e. If not leaf: load children. Ray-sphere test for both. Set `left_descend`, `right_descend` [S1, unofficial].
   f. If neither: continue (pop next) [S1, unofficial].
   g. If one: set current to that child [S1, unofficial].
   h. If both: push farther to `S`, set current to nearer [S1, unofficial].
3. After loop: `o_t = min_t` [S1, unofficial].
4. Return `min_t < 10000.0` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_walk(ray_origin, ray_direction, bvh_nodes, bvh_leaves, svo_data, max_t=10000.0):
    stack = [0]
    min_t = float('inf')
    o_t = 0.0
    o_color = (0.0, 0.0, 0.0, 0.0)
    o_normal = (0.0, 0.0, 0.0)

    while True:
        if not stack:
            break
        idx = stack.pop()
        node = bvh_nodes[idx]

        is_leaf = (node.left_child & 0x80000000) != 0
        if is_leaf:
            for child_mask in [node.left_child, node.right_child]:
                leaf_idx = child_mask ^ 0x80000000
                hit_t, hit_color, hit_normal = trace_leaf(ray_origin, ray_direction, bvh_leaves[leaf_idx], svo_data, min_t)
                if hit_t < min_t:
                    min_t = hit_t
                    o_t = hit_t
                    o_color = hit_color
                    o_normal = hit_normal
        else:
            left_node = bvh_nodes[node.left_child]
            right_node = bvh_nodes[node.right_child]

            left_descend = ray_sphere_intersect(ray_origin, ray_direction, left_node.center, left_node.radius, min_t)
            right_descend = ray_sphere_intersect(ray_origin, ray_direction, right_node.center, right_node.radius, min_t)

            if not left_descend and not right_descend:
                continue
            elif left_descend and right_descend:
                dist_left = length(left_node.center - ray_origin)
                dist_right = length(right_node.center - ray_origin)
                if dist_left > dist_right:
                    if len(stack) < 64:
                        stack.append(node.left_child)
                    idx = node.right_child
                else:
                    if len(stack) < 64:
                        stack.append(node.right_child)
                    idx = node.left_child
            elif left_descend:
                idx = node.left_child
            else:
                idx = node.right_child

            stack.append(idx)

    o_t = min_t
    return min_t < max_t, o_t, o_color, o_normal

def trace_leaf(ray_origin, ray_direction, leaf, svo_data, min_t):
    inv = leaf.inv_model
    local_origin = transform_point(inv, ray_origin)
    local_direction = transform_vector(inv, ray_direction)
    return svo_march(local_origin, local_direction, svo_data, leaf.address, min_t)

def ray_sphere_intersect(origin, direction, center, radius, min_t):
    oc = origin - center
    b = dot(oc, direction)
    c = dot(oc, oc) - radius * radius
    disc = b * b - c
    if disc < 0:
        return False
    sqrt_disc = sqrt(disc)
    t_hit = -b - sqrt_disc
    if t_hit < 0:
        return False
    return t_hit < min_t

def svo_march(origin, direction, svo_data, root_address, min_t):
    # Standard SVO ray marcher [G]
    # Traverses SVO from root_address, accumulating color/normal until min_t or empty
    # Returns (t_hit, color, normal) or (inf, (0,0,0,0), (0,0,0))
    return (float('inf'), (0.0, 0.0, 0.0, 0.0), (0.0, 0.0, 0.0))

def transform_point(matrix, point):
    w = matrix[3][0]*point.x + matrix[3][1]*point.y + matrix[3][2]*point.z + matrix[3][3]
    x = (matrix[0][0]*point.x + matrix[0][1]*point.y + matrix[0][2]*point.z + matrix[0][3]) / w
    y = (matrix[1][0]*point.x + matrix[1][1]*point.y + matrix[1][2]*point.z + matrix[1][3]) / w
    z = (matrix[2][0]*point.x + matrix[2][1]*point.y + matrix[2][2]*point.z + matrix[2][3]) / w
    return (x, y, z)

def transform_vector(matrix, vector):
    x = matrix[0][0]*vector.x + matrix[0][1]*vector.y + matrix[0][2]*vector.z
    y = matrix[1][0]*vector.x + matrix[1][1]*vector.y + matrix[1][2]*vector.z
    z = matrix[2][0]*vector.x + matrix[2][1]*vector.y + matrix[2][2]*vector.z
    return (x, y, z)
```

**Edge Cases and Pitfalls:**
- Stack overflow: `BVH_STACK_SIZE` is fixed at 64. Deep BVHs may cause stack overflow if not handled [S1, unofficial].
- Precision: Ray-sphere intersection precision can affect culling accuracy.
- Leaf Index Masking: The leaf index is stored with the MSB set. It must be XORed with `0x80000000` to get the actual index [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Efficient for large, sparse datasets. Good balance between memory usage and traversal speed.
- **Limitations:** Stack size limitation. Complexity of implementation due to hybrid nature.

## Background
The hybrid BVH-SVO traversal approach is described in the unofficial source [S1, unofficial]. It is part of the `mxbrt/svo` project. The approach combines BVH and SVO techniques, which are common in volume rendering.

## Sources
[S1] mxbrt/svo, https://github.com/mxbrt/svo, fetched 2026-10-26 (unofficial)
