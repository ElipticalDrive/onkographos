# Normal-Based Sign Determination


## Overview
Normal-Based Sign Determination assigns the sign of a Signed Distance Field (SDF) value at a query point by evaluating the dot product between the point's position relative to the closest surface point and the surface normal at that point [S1, unofficial]. It is distinguished from Depth Buffer Sign Determination by its ability to handle meshes with holes, though it is prone to cone-shaped artifacts on meshes with inconsistent triangle orientations [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A query point $P$, the closest surface point $C$ on the mesh, and the surface normal $N$ at $C$ [S1, unofficial].
- Output: The sign of the SDF ($+1$ for outside, $-1$ for inside) [S1, unofficial].

**Data Structures:**
- The method relies on the surface normal vector $N$ associated with the closest triangle or interpolated from vertices at point $C$ [S1, unofficial].
- A parameter `normal_sample_count` (default: 11) determines the number of nearby surface points checked to determine the sign via majority vote [S1, unofficial].

**Steps:**
1. Identify the closest surface point $C$ to the query point $P$ [S1, unofficial].
2. Retrieve the surface normal $N$ at $C$ [S1, unofficial].
3. Calculate the vector $V = P - C$ [S1, unofficial].
4. Compute the dot product $D = V \cdot N$ [S1, unofficial].
5. If `normal_sample_count` > 1, repeat steps 1-4 for `normal_sample_count` nearby surface points to gather votes [S1, unofficial].
6. Determine the sign based on the majority vote of the dot products or individual signs derived from them [S1, unofficial].
7. If the dot product $D$ is positive, the point is considered outside (sign $+1$); if negative, inside (sign $-1$) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def determine_sign_normal(query_point, mesh, normal_sample_count=11):
    # Find the closest surface point and its normal
    closest_point, closest_normal = mesh.get_closest_surface_info(query_point)
    
    # Vector from surface to query point
    vector_to_query = query_point - closest_point
    
    # Primary dot product
    dot_product = np.dot(vector_to_query, closest_normal)
    
    # If multiple samples are requested, gather votes
    if normal_sample_count > 1:
        votes = []
        # Add the primary sample
        votes.append(1 if dot_product > 0 else -1)
        
        # Sample nearby points (implementation detail: [G] specific sampling strategy not provided in source)
        # The source states signs are determined by checking a number of surface points
        # and using a majority vote.
        for _ in range(normal_sample_count - 1):
            # [G] Placeholder for retrieving nearby surface points and normals
            # In practice, this involves sampling neighbors of the closest triangle
            # or vertices.
            nearby_point, nearby_normal = get_nearby_surface_info(query_point, mesh)
            nearby_vector = query_point - nearby_point
            nearby_dot = np.dot(nearby_vector, nearby_normal)
            votes.append(1 if nearby_dot > 0 else -1)
            
        # Majority vote
        sign = 1 if sum(votes) > 0 else -1
    else:
        sign = 1 if dot_product > 0 else -1
        
    return sign
```

**Parameters:**
- `normal_sample_count`: Number of nearby surface points to check. Default is 11 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Cone-shaped artifacts:** These appear in the SDF volume, particularly when the mesh contains tiny triangles facing differently than surrounding areas [S1, unofficial].
- **Ineffectiveness of sampling:** Increasing `normal_sample_count` does not mitigate cone artifacts [S1, unofficial].
- **Smoothing:** Smoothing the mesh does not seem to help with these artifacts [S1, unofficial].

## Subtopics

### Depth Buffer Sign Determination
Determines the sign of the SDF by projecting the query point into the frame of each render and comparing its depth against the depth buffer [S1, unofficial]. The point is projected into the camera frame, and if the query point's depth is less than the depth buffer value, it is considered seen by the camera [S1, unofficial]. The sign is positive if the point is seen by any camera [S1, unofficial]. This method avoids bubble artifacts but fails on non-watertight meshes, producing ray-shaped artifacts when holes allow cameras to see "inside" the mesh [S1, unofficial]. It is less accurate than the normal method for watertight meshes but preferred for meshes with holes [S1, unofficial].

## Operating characteristics
- **Strengths:** Works better for meshes with holes compared to the depth method [S1, unofficial].
- **Limitations:** Produces cone-shaped artifacts in the SDF volume, especially with tiny triangles facing different directions [S1, unofficial].
- **Performance:** The method is part of a pipeline that can process about 60% of meshes in the ShapeNet dataset [S1, unofficial].
- **Usage:** Use when the mesh has holes and depth-based methods fail; try the alternative method if results are unusable [S1, unofficial].

## Background
This method is part of the `mesh_to_sdf` library by marian42, which provides approximate methods for generating SDFs from meshes [S1, unofficial]. It is designed to handle cases where exact watertightness cannot be guaranteed, offering a fallback to the depth-based method when one fails [S1, unofficial].

## Sources
[S1] mesh_to_sdf, https://github.com/marian42/mesh_to_sdf, fetched 2026-10-28 (unofficial)
