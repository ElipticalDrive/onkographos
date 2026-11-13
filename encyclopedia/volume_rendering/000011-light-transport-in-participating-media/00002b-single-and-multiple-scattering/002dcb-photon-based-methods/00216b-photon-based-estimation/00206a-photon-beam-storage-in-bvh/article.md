# Photon Beam Storage in BVH

Also called: BVH-based Photon Beam Storage

## Overview
Photon Beam Storage in BVH accelerates the gathering phase of photon-based radiance estimation by organizing photon beams into a Bounding Volume Hierarchy (BVH) [S1]. Unlike KD-Trees or grid-based storage which may struggle with the high overlap of volumetric photon paths, this method uses a BVH constructed with the Surface Area Heuristic (SAH) to efficiently cull non-interacting beams during camera ray intersection [S1]. It sets itself apart from sibling methods like [Volumetric Photon Mapping](2190-volumetric-photon-mapping.md) by explicitly handling beam geometry and overlap through sub-beam splitting, enabling rapid retrieval of all beams intersected by an eye ray [S1].

## How it works
**Inputs:** A set of photon beams generated during the photon tracing phase. Each beam represents a segment of a photon path through a participating medium, defined by its start and end points, direction, and associated radiance/power information.
**Outputs:** A BVH data structure containing the photon beams, ready for runtime intersection queries.

**Data Structures:**
The core data structure is a BVH where each leaf node contains one or more photon beams. To handle the inefficiency of naive BVH application caused by overlapping beams (e.g., multiple beams emanating from a point light source), the storage structure includes:
*   **Sub-beams:** Original photon beams are split into smaller segments.
*   **Bounding Volumes:** Each sub-beam is enclosed in a bounding volume (typically an axis-aligned bounding box or sphere) for BVH traversal.

**Construction Algorithm:**
1.  **Beam Splitting:** During BVH construction, each photon beam is split into sub-beams. The splitting length is determined automatically by the construction algorithm to ensure that each sub-beam has an approximately unit aspect ratio [S1]. This improves the partitioning quality of the BVH.
2.  **BVH Construction:** The sub-beams are inserted into a BVH using the Surface Area Heuristic (SAH) for node splitting [S1, MacDonald and Booth 1990]. The SAH minimizes the expected cost of ray traversal by balancing the number of primitives in child nodes against the surface area of their bounding volumes.

**Runtime Query:**
1.  **Ray Intersection:** For each pixel, a camera ray is cast.
2.  **BVH Traversal:** The camera ray is intersected with the BVH.
3.  **Beam Retrieval:** All leaf nodes (and thus all sub-beams) intersected by the camera ray are collected.
4.  **Radiance Estimation:** The radiance is estimated using the gathered beams, applying the standard photon density estimation formula from the parent [Parent](2521-photon-based-estimation.md), using the power and phase function of the intersected beams.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class PhotonBeam:
    def __init__(self, start_point, end_point, power, direction):
        self.start = start_point
        self.end = end_point
        self.power = power
        self.direction = direction
        self.bounding_volume = compute_bounding_volume(start_point, end_point)

class BVHNode:
    def __init__(self, bounding_volume, children=None, beams=None):
        self.bounding_volume = bounding_volume
        self.children = children if children else []
        self.beams = beams if beams else []

def split_beam(beam, target_aspect_ratio=1.0):
    """
    Splits a photon beam into sub-beams with approximately unit aspect ratio.
    Implementation detail: [G] Calculate beam length, determine number of splits
    based on target aspect ratio, and generate new PhotonBeam instances.
    """
    length = distance(beam.start, beam.end)
    # Logic to determine split count and generate sub-beams
    # Not fully specified in source, but implied to be automatic
    return [sub_beam for sub_beam in generate_sub_beams(beam, target_aspect_ratio)]

def build_bvh(beams):
    """
    Constructs a BVH using SAH from a list of photon beams.
    """
    # Step 1: Split beams into sub-beams
    sub_beams = []
    for beam in beams:
        sub_beams.extend(split_beam(beam))
    
    # Step 2: Build BVH using SAH
    # Implementation detail: [G] Recursive function that partitions sub-beams
    # based on SAH cost metric until a leaf node is reached.
    return recursive_bvh_build(sub_beams)

def query_bvh(bvh_root, camera_ray):
    """
    Intersects a camera ray with the BVH to retrieve all intersected beams.
    """
    intersected_beams = []
    
    def traverse(node):
        if not ray_intersects_bounding_volume(camera_ray, node.bounding_volume):
            return
        
        if node.children:
            for child in node.children:
                traverse(child)
        else:
            # Leaf node: collect beams
            for beam in node.beams:
                if ray_intersects_beam(camera_ray, beam):
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
*   **Time Complexity:** BVH construction is $O(N \log N)$ on average, where $N$ is the number of sub-beams. Runtime query complexity is $O(\log N)$ for traversal, with intersection tests performed only on intersected nodes [S1].
*   **Memory Complexity:** $O(N)$, where $N$ is the number of sub-beams. Splitting beams increases memory usage compared to storing original beams.
*   **Performance:** Measured on a MacBook Pro with a 3.06 GHz Intel Core 2 Duo processor and 4 GB of RAM. The implementation uses only one core, but radiance estimation is read-only and parallelizable [S1].
*   **Strengths:** Efficient handling of overlapping beams; fast runtime retrieval of intersected beams; effective partitioning via SAH and sub-beam splitting [S1].
*   **Limitations:** Increased memory usage due to beam splitting; construction overhead from splitting and BVH building; single-core implementation in the reference study [S1].
*   **When to use:** When photon beams have significant overlap and efficient spatial querying is required.
*   **When not to use:** When memory is constrained and beam splitting overhead is prohibitive; when hardware-accelerated ray tracing is available (consider sibling method).

## Background
This method was explored by the authors of [S1] as an acceleration scheme for volumetric radiance estimation using photon points and beams. After experimenting with KD-Trees, BVHs, and grids, they found that a BVH with modifications (sub-beam splitting) performed well for their problem domain [S1]. The BVH construction uses the Surface Area Heuristic [MacDonald and Booth 1990].

## Sources
[S1] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
