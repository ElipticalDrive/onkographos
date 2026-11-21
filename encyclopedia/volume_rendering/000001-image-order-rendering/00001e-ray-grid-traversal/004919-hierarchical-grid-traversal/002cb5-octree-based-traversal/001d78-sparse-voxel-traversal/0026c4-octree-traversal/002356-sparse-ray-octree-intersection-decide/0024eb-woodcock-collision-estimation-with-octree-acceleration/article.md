# Woodcock Collision Estimation with Octree Acceleration

Also called: castRayWoodcockFast

## Overview
An optimized ray marching method that combines Woodcock collision estimation with an octree traversal to skip empty regions and nodes that do not intersect the ray's current segment, reducing the number of volume samples [S1, unofficial]. It differs from standard ray marching by using the octree to cull non-intersecting nodes, and from its parent [Sparse Ray-Octree Intersection (DECIDE)](3006-sparse-ray-octree-intersection-decide.md) by employing stochastic collision estimation within leaf nodes rather than deterministic traversal [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray with origin $\mathbf{o}$, direction $\mathbf{d}$, and valid interval $[ray.minT, ray.maxT]$ [S1, unofficial]. An octree representing the volume [S1, unofficial]. A random number generator `sampler` [S1, unofficial]. Configuration `settings` containing `stepSize` and `densityScale` [S1, unofficial].
*   **Output:** Accumulated color and opacity along the ray segment [S1, unofficial].

**Data Structures:**
*   **State Struct:** A stack element containing:
    *   `Octree const* node`: Pointer to the current octree node [S1, unofficial].
    *   `float minT`: Entry distance of the ray into the node's bounding box, clamped to the ray's current valid range [S1, unofficial].
    *   `float maxT`: Exit distance of the ray from the node's bounding box, clamped to the ray's current valid range [S1, unofficial].
    *   `int nodesMask`: Bitmask tracking which of the 8 children of the current node have already been visited or checked for intersection [S1, unofficial].

**Steps:**
1.  **Initialization:**
    *   Create an empty stack `st` [S1, unofficial].
    *   Compute the bounding box intersection range for the root node [S1, unofficial].
    *   If the ray intersects the root bounding box, push the root node with the intersection range onto `st` [S1, unofficial].
    *   Initialize `ray.minT` with a random offset: $ray.minT \leftarrow ray.minT + (-\ln(\text{sampler.nextFloat()})) \times \text{settings.stepSize}$ [S1, unofficial].
