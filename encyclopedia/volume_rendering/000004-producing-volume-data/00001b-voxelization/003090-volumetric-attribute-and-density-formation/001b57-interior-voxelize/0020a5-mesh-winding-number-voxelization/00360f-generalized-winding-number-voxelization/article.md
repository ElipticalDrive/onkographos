# Generalized Winding Number Voxelization

Also called: GWN Voxelization, WindingNumberShape

## Overview
Generalized Winding Number Voxelization constructs a binary image (voxelization) from a set of oriented point clouds in $\mathbb{R}^3$ by evaluating an implicit surface defined by the generalized winding number framework [S1, S2]. It differs from standard mesh voxelization by operating directly on oriented point clouds without requiring a manifold mesh, using the winding number to approximate the exterior envelope of the shape [S1, S2]. It builds on [Mesh Winding Number Voxelization](2328-mesh-winding-number-voxelization.md) by providing the specific implementation for point clouds using the libIGL backend [S1, S2].

## How it works
**Inputs and Outputs:**
- **Input:** An $m \times 3$ matrix of point positions $P = \{p_i\}$ and an $m \times 3$ matrix of normal vectors $N = \{n_i\}$ [S1, S2]. Optionally, an $m$-vector of area measures $A = \{a_i\}$ [S1, S2].
- **Output:** A binary classification (INSIDE, OUTSIDE, ON) for query points, or a voxel grid generated via a digitizer [S1, S2].

**Data Structures:**
The method relies on a hierarchical data structure (e.g., a balanced tree) containing the input points to accelerate query traversal [S1, S2]. The specific internal layout is not detailed in the sources, but it supports $O(\log m)$ expected time per query [S1, S2].

**Implicit Function:**
The generalized winding number $w(q)$ for a query point $q$ is approximated by [S1, S2]:
$$
w(q) = \sum_{i=1}^m a_i \frac{(p_i - q) \cdot n_i}{4\pi \|p_i - q\|^3}
$$
where:
- $q$ is the query point [S1, S2].
- $p_i$ is the position of the $i$-th input point [S1, S2].
- $n_i$ is the normal vector of the $i$-th input point [S1, S2].
- $a_i$ is the area contribution of point $p_i$ [S1, S2].

**Area Measure Estimation:**
The area contribution $a_i$ is computed from the projection of the $k$-nearest neighbors to $p_i$ onto the tangent plane at $p_i$ [S1, S2]. This estimation is performed automatically by libIGL/CGAL if not provided by the user [S1, S2].

