# Exterior-Face Ray Casting

Also called: Distributed-memory volume ray-casting, Parallel Volume Ray-Casting on Unstructured Grids, Visible Exterior Face Rendering

## Overview
Exterior-Face Ray Casting is a parallel volume rendering technique for unstructured grids that restricts ray casting to visible exterior faces of subvolumes, traversing elements until exiting another exterior face [S1, S2, unofficial]. It distinguishes itself from sibling methods by explicitly separating view-independent connectivity preprocessing from view-dependent face visibility culling, enabling efficient distributed compositing via associative ray segmentation [S1, S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Unstructured grid data (elements, nodes, connectivity).
*   Scalar field values at nodes.
*   Transfer function (mapping scalar values to color and opacity).
*   Camera parameters (eye position, viewing direction).
*   Image resolution and partitioning scheme.

**Outputs:**
*   Final composited image (subimages per processor, merged globally).

### Data Structures

**Exterior Face List:**
*   A list of all faces on the boundary of the local subvolume.
*   Each face entry contains:
    *   Vertex indices (defining the polygon).
    *   Face normal vector (precomputed in view-independent step).
    *   Connectivity to adjacent elements (for traversal).

**Ray Buffer:**
*   A two-dimensional array of linked lists.
*   Dimensions correspond to the image resolution assigned to the processor.
*   Each entry points to a linked list of ray segments contributing to that pixel.
*   Segments are sorted by depth (visibility order) upon insertion or at compositing time [S2, unofficial].

**Element Connectivity:**
*   Adjacency graph of elements within the subvolume.
*   Used to traverse from one element to the next along a ray [S1, S2, unofficial].

### Preprocessing

1.  **View-Independent Preprocessing (Done Once):**
    *   Determine connectivity between elements within the subvolume [S1, S2, unofficial].
    *   Identify all exterior faces (faces with no adjacent element in the subvolume) [S1, S2, unofficial].
    *   Calculate face normals for all exterior faces [S1, S2, unofficial].

2.  **View-Dependent Preprocessing (Per View):**
    *   Iterate through the exterior face list.
    *   Test visibility of each face from the current eye position (e.g., using dot product of face normal and view vector).
    *   Collect all visible exterior faces into a "visible face list" [S1, S2, unofficial].
    *   Orthographically project these visible faces onto the screen to determine the exact screen area (pixels) for ray casting [S1, unofficial].

### Ray Casting Loop

1.  **Initialization:**
    *   For each pixel in the projected area of each visible exterior face, generate a ray from the eye [S1, S2, unofficial].
    *   Initialize ray parameters: origin $P(0)$, direction $d$, current parameter $t=0$ [S2, unofficial].

2.  **Entry Detection:**
    *   Find the intersection of the ray with the visible exterior face.
    *   Determine the entry point $a$ (parameter $t_a$) [S1, S2, unofficial].

3.  **Element Traversal:**
    *   While the ray has not exited the data domain:
        *   Identify the current element the ray is traversing [S1, S2, unofficial].
        *   Determine the exit point $b$ (parameter $t_b$) from the current element [S1, S2, unofficial].
        *   **Segmentation:** Define the ray segment from $a$ to $b$ [S1, unofficial].
        *   **Sampling:** Sample the scalar field along the segment $[t_a, t_b]$ (see Sampling below) [S1, S2, unofficial].
        *   **Local Compositing:** Compute the intensity contribution of this segment using front-to-back compositing [S1, S2, unofficial].
        *   **Segment Output:** Store the segment's contribution (or intermediate state) for later global compositing [S1, S2, unofficial].
        *   Update $a = b$ [S1, S2, unofficial].
        *   Use connectivity to find the next element [S1, S2, unofficial].
        *   If the ray exits the subvolume through an exterior face, add this segment to the Ray Buffer for the corresponding pixel(s) and terminate local traversal for this ray [S1, S2, unofficial].

4.  **Exit Handling:**
    *   If the ray exits through an exterior face, it may re-enter the domain if the geometry is concave or if the partition boundary is saw-toothed. In such cases, the ray is treated as a new ray entering at the new exterior face [S1, S2, unofficial].

### Sampling

**Fixed Sampling (Recommended Implementation):**
*   Precompute a fixed sampling interval $dt$ [S1, S2, unofficial].
*   Estimate $dt$ using:
    $$ dt = 0.5 \times \min\left(\frac{l_x}{n_e}, \frac{l_y}{n_e}, \frac{l_z}{n_e}\right) $$
    where $l_x, l_y, l_z$ are the bounding box sizes in x, y, z directions, and $n_e$ is the total number of elements in the domain [S1, S2, unofficial].
    *   *Note:* Source [S1, unofficial] also mentions an alternative estimation: $dt = \frac{l_x + l_y + l_z}{n_e}$ [S1, unofficial]. The minimum-based formula is cited as generating "reasonably good sampling interval values" [S2, unofficial].
*   Sample at least one additional point $c$ between entry $a$ and exit $b$ for linear elements [S1, S2, unofficial].
*   Use the **Trapezoid Rule** to calculate the value of the interval between sample points [S1, S2, unofficial].

**Adaptive Sampling (Expensive):**
*   For a linear element with interpolation function $f(x,y,z)$, ray $P(t) = P(0) + td$, entry $a$, exit $b$:
    $$ r = \frac{|f(P(b)) - f(P(a))|}{b - a} \cdot k $$
    where $k$ is a constant sampling rate for a unit change in function value [S1, S2, unofficial].
*   *Note:* Adaptive sampling is noted as significantly increasing rendering time and is generally avoided in favor of fixed sampling [S1, S2, unofficial].

**Gaussian Quadrature Approximation:**
*   For non-linear elements, Gaussian Quadrature is used:
    *   Linear element: Sample at the middle of the segment (1 point).
    *   Quadratic element: Sample 2 points.
    *   Cubic element: Sample 3 points.
*   This approximates a polynomial of degree $2n-1$ using $n$ points, providing precision order $2n-1$ [S1, S2, unofficial].
*   *Note:* Since transfer functions are usually not polynomials, multiple samples may still be needed even for linear elements if the transfer function varies rapidly [S1, S2, unofficial].

### Compositing

**Intensity Calculation:**
*   At each sample point, retrieve color and opacity from the transfer function look-up table [S1, S2, unofficial].
*   Compute intensity $I(t)$ at point $t$ [S1, S2, unofficial].
*   The intensity $I(a, b)$ of a ray segment from entry $a$ to exit $b$ is given by:
    $$ I(a, b) = \int_a^b e^{-\int_a^t \alpha(s) ds} I(t) dt $$
    where $\alpha$ is the attenuation coefficient and $I(t)$ is the intensity at point $t$ [S1, S2, unofficial].
    *   *Note:* Source [S2, unofficial] uses $c$ for attenuation coefficient in the formula: $I (a, b) = \int_a^b e^{-\int_a^s c ds'} I(t) dt$. The notation differs but the concept is identical [S1, S2, unofficial].

**Associative Compositing:**
*   The compositing operation is associative, allowing rays to be broken into segments [S1, S2, unofficial].
*   Each segment is processed independently (sampling and local compositing) [S1, S2, unofficial].
*   Segments are combined via a final compositing step [S1, S2, unofficial].

**Parallel Compositing Strategies:**
1.  **Separate Local Rendering and Global Compositing:**
    *   Local ray tracing finishes completely.
    *   All ray segments are sent to responsible processors.
    *   Segments are sorted by visibility order.
    *   Global compositing occurs.
    *   *Drawback:* Large messages, network congestion [S2, unofficial].
2.  **Overlap Rendering and Compositing:**
    *   Ray segments are sent immediately after generation.
    *   Responsible processors sort and composite segments as they arrive.
    *   *Benefit:* Improves overall image compositing performance by up to 40% [S2, unofficial].

**Ray Buffer Management:**
*   Incoming ray segments are sorted into the linked list pointed to by the corresponding pixel address in the Ray Buffer [S2, unofficial].
*   Sorted ray segments are composited in order at the end of rendering [S2, unofficial].
*   Final subimages are sent back to the host computer [S2, unofficial].

**Early Ray Termination:**
*   A ray is terminated when the accumulated opacity value reaches unity [S2, unofficial].
*   This affects load balancing: subvolumes with larger projected areas of high-density regions cause rays to terminate earlier, traversing fewer elements [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exterior_face_ray_casting(grid, transfer_function, camera, image_width, image_height, num_processors):
    # 1. View-Independent Preprocessing (Done Once)
    exterior_faces = find_exterior_faces(grid)
    for face in exterior_faces:
        face.normal = compute_normal(face.vertices)
        face.adjacent_element = find_adjacent_element(face, grid)

    # 2. View-Dependent Preprocessing (Per View)
    visible_faces = []
    for face in exterior_faces:
        if is_visible(face, camera.eye_position):
            visible_faces.append(face)

    # Project visible faces to screen to determine pixel coverage
    screen_pixels = project_faces_to_screen(visible_faces, camera, image_width, image_height)

    # 3. Fixed Sampling Parameter Estimation
    bounding_box = get_bounding_box(grid)
    n_e = grid.num_elements
    l_x, l_y, l_z = bounding_box.width, bounding_box.height, bounding_box.depth
    dt = 0.5 * min(l_x / n_e, l_y / n_e, l_z / n_e)

    # 4. Ray Casting Loop
    for face in visible_faces:
        for pixel in screen_pixels[face]:
            ray = create_ray(camera.eye_position, pixel, camera)
            current_t = 0.0
            current_element = find_element_at(ray.origin, grid) # Initial element if inside
            
            # If ray starts outside, find first intersection with exterior face
            if current_element is None:
                intersection = find_intersection(ray, face)
                if intersection is None:
                    continue
                current_t = intersection.t
                current_element = face.adjacent_element
            
            ray_segments = []
            
            while current_element is not None:
                # Find exit point from current element
                exit_intersection = find_exit_intersection(ray, current_element, current_t)
                if exit_intersection is None:
                    break # Ray escaped domain without hitting another element boundary
                
                t_exit = exit_intersection.t
                segment_start_t = current_t
                segment_end_t = t_exit
                
                # Sampling
                samples = []
                t_sample = segment_start_t
                while t_sample < segment_end_t:
                    point = ray.origin + ray.direction * t_sample
                    value = interpolate_value(point, current_element, grid)
                    color, opacity = transfer_function(value)
                    samples.append((point, color, opacity))
                    t_sample += dt
                
                # Add final point if not already included
                if samples[-1].point != exit_intersection.point:
                    point = exit_intersection.point
                    value = interpolate_value(point, current_element, grid)
                    color, opacity = transfer_function(value)
                    samples.append((point, color, opacity))
                
                # Local Compositing for Segment
                segment_intensity = composite_segment(samples)
                
                # Store segment for global compositing
                ray_segments.append({
                    'pixel': pixel,
                    'depth': segment_start_t, # For sorting
                    'intensity': segment_intensity,
                    'opacity': compute_total_opacity(samples)
                })
                
                # Move to next element
                current_t = t_exit
                current_element = find_next_element(ray, current_element, t_exit, grid)
                
                # Early Ray Termination
                if segment_intensity.opacity >= 1.0:
                    break
            
            # Send segments to Ray Buffer
            for seg in ray_segments:
                ray_buffer[seg['pixel']].append(seg)

    # 5. Global Compositing
    for pixel in all_pixels:
        segments = ray_buffer[pixel]
        segments.sort(key=lambda s: s['depth']) # Sort by visibility order
        final_color = composite_segments(segments)
        image[pixel] = final_color

    return image
```

### Edge Cases and Pitfalls
*   **Concavity and Holes:** Rays may enter and exit multiple exterior faces due to concavity, holes, or saw-toothed partition boundaries [S1, S2, unofficial]. The algorithm must handle re-entry correctly [S1, S2, unofficial].
*   **Degenerate Elements:** Casting is straightforward except for degenerate elements, which require special handling [S1, S2, unofficial].
*   **Load Imbalance:** Subvolumes with larger projected areas of high-density regions cause rays to terminate earlier, leading to load imbalance [S2, unofficial].
*   **Network Congestion:** Sending all segments after local rendering can cause large messages and network congestion [S2, unofficial]. Overlapping rendering and compositing mitigates this [S2, unofficial].
*   **Precision:** Fixed sampling may over-sample large elements with constant values [S2, unofficial]. Adaptive sampling is more accurate but expensive [S1, S2, unofficial].

## Operating characteristics

**Time Complexity:**
*   Preprocessing: $O(F)$ where $F$ is the number of exterior faces [S1, S2, unofficial].
*   Ray Casting: Proportional to the number of pixels and the average number of elements traversed per ray [S1, S2, unofficial].
*   Compositing: Depends on the number of ray segments per pixel and the sorting cost [S2, unofficial].

**Memory Complexity:**
*   Ray Buffer: Size depends on the number of ray segments per pixel, which varies with viewing position, data shape, and partitioning [S2, unofficial].
*   Data Structures: Connectivity and exterior face lists are stored in memory [S1, S2, unofficial].

**Measured Performance:**
*   Overlapping ray casting and compositing can improve overall image compositing performance by up to 40% [S2, unofficial].
*   Adaptive sampling significantly increases rendering time compared to fixed sampling [S1, S2, unofficial].

**Strengths:**
*   Efficient culling of invisible faces [S1, S2, unofficial].
*   Parallelizable via associative compositing [S1, S2, unofficial].
*   Handles unstructured grids and complex geometries [S1, S2, unofficial].

**Limitations:**
*   Load imbalance due to varying ray traversal lengths [S2, unofficial].
*   Network congestion if compositing is not overlapped [S2, unofficial].
*   Complexity of handling concave and partitioned boundaries [S1, S2, unofficial].

**When to Use:**
*   Parallel volume rendering of unstructured grids on distributed-memory architectures [S1, S2, unofficial].
*   When efficient culling of invisible geometry is critical [S1, S2, unofficial].

**When Not to Use:**
*   Interactive rendering rates may require more elaborate image partitioning [S2, unofficial].
*   If network bandwidth is severely limited and overlapping compositing is not feasible [S2, unofficial].

## Background

**Origin:**
*   Developed for parallel volume rendering of unstructured-grid data on distributed-memory architectures [S1, S2, unofficial].
*   First described in 1996 [S1, S2, unofficial].

**History and Development:**
*   The method builds on the principles of volume ray casting and parallel rendering [S1, S2, unofficial].
*   It addresses the challenges of irregular shapes, concavities, and holes in unstructured domains, which are exacerbated by data partitioning [S1, S2, unofficial].

**Applications:**
*   Scientific visualization of unstructured grid data [S1, S2, unofficial].
*   Distributed-memory parallel rendering systems [S1, S2, unofficial].

## Sources

[S1] DTIC ADA301855: Parallel Volume Ray-Casting for Unstructured-Grid Data on Distributed-Memory Architectures., https://archive.org/details/DTIC_ADA301855, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19960003512: Parallel volume ray-casting for unstructured-grid data on distributed-memory architectures, https://archive.org/details/NASA_NTRS_Archive_19960003512, fetched 2026-10-26 (unofficial)