2.  **Traversal Loop:**
    *   While `st` is not empty and $ray.minT \le ray.maxT$:
        1.  Pop the top state from `st` to get `node`, `minT`, and `maxT` [S1, unofficial].
        2.  Clamp `minT` and `maxT` to the node's bounding box intersection and the current $ray.minT$ [S1, unofficial].
        3.  If $minT > maxT$, the ray has passed this node; pop the stack and continue to the next iteration [S1, unofficial].
        4.  If `node` is not a leaf:
            *   Determine the child ID to traverse first based on the ray's position relative to the node's center: $id = (sideX \ll 2) \mid (sideY \ll 1) \mid (sideZ)$, where $sideX, sideY, sideZ$ indicate which half of the node the ray occupies [S1, unofficial].
            *   If $id$ is already in `nodesMask`, skip it [S1, unofficial].
            *   Compute the bounding box intersection for the child node [S1, unofficial].
            *   If valid, push the child to `st` with its intersection range and break the inner loop to process it immediately [S1, unofficial].
        5.  If `node` is a leaf:
            *   Calculate `invMaxOpacity` as the inverse of the maximum opacity in the node, defaulting to $1.0$ if not pre-computed [S1, unofficial].
            *   While $minT \le maxT$:
                *   Sample the volume and evaluate the transfer function to obtain color $C$ and opacity $\alpha$ (stored as `out.w`) [S1, unofficial].
                *   Generate a random number $u = \text{sampler.nextFloat()}$ [S1, unofficial].
                *   Test collision: if $u < (\alpha \times \text{settings.densityScale}) \times \text{invMaxOpacity} \times \text{settings.stepSize}$, accumulate $C$ and $\alpha$ into the ray's result [S1, unofficial].
                *   Advance $minT$: $minT \leftarrow minT + (-\ln(\text{sampler.nextFloat()})) \times \text{invMaxOpacity} \times \text{settings.stepSize}$ [S1, unofficial].
            *   After the leaf loop, update $ray.minT \leftarrow minT$ and pop the stack to process the next node [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def castRayWoodcockFast(ray, octree, sampler, settings):
    stack = []
    root_bbox = octree.root.bounding_box
    if not root_bbox.intersects(ray):
        return None
    root_min_t, root_max_t = root_bbox.get_intersection(ray)
    stack.append({'node': octree.root, 'minT': root_min_t, 'maxT': root_max_t, 'nodesMask': 0})
    ray.minT += (-math.log(sampler.next_float())) * settings.step_size

    while stack and ray.minT <= ray.maxT:
        state = stack.pop()
        node = state['node']
        min_t = state['minT']
        max_t = state['maxT']

        node_bbox = node.bounding_box
        if node_bbox.intersects(ray):
            n_min_t, n_max_t = node_bbox.get_intersection(ray)
            min_t = max(min_t, n_min_t, ray.minT)
            max_t = min(max_t, n_max_t)
        else:
            min_t = max_t + 1.0

        if min_t > max_t:
            continue

        if not node.is_leaf:
            side_x = 1 if ray.origin.x >= node.center.x else 0
            side_y = 1 if ray.origin.y >= node.center.y else 0
            side_z = 1 if ray.origin.z >= node.center.z else 0
            child_id = (side_x << 2) | (side_y << 1) | side_z

            if not (state['nodesMask'] & (1 << child_id)):
                state['nodesMask'] |= (1 << child_id)
                child = node.children[child_id]
                child_bbox = child.bounding_box
                if child_bbox.intersects(ray):
                    c_min_t, c_max_t = child_bbox.get_intersection(ray)
                    stack.append({'node': child, 'minT': c_min_t, 'maxT': c_max_t, 'nodesMask': 0})
                    break
        else:
            inv_max_opacity = 1.0 / node.max_opacity if node.max_opacity > 0 else 1.0
            while min_t <= max_t:
                pos = ray.origin + ray.direction * min_t
                out = sample_volume(pos)
                u = sampler.next_float()
                collision_prob = (out.w * settings.density_scale) * inv_max_opacity * settings.step_size
                if u < collision_prob:
                    accumulate_result(out.color, out.w)
                min_t += (-math.log(sampler.next_float())) * inv_max_opacity * settings.step_size
            ray.minT = min_t
```

**Parameters:**
*   `settings.stepSize`: Base step size for sampling. Typical value: Not given in the sources.
*   `settings.densityScale`: Scaling factor for density. Typical value: Not given in the sources.
*   `invMaxOpacity`: Inverse of maximum opacity in the node. Default: $1.0$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Ray Passed Node:** If $minT > maxT$ after clamping, the ray has exited the node's bounds; the algorithm must pop the stack and continue [S1, unofficial].
*   **Precision:** Floating point precision in logarithmic calculations for step advancement may cause jitter or missed collisions if not handled carefully [G].
*   **Empty Nodes:** The octree traversal naturally skips empty nodes if they are not pushed to the stack or if their intersection is invalid [S1, unofficial].

## Subtopics

### Woodcock Collision Estimation with Octree and Adaptive Step Size (Fast2)
A variant of the fast Woodcock method that uses a fixed step size within leaf nodes, accumulating opacity, and compares the accumulated opacity against a random threshold $S$ derived from the exponential distribution [S1, unofficial]. It computes `maxOpacity` per leaf node to adapt the step size [S1, unofficial]. Initialization sets $S = -\ln(\text{sampler.nextFloat()})) / \text{settings.densityScale}$ and `sum = 0.0f` [S1, unofficial]. When skipping a node (empty or passed), `needsJitter` is set to true to ensure the next entry into a volume region is jittered [S1, unofficial]. For leaf nodes, `maxOpacity` is computed by iterating through `settings.maxOpacity` for masks present in the node [S1, unofficial]. The `stepSize` is calculated using linear interpolation between $1.0$ and `settings.stepSize` based on `settings.densityScale * maxOpacity` [S1, unofficial]. If `needsJitter` is true, `minT` is jittered by `sampler.nextFloat() * stepSize - dT` and `needsJitter` is set to false [S1, unofficial]. The loop runs while `minT <= maxT`, sampling the volume and accumulating opacity: `sum += settings.densityScale * out.w * stepSize` [S1, unofficial]. If `sum >= S`, the intersection is recorded and the function returns true [S1, unofficial]. Finally, `minT` is advanced by `stepSize` [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the density of the volume and the octree depth; fewer samples are taken in empty regions due to octree culling [S1, unofficial].
*   **Memory Complexity:** Stack depth is proportional to the octree depth [S1, unofficial].
*   **Strengths:** Efficiently skips empty space and reduces sampling rate in low-density regions via Woodcock estimation [S1, unofficial].
*   **Limitations:** Stochastic nature introduces noise and requires careful tuning of `stepSize` and `densityScale` [S1, unofficial].
*   **When to use:** Sparse volumes with large empty regions [S1, unofficial].
*   **When not to use:** Dense volumes where deterministic marching might be more efficient or less noisy [S1, unofficial].

## Background
Defined in the PathTracer repository by thewildnath [S1, unofficial]. It adapts Woodcock collision estimation (delta tracking) for use with hierarchical voxel structures (octrees) to accelerate volume rendering [S1, unofficial].

## Sources
[S1] thewildnath/PathTracer, https://github.com/thewildnath/PathTracer, fetched 2026-10-26 (unofficial)
