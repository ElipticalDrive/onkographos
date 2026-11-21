# Photon Beam Storage in BVH


## Overview
Photon Beam Storage in BVH accelerates the gathering phase of photon-based radiance estimation by organizing photon beams into a Bounding Volume Hierarchy (BVH) [S1]. Unlike KD-Trees or grid-based storage, which struggle with the high overlap of volumetric photon paths, this method uses a BVH constructed with the Surface Area Heuristic (SAH) to efficiently cull non-interacting beams during camera ray intersection [S1]. It sets itself apart from sibling methods like [Volumetric Photon Mapping](2190-volumetric-photon-mapping.md) by explicitly handling beam geometry and overlap through sub-beam splitting, enabling rapid retrieval of all beams intersected by an eye ray [S1].

## How it works
**Inputs:** A set of photon beams generated during the photon tracing phase [S1].
**Outputs:** A BVH data structure containing the photon beams, ready for runtime intersection queries [S1].

**Data Structures:**
The core data structure is a BVH where each leaf node contains one or more photon beams [S1]. To handle the inefficiency of naive BVH application caused by overlapping beams (e.g., multiple beams emanating from a point light source), the storage structure splits original photon beams into smaller segments called sub-beams [S1]. Each sub-beam is enclosed in a bounding volume for BVH traversal [S1].

**Construction Algorithm:**
1.  **Beam Splitting:** During BVH construction, each photon beam is split into sub-beams [S1]. The splitting length is determined automatically by the construction algorithm to ensure that each sub-beam has an approximately unit aspect ratio [S1]. This improves the partitioning quality of the BVH [S1].
2.  **BVH Construction:** The sub-beams are inserted into a BVH using the Surface Area Heuristic (SAH) for node splitting [S1, MacDonald and Booth 1990].

**Runtime Query:**
1.  **Ray Intersection:** For each pixel, a camera ray is cast.
2.  **BVH Traversal:** The camera ray is intersected with the BVH [S1].
3.  **Beam Retrieval:** All leaf nodes (and thus all sub-beams) intersected by the camera ray are collected [S1].
4.  **Radiance Estimation:** The radiance is estimated using the gathered beams, applying the standard photon density estimation formula from the parent [Parent](2521-photon-based-estimation.md), using the power and phase function of the intersected beams.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __sub__(self, o): return Vec3(self.x-o.x, self.y-o.y, self.z-o.z)
    def dot(self, o): return self.x*o.x + self.y*o.y + self.z*o.z
    def length(self): return math.sqrt(self.x**2 + self.y**2 + self.z**2)

class PhotonBeam:
    def __init__(self, start, end, power, direction):
        self.start = start
        self.end = end
        self.power = power
        self.direction = direction
        self.bounding_volume = self._compute_aabb()

    def _compute_aabb(self):
        return {
            'min': Vec3(min(self.start.x, self.end.x), min(self.start.y, self.end.y), min(self.start.z, self.end.z)),
            'max': Vec3(max(self.start.x, self.end.x), max(self.start.y, self.end.y), max(self.start.z, self.end.z))
        }

class BVHNode:
    def __init__(self, bounding_volume, children=None, beams=None):
        self.bounding_volume = bounding_volume
        self.children = children if children else []
        self.beams = beams if beams else []

def split_beam(beam, target_aspect_ratio=1.0):
    diff = beam.start - beam.end
    length = diff.length()
    num_splits = max(1, int(length / target_aspect_ratio))
    sub_beams = []
    for i in range(num_splits):
        t0 = i / num_splits
        t1 = (i + 1) / num_splits
        s = Vec3(beam.start.x + (beam.end.x - beam.start.x)*t0,
                 beam.start.y + (beam.end.y - beam.start.y)*t0,
                 beam.start.z + (beam.end.z - beam.start.z)*t0)
        e = Vec3(beam.start.x + (beam.end.x - beam.start.x)*t1,
                 beam.start.y + (beam.end.y - beam.start.y)*t1,
                 beam.start.z + (beam.end.z - beam.start.z)*t1)
        sub_beams.append(PhotonBeam(s, e, beam.power, beam.direction))
    return sub_beams

def build_bvh(beams, max_depth=10):
    if not beams or max_depth == 0:
        bv = beams[0].bounding_volume if beams else None
        return BVHNode(bounding_volume=bv, beams=beams)

    min_x = min(b.bounding_volume['min'].x for b in beams)
    max_x = max(b.bounding_volume['max'].x for b in beams)
    min_y = min(b.bounding_volume['min'].y for b in beams)
    max_y = max(b.bounding_volume['max'].y for b in beams)
    min_z = min(b.bounding_volume['min'].z for b in beams)
    max_z = max(b.bounding_volume['max'].z for b in beams)
    extents = (max_x - min_x, max_y - min_y, max_z - min_z)
    axis = extents.index(max(extents))

    centroids = []
    for b in beams:
        cx = (b.bounding_volume['min'].x + b.bounding_volume['max'].x) / 2
        cy = (b.bounding_volume['min'].y + b.bounding_volume['max'].y) / 2
        cz = (b.bounding_volume['min'].z + b.bounding_volume['max'].z) / 2
        centroids.append(((cx, cy, cz)[axis], b))
    centroids.sort(key=lambda x: x[0])
    mid = len(centroids) // 2
    left_beams = [c[1] for c in centroids[:mid]]
    right_beams = [c[1] for c in centroids[mid:]]

    left_node = build_bvh(left_beams, max_depth - 1)
    right_node = build_bvh(right_beams, max_depth - 1)

    merged_min = Vec3(min(left_node.bounding_volume['min'].x, right_node.bounding_volume['min'].x),
                      min(left_node.bounding_volume['min'].y, right_node.bounding_volume['min'].y),
                      min(left_node.bounding_volume['min'].z, right_node.bounding_volume['min'].z))
    merged_max = Vec3(max(left_node.bounding_volume['max'].x, right_node.bounding_volume['max'].x),
                      max(left_node.bounding_volume['max'].y, right_node.bounding_volume['max'].y),
                      max(left_node.bounding_volume['max'].z, right_node.bounding_volume['max'].z))
    merged_bv = {'min': merged_min, 'max': merged_max}

    return BVHNode(bounding_volume=merged_bv, children=[left_node, right_node])

