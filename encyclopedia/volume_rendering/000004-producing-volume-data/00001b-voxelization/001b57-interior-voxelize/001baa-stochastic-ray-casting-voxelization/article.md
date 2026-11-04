# Stochastic Ray-Casting Voxelization

Also called: Probabilistic Inside-Outside Test, Randomized Direction Voxelization

## Overview
Stochastic Ray-Casting Voxelization determines the interior status of voxels by casting multiple random rays from each voxel center and counting intersections with the mesh triangles. It differs from its sibling [Ray-Cast Style Voxel Filling](1502-ray-cast-style-voxel-filling.md) and [Front/Back Face Flagging](1506-front-back-face-flagging.md) by relying on statistical majority voting over randomized directions rather than deterministic axis-aligned rays or face orientation flags, making it robust against non-manifold geometry, broken meshes, and self-intersections where a single deterministic ray might fail [S1, unofficial].

## How it works
**Inputs:** A watertight or non-watertight triangle mesh (vertices and indices), a target voxel grid resolution ($N \times N \times N$ or $D_x \times D_y \times D_z$), the number of random samples per voxel ($S$), and a boolean flag for double-thick meshes ($D$).
**Outputs:** A voxel grid where each voxel is marked as inside or outside.

**Data Structures:**
*   **Triangle:** A struct containing three 3D vertices ($v_1, v_2, v_3$). Each vertex is a `Vec3` with components $x, y, z$ stored as floating-point numbers (typically `float`) [S1, unofficial].
*   **VoxelGrid:** A 3D boolean array `m_insideArray` of size $D_x \cdot D_y \cdot D_z$. The layout is Z-major (or row-major depending on implementation, but typically accessed as `grid[k][y][x]` or linear index `k*(D_x*D_y) + y*D_x + x` where $k$ is the z-index) [S1, unofficial].
*   **Random State:** A pseudo-random number generator (PRNG) state, often seeded per voxel or globally, to generate uniform random numbers [S1, unofficial].

**Algorithm Steps:**
1.  **Grid Initialization:** Create a boolean array for the voxel grid, initialized to `false` (outside).
2.  **Voxel Loop:** Iterate through every voxel $(i, j, k)$ in the grid.
3.  **Voxel Center Calculation:** Compute the 3D position $P$ of the voxel center in world space.
    $$ P = \left( \frac{i + 0.5}{D_x}, \frac{j + 0.5}{D_y}, \frac{k + 0.5}{D_z} \right) $$
    *Note: This assumes the mesh is normalized to the unit cube $[0,1]^3$. If not, scale $P$ accordingly.*
4.  **Sample Loop:** Initialize `votes = 0`. Loop $s$ from $0$ to $S-1$:
    a.  **Generate Random Direction:** Generate a random unit vector $d$ uniformly distributed on the sphere.
        *   Generate two uniform random numbers $u_1, u_2 \in [0, 1)$ from the PRNG.
        *   Calculate spherical coordinates:
            $$ \theta = 2\pi u_1 $$
            $$ z = 2u_2 - 1 $$
            $$ \rho = \sqrt{1 - z^2} $$
        *   Convert to Cartesian coordinates:
            $$ d_x = \rho \cos(\theta) $$
            $$ d_y = \rho \sin(\theta) $$
            $$ d_z = z $$
        *   *Pitfall:* Some implementations incorrectly compute $d_z$ as $\rho \cos(\theta)$ or similar, leading to biased distributions. The correct $z$ component is the linear random variable $z$ itself [S1, unofficial].
    b.  **Ray Casting:** Initialize `intersections = 0`.
    c.  **Triangle Intersection Loop:** For each triangle $T$ in the mesh:
        *   Check if the ray $R(t) = P + t \cdot d$ ($t > 0$) intersects triangle $T$.
        *   If an intersection exists, increment `intersections`.
    d.  **Inside Check:** Determine if the voxel is inside based on `intersections` and the `double_thick` flag $D$.
        *   If $D$ is false (standard single-thick mesh):
            $$ \text{is\_inside} = (\text{intersections} \pmod 2 == 1) $$
        *   If $D$ is true (double-thick/watertight mesh):
            $$ \text{is\_inside} = ((\text{intersections} / 2) \pmod 2 == 1) $$
            *Note: Integer division is used here to handle paired intersections from double-sided geometry.*
        *   If `is_inside` is true, increment `votes`.
