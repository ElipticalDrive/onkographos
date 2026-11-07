# Stochastic Ray-Casting Voxelization

Also called: Probabilistic Inside-Outside Test, Randomized Direction Voxelization

## Overview
Stochastic Ray-Casting Voxelization determines the interior status of voxels by casting multiple random rays from each voxel center and counting intersections with mesh triangles [S1, unofficial]. It differs from its siblings [Ray-Cast Style Voxel Filling](1502-ray-cast-style-voxel-filling.md) and [Front/Back Face Flagging](1506-front-back-face-flagging.md) by relying on statistical majority voting over randomized directions rather than deterministic axis-aligned rays or face orientation flags, making it robust against non-manifold geometry, broken meshes, and self-intersections where a single deterministic ray might fail [S1, unofficial].

## How it works
**Inputs:** A triangle mesh (vertices and indices), a target voxel grid resolution ($D_x \times D_y \times D_z$), the number of random samples per voxel ($S$), and a boolean flag for double-thick meshes ($D$) [S1, unofficial].
**Outputs:** A voxel grid where each voxel is marked as inside or outside [S1, unofficial].

**Data Structures:**
*   **Triangle:** Struct containing three `Vec3` vertices ($v_1, v_2, v_3$) [S1, unofficial].
*   **Vec3:** Struct with a union containing `m_pos[3]` and individual members `m_x`, `m_y`, `m_z`; `precision_type` defaults to `float` (`USE_DOUBLE false`) [S1, unofficial].
*   **VoxelGrid:** Boolean array `m_insideArray` of size $D_x \cdot D_y \cdot D_z$. Layout is Z-major; accessor `isInside(i, j, k)` returns `m_insideArray[k*(D_x*D_y)+j*D_y + i]` [S1, unofficial].

**Algorithm Steps:**
1.  **Grid Initialization:** Create `m_insideArray` initialized to `false` [S1, unofficial].
2.  **Voxel Loop:** Iterate through every voxel $(i, j, k)$ in the grid [S1, unofficial].
3.  **Voxel Center Calculation:** Compute the 3D position $P$ of the voxel center in world space [G].
4.  **Sample Loop:** Initialize `votes = 0`. Loop $j$ from $0$ to $S-1$:
    a.  **Generate Random Direction:** Generate a random unit vector $d$ uniformly distributed on the sphere using polar-to-Euclidean conversion [S1, unofficial].
    b.  **Ray Casting:** Initialize `intersections = 0` [S1, unofficial].
    c.  **Triangle Intersection Loop:** Loop over all triangles (index $i$ from $0$ to `numTriangles`) [S1, unofficial]. Check if the ray starting at $P$ with direction $d$ intersects the triangle [S1, unofficial]. If it intersects, increment `intersections` [S1, unofficial].
    d.  **Inside Check:** Determine if the voxel is inside based on `intersections` and the `double_thick` flag $D$ [S1, unofficial]. If `inside(intersections, double_thick)` returns true, increment `votes` [S1, unofficial].
5.  **Majority Vote:** After $S$ samples, if `votes > S / 2.0`, set the voxel to inside (`true`), otherwise outside (`false`) [S1, unofficial].

