# VF-Ray-GPU

Also called: Memory Efficient GPU-Based Ray Casting

## Overview
VF-Ray-GPU is a GPGPU implementation of the VF-Ray algorithm designed for memory-efficient ray casting on unstructured tetrahedral volumes [S1]. It solves the problem of rendering large datasets on GPUs with limited memory by avoiding explicit adjacency structures (like `Use_set`) and pre-computed normals, instead computing face parameters and volume integrals on-the-fly [S1]. It sets itself apart from its parent, Visible Faces Driven Ray Casting, by fully parallelizing the visibility determination, projection, and ray traversal across CUDA kernels, utilizing a hash-based local memory buffer for traversal state to minimize global memory bandwidth [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** `vertList` (vertex coordinates and scalar values), `tetList` (tetrahedron vertex indices), `conTet` (tetrahedron connectivity), `extFaces` (pre-computed external faces) [S1].
*   **Outputs:** Rendered image [S1].

**Data Structures:**
1.  `vertList`: Stores $x, y, z$ coordinates and scalar value $s$ for each vertex [S1].
2.  `tetList`: Stores vertex IDs $v_i$ composing each tetrahedron [S1].
3.  `conTet`: Stores tetrahedron ID $t_i$ sharing a specific face [S1].
4.  `extFaces`: List of external faces, pre-computed similarly to Bunyk's method [S1].
5.  `visFaces`: List of visible faces storing tetrahedron ID $t_i$, face ID $f_i$, and interpolation parameters $z$ and $s$ [S1].
6.  `projVisFaces`: List of projected visible faces storing bounding boxes in screen space [S1].
7.  `computedFaces`: 1D buffer in CUDA local memory per thread. Stores $t_i$, $f_i$, and face parameters. Indexed by a hash table based on the $z$-coordinate centroid of the face. MinZ maps to the first position, maxZ to the last [S1].

**Face Identification:**
The vertices of face $f_i$ of tetrahedron $t_i$ are identified by indices $v_i$, $v_{(i+1)\mod 4}$, and $v_{(i+2)\mod 4}$ from `tetList`, avoiding explicit face storage [S1].

**Algorithm Steps:**
The algorithm is divided into three CUDA kernels [S1].

**Kernel 1: Find Visible Faces**
*   **Thread Organization:** One thread per external face. Total threads = number of external faces [S1].
*   **Process:**
    1.  Read external face from `extFaces` (texture memory) [S1].
    2.  Compute plane equation coefficients for $z$ and $s$ interpolation by solving two $3 \times 3$ linear systems [S1].
    3.  Perform visibility test: Compare the $z$-coordinate of the fourth vertex (not on the face) with the projected $z$-coordinate on the face using the computed interpolation parameters [S1].
    4.  If projected $z <$ fourth vertex $z$, the face is visible [S1].
    5.  Store $z$ and $s$ interpolation parameters in `visFaces` (global memory) [S1].
*   **Cost:** Less than 5% of total time and memory [S1].

**Kernel 2: Project Visible Faces**
*   **Thread Organization:** One thread per visible face [S1].
*   **Process:**
    1.  Read vertex coordinates from `vertList` (texture memory) using IDs from `tetList` (global memory) [S1].
    2.  Project visible faces to screen space [S1].
    3.  Compute bounding box of the projection [S1].
    4.  Store bounding box in `projVisFaces` (global memory) [S1].

**Kernel 3: Ray Casting**
*   **Thread Organization:** CUDA grid divided into blocks; one block per visible face. Each block divided into threads; each thread computes a small set of pixels [S1]. Threads finishing early are reused by the hardware driver [S1].
*   **Process:**
    1.  Read `visFaces` and `projVisFaces` [S1].
    2.  Determine visible face pixels using the bounding box and a point-in-triangle test (cross product) [S1].
    3.  For each pixel, trigger ray casting using the visible face as the first entry face [S1].
    4.  Traverse tetrahedra:
        *   Compute face parameters on-the-fly (replacing pre-computed normals/gradients) [S1].
        *   Store next faces in `computedFaces` buffer.
        *   Hash index is the $z$-coordinate centroid of the face [S1].
        *   Handle hash collisions: Check stored $t_i$ and $f_i$ to ensure the correct face is read [S1].
    5.  Compute volume rendering integral on-the-fly for each cell contribution using Bunyk's method [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
# vertList: array of (x, y, z, s)
# tetList: array of [v0, v1, v2, v3]
# extFaces: array of face indices
# visFaces: array of (t_id, f_id, z_param, s_param)
# projVisFaces: array of (t_id, f_id, min_x, min_y, max_x, max_y)

def kernel_find_visible_faces(ext_face_idx):
    # Get face vertices from tetList using extFace data
    # Face fi of tetrahedron ti uses vertices vi, v(i+1)%4, v(i+2)%4
    t_id, f_id = get_face_info(ext_face_idx)
    v0 = vertList[tetList[t_id][f_id]]
    v1 = vertList[tetList[t_id][(f_id + 1) % 4]]
    v2 = vertList[tetList[t_id][(f_id + 2) % 4]]
    v3 = vertList[tetList[t_id][(f_id + 3) % 4]] # Fourth vertex

    # Compute plane equation coefficients for z and s interpolation
    # Solve 3x3 linear systems for z and s
    z_coeffs = solve_linear_system(v0, v1, v2, v0.z, v1.z, v2.z)
    s_coeffs = solve_linear_system(v0, v1, v2, v0.s, v1.s, v2.s)

    # Project v3 onto the plane defined by v0, v1, v2
    projected_z = interpolate_z(v0, v1, v2, v3, z_coeffs)

    # Visibility test: back-face culling
    if projected_z < v3.z:
        # Store visible face parameters
        visFaces.append((t_id, f_id, z_coeffs, s_coeffs))

def kernel_project_visible_faces(visible_face_idx):
    t_id, f_id, z_coeffs, s_coeffs = visFaces[visible_face_idx]
    
    # Get vertex coordinates
    v0 = vertList[tetList[t_id][f_id]]
    v1 = vertList[tetList[t_id][(f_id + 1) % 4]]
    v2 = vertList[tetList[t_id][(f_id + 2) % 4]]

    # Project to screen space
    p0 = project_to_screen(v0)
    p1 = project_to_screen(v1)
    p2 = project_to_screen(v2)

    # Compute bounding box
    min_x = min(p0.x, p1.x, p2.x)
    min_y = min(p0.y, p1.y, p2.y)
    max_x = max(p0.x, p1.x, p2.x)
    max_y = max(p0.y, p1.y, p2.y)

    projVisFaces.append((t_id, f_id, min_x, min_y, max_x, max_y))

def kernel_ray_casting(pixel_x, pixel_y, visible_face_idx):
    # Check if pixel is inside the bounding box
    bbox = projVisFaces[visible_face_idx]
    if not (bbox.min_x <= pixel_x <= bbox.max_x and 
            bbox.min_y <= pixel_y <= bbox.max_y):
        return

    # Point-in-triangle test
    t_id, f_id, z_coeffs, s_coeffs = visFaces[visible_face_idx]
    v0 = vertList[tetList[t_id][f_id]]
    v1 = vertList[tetList[t_id][(f_id + 1) % 4]]
    v2 = vertList[tetList[t_id][(f_id + 2) % 4]]
    
    if not point_in_triangle(pixel_x, pixel_y, v0, v1, v2):
        return

    # Initialize ray
    ray_origin = screen_to_world(pixel_x, pixel_y)
    ray_direction = compute_ray_direction(pixel_x, pixel_y)
    
    # First entry face
    current_t_id = t_id
    current_f_id = f_id
    
    # Local buffer for computed faces
    computed_faces = [] # Fixed size buffer in local memory

    while true:
        # Get current face vertices
        v0 = vertList[tetList[current_t_id][current_f_id]]
        v1 = vertList[tetList[current_t_id][(current_f_id + 1) % 4]]
        v2 = vertList[tetList[current_t_id][(current_f_id + 2) % 4]]
        
        # Compute face parameters on-the-fly
        # Replace pre-computed normals/gradients
        face_normal = cross_product(v1 - v0, v2 - v0)
        
        # Find intersection with ray
        t, hit_point = ray_plane_intersection(ray_origin, ray_direction, v0, face_normal)
        
        if t < 0:
            break # Ray missed or went backwards

        # Compute integral contribution for the segment
        # Use Bunyk's method for integration
        contribution = compute_volume_integral(ray_origin, hit_point, current_t_id)
        accumulate_color(contribution)

        # Find next face
        # Compute centroid z for hashing
        centroid_z = (v0.z + v1.z + v2.z) / 3.0
        hash_index = map_z_to_buffer_index(centroid_z)
        
        # Check local buffer for cached face
        next_face = None
        if hash_index < len(computed_faces):
            cached = computed_faces[hash_index]
            if cached.t_id == current_t_id and cached.f_id != current_f_id:
                # Verify it's the correct face (handle collisions)
                # Check if this face is shared by current tetrahedron
                next_face = cached

        if next_face is None:
            # Compute next face explicitly
            # Determine which vertex was not hit (or use connectivity)
            # Find adjacent tetrahedron sharing the face
            # This replaces Use_set
            next_t_id, next_f_id = find_next_face(current_t_id, current_f_id, hit_point)
            
            # Store in local buffer
            if hash_index < len(computed_faces):
                computed_faces[hash_index] = (next_t_id, next_f_id, ...)
            else:
                computed_faces.append((next_t_id, next_f_id, ...))
            
            next_face = (next_t_id, next_f_id)
        
        current_t_id, current_f_id = next_face
        
        # Handle degeneracies by perturbing ray
        if is_degenerate(hit_point):
            ray_origin = hit_point + epsilon * ray_direction
        else:
            ray_origin = hit_point

def solve_linear_system(v0, v1, v2, val0, val1, val2):
    # Solve for plane equation ax + by + cz = d
    # Returns coefficients for interpolation
    pass

def interpolate_z(v0, v1, v2, v3, coeffs):
    # Project v3 onto plane of v0,v1,v2 using z interpolation
    pass

def project_to_screen(vertex):
    # Apply projection matrix
    pass

def point_in_triangle(x, y, v0, v1, v2):
    # Cross product test
    pass

def ray_plane_intersection(origin, direction, point, normal):
    # Standard ray-plane intersection
    pass

def compute_volume_integral(origin, hit, t_id):
    # On-the-fly integration using Bunyk's method
    pass

def find_next_face(t_id, f_id, hit_point):
    # Find adjacent tetrahedron and face
    # Uses conTet or vertex connectivity
    pass

def is_degenerate(hit_point):
    # Check if hit is on edge or vertex
    pass
```

## Operating characteristics
*   **Time Complexity:** Slower than HARC and HARC-Partial due to on-the-fly face parameter computations and lack of pre-computed normals/gradients [S1].
*   **Memory Complexity:** Very low. 38 bytes/tetrahedron [S1].
    *   Blunt Fin: 7,029 KB [S1].
    *   Oxygen Post: 19,494 KB [S1].
    *   Fighter+: 426 MB [S1].
    *   F16: 239 MB [S1].
*   **Performance (Hardware):** Intel Pentium Core 2 Duo 6400 (2.13 GHz), 2 GB RAM, nVidia GeForce 8800 Ultra (768 MB) [S1].
    *   Blunt Fin: 186 ms [S1].
    *   Oxygen Post: 370 ms [S1].
    *   Fighter+: 3,081 ms [S1].
    *   F16: 804 ms [S1].
*   **Scaling:** Handles large datasets (1.1M-1.9M vertices) that exceed the 768 MB memory of the GeForce 8800 Ultra for other algorithms [S1].
*   **Strengths:** Low memory footprint allows rendering of large datasets on limited hardware; ~4x faster and ~50% less memory than CPU-based VF-Ray [S1].
*   **Limitations:** Slower than HARC/HARC-Partial; approximate results for degenerate cases due to ray perturbation [S1].
*   **When to use:** When memory is the primary constraint and the dataset is too large for other GPU ray casting methods [S1].
*   **When not to use:** When maximum performance is required and memory is sufficient (use HARC) [S1].

## Background
*   **Origin:** Based on VF-Ray, described in [APCBRF07] A. Maximo et al., 'Memory efficient and robust software implementation of the raycast algorithm', WSCG’07 (2007) [S1].
*   **Development:** Implemented using CUDA to exploit GPGPU parallelism [S1].
*   **Usage:** Tested on Blunt Fin, Oxygen Post, Fighter, and F16 datasets [S1].

## Sources
[S1] Memory Efficient GPU-Based Ray Casting for Unstructured Volume Rendering, https://doi.org/10.2312/vg.vg-pbg08.155-162, fetched 2026-11-05