def ray_intersects_aabb(ray_origin, ray_direction, aabb):
    t_min = (aabb['min'].x - ray_origin.x) / ray_direction.x if ray_direction.x != 0 else float('-inf')
    t_max = (aabb['max'].x - ray_origin.x) / ray_direction.x if ray_direction.x != 0 else float('inf')
    if t_min > t_max: t_min, t_max = t_max, t_min

    ty_min = (aabb['min'].y - ray_origin.y) / ray_direction.y if ray_direction.y != 0 else float('-inf')
    ty_max = (aabb['max'].y - ray_origin.y) / ray_direction.y if ray_direction.y != 0 else float('inf')
    if ty_min > ty_max: ty_min, ty_max = ty_max, ty_min

    if (t_min > ty_max) or (ty_min > t_max): return False
    t_min = max(t_min, ty_min)
    t_max = min(t_max, ty_max)

    tz_min = (aabb['min'].z - ray_origin.z) / ray_direction.z if ray_direction.z != 0 else float('-inf')
    tz_max = (aabb['max'].z - ray_origin.z) / ray_direction.z if ray_direction.z != 0 else float('inf')
    if tz_min > tz_max: tz_min, tz_max = tz_max, tz_min

    if (t_min > tz_max) or (tz_min > t_max): return False
    return True

def ray_intersects_beam(ray_origin, ray_direction, beam):
    diff = beam.end - beam.start
    denom = ray_direction.dot(diff)
    if abs(denom) < 1e-9: return False
    t = ((ray_origin - beam.start).dot(diff)) / denom
    if t < 0 or t > 1: return False
    closest = Vec3(ray_origin.x + ray_direction.x * t,
                   ray_origin.y + ray_direction.y * t,
                   ray_origin.z + ray_direction.z * t)
    return closest.length() < 1e-6

def query_bvh(bvh_root, ray_origin, ray_direction):
    intersected_beams = []
    def traverse(node):
        if not ray_intersects_aabb(ray_origin, ray_direction, node.bounding_volume):
            return
        if node.children:
            for child in node.children:
                traverse(child)
        else:
            for beam in node.beams:
                if ray_intersects_beam(ray_origin, ray_direction, beam):
                    intersected_beams.append(beam)
    traverse(bvh_root)
    return intersected_beams
```

**Parameters:**
*   **Aspect Ratio:** Target aspect ratio for sub-beams is approximately 1.0 [S1].
*   **Heuristic:** Surface Area Heuristic (SAH) is used for BVH construction [S1].

**Edge Cases:**
*   **High Overlap:** Naive BVH application is inefficient when many beams overlap the same region (e.g., from a point light). Sub-beam splitting mitigates this [S1].
*   **Precision:** Not given in the sources.

## Subtopics

### [Photon Query Using Ray Tracing (BVH Acceleration)](10614-photon-query-using-ray-tracing-bvh-acceleration.md)
Accelerates photon queries by formulating spatial search as a ray-tracing problem using hardware-accelerated BVHs (e.g., NVIDIA OptiX). It stores geometric bounding volumes (cylinders/spheres) for kernel support, leveraging GPU ray-tracing cores for culling. Choose this when hardware acceleration is available and photon densities are high.

## Operating characteristics
*   **Performance:** Measured on a MacBook Pro with a 3.06 GHz Intel Core 2 Duo processor and 4 GB of RAM [S1]. The reference implementation uses only one core, but radiance estimation is a read-only operation and can be parallelized in an optimized implementation [S1]. Test scenes include the Cornell box, bumpy sphere, and sphere caustic at 512 pixels across their larger dimension [S1].
*   **Strengths:** Efficient handling of overlapping beams; fast runtime retrieval of intersected beams; effective partitioning via SAH and sub-beam splitting [S1].
*   **Limitations:** Construction overhead from splitting and BVH building; the reference study uses a single-core implementation [S1].

## Background
This method was explored by the authors of [S1] as an acceleration scheme for volumetric radiance estimation using photon points and beams. After experimenting with KD-Trees, BVHs, and grids, they found that a BVH with modifications (sub-beam splitting) performed well for their problem domain [S1]. The BVH construction uses the Surface Area Heuristic [MacDonald and Booth 1990].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
