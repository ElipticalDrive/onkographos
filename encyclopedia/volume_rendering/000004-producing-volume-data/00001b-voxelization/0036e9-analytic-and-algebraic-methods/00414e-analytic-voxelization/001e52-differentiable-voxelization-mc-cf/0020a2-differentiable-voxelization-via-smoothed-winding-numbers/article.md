# Differentiable Voxelization via Smoothed Winding Numbers

Also called: DVX, Smoothed Winding Number Voxelization

## Overview
Differentiable Voxelization via Smoothed Winding Numbers (DVX) is a method for voxelizing triangle meshes into smoothed winding number voxel grids that provides closed-form gradients with respect to mesh vertex positions, enabling gradient-based optimization of surface representations based on volumetric properties [S1]. Unlike [Differentiable Voxelization (MC/CF)](1742-differentiable-voxelization-mc-cf.md) which samples a non-smoothed winding number and uses automatic differentiation (yielding sparse near-surface derivative signals), this method yields a broader gradient pattern matching finite-difference references [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A triangle mesh defined by vertex positions $V$ and connectivity; a 3D voxel grid of resolution $N^3$ [S1].
*   **Output:** A voxel grid of smoothed winding numbers $W$; the gradient of a scalar objective function $L$ with respect to vertex positions $\nabla_V L$ [S1].

**Data Structures:**
*   **Voxel Grid:** A 3D array of size $N \times N \times N$ [S1]. Each cell stores a floating-point value representing the average winding number within that voxel volume [S1].
*   **Mesh:** Defined by a list of vertices $v_i \in \mathbb{R}^3$ and a list of triangles $t_j = (v_a, v_b, v_c)$ [G].

**Method:**
The core challenge is that point-sampling the raw winding number gives a piecewise-constant signal, so its derivative with respect to mesh motion is zero almost everywhere or undefined at the boundaries [S1]. DVX solves this by smoothing the winding-number field with a filter before sampling voxel values [S1].

1.  **Voxel Value Computation:**
    For a voxel-aligned box filter, a voxel's filtered value is the average winding number within the voxel [S1]. To compute this efficiently:
    *   Clip each triangle of the mesh against the boundaries of the voxel grid [S1].
    *   For each triangle, determine its intersection with the voxel grid [S1].
    *   Divide the intersections into sub-triangles [S1].
    *   Add up their signed shadow-volume contributions along grid columns to compute the voxel value [S1].
    
    The winding number $w(p)$ for a point $p$ is the signed count of how the surface wraps around $p$ [G]. The smoothed value $W_{ijk}$ for voxel $(i,j,k)$ is:
    $$ W_{ijk} = \frac{1}{V_{\text{voxel}}} \int_{V_{ijk}} w(p) \, dp $$
    where $V_{ijk}$ is the volume of the voxel and $V_{\text{voxel}}$ is the voxel volume [G]. The algorithm computes this integral by summing the contributions of clipped sub-triangles [S1].

2.  **Gradient Computation:**
    The gradient is computed using Reynolds' transport theorem [S1]. The general derivation states that the moving boundary contributes a term, while the interior term vanishes because the integrand does not otherwise depend on the mesh parameter [S1]. This holds for general surfaces and allows self-intersections [S1].
    
    For the efficient closed-form algorithm specialized for triangle meshes:
    *   Integrate the triangles' normal motion over the clipped pieces [S1].
    *   Map those contributions back to mesh vertices [S1].
    
    The gradient $\frac{\partial W_{ijk}}{\partial v_m}$ for a vertex $v_m$ is non-zero only if $v_m$ is part of a triangle that intersects the voxel or its vicinity [G]. The contribution is derived from the boundary term of the transport theorem, effectively integrating the displacement of the triangle edges against the voxel boundaries [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b): return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]
def sub(a, b): return (a[0]-b[0], a[1]-b[1], a[2]-b[2])
def cross(a, b): return (a[1]*b[2]-a[2]*b[1], a[2]*b[0]-a[0]*b[2], a[0]*b[1]-a[1]*b[0])
def scale(a, s): return (a[0]*s, a[1]*s, a[2]*s)
def add(a, b): return (a[0]+b[0], a[1]+b[1], a[2]+b[2])

