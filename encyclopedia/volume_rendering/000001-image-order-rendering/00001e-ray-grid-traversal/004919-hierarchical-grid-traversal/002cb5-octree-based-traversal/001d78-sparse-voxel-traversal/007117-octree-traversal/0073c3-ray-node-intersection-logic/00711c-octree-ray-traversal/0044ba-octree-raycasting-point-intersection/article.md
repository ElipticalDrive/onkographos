# Octree Raycasting (Point Intersection)

Also called: PointOctant

## Overview
This method finds points within a pointer-based octree that intersect with a given ray by iterating through intersecting octants and testing the points contained within them [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A raycaster (defining origin and direction) and a `PointOctree` instance [S1, unofficial].
- Output: A list of `RayPointIntersection` objects representing the intersected points [S1, unofficial].

**Data Structures:**
- `PointOctree<T>`: Extends `Octree<PointOctant<T>>` [S1, unofficial].
  - `bias`: Number. Octant boundary bias. Defaults to $0.0$ [S1, unofficial].
  - `maxPoints`: Number. Maximum distinct points per leaf octant before a split occurs. Defaults to $8$ [S1, unofficial].
  - `maxDepth`: Number. Maximum tree depth level, starting at $0$. Defaults to $8$ [S1, unofficial].
- `PointOctant<T>`: Represents a node in the octree [S1, unofficial].
  - `pointData`: Holds the points stored in this octant [S1, unofficial].

**Parameters:**
- `bias`: Default $0.0$. Must be $\ge 0$ [S1, unofficial].
- `maxPoints`: Default $8$. Must be $\ge 1$ [S1, unofficial].
- `maxDepth`: Default $8$. Must be $\ge 0$ [S1, unofficial].

**Steps:**
1. Initialize an empty list `result` of type `RayPointIntersection` [S1, unofficial].
2. Retrieve the list of octants intersecting the ray by calling the parent traversal method `get_intersecting_octants(raycaster)` [S1, unofficial].
3. If the list of intersecting octants is not empty:
   a. Iterate through each octant in the list.
   b. Check if the octant contains point data (`pointData` is not `None`) [S1, unofficial].
   c. If point data exists, call `pointData.testPoints(raycaster, result)` to identify specific points within the octant that intersect the ray and add them to `result` [S1, unofficial].
4. Return the `result` list [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_octree_raycast(raycaster, octree):
    # 1. Initialize result list
    result = []
    
    # 2. Get intersecting octants from parent traversal logic [G]
    intersecting_octants = octree.get_intersecting_octants(raycaster)
    
    # 3. Iterate through octants
    if len(intersecting_octants) > 0:
        for octant in intersecting_octants:
            # 4. Check for point data
            if octant.pointData is not None:
                # 5. Test points within this octant against the ray
                octant.pointData.testPoints(raycaster, result)
                
    # 6. Return results
    return result
```

**Edge Cases and Pitfalls:**
- Allowing `maxDepth` to be infinite can result in poor performance due to infinitely small octants [S1, unofficial].
- The `bias` parameter must be non-negative and affects the looseness of the octree boundaries [S1, unofficial].
- Octants without `pointData` are skipped, optimizing traversal for sparse data [S1, unofficial].

## Subtopics

### [Octree Raycasting with Refraction and Reflection](16931-octree-raycasting-with-refraction-and-reflection.md)
A GPU-based technique extending this method by adding recursive shading for non-opaque surfaces to simulate transparent and reflective materials.

## Operating characteristics
Time and memory complexity are not given in the sources. Performance scales linearly with the number of intersecting octants and the number of points per octant [S1, unofficial].

## Background
This method originates from the `sparse-octree` library implementation [S1, unofficial]. It is designed to work with the `PointOctant` class to provide efficient raycasting against discrete point sets [S1, unofficial].

## Sources
[S1] sparse-octree module, https://github.com/vanruesc/sparse-octree, fetched 2026-10-27 (unofficial)
