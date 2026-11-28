# Watertight BVH Box Traversal Fix

Also called: BVH Box Traversal Precision Fix, Robust BVH Ray Traversal with ULP Expansion, Slab Test Zero-Inversion Fix, Treat BVH boxes as closed sets in ray traversal

## Overview
This method fixes a robustness bug in shared slab tests where a ray with a zero direction component lying exactly in the plane of a box face misses the box on one side [S1]. It ensures that grazing rays, such as orthographic camera rays landing exactly on a bound, correctly intersect triangles whose edges they touch, by treating BVH boxes as closed sets [S1]. Unlike standard slab tests which may compute an empty slab for $t > 0$ due to zero-inversion artifacts, this fix guarantees consistent intersection decisions for axis-parallel rays on face planes [S1].

## How it works
The method modifies the bounding volume hierarchy (BVH) construction phase to expand all node and instance bounding boxes by one unit in the last place (ULP) outward [S1]. This expansion ensures that rays with zero direction components lying exactly on a box face plane are strictly inside the slab, allowing the subsequent watertight triangle test to decide the intersection consistently [S1].

**Inputs and Outputs:**
- **Input:** Scene geometry and existing BVH structure.
- **Output:** A BVH with expanded bounding boxes.

**Data Structures:**
- **BVH Node/Instance Box:** Axis-Aligned Bounding Box (AABB) defined by minimum $(x_{min}, y_{min}, z_{min})$ and maximum $(x_{max}, y_{max}, z_{max})$ coordinates [S1].
- **ULP Expansion:** The expansion is applied to each coordinate of the AABB [S1].

**Steps:**
1. During BVH construction (`SceneAccel.build`), for every node and instance bounding box:
   a. Calculate the ULP of the box's minimum and maximum coordinates [S1].
   b. Expand the minimum coordinates by subtracting one ULP: $x_{min} \leftarrow x_{min} - \text{ulp}(x_{min})$, $y_{min} \leftarrow y_{min} - \text{ulp}(y_{min})$, $z_{min} \leftarrow z_{min} - \text{ulp}(z_{min})$ [S1].
   c. Expand the maximum coordinates by adding one ULP: $x_{max} \leftarrow x_{max} + \text{ulp}(x_{max})$, $y_{max} \leftarrow y_{max} + \text{ulp}(y_{max})$, $z_{max} \leftarrow z_{max} + \text{ulp}(z_{max})$ [S1].
2. Proceed with standard ray traversal using the expanded boxes [S1].

**Formula:**
The bug occurs when the slab test computes $(hi - o) * inv = 0 * inv = 0$, making the slab appear empty for $t > 0$ [S1]. The fix ensures the ray origin $o$ is strictly inside the expanded slab, so $(hi - o) * inv > 0$ for the relevant side [S1].

**Parameters:**
- **Expansion Amount:** One ULP (Unit in the Last Place) [S1].

**Edge Cases and Pitfalls:**
- **Zero Direction Components:** Rays with zero direction components lying exactly in the plane of a box face previously missed the box on one side [S1].
- **Grazing Rays:** Straight-down rays along the side of a cube missed the top face's edge that it passes through [S1].
- **Orthographic Camera Rays:** Rays landing exactly on a bound previously skipped triangles whose edges they touched [S1].
- **Precision:** The fix relies on floating-point precision; the ULP expansion must be calculated correctly for the specific coordinate values [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def expand_box_by_ulp(box):
    """
    Expands an AABB by one ULP in all directions.
    box: tuple of (min_x, min_y, min_z, max_x, max_y, max_z)
    """
    min_x, min_y, min_z, max_x, max_y, max_z = box
    
    # Calculate ULP for each coordinate
    # Note: ulp(x) is the distance between x and the next representable float
    ulp_min_x = ulp(min_x)
    ulp_min_y = ulp(min_y)
    ulp_min_z = ulp(min_z)
    ulp_max_x = ulp(max_x)
    ulp_max_y = ulp(max_y)
    ulp_max_z = ulp(max_z)
    
    # Expand the box
    new_min_x = min_x - ulp_min_x
    new_min_y = min_y - ulp_min_y
    new_min_z = min_z - ulp_min_z
    new_max_x = max_x + ulp_max_x
    new_max_y = max_y + ulp_max_y
    new_max_z = max_z + ulp_max_z
    
    return (new_min_x, new_min_y, new_min_z, new_max_x, new_max_y, new_max_z)

def build_scene_accel(scene):
    """
    Builds the scene acceleration structure with watertight BVH boxes.
    """
    bvh = BVH()
    for node in scene.nodes:
        # Get original bounding box
        original_box = node.get_bounding_box()
        # Expand by one ULP
        expanded_box = expand_box_by_ulp(original_box)
        # Set the expanded box to the node
        node.set_bounding_box(expanded_box)
        bvh.add_node(node)
    return bvh
```

## Operating characteristics
- **Time Complexity:** BVH construction time increases slightly due to ULP calculation and box expansion, but traversal cost is unchanged [S1].
- **Memory Complexity:** No additional memory is required for the BVH structure itself, as the expansion is applied to existing box coordinates [S1].
- **Performance:** The fix does not impact traversal performance; the GPU path tracer reads the same boxes [S1].
- **Strengths:** Fixes robustness issues with grazing rays and axis-parallel rays on face planes without changing golden images or reference outputs for other rays [S1].
- **Limitations:** Only affects rays with zero direction components lying exactly on box face planes; no other rays are affected [S1].

## Background
This fix was introduced in the Aether3D project (commit b43ebc2) to address a robustness bug in the shared slab test [S1]. It was developed to ensure that watertight triangle tests can correctly decide intersections for grazing rays, which previously missed boxes due to precision errors in the slab test [S1]. The fix is part of Stage 10 of the Aether3D project, which focuses on scene intelligence and spatial queries [S1].

## Sources
[S1] Stage 10: Scene intelligence and spatial queries (v0.10.0), https://github.com/Murisixz/Aether3D/pull/11, fetched 2026-10-29