def clip_triangle_to_box(v0, v1, v2, min_c, max_c):
    """Clips a triangle against an AABB using Sutherland-Hodgman plane clipping."""
    poly = [v0, v1, v2]
    bounds = [min_c, max_c]
    for axis in range(3):
        new_poly = []
        for i in range(len(poly)):
            curr = poly[i]
            prev = poly[i-1]
            curr_inside = curr[axis] >= bounds[0][axis]
            prev_inside = prev[axis] >= bounds[0][axis]
            if curr_inside:
                if not prev_inside:
                    t = (bounds[0][axis] - prev[axis]) / (curr[axis] - prev[axis])
                    new_poly.append(add(prev, scale(sub(curr, prev), t)))
                new_poly.append(curr)
            elif prev_inside:
                t = (bounds[0][axis] - prev[axis]) / (curr[axis] - prev[axis])
                new_poly.append(add(prev, scale(sub(curr, prev), t)))
        poly = new_poly
        if not poly: return []
        new_poly = []
        for i in range(len(poly)):
            curr = poly[i]
            prev = poly[i-1]
            curr_inside = curr[axis] <= bounds[1][axis]
            prev_inside = prev[axis] <= bounds[1][axis]
            if curr_inside:
                if not prev_inside:
                    t = (bounds[1][axis] - prev[axis]) / (curr[axis] - prev[axis])
                    new_poly.append(add(prev, scale(sub(curr, prev), t)))
                new_poly.append(curr)
            elif prev_inside:
                t = (bounds[1][axis] - prev[axis]) / (curr[axis] - prev[axis])
                new_poly.append(add(prev, scale(sub(curr, prev), t)))
        poly = new_poly
        if not poly: return []
    return [poly]

def signed_tetrahedron_volume(p0, p1, p2, p3):
    """Computes signed volume of tetrahedron formed by four points."""
    return dot(sub(p1, p0), cross(sub(p2, p0), sub(p3, p0))) / 6.0

def compute_signed_shadow_volume(sub_tri_points, min_c, max_c):
    """Approximates the integral of the winding number over a voxel using tetrahedral decomposition."""
    center = add(min_c, scale(sub(max_c, min_c), 0.5))
    vol = 0.0
    for i in range(len(sub_tri_points)):
        p1 = sub_tri_points[i]
        p2 = sub_tri_points[(i+1)%len(sub_tri_points)]
        vol += signed_tetrahedron_volume(center, p1, p2, (0.0, 0.0, 0.0))
    return vol

def compute_voxelization_and_gradient(mesh_vertices, mesh_triangles, grid_resolution, grid_origin, grid_spacing):
    """
    Computes the smoothed winding number voxel grid and the gradient of the grid values
    with respect to mesh vertices.
    """
    voxel_grid = [[[0.0]*grid_resolution for _ in range(grid_resolution)] for _ in range(grid_resolution)]
    vertex_gradients = [[0.0, 0.0, 0.0] for _ in range(len(mesh_vertices))]
    voxel_volume = grid_spacing ** 3

    for tri_idx in range(len(mesh_triangles)):
        v0 = mesh_vertices[mesh_triangles[tri_idx][0]]
        v1 = mesh_vertices[mesh_triangles[tri_idx][1]]
        v2 = mesh_vertices[mesh_triangles[tri_idx][2]]

        for i in range(grid_resolution):
            for j in range(grid_resolution):
                for k in range(grid_resolution):
                    min_c = add(grid_origin, (i*grid_spacing, j*grid_spacing, k*grid_spacing))
                    max_c = add(min_c, (grid_spacing, grid_spacing, grid_spacing))

                    clipped_polys = clip_triangle_to_box(v0, v1, v2, min_c, max_c)
                    if not clipped_polys: continue

                    voxel_winding_sum = 0.0
                    for poly in clipped_polys:
                        voxel_winding_sum += compute_signed_shadow_volume(poly, min_c, max_c)

                    voxel_grid[i][j][k] += voxel_winding_sum / voxel_volume

                    # [G] Exact gradient formula from Reynolds' transport theorem boundary term
                    # is not provided in the sources. Standard implementation integrates
                    # triangle normal motion over clipped boundaries and maps to vertices.
                    # grad_v0, grad_v1, grad_v2 = compute_gradient_for_subtriangles(...)
                    # vertex_gradients[mesh_triangles[tri_idx][0]] = add(vertex_gradients[...], grad_v0)

    return voxel_grid, vertex_gradients