5.  **Majority Vote:** After $S$ samples, if `votes > S / 2`, set `m_insideArray[i][j][k] = true`. Otherwise, it remains `false`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def stochastic_ray_casting_voxelization(mesh_triangles, grid_dims, num_samples, double_thick=False):
    """
    Voxelize a mesh using stochastic ray casting.
    
    Args:
        mesh_triangles: List of Triangle objects, each with v1, v2, v3 (Vec3).
        grid_dims: Tuple (dx, dy, dz) representing grid resolution.
        num_samples: Number of random rays to cast per voxel.
        double_thick: Boolean flag for double-sided mesh handling.
        
    Returns:
        3D boolean array (dx, dy, dz) indicating inside voxels.
    """
    dx, dy, dz = grid_dims
    voxel_grid = [[[False for _ in range(dx)] for _ in range(dy)] for _ in range(dz)]
    
    # Helper: Ray-Triangle Intersection (Möller-Trumbore)
    def ray_triangle_intersect(ray_origin, ray_dir, tri):
        # Vector from v1 to v2
        v1v2 = Vec3_sub(tri.v2, tri.v1)
        # Vector from v1 to v3
        v1v3 = Vec3_sub(tri.v3, tri.v1)
        # Normal to the triangle
        n = Vec3_cross(v1v2, v1v3)
        
        # Distance from v1 to ray origin
        v1o = Vec3_sub(ray_origin, tri.v1)
        
        # Determinant
        det = Vec3_dot(ray_dir, n)
        
        # If det is near zero, ray is parallel to triangle
        if abs(det) < 1e-8:
            return False
        
        inv_det = 1.0 / det
        
        # Calculate u parameter
        u = Vec3_dot(v1o, n) * inv_det
        if u < 0.0 or u > 1.0:
            return False
            
        # Prepare to test v parameter
        v1o_cross_ray = Vec3_cross(v1o, ray_dir)
        
        # Calculate v parameter
        v = Vec3_dot(v1v3, v1o_cross_ray) * inv_det
        if v < 0.0 or u + v > 1.0:
            return False
            
        # Calculate t, distance from ray origin to intersection
        t = Vec3_dot(v1v2, v1o_cross_ray) * inv_det
        
        # t must be positive (ray direction)
        if t <= 0.0:
            return False
            
        return True

    # Helper: Vector operations
    class Vec3:
        def __init__(self, x, y, z):
            self.x = x
            self.y = y
            self.z = z
        def __sub__(self, other):
            return Vec3(self.x - other.x, self.y - other.y, self.z - other.z)
        def __add__(self, other):
            return Vec3(self.x + other.x, self.y + other.y, self.z + other.z)
        def __mul__(self, scalar):
            return Vec3(self.x * scalar, self.y * scalar, self.z * scalar)
        def dot(self, other):
            return self.x * other.x + self.y * other.y + self.z * other.z
        def cross(self, other):
            return Vec3(
                self.y * other.z - self.z * other.y,
                self.z * other.x - self.x * other.z,
                self.x * other.y - self.y * other.x
            )

    def Vec3_sub(a, b): return a - b
    def Vec3_cross(a, b): return a.cross(b)
    def Vec3_dot(a, b): return a.dot(b)

    # Main Loop
    for k in range(dz):
        for j in range(dy):
            for i in range(dx):
                # 1. Voxel Center
                # Assuming mesh is in [0,1]^3. If not, scale coordinates.
                px = (i + 0.5) / dx
                py = (j + 0.5) / dy
                pz = (k + 0.5) / dz
                origin = Vec3(px, py, pz)
                
                votes = 0
                
                # 2. Sample Loop
                for s in range(num_samples):
                    # Generate random direction uniformly on sphere
                    u1 = random.random()
                    u2 = random.random()
                    
                    theta = 2.0 * math.pi * u1
                    z_val = 2.0 * u2 - 1.0
                    rho = math.sqrt(1.0 - z_val * z_val)
                    
                    dir_x = rho * math.cos(theta)
                    dir_y = rho * math.sin(theta)
                    dir_z = z_val # Correct z component
                    
                    direction = Vec3(dir_x, dir_y, dir_z)
                    
                    # 3. Ray Casting
                    intersections = 0
                    for tri in mesh_triangles:
                        if ray_triangle_intersect(origin, direction, tri):
                            intersections += 1
                    
                    # 4. Inside Check
                    is_inside = False
                    if double_thick:
                        # Double-thick: ignore paired intersections
                        if (intersections // 2) % 2 == 1:
                            is_inside = True
                    else:
                        # Standard: odd number of intersections
                        if intersections % 2 == 1:
                            is_inside = True
                            
                    if is_inside:
                        votes += 1
                
                # 5. Majority Vote
                if votes > num_samples / 2.0:
                    voxel_grid[k][j][i] = True

    return voxel_grid
```

**Parameters:**
*   `num_samples` ($S$): Number of random rays per voxel. Default is often 1 (deterministic) or higher (e.g., 5, 11) for broken meshes [S1, unofficial].
*   `double_thick` ($D$): Boolean flag. If true, the mesh is treated as double-thick (watertight), affecting the logic in the inside check [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Direction Bias:** A common bug in implementations is calculating the $z$-component of the random direction using $\cos(\theta)$ instead of the linear random variable $z$. This biases the distribution towards the x-y plane projection [S1, unofficial].
*   **Precision:** Floating-point precision issues in ray-triangle intersection tests can lead to incorrect intersection counts, especially for voxels near mesh boundaries.
*   **Degenerate Input:** Non-watertight meshes with holes may require a higher number of samples to achieve a reliable majority vote.

## Subtopics

### [Solid Voxelization via 6-Direction Odd-Even Ray Parity](2320-solid-voxelization-via-6-direction-odd-even-ray-parity.md)
This method converts a normalized, watertight mesh into a solid voxel target by determining the inside/outside status of each voxel center using the parity of ray intersections along six axis-aligned directions. It differs from its parent [Stochastic Ray-Casting Voxelization](1067-stochastic-ray-casting-voxelization.md) by using deterministic, axis-aligned rays and a majority vote on parity rather than random rays and intersection counts, providing a robust, non-stochastic solution for solid interior filling.

### GPU Stochastic Voxelization with Parity Check
GPU-accelerated voxelization using CUDA that handles incomplete blocks via multiple random sampling directions and resolves double-thickness meshes using parity checks. It uses the `-s` or `--samples` argument to specify the number of random directions tested per voxel block. For occupancy determination, it computes `(num_intersections / 2) % 2`. This parity check correctly handles meshes with double-thickness geometry by ignoring paired intersections. GPU speedup on NVIDIA GTX970: 396x for teapot (2464 tris, 64³), 2750x for bunny (69664 tris, 64³). GPU implementation is 400-2750 times faster than the provided CPU sample executable [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(V \cdot S \cdot T)$, where $V$ is the number of voxels, $S$ is the number of samples per voxel, and $T$ is the number of triangles. This is significantly slower than deterministic methods for simple meshes but necessary for complex/non-manifold ones.
*   **Memory Complexity:** $O(V + T)$, for storing the voxel grid and the mesh triangles.
*   **Performance:** The method is computationally expensive due to the large number of ray-triangle intersection tests. However, GPU implementations can achieve significant speedups (400x-2750x) compared to CPU counterparts [S1, unofficial].
*   **Strengths:** Robustness to non-manifold geometry, broken meshes, and self-intersections. It does not require the mesh to be watertight or manifold.
*   **Limitations:** High computational cost. Results are probabilistic and may vary slightly between runs if the PRNG seed is not fixed. Sensitive to the number of samples; too few samples may lead to incorrect classification for complex geometries.
*   **When to Use:** When dealing with non-manifold, broken, or self-intersecting meshes where deterministic methods fail.
*   **When Not to Use:** For simple, watertight meshes where deterministic methods (like [Solid Voxelization via 6-Direction Odd-Even Ray Parity](2320-solid-voxelization-via-6-direction-odd-even-ray-parity.md)) are faster and more accurate.

## Background
Stochastic Ray-Casting Voxelization is a technique used in computer graphics for converting polygonal meshes into voxel grids. It is particularly useful in applications where mesh quality is uncertain, such as 3D scanning data or procedurally generated models. The method was popularized in open-source voxelization tools like `voxelizer` by kctess5, which provides a GPU-accelerated implementation [S1, unofficial].

## Sources
[S1] kctess5/voxelizer, https://github.com/kctess5/voxelizer, fetched 2026-10-26 (unofficial)
