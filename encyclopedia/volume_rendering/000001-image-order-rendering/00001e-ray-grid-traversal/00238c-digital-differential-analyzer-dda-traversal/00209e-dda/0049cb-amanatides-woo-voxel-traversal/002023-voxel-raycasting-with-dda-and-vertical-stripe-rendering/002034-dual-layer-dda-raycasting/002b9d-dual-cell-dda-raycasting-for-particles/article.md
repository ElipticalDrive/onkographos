# Dual-Cell DDA Raycasting for Particles

Also called: Dual-Cell DDA, Ray-Traced Mirror and Glass Particles

## Overview
A ray-tracing optimization for particle volumes that uses a 3D Digital Differential Analyzer (DDA) over dual cells to efficiently test for ray-sphere intersections, enabling real-time rendering of glossy mirrors and refractive glass particles [S1, unofficial]. It distinguishes itself from standard voxel raycasting by leveraging a pre-sorted neighbor grid and a 2×2×2 cell expansion strategy to guarantee hit detection within a dual cell without exhaustive search, significantly reducing shading overhead for deep particle piles [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin and direction; particle system data including `sorted_positions_buffer`, `sorted_indices_buffer`, `cell_offsets_buffer`, `cell_size`, and `grid_origin` [S1, unofficial].
*   **Output:** Final pixel color, accounting for reflections, refractions, and room geometry intersections [S1, unofficial].

**Data Structures:**
The method relies on additive accessors added to the `FluidSimulation` structure, requiring no restructuring of existing files [S1, unofficial]:
*   `sorted_positions_buffer`: Array of particle positions, sorted spatially.
*   `sorted_indices_buffer`: Array of indices corresponding to the sorted positions.
*   `cell_offsets_buffer`: Offsets defining the neighbor grid topology.
*   `cell_size`: Scalar size of a single grid cell.
*   `grid_origin`: Origin point of the grid.

**Parameters:**
*   `set_rt_bounces`: Integer controlling ray bounce depth. Default is 2 [S1, unofficial].
*   Particle count: Default 8192 larger particles [S1, unofficial].
*   Particle Radius: Must be less than half a cell size to guarantee the 2×2×2 test coverage [S1, unofficial].

**Algorithm Steps:**
1.  **Ray Initialization:** Cast primary rays from the camera [G]. If `rt=on`, assign materials: 30% glossy black mirrors, 35% glass (IOR 1.5, Fresnel), and 35% matte grey [S1, unofficial].
2.  **DDA Traversal:** Perform a 3D DDA over the dual cells of the fluid's neighbor grid [S1, unofficial].
3.  **Dual Cell Testing:** For each DDA step at a current dual cell, test the 8 surrounding grid cells (forming a 2×2×2 block) for ray-sphere intersections [S1, unofficial].
    *   *Geometric Guarantee:* Since particle radius $r < \frac{1}{2} \times \text{cell\_size}$, any sphere intersecting the dual cell must intersect at least one of the 8 surrounding grid cells [S1, unofficial].
4.  **Material Interaction:**
    *   If a sphere is hit, compute reflection (for mirrors) or refraction (for glass, exiting the far side) [S1, unofficial].
    *   Recursively cast new rays up to `set_rt_bounces` [S1, unofficial].
5.  **Room Intersection:** If a ray misses all spheres during traversal, it intersects the room geometry [S1, unofficial].
    *   Determine color based on the panel's mean irradiance, walls lit by the panel plus one voxel cone, or black for the open front [S1, unofficial].
    *   The sky gradient substitutes for the ceiling and open front [S1, unofficial].
6.  **Impostor Depth Optimization:** For depth testing, render camera-facing quads at the sphere's nearest point. Do not write `frag_depth` to avoid running the fragment shader (including ray tracing) before the depth test, which is critical for deep piles [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_sphere_intersect(ox, oy, oz, dx, dy, dz, sx, sy, sz, r):
    lx, ly, lz = ox - sx, oy - sy, oz - sz
    a = dx*dx + dy*dy + dz*dz
    b = 2.0 * (lx*dx + ly*dy + lz*dz)
    c = lx*lx + ly*ly + lz*lz - r*r
    disc = b*b - 4*a*c
    if disc < 0: return None, None
    t = (-b - disc**0.5) / (2*a)
    if t < 1e-6: t = (-b + disc**0.5) / (2*a)
    if t < 1e-6: return None, None
    nx, ny, nz = (ox + dx*t - sx)/r, (oy + dy*t - sy)/r, (oz + dz*t - sz)/r
    return t, (nx, ny, nz)

def dual_cell_dda_raytrace(ray_origin, ray_direction, max_bounces):
    x, y, z = ray_origin
    dx, dy, dz = ray_direction
    cell_size = 1.0  # Not given in the sources.
    grid_origin = (0.0, 0.0, 0.0)  # Not given in the sources.
    
    # DDA initialization
    step_x = 1 if dx > 0 else -1
    step_y = 1 if dy > 0 else -1
    step_z = 1 if dz > 0 else -1
    t_max_x = (grid_origin[0] + (1 if dx > 0 else 0) * cell_size - x) / dx if dx != 0 else float('inf')
    t_max_y = (grid_origin[1] + (1 if dy > 0 else 0) * cell_size - y) / dy if dy != 0 else float('inf')
    t_max_z = (grid_origin[2] + (1 if dz > 0 else 0) * cell_size - z) / dz if dz != 0 else float('inf')
    t_delta_x = cell_size / dx if dx != 0 else float('inf')
    t_delta_y = cell_size / dy if dy != 0 else float('inf')
    t_delta_z = cell_size / dz if dz != 0 else float('inf')
    cx = int((x - grid_origin[0]) / cell_size)
    cy = int((y - grid_origin[1]) / cell_size)
    cz = int((z - grid_origin[2]) / cell_size)
    
    for bounce in range(max_bounces):
        hit_t = float('inf')
        hit_normal = None
        hit_material = None
        
        # Test 2x2x2 grid cells surrounding the current dual cell
        for ox in range(cx, cx + 2):
            for oy in range(cy, cy + 2):
                for oz in range(cz, cz + 2):
                    # Retrieve spheres in this cell
                    start_idx = cell_offsets_buffer[ox, oy, oz]
                    end_idx = cell_offsets_buffer[ox, oy, oz + 1]
                    for i in range(start_idx, end_idx):
                        sphere_idx = sorted_indices_buffer[i]
                        sx, sy, sz = sorted_positions_buffer[sphere_idx]
                        r = 0.4 * cell_size  # Not given in the sources.
                        
                        t, normal = ray_sphere_intersect(x, y, z, dx, dy, dz, sx, sy, sz, r)
                        if t is not None and t < hit_t:
                            hit_t = t
                            hit_normal = normal
                            hit_material = "mirror" if sphere_idx % 10 < 3 else ("glass" if sphere_idx % 10 < 7 else "grey")
                        
                        if hit_t < 1e-6: break
                    if hit_t < 1e-6: break
                if hit_t < 1e-6: break
            if hit_t < 1e-6: break
            if hit_t < 1e-6: break
        
        if hit_t is not None and hit_t < float('inf'):
            x, y, z = x + dx * hit_t, y + dy * hit_t, z + dz * hit_t
            if hit_material == "mirror":
                dot = dx*hit_normal[0] + dy*hit_normal[1] + dz*hit_normal[2]
                dx, dy, dz = dx - 2*dot*hit_normal[0], dy - 2*dot*hit_normal[1], dz - 2*dot*hit_normal[2]
            elif hit_material == "glass":
                dot = dx*hit_normal[0] + dy*hit_normal[1] + dz*hit_normal[2]
                eta = 1.0 / 1.5
                cos_i = -dot
                sin2_t = eta*eta * (1.0 - cos_i*cos_i)
                if sin2_t > 1.0:
                    dot = dx*hit_normal[0] + dy*hit_normal[1] + dz*hit_normal[2]
                    dx, dy, dz = dx - 2*dot*hit_normal[0], dy - 2*dot*hit_normal[1], dz - 2*dot*hit_normal[2]
                else:
                    dx, dy, dz = dx*eta + (dx*eta*cos_i - (1.0 - sin2_t)**0.5)*hit_normal[0], \
                                  dy*eta + (dy*eta*cos_i - (1.0 - sin2_t)**0.5)*hit_normal[1], \
                                  dz*eta + (dz*eta*cos_i - (1.0 - sin2_t)**0.5)*hit_normal[2]
            else:
                return (0.5, 0.5, 0.5)
            
            x += dx * 1e-4
            y += dy * 1e-4
            z += dz * 1e-4
        else:
            return shade_room_geometry(x, y, z, dx, dy, dz)
            
    return (0.0, 0.0, 0.0)

def shade_room_geometry(x, y, z, dx, dy, dz):
    # Determines color based on panel irradiance, wall lit by panel + voxel cone, or black for open front
    # Sky gradient substitutes for ceiling and open front
    return (0.0, 0.0, 0.0)  # Not given in the sources.
```

**Edge Cases and Pitfalls:**
*   **Depth Testing:** Writing `frag_depth` before the depth test causes the fragment shader (including ray tracing) to run unnecessarily for occluded layers in deep piles, degrading performance from ~92 ms to ~16.7 ms per frame [S1, unofficial].
*   **Radius Constraint:** The guarantee of finding every hit relies on the particle radius being strictly less than half the cell size [S1, unofficial].

## Operating characteristics
*   **Performance:** Reduces frame time from approximately 92 ms to 16.7 ms (vsync) for scenes with 8192 particles by avoiding full shading for occluded layers via impostor depth quads [S1, unofficial].
*   **Strengths:** Supports complex materials (mirror, glass) in real-time; additive implementation requires no core restructuring [S1, unofficial].
*   **Limitations:** Limited to spherical particles; requires pre-sorted data structures; performance degrades if particle radius approaches half cell size [S1, unofficial].

## Background
This method was introduced in the "voxel-gi-particles" project as a bonus ray-tracing mode for real-time global illumination with particles [S1, unofficial].

## Sources
[S1] voxel-gi-particles: the article's lightbox look, plus its ray-traced mirror and glass bonus, https://github.com/Siroko/kansei/pull/108, fetched 2026-10-26 (unofficial)
