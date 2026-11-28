# Pixel Ray Image (PRI)

Also called: Parametric Ray Integral Construction, Ray-cell intersection knot listing, PRI Spline Construction, PRI knot generation

## Overview
The Pixel Ray Image (PRI) is a semi-regular data structure for volume rendering of unstructured grids that replaces the irregular mesh with a pixelized projection plane and 1D linear splines defined on sampling rays [S1, S2, unofficial]. It solves the difficulty of mapping irregular unstructured grid data efficiently to GPU hardware by leveraging coherent sample placement and 2D texture storage, avoiding the cell-ordering bottlenecks of direct ray-casting [S1, S2, unofficial]. Unlike sibling methods like [GPU-based Ray Traversal on Tetrahedral Meshes](3165-gpu-based-ray-traversal-on-tetrahedral-meshes.md) which traverse the mesh directly, PRI pre-computes a texture-based representation [S1, S2, unofficial].

## How it works
**Inputs:** Unstructured grid $V$ with scalar values at vertices; camera parameters (position, up vector, view direction) [S1, S2, unofficial].
**Outputs:** A 2D texture containing packed linear splines representing the PRI [S2, unofficial].

**Data Structures:**
1. **Sampling Plane:** A 2D image plane with finite dimensions, pixelized with a regular grid of pixel centers at integer offsets from the origin [S1, S2, unofficial].
2. **Knot List:** For each pixel, a list of knots $K = [(s_0, t_0, p_0), (s_1, t_1, p_1), \dots, (s_n, t_n, p_n)]$ [S1, S2, unofficial].
   - $s$: Scalar value [S1, S2, unofficial].
   - $t$: Distance from the sampling plane along the ray [S1, S2, unofficial].
   - $p$: 3D intersection point [G].
   - Ordered such that $t_k < t_{k+1}$ [S1, S2, unofficial].
3. **PRI Texture:** A 2D texture storing the knot lists [S2, unofficial]. The specific packing format is detailed in [PRI Texture Storage](2493-pri-texture-storage.md).

**Steps:**

1. **Define Sampling Geometry:**
   - Compute a tight axis-aligned bounding box (AABB) around $V$ [S1, S2, unofficial].
   - Select one face of the AABB as the sampling plane [S1, S2, unofficial].
   - The associated axial direction of the box face defines the sampling direction (view direction) [S1, S2, unofficial].
   - The view direction must point towards the center of the grid [S1, S2, unofficial].
   - Set pixel resolution $R \times R$ (e.g., $256 \times 256$) [S1, S2, unofficial]. Resolution must be high enough that multiple rays pierce every cell face to ensure sufficient resampling [S1, S2, unofficial].

2. **Resample Grid onto Sampling Rays (Knot Generation):**
   - For each pixel $(u, v)$ on the sampling plane:
     - Cast a sampling ray $r$ into the grid [S1, S2, unofficial].
     - Find all intersections between $r$ and the grid cell faces [S1, S2, unofficial].
     - For each intersection, compute:
       - $t$: Distance from the pixel to the intersection [S1, S2, unofficial].
       - $s$: Reconstructed scalar value at the intersection location [S1, S2, unofficial].
         - If the scalar field is linear within cells, $s$ is interpolated from vertex values at the intersection point [S1, S2, unofficial].
         - If not linear, samples may be located uniformly along the ray, and $s$ is reconstructed using a filter [S1, S2, unofficial].
     - Insert the triple $(s, t, p)$ into the knot list for this pixel [S1, S2, unofficial].
   - *Alternative Implementation:* Use depth peeling on graphics hardware to render layers of cell faces in depth-sorted order and compute knot values $(s, t)$ directly on the GPU [S1, S2, unofficial].

3. **Handle Non-Convexity (Boundary Knots):**
   - For non-convex grids, a ray may enter and exit the grid multiple times [S1, S2, unofficial].
   - For every boundary knot $(s_b, t_b, p_b)$ (where the ray enters or exits the volume):
     - Insert an additional knot $(0, t_b, p_b)$ into the knot list immediately following $(s_b, t_b, p_b)$ [S1, S2, unofficial].
   - This creates consecutive pairs of knots with multiplicity two, bounding portions of the ray outside the grid with a scalar value of 0 [S1, S2, unofficial].

