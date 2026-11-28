# Normal-Based Sign Determination
## Overview
Normal-Based Sign Determination assigns the sign of a Signed Distance Field (SDF) value at a query point by evaluating the dot product between the vector from the closest surface point to the query point and the surface normal at that point [S1, unofficial]. It is distinguished from Depth Buffer Sign Determination by its ability to handle meshes with holes, though it is prone to cone-shaped artifacts on meshes with inconsistent triangle orientations [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A query point $P$, a list of `normal_sample_count` nearby surface points, and their corresponding surface normals [S1, unofficial].
- Output: The sign of the SDF ($+1$ for outside, $-1$ for inside) [S1, unofficial].

**Data Structures:**
- The method stores a list of surface points and their normals to evaluate the sign via majority voting [S1, unofficial].
- A parameter `normal_sample_count` (default: 11) specifies how many nearby surface points are checked [S1, unofficial].

**Steps:**
1. Identify the closest surface point $C$ to the query point $P$ [S1, unofficial].
2. Retrieve the surface normal $N$ at $C$ [S1, unofficial].
3. Compute the vector $V = P - C$ [S1, unofficial].
4. Compute the dot product $D = V \cdot N$ [S1, unofficial].
5. If `normal_sample_count` > 1, repeat steps 1–4 for `normal_sample_count` nearby surface points to collect votes [S1, unofficial].
6. Determine the sign via majority vote of the collected signs [S1, unofficial].
7. A positive majority indicates outside ($+1$); a negative majority indicates inside ($-1$) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def determine_sign_normal(query_point, surface_points, surface_normals, normal_sample_count=11):
    votes = []
    for i in range(normal_sample_count):
        # The source does not specify how nearby surface points are sampled [G]
        point = surface_points[i]
        normal = surface_normals[i]
        vector_to_query = [query_point[j] - point[j] for j in range(3)]
        dot_product = sum(v * n for v, n in zip(vector_to_query, normal))
        votes.append(1 if dot_product > 0 else -1)
    sign = 1 if sum(votes) > 0 else -1
    return sign
```

**Parameters:**
- `normal_sample_count`: Number of nearby surface points to check [S1, unofficial]. Default is 11 [S1, unofficial].

**Edge Cases and Pitfalls:**
- Cone-shaped artifacts appear in the SDF volume, especially when the mesh contains tiny triangles facing differently than surrounding areas [S1, unofficial].
- Increasing `normal_sample_count` does not mitigate these artifacts [S1, unofficial].
- Mesh smoothing does not resolve the artifacts [S1, unofficial].
## Subtopics

### [Depth Buffer Sign Determination](16360-depth-buffer-sign-determination.md)
Depth Buffer Sign Determination determines the inside/outside sign of SDF voxel corners by projecting them into camera space and comparing their depth against a rendered depth buffer of the surface. It sets apart from sibling methods like Normal-Based Sign Determination by relying on geometric occlusion rather than surface normals, thereby avoiding bubble artifacts caused by noisy normals or tiny triangles. However, it is less accurate on non-watertight geometry.

## Operating characteristics
- Strengths: Works better for meshes with holes compared to the depth method [S1, unofficial].
- Limitations: Produces cone-shaped artifacts in the SDF volume, especially with tiny triangles facing different directions [S1, unofficial].
- Performance: The method is part of a pipeline that can process about 60% of meshes in the ShapeNet dataset [S1, unofficial].
- Usage: Use when the mesh has holes and depth-based methods fail; try the alternative method if results are unusable [S1, unofficial].
## Background
This method is part of the `mesh_to_sdf` library by marian42, which provides approximate methods for generating SDFs from meshes [S1, unofficial]. It is designed to handle cases where exact watertightness cannot be guaranteed, offering a fallback to the depth-based method when one fails [S1, unofficial].
## Sources
[S1] mesh_to_sdf, https://github.com/marian42/mesh_to_sdf, fetched 2026-10-28 (unofficial)
