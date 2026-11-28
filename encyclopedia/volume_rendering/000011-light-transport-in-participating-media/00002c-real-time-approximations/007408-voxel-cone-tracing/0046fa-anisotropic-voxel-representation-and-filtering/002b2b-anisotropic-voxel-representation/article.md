# Anisotropic Voxel Representation

Also called: Anisotropic MIP-mapping, Directional Voxel Representation

## Overview
Anisotropic Voxel Representation is a storage format for non-leaf octree nodes in voxel cone tracing that replaces the single scalar value with six directional values corresponding to the major axes ($+X, -X, +Y, -Y, +Z, -Z$) [S1]. It solves the "two red-green wall problem" and opacity under-estimation caused by isotropic averaging, which incorrectly mixes opaque surfaces with different colors or under-estimates opacity in half-filled voxel clusters, leading to light leaking during large cone traces [S1]. It differs from isotropic MIP-mapping by preserving directional coherence during pre-integration, allowing accurate interpolation at render time without the artifacts of simple averaging [S1].

## How it works
**Inputs:** Irradiance or color values for leaf voxels and the octree structure [S1].
**Outputs:** An octree where non-leaf nodes store 6 directional values per property, and leaf nodes store 1 scalar value [S1].

**Data Structures:**
*   **Leaf Node:** Stores a single scalar value (e.g., irradiance, color) per property [S1].
*   **Non-Leaf Node:** Stores an array of 6 values per property, indexed by direction $d \in \{+X, -X, +Y, -Y, +Z, -Z\}$ [S1].
*   **Memory Layout:** The directional values are stored contiguously or in separate textures (bricks) to allow efficient sampling [S1]. The base level (full resolution) is shared across all directions to reduce overhead [S1].

**Mipmap Generation (Pre-Integration):**
The representation is constructed during the MIP-mapping process when building or updating the octree [S1]. For each non-leaf voxel and for each of the 6 major directions:
1.  Perform a step of volumetric integration in depth along that specific direction [S1].
2.  This involves compositing the 8 child voxels front-to-back along the axis [G].
3.  Average the 4 directional values to produce the final directional value for the parent voxel [S1].

**Runtime Sampling:**
During cone tracing, the voxel value is retrieved by linearly interpolating the values from the three closest directions with respect to the view direction $\vec{v}$ [S1].
1.  Identify the three major axes whose directions have the highest squared dot product with the trace direction $\vec{d}$ [G].
2.  Compute weights for these three directions:
    $$ w_i = \text{dot}(\vec{d}, \vec{a}_i)^2 $$
    where $\vec{a}_i$ is the unit vector of the $i$-th major axis [G].
3.  Normalize the weights and linearly interpolate the stored directional values using these weights [G].

**Parameters:**
*   Number of directional channels: 6 [S1].
*   Number of directions interpolated at render time: 3 [S1].
*   Scope: Directional values are only computed and used for voxels not located in the last octree level (not at full resolution) [S1].