4. **Spline Simplification (Data Reduction):**
   - Apply a linear spline simplification algorithm to reduce the knot count per spline [S1, S2, unofficial].
   - The algorithm exploits coherence in the sampling direction [S1, S2, unofficial].
   - Consecutive runs of samples approximating a line in the scalar domain are collected into single linear segments [S1, S2, unofficial].
   - This is a greedy algorithm that may insert new samples into the final linear spline if necessary to maintain fidelity [S1, S2, unofficial].

5. **Pack and Store:**
   - Pack the entire collection of splines (knot lists) onto a 2D texture [S1, S2, unofficial].
   - Upload the texture to GPU memory once prior to rendering [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_PRI(grid, camera_params):
    # 1. Define Sampling Plane
    aabb_min, aabb_max = compute_aabb(grid)
    sampling_plane = select_aabb_face(aabb_min, aabb_max)
    view_direction = get_axial_direction(sampling_plane)
    grid_center = compute_center_of_grid(grid)
    assert dot(view_direction, grid_center - camera_params.position) > 0
    
    resolution = 256
    pixels = generate_pixel_grid(resolution, sampling_plane)
    
    pri_knot_lists = {}
    
    for pixel in pixels:
        ray_origin = pixel.position
        ray_direction = view_direction
        knots = []
        
        intersections = find_ray_cell_face_intersections(ray_origin, ray_direction, grid)
        
        for t, point in intersections:
            s = reconstruct_scalar_field(point, grid)
            knots.append((s, t, point))
            
        knots.sort(key=lambda k: k[1])
        
        processed_knots = []
        for s, t, point in knots:
            processed_knots.append((s, t, point))
            # Sources do not specify exact algorithm to check if a knot is on the grid boundary [G]
            if is_on_grid_boundary(point, grid):
                processed_knots.append((0, t, point))
                
        pri_knot_lists[pixel.index] = processed_knots
        
    simplified_knot_lists = {}
    for pixel_index, knots in pri_knot_lists.items():
        simplified_knots = simplify_linear_spline(knots)
        simplified_knot_lists[pixel_index] = simplified_knots
        
    pri_texture = pack_knots_to_texture(simplified_knot_lists)
    return pri_texture

def compute_aabb(grid):
    return min(v.position for v in grid.vertices), max(v.position for v in grid.vertices)

def select_aabb_face(aabb):
    return aabb[0]

def get_axial_direction(face):
    return face.normal

def compute_center_of_grid(grid):
    return sum(v.position for v in grid.vertices) / len(grid.vertices)

def generate_pixel_grid(res, plane):
    return [(u, v) for u in range(res) for v in range(res)]

def find_ray_cell_face_intersections(origin, direction, grid):
    intersections = []
    for face in grid.faces:
        t = ray_face_intersection(origin, direction, face)
        if t > 0:
            intersections.append((t, origin + t * direction))
    intersections.sort(key=lambda x: x[0])
    return intersections

def reconstruct_scalar_field(point, grid):
    return interpolate_scalar(point, grid.vertices)

def is_on_grid_boundary(point, grid):
    # Sources do not specify exact algorithm to check if a knot is on the grid boundary [G]
    return False

def pack_knots_to_texture(knot_lists):
    # Sources do not specify exact packing format [G]
    return None

def simplify_linear_spline(knots):
    if len(knots) <= 2:
        return knots
    simplified = [knots[0]]
    start_s, start_t = knots[0][0], knots[0][1]
    for i in range(1, len(knots) - 1):
        curr_s, curr_t = knots[i][0], knots[i][1]
        next_s, next_t = knots[i+1][0], knots[i+1][1]
        # Check if current point is collinear with start and next
        cross = (curr_s - start_s) * (next_t - start_t) - (curr_t - start_t) * (next_s - start_s)
        if abs(cross) > 1e-6:
            simplified.append(knots[i])
            start_s, start_t = knots[i][0], knots[i][1]
    simplified.append(knots[-1])
    return simplified

def is_collinear(s1, t1, s2, t2, s3, t3):
    cross = (s2 - s1) * (t3 - t1) - (t2 - t1) * (s3 - s1)
    return abs(cross) < 1e-6
```

**Edge Cases and Pitfalls:**
*   **Non-Convex Grids:** Rays may enter and exit multiple times. Failure to insert the $(0, t_b)$ boundary knots will result in incorrect rendering of regions outside the volume [S1, S2, unofficial].
*   **Resolution:** If the pixel resolution is too low, multiple rays may not pierce every cell face, leading to insufficient resampling and artifacts [S1, S2, unofficial].
*   **Scalar Field Linearity:** The method assumes linearity within cells for knot placement at face intersections. If the field is non-linear, uniform sampling and filtering are required, which may increase knot count [S1, S2, unofficial].

## Subtopics

### [PRI Texture Storage](2493-pri-texture-storage.md)
GPU-friendly data layout for the PRI, focusing on compact, texture-based representation of splines.

### [PRI Approximation Error Measurement](2893-pri-approximation-error-measurement.md)
Quantifies fidelity of the PRI representation relative to the source unstructured grid by evaluating sampling error at grid vertices.

## Operating characteristics
**Time Complexity:**
*   **Construction:** Depends on the number of pixels and intersections. Table 1 in [S1, unofficial] shows construction times for $256 \times 256$ resolution:
    *   Blunt Fin (187,395 tets): 29.20 seconds [S1, S2, unofficial].
    *   Combustor (215,040 tets): 86.04 seconds [S1, S2, unofficial].
    *   Liquid Oxygen Post (513,375 tets): 32.84 seconds [S1, S2, unofficial].
    *   SPX (12,936 tets): 4.39 seconds [S1, S2, unofficial].
*   **Rendering:** Efficient, using slice-based rendering similar to hardware-accelerated regular grid rendering [S1, S2, unofficial].

**Memory Complexity:**
*   Proportional to the number of pixels and the average number of knots per spline [S1, S2, unofficial].
*   Average knots per spline for $256 \times 256$ resolution (from [S1, unofficial]):
    *   Blunt Fin: 80 knots/spline [S1, S2, unofficial].
    *   Combustor: 63 knots/spline [S1, S2, unofficial].
    *   Liquid Oxygen Post: 85 knots/spline [S1, S2, unofficial].
    *   SPX: 27 knots/spline [S1, S2, unofficial].

**Strengths:**
*   Avoids cell-ordering bottlenecks of direct ray-casting [S1, S2, unofficial].
*   Leverages GPU hardware acceleration via 2D textures [S1, S2, unofficial].
*   Flexible sample placement [S1, unofficial].
*   Fewer samples needed than regular resampling of the grid [S1, unofficial].

**Limitations:**
*   Preprocessing time can be significant for large grids [S1, unofficial].
*   Requires sufficient pixel resolution to ensure multiple rays pierce every cell face [S1, S2, unofficial].
*   Restricted to sampling directions pointing towards the center of the grid (based on AABB face selection) [S1, S2, unofficial].

## Background
The Pixel Ray Image (PRI) was introduced by Shareef et al. in 2006 [S1, S2, unofficial]. It is an image-based approach designed to allow flexible sample placement and efficient mapping to graphics hardware [S1, S2, unofficial]. It builds on the idea of replacing the unstructured grid with a semi-regular data structure to leverage current trends in hardware performance [S1, S2, unofficial].

## Sources
[S1] An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume Rendering, https://doi.org/10.2312/vg/vg06/031-038, fetched 2026-10-24 (unofficial)
[S2] PDF An Image-Based Modelling Approach To GPU-based Unstructured Grid Volume ..., https://www3.cs.stonybrook.edu/~mueller/papers/vg06_ibr.pdf, fetched 2026-10-27 (unofficial)
