# Power Distance Sort

Also called: Power Distance Sorting, Power Sorting, Radical axis sorting, Visibility sorting via circumcircles

## Overview
Power Distance Sort is a visibility sorting method that orders primitives (triangles or tetrahedra) front-to-back relative to a camera origin by comparing the power of the viewpoint with respect to the circumcircles or circumspheres of the primitives [S2, unofficial]. It serves as a fast, constant-time sorting criterion for Delaunay triangulations, distinguishing itself from general-purpose sorts by leveraging geometric properties of the mesh topology [S1][S4, unofficial]. Unlike exact visibility algorithms, it is not guaranteed to produce correct ordering for non-Delaunay meshes [S1][S3, unofficial].

## How it works
**Inputs:**
- A set of primitives $T_i$ (triangles in 2D or tetrahedra in 3D) forming a Delaunay triangulation/tetrahedralization.
- A camera origin $\mathbf{o}$.

**Outputs:**
- A sorted list of primitives in back-to-front order relative to $\mathbf{o}$.

**Data Structures:**
- Each primitive $T$ stores its circumcenter $\mathbb{O}(T)$ and circumradius $\mathbb{R}(T)$. These are precomputed based on the vertices of $T$.

**Steps:**
1. For each primitive $T$, compute the power of the viewpoint $\mathbf{o}$ relative to the circumcircle/circumsphere of $T$.
2. Sort all primitives in ascending order of this power value.

**Formula:**
The power $P(T)$ of the viewpoint $\mathbf{o}$ with respect to the circumsphere of primitive $T$ is defined as [S2, unofficial]:

$$
P(T) = \|\mathbb{O}(T) - \mathbf{o}\|_2^2 - \mathbb{R}(T)^2
$$

Where:
- $\mathbb{O}(T)$ is the center of the circumsphere of $T$.
- $\mathbb{R}(T)$ is the radius of the circumsphere of $T$.
- $\mathbf{o}$ is the camera origin.
- $\|\cdot\|_2$ denotes the Euclidean norm.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_power_distance_sort(primitives, camera_origin):
    """
    Sorts primitives by power distance relative to camera_origin.
    Assumes primitives form a Delaunay triangulation/tetrahedralization.
    
    Args:
        primitives: List of primitive objects, each having attributes:
            - circumcenter: tuple (x, y, z) or (x, y)
            - circumradius: float
        camera_origin: tuple (x, y, z) or (x, y)
        
    Returns:
        List of primitives sorted from back to front (ascending power distance).
    """
    # Step 1: Compute power distance for each primitive
    scored_primitives = []
    for prim in primitives:
        # Calculate squared Euclidean distance between circumcenter and camera origin
        dx = prim.circumcenter[0] - camera_origin[0]
        dy = prim.circumcenter[1] - camera_origin[1]
        # Add z-component if 3D
        if len(prim.circumcenter) == 3:
            dz = prim.circumcenter[2] - camera_origin[2]
            dist_sq = dx*dx + dy*dy + dz*dz
        else:
            dist_sq = dx*dx + dy*dy
            
        # Power distance formula: ||O - o||^2 - R^2
        power = dist_sq - (prim.circumradius ** 2)
        scored_primitives.append((power, prim))
        
    # Step 2: Sort by power distance ascending
    # Ascending order corresponds to back-to-front for Delaunay meshes relative to origin
    scored_primitives.sort(key=lambda x: x[0])
    
    # Return just the primitives in sorted order
    return [prim for _, prim in scored_primitives]
```

**Edge Cases and Pitfalls:**
- **Non-Delaunay Meshes:** The method is not guaranteed to produce an accurate visibility ordering unless the mesh is a Delaunay triangulation [S1][S4, unofficial]. There is no way to tell in advance whether a given non-Delaunay mesh will be correctly sorted [S1].
- **Viewpoint Position:** The sort order is relative to the ray origin, making it valid for fisheye lenses and distorted projections [S2, unofficial]. However, simple schemes like power distance sorts may fail from certain viewpoints or viewing directions even for Delaunay meshes if the geometric assumptions are violated (e.g., degenerate configurations) [S3, unofficial].
- **Precision:** Computing circumcenters and radii can be numerically unstable for nearly cospherical points, which may lead to incorrect sorting if precision is insufficient [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(n \log n)$ due to sorting, where $n$ is the number of primitives. The computation of power distance per primitive is $O(1)$ [S4, unofficial].
- **Memory Complexity:** $O(n)$ to store the power values and primitive references.
- **Performance:** Fast with very small constant computations and straightforward implementation for Delaunay meshes [S1][S4, unofficial].
- **Strengths:** Works with distorted projections (e.g., fisheye) because it is based on the camera origin rather than a view direction [S2, unofficial].
- **Limitations:** Not applicable to general non-Delaunay meshes without guarantees of correctness [S1]. Accuracy is not predictable for arbitrary meshes [S1].

## Background
Power Distance Sort was described by Cignoni et al. and Wittenbrink [S1][S4, unofficial]. It leverages the geometric property that for two adjacent triangles in a Delaunay triangulation, the shared edge lies along the radical axis of their circumcircles. The radical axis divides the plane into two regions where the viewpoint has a lower power distance to one circumcircle or the other. Since the relative depth order of the two triangles flips only when the viewpoint crosses this axis, sorting by power distance yields the correct visibility order [S2, unofficial]. This property generalizes to 3D, where the shared face of two tetrahedra lies on the radical plane [S2, unofficial].

## Sources

[S1] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://doi.org/10.1109/tvcg.2004.45, fetched 2026-10-25
[S2] Radiance Meshes for Volumetric Reconstruction, https://arxiv.org/abs/2512.04076v1, fetched 2026-10-26 (unofficial)
[S3] (PDF) Image-space visibility ordering for cell projection volume ..., https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S4] PDF 0078-0903-2 1..13, https://sci.utah.edu/~csilva/papers/tvcg2004cr.pdf, fetched 2026-10-28 (unofficial)