```

**Parameters:**
*   **Grid Resolution:** The method is tested on a $32^3$ grid and reported to handle $1024^3$ resolution [S1].
*   **Filter:** A voxel-aligned box filter is used, which corresponds to averaging the winding number within the voxel [S1].

**Edge Cases and Pitfalls:**
*   **Self-intersection removal works at voxel resolution:** Overlaps hidden by voxel averaging in partly occupied cells, or by negative winding numbers from volume inversions, can escape the objective [S1].
*   **Approximate nature:** The method is approximate; the resulting shapes may not fill the whole volume or be intersection-free, and are described as 'near-tileable' [S1].

## Subtopics

### Bandsaw Cutability Optimization
An optimization process that deforms a mesh to minimize the volume difference between the object and its extruded silhouettes from three orthogonal directions [S1].
*   **Steps:** Project the voxelized mesh along each of three orthogonal directions to obtain filled silhouettes. Extrude the silhouettes. Compare the resulting cut shape with the voxelized mesh to compute the objective [S1].
*   **Parameters:** An as-rigid-as-possible (ARAP) regularizer is used to discourage unnecessary deformation [S1].
*   **Performance:** On the TOSCA cat category, the average cut-volume difference (measured with the voxel-based objective) improves by 70.775%, going from $3.425 \times 10^3$ to $0.454 \times 10^3$ [S1]. Relative improvements for four examples (voxel-based comparison) are 76.0%, 69.9%, 58.7%, and 62.8% [S1].

### Space-Filling Tiling Optimization
Optimizing repeated shapes toward space-filling tilings [S1].
*   **Pitfalls:** The method is approximate; the resulting shapes may not fill the whole volume or be intersection-free, and are described as 'near-tileable' [S1].

### Self-Intersection Removal via Mesh Deformation
Deforming a mesh to reduce voxel regions with winding number above one to remove self-intersections [S1].
*   **Pitfalls:** The method is approximate; overlaps hidden by voxel averaging in partly occupied cells or negative winding numbers from volume inversions can escape the objective [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [G].
*   **Memory Complexity:** Not given in the sources [G].
*   **Measured Performance:** On a 2,418-triangle mesh and $32^3$ grid, voxelization takes 418.4 ms with DOPH and 2.2 ms with DVX [S1]. The authors report sub-second CPU voxelization and backward-mode gradient computation for meshes above 100,000 faces at $1024^3$ resolution, averaged over 30 runs on an Intel i9-12900K [S1].
*   **Scaling:** The method is reported to handle $1024^3$ resolution [S1].
*   **Strengths:** Provides a broader gradient pattern matching finite-difference references, unlike DOPH which yields sparse near-surface derivative signals [S1]. Efficient closed-form algorithm for triangle meshes with voxel-aligned box filters [S1].
*   **Limitations:** Approximate; self-intersection removal and tiling optimization may not be perfect [S1]. The reported timings are for CPU implementation, not fully optimized or parallelized, and GPU implementation is left to future work [S1].
*   **When to use:** When gradient-based optimization of surface representations based on volumetric properties is required, such as deforming meshes to resolve intersections, ensuring manufacturability, or creating shapes close to tiling 3D space [S1].
*   **When not to use:** When exact intersection-free or perfect space-filling shapes are required, as the method is approximate [S1].

## Background
*   **Origin:** The method is described in the paper "Differentiable Voxelization of Surface Representations" [S1].
*   **History:** The paper demonstrates the practical use of this approach in deforming meshes to resolve intersections, being manufacturable by cutting with a bandsaw from three directions, and creating shapes that are close to tiling 3D space [S1].
*   **Usage:** Used for deforming meshes to resolve intersections, ensuring manufacturability by cutting with a bandsaw from three directions, and creating shapes close to tiling 3D space [S1].

## Sources
[S1] Differentiable Voxelization of Surface Representations, https://www.alphaxiv.org/abs/2608.15934, fetched 2026-10-29
