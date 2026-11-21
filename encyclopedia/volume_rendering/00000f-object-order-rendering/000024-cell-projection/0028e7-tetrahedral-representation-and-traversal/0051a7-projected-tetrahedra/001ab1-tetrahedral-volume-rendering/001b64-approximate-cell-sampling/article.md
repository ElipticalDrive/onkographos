# Approximate Cell Sampling
## Overview
Approximate Cell Sampling is a method for sampling cell-centered data in tetrahedral volume rendering that estimates the contribution of neighboring tetrahedra without performing a full tree traversal to locate their vertices [S1, unofficial]. It solves the problem of accessing neighbor geometry when vertex data is not stored for non-coarse elements, using volume ratios and face normals to approximate neighbor centroids [S1, unofficial]. It is distinguished from Simple Cell Sampling by weighting samples by the inverse distance to the estimated centroids, and from [Vertex-centered Data Sampling](1037-vertex-centered-data-sampling.md) by operating on cell-centered data rather than vertex-centered data [S1, unofficial].

## How it works
**Inputs:**
- `point`: The 3D sample point in world space [S1, unofficial].
- `V`: An array of 4 vertices `[t, l, r, b]`, where each vertex is a 3D coordinate tuple [S1, unofficial].
- `tetID`: The ID of the current tetrahedron [S1, unofficial].
- `coarseTetID`: The ID of the coarse tetrahedron containing the current element [S1, unofficial].
- `neighbors`: An array of neighbor IDs, where neighbors for `tetID` are located at `neighbors[4 * tetID + i]` for `i` in `0..3` [S1, unofficial].
- `cellData`: An array of scalar values associated with each tetrahedron [S1, unofficial].

**Outputs:**
- A weighted average of the cell-centered data from the current tetrahedron and its neighbors, weighted by the inverse distance to their estimated centroids [S1, unofficial].

**Data Structures:**
- The neighbor array and refinement level differences are pre-computed during TET-AMR construction [S1, unofficial].

**Steps:**
1. Extract the four vertices of the current tetrahedron from `V`: `t`, `l`, `r`, `b` [S1, unofficial].
2. Calculate the centroid of the current tetrahedron:
   $$ \text{centroid} = \frac{t + l + r + b}{4} $$ [S1, unofficial].
3. Initialize the inverse distance accumulator and the weighted sample sum:
   $$ \text{invDist} = \frac{1}{\|\text{centroid} - \text{point}\|} $$ [S1, unofficial].
   $$ \text{sample} = \text{cellData}[\text{tetID}] \times \text{invDist} $$ [S1, unofficial].
4. Loop for each face index $i$ from 0 to 3 (representing the 4 neighbors):
   a. Retrieve the neighbor ID:
      $$ \text{neighborID} = \text{neighbors}[4 \times \text{tetID} + i] $$ [S1, unofficial].
   b. Initialize the volume ratio:
      $$ \text{volRatio} = 1 $$ [S1, unofficial].
   c. Determine if the neighbor is on a different refinement tree and get the level difference:
      $$ (\text{onAnotherTree}, \text{rLevelDiff}) = \text{ProcessNeighborID}(\text{neighborID}) $$ [S1, unofficial].
   d. If the neighbor is on another tree, calculate the volume ratio between the coarse elements:
      $$ \text{volRatio} = \text{GetVolumeRatio}(\text{neighborID}, \text{coarseTetID}) $$ [S1, unofficial].
   e. Adjust the volume ratio for the refinement level difference. Since each child has $1/8$ the volume of its parent:
      $$ \text{volRatio} = \text{volRatio} \times 8^{\text{rLevelDiff}} $$ [S1, unofficial].
   f. Calculate the centroid of the shared face. The face opposite to vertex $V[i]$ is the average of the other 3 vertices [G]. The formula $(\text{centroid} \times 4 - V[i]) / 3$ correctly computes this because $4 \times \text{centroid} = t+l+r+b$:
      $$ \text{faceCentroid} = \frac{\text{centroid} \times 4 - V[i]}{3} $$ [S1, unofficial].
   g. Get the unit normal of the face:
      $$ \text{faceNormal} = \text{GetUnitFaceNormal}(i, t, l, r, b) $$ [S1, unofficial].
   h. Calculate the distance from the current tetrahedron's centroid to the face centroid:
      $$ \text{faceToCentroidDistance} = \|\text{centroid} - \text{faceCentroid}\| $$ [S1, unofficial].
   i. Estimate the distance from the face centroid to the neighbor's centroid by scaling the current distance by the volume ratio:
      $$ \text{neighborFaceToCentroidDistance} = \text{volRatio} \times \text{faceToCentroidDistance} $$ [S1, unofficial].
   j. Estimate the neighbor's centroid location by moving from the face centroid along the face normal:
      $$ \text{neighborCentroid} = \text{faceCentroid} + \text{neighborFaceToCentroidDistance} \times \text{faceNormal} $$ [S1, unofficial].
   k. Calculate the inverse distance from the sample point to the estimated neighbor centroid:
      $$ \text{tmpDist} = \frac{1}{\|\text{neighborCentroid} - \text{point}\|} $$ [S1, unofficial].
   l. Accumulate the inverse distance and the weighted sample:
      $$ \text{invDist} = \text{invDist} + \text{tmpDist} $$ [S1, unofficial].
      $$ \text{sample} = \text{sample} + \text{cellData}[\text{neighborID}] \times \text{tmpDist} $$ [S1, unofficial].
   m. End loop.