**Formulas:**
Random direction generation uses two uniform random numbers $u_1, u_2 \in [0, 1)$:
$$ \theta = u_1 \cdot 2\pi $$
$$ z = 2u_2 - 1 $$
$$ d_x = \sqrt{1 - z^2} \cos(\theta) $$
$$ d_y = \sqrt{1 - z^2} \sin(\theta) $$
$$ d_z = z $$
*Pitfall:* The reference implementation contains a bug where $d_z$ is incorrectly computed as $\sqrt{1 - z^2} \cos(\theta)$ instead of $z$, resulting in a non-uniform distribution biased towards the $xy$-plane [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def stochastic_ray_casting_voxelization(triangles, grid_dims, num_samples, double_thick=False):
    dx, dy, dz = grid_dims
    total_voxels = dx * dy * dz
    voxel_grid = [False] * total_voxels
    
    class Vec3:
        def __init__(self, x, y, z):
            self.x, self.y, self.z = x, y, z
        def __sub__(self, other): return Vec3(self.x - other.x, self.y - other.y, self.z - other.z)
        def dot(self, other): return self.x*other.x + self.y*other.y + self.z*other.z
        def cross(self, other): return Vec3(self.y*other.z - self.z*other.y, self.z*other.x - self.x*other.z, self.x*other.y - self.y*other.x)

    def ray_triangle_intersect(origin, direction, tri):
        v1v2 = tri.v2 - tri.v1
        v1v3 = tri.v3 - tri.v1
        n = v1v2.cross(v1v3)
        v1o = origin - tri.v1
        det = direction.dot(n)
        if abs(det) < 1e-8: return False
        inv_det = 1.0 / det
        u = v1o.dot(n) * inv_det
        if u < 0.0 or u > 1.0: return False
        v1o_cross_ray = v1o.cross(direction)
        v = v1v3.dot(v1o_cross_ray) * inv_det
        if v < 0.0 or u + v > 1.0: return False
        t = v1v2.dot(v1o_cross_ray) * inv_det
        return t > 0.0

    for k in range(dz):
        for j in range(dy):
            for i in range(dx):
                px, py, pz = (i + 0.5) / dx, (j + 0.5) / dy, (k + 0.5) / dz
                origin = Vec3(px, py, pz)
                votes = 0
                
                for _ in range(num_samples):
                    u1, u2 = random.random(), random.random()
                    theta = u1 * 2.0 * math.pi
                    z_val = 2.0 * u2 - 1.0
                    rho = math.sqrt(1.0 - z_val * z_val)
                    dir_x = rho * math.cos(theta)
                    dir_y = rho * math.sin(theta)
                    dir_z = z_val
                    direction = Vec3(dir_x, dir_y, dir_z)
                    
                    intersections = 0
                    for tri in triangles:
                        if ray_triangle_intersect(origin, direction, tri):
                            intersections += 1
                    
                    is_inside = False
                    if double_thick:
                        is_inside = ((intersections // 2) % 2 == 1)
                    else:
                        is_inside = (intersections % 2 == 1)
                    if is_inside:
                        votes += 1
                
                if votes > num_samples / 2.0:
                    voxel_grid[k * (dx * dy) + j * dy + i] = True
    return voxel_grid
```

**Parameters:**
*   `num_samples` ($S$): Number of random rays cast per voxel. Default is often 1 (deterministic) or higher (e.g., 5, 11) for broken meshes. Controlled by command-line option `-s` [S1, unofficial].
*   `double_thick` ($D$): Boolean flag. If true, the mesh is treated as double-thick (watertight), affecting the logic in the inside check. Controlled by command-line option `-d` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Direction Bias:** A common bug in implementations is calculating the $z$-component of the random direction using $\sqrt{1-z^2}\cos(\theta)$ instead of the linear random variable $z$, biasing the distribution towards the $xy$-plane [S1, unofficial].
*   **Precision:** Floating-point precision issues in ray-triangle intersection tests can lead to incorrect intersection counts, especially for voxels near mesh boundaries [G].
*   **Degenerate Input:** Non-watertight meshes with holes may require a higher number of samples to achieve a reliable majority vote [S1, unofficial].

## Subtopics

### [Solid Voxelization via 6-Direction Odd-Even Ray Parity](2320-solid-voxelization-via-6-direction-odd-even-ray-parity.md)
This method converts a normalized, watertight mesh into a solid voxel target by determining the inside/outside status of each voxel center using the parity of ray intersections along six axis-aligned directions. It differs from its parent [Stochastic Ray-Casting Voxelization](1067-stochastic-ray-casting-voxelization.md) by using deterministic, axis-aligned rays and a majority vote on parity rather than random rays and intersection counts, providing a robust, non-stochastic solution for solid interior filling.

### GPU Stochastic Voxelization with Parity Check
GPU-accelerated voxelization using CUDA that handles incomplete blocks via multiple random sampling directions and resolves double-thickness meshes using parity checks [S1, unofficial]. It uses the `-s` or `--samples` argument to specify the number of random directions tested per voxel block [S1, unofficial]. For occupancy determination, it computes `(num_intersections / 2) % 2` [S1, unofficial]. This parity check correctly handles meshes with double-thickness geometry by ignoring paired intersections [S1, unofficial]. GPU speedup on NVIDIA GTX970: 396x for teapot (2464 tris, 64³), 2750x for bunny (69664 tris, 64³) [S1, unofficial]. GPU implementation is 400-2750 times faster than the provided CPU sample executable [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Performance:** GPU implementations achieve significant speedups (400x-2750x) compared to CPU counterparts [S1, unofficial].
*   **Strengths:** Robustness to non-manifold geometry, broken meshes, and self-intersections [S1, unofficial].
*   **Limitations:** High computational cost due to multiple ray-triangle intersection tests per voxel [S1, unofficial]. Sensitive to the number of samples; too few samples may lead to incorrect classification for complex geometries [S1, unofficial].
*   **When to Use:** When dealing with non-manifold, broken, or self-intersecting meshes where deterministic methods fail [S1, unofficial].
*   **When Not to Use:** For simple, watertight meshes where deterministic methods (like [Solid Voxelization via 6-Direction Odd-Even Ray Parity](2320-solid-voxelization-via-6-direction-odd-even-ray-parity.md)) are faster and more accurate [S1, unofficial].

## Background
Stochastic Ray-Casting Voxelization is a technique used in computer graphics for converting polygonal meshes into voxel grids [S1, unofficial]. It is particularly useful in applications where mesh quality is uncertain, such as 3D scanning data or procedurally generated models [S1, unofficial]. The method was popularized in open-source voxelization tools like `voxelizer` by kctess5, which provides a GPU-accelerated implementation [S1, unofficial].

## Sources
[S1] kctess5/voxelizer, https://github.com/kctess5/voxelizer, fetched 2026-10-26 (unofficial)
