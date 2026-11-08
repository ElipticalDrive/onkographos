# Dual-Cell DDA Raycasting for Particles

Also called: Dual-Cell DDA, Ray-Traced Mirror and Glass Particles

## Overview
A ray-tracing optimization for particle volumes that uses a 3D Digital Differential Analyzer (DDA) over dual cells to efficiently test for ray-sphere intersections, enabling real-time rendering of glossy mirrors and refractive glass particles [S1, unofficial]. It distinguishes itself from standard voxel raycasting by leveraging a pre-sorted neighbor grid and a 2x2x2 cell expansion strategy to guarantee hit detection within a dual cell without exhaustive search, significantly reducing shading overhead for deep particle piles [S1, unofficial].

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
*   Particle Radius: Must be less than half a cell size to guarantee the 2x2x2 test coverage [S1, unofficial].

**Algorithm Steps:**
1.  **Ray Initialization:** Cast primary rays from the camera. If `rt=on`, assign materials: 30% glossy black mirrors, 35% glass (IOR 1.5, Fresnel), and 35% matte grey [S1, unofficial].
2.  **DDA Traversal:** Perform a 3D DDA over the dual cells of the fluid's neighbor grid [S1, unofficial].
3.  **Dual Cell Testing:** For each DDA step at a current dual cell, test the 8 surrounding grid cells (forming a 2x2x2 block) for ray-sphere intersections [S1, unofficial].
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
def dual_cell_dda_raytrace(ray_origin, ray_direction, bounces, max_bounces):
    # Initialize DDA state for dual cells
    current_dual_cell = get_dual_cell(ray_origin)
    
    for bounce in range(max_bounces):
        # Step 1: DDA traversal over dual cells
        hit_sphere = False
        hit_point = None
        hit_normal = None
        hit_material = None
        
        # Step 2: Test 2x2x2 grid cells surrounding the current dual cell
        surrounding_cells = get_surrounding_cells(current_dual_cell)
        
        for cell in surrounding_cells:
            # Test all spheres in this grid cell
            for sphere_index in get_spheres_in_cell(cell):
                sphere_pos = sorted_positions_buffer[sphere_index]
                sphere_radius = get_radius(sphere_index)
                
                # Ray-sphere intersection test
                t, normal = ray_sphere_intersect(ray_origin, ray_direction, sphere_pos, sphere_radius)
                
                if t is not None and t > 0:
                    hit_sphere = True
                    hit_point = ray_origin + ray_direction * t
                    hit_normal = normal
                    hit_material = get_material(sphere_index)
                    break
            if hit_sphere:
                break
        
        if hit_sphere:
            # Step 3: Material interaction
            if hit_material == "mirror":
                # Reflect ray
                ray_direction = reflect(ray_direction, hit_normal)
                ray_origin = hit_point + epsilon * ray_direction
            elif hit_material == "glass":
                # Refract ray (IOR 1.5)
                ray_direction = refract(ray_direction, hit_normal, 1.0, 1.5)
                ray_origin = hit_point + epsilon * ray_direction
            else:
                # Matte grey: diffuse shading or termination
                return shade_diffuse(hit_point, hit_normal)
            
            # Recurse or continue loop for bounces
            if bounce < max_bounces - 1:
                return dual_cell_dda_raytrace(ray_origin, ray_direction, bounces, max_bounces)
            else:
                return shade_diffuse(hit_point, hit_normal)
        else:
            # Step 4: Room intersection
            return shade_room_geometry(ray_origin, ray_direction)
            
    return shade_diffuse(ray_origin, ray_direction)
```

**Edge Cases and Pitfalls:**
*   **Depth Testing:** Writing `frag_depth` before the depth test causes the fragment shader (including ray tracing) to run unnecessarily for occluded layers in deep piles, degrading performance from ~16.7 ms to ~92 ms per frame [S1, unofficial].
*   **Radius Constraint:** The guarantee of finding every hit relies on the particle radius being strictly less than half the cell size [S1, unofficial].

## Operating characteristics
*   **Performance:** Reduces frame time from approximately 92 ms to 16.7 ms (vsync) for scenes with 8192 particles by avoiding full shading for occluded layers via impostor depth quads [S1, unofficial].
*   **Complexity:** Time complexity depends on the number of DDA steps and the density of spheres in the 2x2x2 neighborhood. Memory complexity is linear with the number of particles for the sorted buffers [S1, unofficial].
*   **Strengths:** Efficient handling of deep particle piles; supports complex materials (mirror, glass) in real-time; additive implementation requires no core restructuring [S1, unofficial].
*   **Limitations:** Limited to spherical particles; requires pre-sorted data structures; performance degrades if particle radius approaches half cell size [S1, unofficial].

## Background
This method was introduced in the "voxel-gi-particles" project as a bonus ray-tracing mode for real-time global illumination with particles [S1, unofficial]. It was developed to enable glossy and refractive materials in a voxel-based GI system without prohibitive performance costs, leveraging the spatial coherence of particle distributions [S1, unofficial].

## Sources
[S1] voxel-gi-particles: the article's lightbox look, plus its ray-traced mirror and glass bonus, https://github.com/Siroko/kansei/pull/108, fetched 2026-10-26 (unofficial)
