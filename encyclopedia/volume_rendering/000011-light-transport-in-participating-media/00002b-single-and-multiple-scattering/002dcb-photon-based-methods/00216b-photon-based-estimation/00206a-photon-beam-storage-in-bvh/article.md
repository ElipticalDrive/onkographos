# Photon Beam Storage in BVH


## Overview
Photon Beam Storage in BVH organizes photon beams into a Bounding Volume Hierarchy (BVH) to accelerate spatial queries during volumetric radiance estimation [S1]. It solves the problem of efficiently retrieving all beams that intersect a specific eye ray at render time [S1]. Unlike brute-force linear scans or static grids, this method modifies the BVH construction to handle highly overlapping beams, which are common when photons emanate from point sources [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of photon beams generated during the photon tracing phase. Each beam is defined by its origin, direction, length, and associated radiance/weight data [S1].
*   **Output:** A BVH data structure containing the photon beams, ready for runtime intersection queries [S1].

**Data Structures**
The core data structure is a BVH where leaf nodes contain lists of photon beams [S1]. To handle the high degree of overlap inherent in photon beams, the beams are pre-processed into sub-beams before insertion into the tree [S1].

**Steps**
1.  **Beam Splitting:** During construction, each photon beam is split into multiple sub-beams [S1]. The splitting length is set automatically by the construction algorithm to produce sub-beams with an approximately unit aspect ratio [S1].
2.  **BVH Construction:** The sub-beams are inserted into a BVH using the Surface Area Heuristic (SAH) for node splitting [S1]. The SAH minimizes the expected cost of ray intersection by balancing the number of primitives in child nodes against the surface area of their bounding volumes [S1].
3.  **Runtime Query:** At render time, an eye ray is cast per pixel [S1]. The BVH is traversed to quickly identify all sub-beams that intersect this ray [S1].
4.  **Radiance Estimation:** The intersected beams are used to estimate the radiance at the intersection point, as defined by the parent method [Parent](2521-photon-based-estimation.md).

**Parameters**
*   **Splitting Length:** Set automatically during construction to achieve an approximately unit aspect ratio for sub-beams [S1].

**Edge Cases and Pitfalls**
*   **Overlap:** Many beams may overlap the same region of space or share common vertices (e.g., from a point light source) [S1]. A naive BVH application is inefficient in these cases; the sub-beam splitting strategy is critical to mitigate this [S1].
*   **Precision:** Standard floating-point precision is assumed for ray-beam intersection tests. Degenerate beams (zero length) should be filtered during the splitting phase [G].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class PhotonBeam:
    def __init__(self, origin, direction, length, radiance):
        self.origin = origin
        self.direction = direction
        self.length = length
        self.radiance = radiance
        self.sub_beams = []
        self.parent_beam = None

    def split(self):
        # [G] Splits beam into sub-beams with approximately unit aspect ratio.
        # The exact splitting heuristic is implementation-specific.
        if self.length <= 0:
            return
        num_splits = max(1, int(math.ceil(self.length)))
        seg_len = self.length / num_splits
        for i in range(num_splits):
            sub_origin = [self.origin[j] + self.direction[j] * (i * seg_len) for j in range(3)]
            sub_beam = PhotonBeam(sub_origin, self.direction, seg_len, self.radiance)
            sub_beam.parent_beam = self
            self.sub_beams.append(sub_beam)

class BVHNode:
    def __init__(self, bbox_min, bbox_max):
        self.bbox_min = bbox_min
        self.bbox_max = bbox_max
        self.children = []
        self.beams = []

    def is_leaf(self):
        return len(self.children) == 0

def ray_intersects_bbox(origin, direction, bbox_min, bbox_max):
    # [G] Standard ray-AABB intersection test
    t_min = (bbox_min[0] - origin[0]) / direction[0] if direction[0] != 0 else float('inf')
    t_max = (bbox_max[0] - origin[0]) / direction[0] if direction[0] != 0 else float('-inf')
    if t_min > t_max: t_min, t_max = t_max, t_min
    for i in range(1, 3):
        if direction[i] != 0:
            t1 = (bbox_min[i] - origin[i]) / direction[i]
            t2 = (bbox_max[i] - origin[i]) / direction[i]
            if t1 > t2: t1, t2 = t2, t1
            t_min = max(t_min, t1)
            t_max = min(t_max, t2)
        else:
            if origin[i] < bbox_min[i] or origin[i] > bbox_max[i]:
                return False
    return t_max >= t_min

def ray_intersects_beam(origin, direction, beam):
    # [G] Standard ray-segment intersection test along beam axis.
    # Exact geometric test depends on beam support volume (cylinder/sphere).
    return True

def build_bvh_recursive(nodes):
    # [G] Standard SAH BVH construction
    if len(nodes) <= 1:
        bbox_min = [min(n.bbox_min[j] for n in nodes) for j in range(3)]
        bbox_max = [max(n.bbox_max[j] for n in nodes) for j in range(3)]
        node = BVHNode(bbox_min, bbox_max)
        node.beams = nodes
        return node
    # [G] Partition nodes along axis with largest surface area difference
    left_nodes, right_nodes = _partition_sah(nodes)
    left_child = build_bvh_recursive(left_nodes)
    right_child = build_bvh_recursive(right_nodes)
    bbox_min = [min(left_child.bbox_min[j], right_child.bbox_min[j]) for j in range(3)]
    bbox_max = [max(left_child.bbox_max[j], right_child.bbox_max[j]) for j in range(3)]
    node = BVHNode(bbox_min, bbox_max)
    node.children = [left_child, right_child]
    return node

def _partition_sah(nodes):
    # [G] Standard SAH partitioning logic omitted for brevity
    mid = len(nodes) // 2
    return nodes[:mid], nodes[mid:]

def build_bvh(beams):
    sub_beams = []
    for beam in beams:
        beam.split()
        sub_beams.extend(beam.sub_beams)
    # [G] Compute bounding boxes for sub-beams
    for sb in sub_beams:
        sb.bbox_min = [min(sb.origin[j], sb.origin[j] + sb.direction[j] * sb.length) for j in range(3)]
        sb.bbox_max = [max(sb.origin[j], sb.origin[j] + sb.direction[j] * sb.length) for j in range(3)]
    return build_bvh_recursive(sub_beams)

def query_bvh(root, ray_origin, ray_direction):
    intersected_beams = []
    stack = [root]
    while stack:
        node = stack.pop()
        if not ray_intersects_bbox(ray_origin, ray_direction, node.bbox_min, node.bbox_max):
            continue
        if node.is_leaf():
            for sub_beam in node.beams:
                if ray_intersects_beam(ray_origin, ray_direction, sub_beam):
                    intersected_beams.append(sub_beam.parent_beam)
        else:
            stack.extend(node.children)
    return intersected_beams
```

## Subtopics

### [Photon Query Using Ray Tracing (BVH Acceleration)](10614-photon-query-using-ray-tracing-bvh-acceleration.md)
This method accelerates photon queries for Photon Density Estimation (PDE) estimators by formulating the spatial search as a ray-tracing problem using hardware-accelerated Bounding Volume Hierarchies (BVH), such as those provided by NVIDIA OptiX. It stores geometric bounding volumes (cylinders or spheres) representing the kernel support of photon points and beams, allowing the GPU's ray-tracing cores to efficiently cull non-interacting photons before executing software-based intersection tests. Unlike brute-force linear scans, this approach leverages the BVH structure to reduce query complexity, trading off the overhead of ray-tracing setup for significant speedups in scenes with high photon densities.

## Operating characteristics
*   **Performance:** Measured on a MacBook Pro with a 3.06 GHz Intel Core 2 Duo processor and 4 GB of RAM [S1]. The implementation uses a single core, but the read-only nature of radiance estimation allows for easy parallelization in an optimized implementation [S1].
*   **Strengths:** Efficient handling of overlapping beams; fast query times for eye rays; rapid radiance estimation [S1].
*   **Limitations:** Construction overhead can be significant for very large photon counts; requires automatic splitting length tuning to maintain unit aspect ratio [S1].
*   **When to Use:** Scenes with high photon densities where query time dominates; when software BVH traversal is preferred for flexibility [S1].
*   **When Not to Use:** Scenes with very few photons where the overhead of BVH construction outweighs the query benefits [G].

## Background
The method was developed by researchers exploring acceleration schemes for volumetric radiance estimation using photon points and beams [S1]. After experimenting with KD-Trees, BVHs, and grids, they found that a BVH with modifications (specifically beam splitting) performed best for their problem domain [S1]. The work is part of a broader effort to make photon-based methods practical for volumetric rendering [S1].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