**Edge Cases and Pitfalls:**
*   **Two Red-Green Wall Problem:** Averaging opaque voxels with different colors in upper octree levels mixes them as if semi-transparent, causing light leaking [S1].
*   **Opacity Under-estimation:** If a 2x2x2 set of voxels is half opaque and half transparent, the averaged voxel becomes half-transparent, causing light leaking for large cones [S1].
*   **Memory Overhead:** Storing directional values for all properties increases memory consumption by 1.5x compared to isotropic storage [S1]. The total storage overhead is cited as 1.75x in some contexts [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Major axes directions
AXES = [
    (1.0, 0.0, 0.0), (-1.0, 0.0, 0.0),
    (0.0, 1.0, 0.0), (0.0, -1.0, 0.0),
    (0.0, 0.0, 1.0), (0.0, 0.0, -1.0)
]

def composite_front_to_back(children, axis):
    """Standard front-to-back volumetric compositing of 8 child voxels along an axis."""
    # children: list of 8 tuples (color, opacity) ordered front-to-back
    # axis: unit vector indicating integration direction
    result_color = (0.0, 0.0, 0.0)
    result_opacity = 0.0
    for c_color, c_opacity in children:
        alpha = c_opacity * (1.0 - result_opacity)
        result_color = tuple(r + a * c for r, a, c in zip(result_color, alpha, c_color))
        result_opacity += alpha
    return result_color, result_opacity

def get_top_3(scores):
    """Returns the 3 keys with the highest scores."""
    sorted_items = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return [item[0] for item in sorted_items[:3]]

def compute_directional_value(children, axis):
    """Computes the directional value for a parent voxel."""
    # Perform volumetric integration in depth along the axis
    integrated = composite_front_to_back(children, axis)
    # Average the 4 directional values to get the resulting value for one given direction [S1]
    # Note: The exact method to derive the 4 values from integration is not specified in the sources.
    # Assuming a standard 4-sample average or direct return of the integrated result for simplicity.
    return integrated

def build_anisotropic_octree(leaf_voxels):
    """Builds the anisotropic octree from leaf nodes."""
    non_leaf_nodes = []
    # Iterate over non-leaf levels (base level is shared and isotropic)
    for parent in leaf_voxels.non_leaf_levels:
        parent_values = {}
        children = parent.get_children() # Returns 8 child values
        for axis in AXES:
            val = compute_directional_value(children, axis)
            parent_values[axis] = val
        non_leaf_nodes.append(parent_values)
    return non_leaf_nodes, leaf_voxels

def sample_anisotropic_voxel(directional_values, trace_direction):
    """Samples the voxel value for a given trace direction."""
    # Compute squared dot products for all 6 directions
    scores = {}
    for axis, val in directional_values.items():
        dot_prod = sum(t1 * t2 for t1, t2 in zip(trace_direction, axis))
        scores[axis] = dot_prod ** 2

    # Select the 3 directions with the highest scores
    top_3_axes = get_top_3(scores)

    # Linearly interpolate the values from the three closest directions [S1]
    total_weight = sum(scores[axis] for axis in top_3_axes)
    result = (0.0, 0.0, 0.0)
    for axis in top_3_axes:
        weight = scores[axis] / total_weight
        val = directional_values[axis]
        result = tuple(r + w * v for r, w, v in zip(result, weight, val))
    return result
```

## Subtopics

### [Voxel Cone Tracing Anisotropic Filtering](2552-voxel-cone-tracing-anisotropic-filtering.md)
A sibling method that applies anisotropic filtering techniques to cone tracing, differing in implementation details but sharing the goal of reducing artifacts.

## Operating characteristics
*   **Time Complexity:** Mipmap generation requires 6 integration passes per non-leaf node, increasing build time compared to isotropic MIP-mapping [G]. Runtime sampling involves computing dot products and interpolating 3 values, which is slightly more expensive than single-value lookup but remains efficient [G].
*   **Memory Complexity:** Increases memory consumption by 1.5x for non-leaf levels compared to isotropic storage [S1]. The base level is shared, reducing the theoretical maximum overhead from 6x to 1.5x [S1]. The total storage overhead is cited as 1.75x in some contexts [S1].
*   **Performance:** Delivers interactive to real-time framerates on complex scenes (e.g., Sponza, 280K triangles, 9-level octree) on hardware such as NVIDIA GTX 480 and Intel Core 2 Duo E6850 [S1].
*   **Strengths:** Prevents light leaking and color mixing artifacts; simple to implement and store in bricks; fully interpolatable [S1].
*   **Limitations:** Increased memory usage; requires careful handling of the last octree level (leaf nodes remain isotropic) [S1].

## Background
The method was introduced in the context of Interactive Indirect Illumination Using Voxel Cone Tracing [S1]. It was chosen over more complicated representations because it can be fully interpolated and easily stored in bricks [S1]. The approach addresses specific quality problems in voxel cone tracing, such as the two red-green wall problem, which were not adequately solved by isotropic MIP-mapping [S1].

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