**Thresholding:**
The orientation is determined by thresholding $w(q)$ [S1, S2]. The default threshold is $0.3$ [S1, S2]. Values in the range $[0, 1]$ are used to define the iso-surface [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
from typing import List, Tuple

class WindingNumberShape:
    def __init__(self, points: List[Tuple[float, float, float]], 
                 normals: List[Tuple[float, float, float]], 
                 area_measures: List[float] = None, skip_point_areas: bool = False):
        """
        Initialize the winding number shape.
        
        Args:
            points: m x 3 matrix of point positions.
            normals: m x 3 matrix of normal vectors.
            area_measures: Optional m-vector of area measures. If provided, 
                           skip_point_areas must be True.
            skip_point_areas: If True, bypass automatic area estimation.
        """
        self.points = points
        self.normals = normals
        self.threshold = 0.3  # Default threshold [S1, S2]
        
        if area_measures is not None:
            if not skip_point_areas:
                raise ValueError("skip_point_areas must be True if area_measures are provided")
            self.area_measures = area_measures
        else:
            # Estimate area measures: project k-nearest neighbors to p_i onto tangent plane at p_i [S1, S2]
            # The sources do not provide the exact implementation [G]
            self.area_measures = [1.0] * len(points)
            
        # Build hierarchical data structure for fast queries [S1, S2]
        # The sources do not provide the exact tree construction algorithm [G]
        self.tree = None

    def orientation(self, q: Tuple[float, float, float]) -> str:
        """
        Query orientation for a single point.
        
        Args:
            q: Query point (3-vector).
            
        Returns:
            'INSIDE', 'OUTSIDE', or 'ON' based on thresholding w(q).
        """
        w = self.winding_number(q)
        if w > self.threshold:
            return 'INSIDE'
        elif w < 1.0 - self.threshold:
            return 'OUTSIDE'
        else:
            return 'ON'

    def winding_number(self, q: Tuple[float, float, float]) -> float:
        """
        Compute the generalized winding number w(q) for a single point.
        
        Formula: w(q) = sum_i (a_i * (p_i - q) . n_i) / (4 * pi * ||p_i - q||^3) [S1, S2]
        """
        w = 0.0
        for i in range(len(self.points)):
            p_i = self.points[i]
            n_i = self.normals[i]
            a_i = self.area_measures[i]
            
            diff = (p_i[0] - q[0], p_i[1] - q[1], p_i[2] - q[2])
            dist_sq = diff[0]**2 + diff[1]**2 + diff[2]**2
            dist_cube = dist_sq ** 1.5
            
            if dist_cube < 1e-9:
                continue  # Avoid division by zero [G]
                
            dot_prod = diff[0]*n_i[0] + diff[1]*n_i[1] + diff[2]*n_i[2]
            term = a_i * dot_prod / (4.0 * math.pi * dist_cube)
            w += term
        return w

    def orientation_batch(self, queries: List[Tuple[float, float, float]]) -> List[str]:
        """
        Query orientations for multiple points, leveraging multithreading [S1, S2].
        
        Args:
            queries: Q x 3 matrix of query points.
            
        Returns:
            List of orientations ('INSIDE', 'OUTSIDE', 'ON') in the same order as queries.
        """
        # Implementation relies on libIGL multithreading [S1, S2]
        # The sources do not provide the exact batch query implementation [G]
        return [self.orientation(q) for q in queries]

    def set_area_measure(self, area_measures: List[float]):
        """
        Set custom area measures.
        """
        self.area_measures = area_measures
```

**Edge Cases and Pitfalls:**
- **Dependencies:** Requires LIBIGL, CGAL, and GMP libraries. CMake flags `WITH_LIBIGL`, `WITH_CGAL`, and `WITH_GMP` must be set to true [S1, S2].
- **Area Measures:** If user-provided area measures are used, `skipPointAreas` must be set to true in the constructor to bypass automatic estimation [S1, S2].
- **Precision:** The formula involves division by $\|p_i - q\|^3$, which can lead to numerical instability if $q$ is very close to $p_i$ [G].

## Operating characteristics
- **Time Complexity:** Expected $O(\log m)$ per single query due to traversal of a hierarchical data structure, assuming balanced trees [S1, S2].
- **Memory Complexity:** Not given in the sources.
- **Performance:** Leverages multithreading for batch queries [S1, S2].
- **Strengths:** Handles oriented point clouds directly without manifold mesh requirements [S1, S2]. Fast query times with hierarchical structures [S1, S2].
- **Limitations:** Requires external dependencies (LIBIGL, CGAL, GMP) [S1, S2]. Area measure estimation relies on libIGL/CGAL [S1, S2].
- **When to Use:** Voxelizing oriented point clouds [S1, S2].
- **When Not to Use:** When external dependencies cannot be used or when a manifold mesh is already available and simpler voxelization methods suffice [G].

## Background
The method is implemented in the DGtal library (version 1.4+) within the Shapes package [S1, S2]. It is based on the generalized winding number framework of [10] (not provided in the sources) [G]. It is used in examples `polyscope_examples/windingNumberShape.cpp` and tests `shapes/testWindingNumberShape.cpp` [S1, S2]. The class `WindingNumberShape<Space>` models `concepts::CEuclideanBoundedShape` [S1, S2].

## Sources
[S1] Voxelization from an oriented point cloud, https://www.dgtal.org/doc/1.4/moduleWinding.html, fetched 2026-10-26
[S2] Voxelization from an oriented point cloud, https://dgtal-team.github.io/doc-pr/pr1697/moduleWinding.html, fetched 2026-10-26