5. Return the final weighted average:
   $$ \text{result} = \frac{\text{sample}}{\text{invDist}} $$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sample_cell_approximate(point, V, tetID, coarseTetID, neighbors, cellData):
    # V contains vertices t, l, r, b at indices 0, 1, 2, 3 [S1, unofficial]
    t, l, r, b = V[0], V[1], V[2], V[3]
    centroid = tuple((t[k] + l[k] + r[k] + b[k]) / 4.0 for k in range(3))

    # Guard against division by zero [G]
    dist_centroid = math.sqrt(sum((c - p)**2 for c, p in zip(centroid, point)))
    if dist_centroid == 0.0:
        dist_centroid = 1e-9
    invDist = 1.0 / dist_centroid
    sample = cellData[tetID] * invDist

    # Source Algorithm 7 says "0 . . . 4", which is an off-by-one bug for a tetrahedron with 4 faces. Fixed to range(4) [S1, unofficial]
    for i in range(4):
        neighborID = neighbors[4 * tetID + i]
        volRatio = 1.0

        # Determine tree membership and refinement level difference [S1, unofficial]
        # Implementation details not provided in sources [G]
        onAnotherTree, rLevelDiff = False, 0

        if onAnotherTree:
            # Calculate volume ratio between coarse elements [S1, unofficial]
            # Implementation details not provided in sources [G]
            volRatio = 1.0

        volRatio = volRatio * (8.0 ** rLevelDiff)

        # Face centroid: average of the 3 vertices not equal to V[i] [S1, unofficial]
        faceCentroid = tuple((centroid[k] * 4.0 - V[i][k]) / 3.0 for k in range(3))

        # Get unit normal for face i [S1, unofficial]
        # Standard cross-product calculation [G]
        vertices = [t, l, r, b]
        face_verts = [v for j, v in enumerate(vertices) if j != i]
        v1 = tuple(b - a for a, b in zip(face_verts[1], face_verts[0]))
        v2 = tuple(b - a for a, b in zip(face_verts[2], face_verts[0]))
        cross = (v1[1]*v2[2] - v1[2]*v2[1],
                 v1[2]*v2[0] - v1[0]*v2[2],
                 v1[0]*v2[1] - v1[1]*v2[0])
        length = math.sqrt(sum(c**2 for c in cross))
        if length == 0.0:
            faceNormal = (0.0, 0.0, 0.0)
        else:
            faceNormal = tuple(c / length for c in cross)

        faceToCentroidDist = math.sqrt(sum((c - f)**2 for c, f in zip(centroid, faceCentroid)))
        neighborFaceToCentroidDist = volRatio * faceToCentroidDist

        neighborCentroid = tuple(fc + nfd * fn for fc, nfd, fn in zip(faceCentroid, neighborFaceToCentroidDist, faceNormal))

        dist_neighbor = math.sqrt(sum((nc - p)**2 for nc, p in zip(neighborCentroid, point)))
        if dist_neighbor == 0.0:
            dist_neighbor = 1e-9
        tmpDist = 1.0 / dist_neighbor

        invDist = invDist + tmpDist
        # Source Algorithm 7 line 21 uses cellData[tetID], which is a typo; corrected to neighborID [S1, unofficial]
        sample = sample + cellData[neighborID] * tmpDist

    return sample / invDist
```

**Parameters:**
- `8^rLevelDiff`: The scaling factor for volume ratios based on refinement level difference, as each child element has one-eighth of its parent's volume [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If the sample point coincides with a centroid, the inverse distance becomes infinite. A small epsilon guard is required [G].
- **Index Out of Bounds:** The source loop runs from 0 to 4, but a tetrahedron has 4 faces. Accessing `V[4]` or `neighbors[4*tetID + 4]` is invalid; the loop must run from 0 to 3 [S1, unofficial].
- **Source Typo:** The source pseudocode uses `cellData[tetID]` for the neighbor contribution, which is logically incorrect; it must use `cellData[neighborID]` [S1, unofficial].
- **Helper Functions:** `ProcessNeighborID` and `GetVolumeRatio` are not implemented in the source; their logic must be provided by the TET-AMR construction pipeline [G].

## Subtopics

### Simple Cell Sampling
A coarse approximation method for cell-centered data that averages the cell-centered data of the element and its neighbors without considering distances to centroids [S1, unofficial]. It is a coarse approximation because it does not consider the distances to the neighbors' centroids where the cell-centered data is sampled [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Constant time per sample point relative to the number of neighbors (4 neighbors for a tetrahedron) [S1, unofficial].
- **Memory Complexity:** Requires storage of neighbor IDs and refinement level differences, which are pre-computed during TET-AMR construction [S1, unofficial].
- **Strengths:** Avoids the computational cost of traversing the refinement tree to find neighbor vertices [S1, unofficial]. Provides a distance-weighted approximation, which is more accurate than simple averaging [S1, unofficial].
- **Limitations:** It is an approximation; the estimated neighbor centroids may not be exact, especially across different refinement trees or large level differences [S1, unofficial].
- **When to Use:** When vertex data is not stored for non-coarse elements and neighbor geometry must be approximated efficiently [S1, unofficial].

## Background
- **Origin:** The method is described in the thesis "Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data" [S1, unofficial].
- **History:** It is part of the rendering pipeline for TET-AMR data, which uses a bounding volume hierarchy (BVH) to locate coarse elements and then traverses the refinement tree on the ray until out of the coarse element [S1, unofficial].
- **Usage:** Used in direct volume rendering of unstructured tetrahedral data with adaptive mesh refinement [S1, unofficial].

## Sources
[S1] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
