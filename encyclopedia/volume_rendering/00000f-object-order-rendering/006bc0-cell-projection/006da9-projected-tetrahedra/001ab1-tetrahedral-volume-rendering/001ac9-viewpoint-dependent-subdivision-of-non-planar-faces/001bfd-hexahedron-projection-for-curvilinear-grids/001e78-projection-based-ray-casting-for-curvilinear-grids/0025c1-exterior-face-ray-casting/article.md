# Exterior-Face Ray Casting

Also called: Distributed-memory volume ray-casting, Parallel Volume Ray-Casting on Unstructured Grids, Visible Exterior Face Rendering

## Overview
Exterior-Face Ray Casting is a parallel volume rendering algorithm for unstructured grids (tetrahedral and hexahedral) that restricts ray casting to visible exterior faces of subvolumes, traversing elements until exiting another exterior face [S1, unofficial]. It sets itself apart from its parent, [Projection-Based Ray Casting for Curvilinear Grids](1756-projection-based-ray-casting-for-curvilinear-grids.md), by explicitly identifying and projecting only the visible exterior faces of partitioned subdomains to define the exact screen area for ray casting, thereby avoiding redundant traversal of invisible internal boundaries [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Unstructured grid data (elements, connectivity, scalar values), viewing position, transfer function (mapping scalar to color/opacity), and image resolution [S1, unofficial].
- **Outputs:** Final composited image [S1, unofficial].

**Data Structures:**
- **Exterior Face List:** A list of all faces on the boundary of the local subvolume [S1, unofficial]. Each face stores vertex coordinates, normal vector, and element connectivity [S1, unofficial].
- **Visible Face List:** A subset of the Exterior Face List containing only faces visible from the current viewing position [S1, unofficial].
- **Ray Buffer:** A two-dimensional array of linked lists [S2, unofficial]. Each processor maintains a local buffer corresponding to its assigned image region [S2, unofficial]. Incoming ray segments are sorted into linked lists by pixel address [S2, unofficial].

**Preprocessing:**
1. **View-Independent (Once):**
   - Find connectivity between elements [S1, unofficial].
   - Identify all exterior faces of the subvolume [S1, unofficial].
   - Calculate face normals [S1, unofficial].
2. **View-Dependent (Per View):**
   - Extract visible faces from the Exterior Face List by checking face normals against the viewing direction [S1, unofficial].
   - Orthographically project visible exterior faces onto the screen to produce a set of convex polygons in screen coordinates [S1, unofficial].

**Ray Casting Loop:**
1. Traverse the Visible Face List in scanline order [S1, unofficial].
2. For each visible face, cast rays from the eye [S1, unofficial].
3. A ray enters the data domain through an exterior face and marches element by element using connectivity information until it exits through another exterior face [S1, unofficial].
   - Due to concavity, holes, or saw-toothed partition boundaries, a ray may enter and exit multiple exterior faces [S1, unofficial].
4. **Segmentation:** Break the ray into segments based on element boundaries [S1, unofficial]. The compositing operation is associative, allowing segments to be processed independently [S1, unofficial].
5. **Sampling:**
   - **Adaptive (Expensive):** For a linear element with interpolation function $f(x,y,z)$, ray $P(t) = P(0) + td$, entry $a$, exit $b$, and constant $k$ (sampling rate for unit change in function value), the sample rate $r$ is defined as:
     $$ r = \frac{|f(P(b)) - f(P(a))|}{b - a} \cdot k $$
     This is noted as expensive and often not used in practice [S1, unofficial].
   - **Fixed (Practical):** Precompute a fixed sampling interval $dt$ [S1, unofficial]. A good principle for linear elements is to sample at least one additional point $c$ between entering point $a$ and leaving point $b$ [S1, unofficial].
     - Estimate $dt$ using:
       $$ dt = 0.5 \times \min\left(\frac{l_x}{n_e}, \frac{l_y}{n_e}, \frac{l_z}{n_e}\right) $$
       where $l_x, l_y, l_z$ are bounding box sizes and $n_e$ is the total number of elements [S2, unofficial].
       *Note:* Source [S1, unofficial] cites an alternative estimation: $dt = \frac{l_x + l_y + l_z}{n_e}$ [S1, unofficial].
     - Use the Trapezoid rule to calculate the value of the interval between sample points [S1, unofficial].
     - **Gaussian Quadrature Approximation:** For linear elements, approximate intensity by computing the value at the middle of the segment [S1, unofficial]. For quadratic elements, sample two points; for cubic elements, three points [S1, unofficial]. This approximates a polynomial of degree $2n-1$ using $n$ points [S1, unofficial].
6. **Color/Opacity Mapping:** At each sample point, use the interpolated function value to obtain color and opacity from a look-up table [S1, unofficial].
7. **Intensity Calculation:** Compute intensity for a segment from entry $a$ to exit $b$ using:
   $$ I(a, b) = \int_a^b e^{-\int_a^s \alpha(s') ds'} I(t) dt $$
   where $\alpha$ (or $c$) is the attenuation coefficient and $I(t)$ is the intensity at point $t$ [S1, S2, unofficial].
8. **Compositing:** Form the final image value by compositing front-to-back the intensity values of sample points along the ray [S1, unofficial].
9. **Early Ray Termination:** Terminate a ray when the accumulated opacity value reaches unity [S2, unofficial].

**Parallel Compositing:**
- Ray segments contributing to a pixel cannot be combined until all are received and sorted in visibility order by the responsible processor [S2, unofficial].
- **Approach 1:** Separate local rendering from global compositing [S2, unofficial]. Send all segments after local tracing finishes [S2, unofficial]. Causes large messages and network congestion [S2, unofficial].
- **Approach 2:** Overlap rendering and compositing [S2, unofficial]. Send segments immediately after generation [S2, unofficial]. Improves overall image compositing performance by as much as 40% [S2, unofficial].
- **Image Partitioning:** Image space is partitioned evenly into horizontal strips for simplicity [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exterior_face_ray_casting(grid, eye_pos, transfer_func, img_w, img_h):
    # 1. View-Independent Preprocessing (Done Once)
    exterior_faces = []
    for elem in grid.elements:
        for face_idx in range(elem.num_faces):
            face = elem.faces[face_idx]
            if not face.is_shared:
                exterior_faces.append(face)
                face.normal = cross_product(face.edge1, face.edge2)
                
    # 2. View-Dependent Preprocessing (Per View)
    visible_faces = []
    view_dir = normalize(eye_pos)
    for face in exterior_faces:
        if dot(face.normal, view_dir) > 0:
            visible_faces.append(face)
            
    screen_polygons = {}
    for face in visible_faces:
        screen_polygons[face] = orthographic_project(face.vertices, eye_pos, img_w, img_h)
        
    # 3. Initialize Ray Buffer for Parallel Compositing
    ray_buffer = [[[] for _ in range(img_w)] for _ in range(img_h)]
    
    # 4. Ray Casting Loop
    for face in visible_faces:
        for pixel in screen_polygons[face]:
            ray_origin = eye_pos
            ray_dir = normalize(pixel - eye_pos)
            segments = trace_ray_through_elements(ray_origin, ray_dir, grid)
            
            segment_results = []
            for seg in segments:
                samples = sample_segment(seg.entry, seg.exit, seg.dir, seg.grid, transfer_func)
                
                intensity = 0.0
                alpha = 0.0
                
                for s in samples:
                    c, a = transfer_func(s.value)
                    # Front-to-back compositing
                    alpha_out = alpha + (1.0 - alpha) * a
                    intensity_out = intensity + (1.0 - alpha) * c * a
                    intensity, alpha = intensity_out, alpha_out
                    
                    if alpha >= 1.0:
                        break
                
                segment_results.append({
                    'x': pixel.x, 'y': pixel.y,
                    'intensity': intensity, 'opacity': alpha,
                    'depth': seg.depth
                })
                
            for res in segment_results:
                ray_buffer[res['y']][res['x']].append(res)
                
    # 7. Global Compositing
    final_image = composite_ray_buffer(ray_buffer, img_w, img_h)
    return final_image

def sample_segment(a, b, ray_dir, grid, transfer_func):
    dt = 0.5 * min(grid.bbox.x / grid.num_elements, 
                   grid.bbox.y / grid.num_elements, 
                   grid.bbox.z / grid.num_elements)
    samples = []
    t = a
    while t <= b:
        point = a + t * ray_dir
        val = interpolate_grid(grid, point)
        samples.append({'point': point, 'value': val})
        t += dt
        if t > b:
            t = b
    return samples

def trace_ray_through_elements(origin, direction, grid):
    segments = []
    current_pos = origin
    current_dir = direction
    max_steps = 1000
    for _ in range(max_steps):
        hit_elem, hit_t, hit_normal = find_first_element_intersection(current_pos, current_dir, grid)
        if hit_elem is None:
            break
        seg_exit = current_pos + hit_t * current_dir
        segments.append({'entry': current_pos, 'exit': seg_exit, 'dir': current_dir, 'depth': hit_t})
        current_pos = seg_exit + 1e-6 * current_dir
        current_dir = reflect_or_transmit(current_dir, hit_normal, hit_elem)
    return segments

def composite_ray_buffer(buffer, w, h):
    image = [[(0.0, 0.0) for _ in range(w)] for _ in range(h)]
    for y in range(h):
        for x in range(w):
            segs = sorted(buffer[y][x], key=lambda s: s['depth'])
            intensity, alpha = 0.0, 0.0
            for s in segs:
                alpha_out = alpha + (1.0 - alpha) * s['opacity']
                intensity_out = intensity + (1.0 - alpha) * s['intensity']
                intensity, alpha = intensity_out, alpha_out
            image[y][x] = (intensity, alpha)
    return image
```

## Subtopics

### [Visible Faces Driven Ray Casting](15651-visible-faces-driven-ray-casting.md)
A memory-efficient variant that reduces memory footprint by storing only the face data of the most recent traversals, exploiting ray coherence among neighboring pixels. It sets itself apart from Exterior-Face Ray Casting by explicitly managing a temporary buffer to cache internal face parameters for the current visible set, thereby avoiding redundant geometric computations for rays traversing the same internal boundaries.

## Operating characteristics
- **Time Complexity:** Dependent on the number of visible exterior faces and the number of elements traversed per ray [S1, S2, unofficial]. Adaptive sampling increases complexity significantly [S1, S2, unofficial].
- **Memory Complexity:** Requires storage for the exterior face list, visible face list, and the Ray Buffer [S2, unofficial]. The Ray Buffer size grows during rendering and depends on viewing position, data shape, and partitioning [S2, unofficial].
- **Measured Performance:** Overlapping ray casting and compositing can improve overall image compositing performance by as much as 40% [S2, unofficial].
- **Scaling:** Load balancing is affected by early ray termination [S2, unofficial]. Subvolumes with larger projected areas of high-density regions (high opacity) cause rays to terminate earlier, traversing fewer elements, leading to uneven load distribution [S2, unofficial].
- **Strengths:** Efficiently handles unstructured grids with concavities and holes; parallelizable via associative compositing [S1, S2, unofficial].
- **Limitations:** Adaptive sampling is expensive; Ray Buffer management can be complex; load balancing issues due to variable ray traversal lengths [S1, S2, unofficial].

## Background
The method was developed for parallel volume rendering on distributed-memory architectures using unstructured grids [S1, S2, unofficial]. It addresses the challenges of rendering irregular, partitioned data volumes where traditional regular grid ray casting is not applicable [S1, S2, unofficial].

## Sources
[S1] DTIC ADA301855: Parallel Volume Ray-Casting for Unstructured-Grid Data on Distributed-Memory Architectures., https://archive.org/details/DTIC_ADA301855, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 19960003512: Parallel volume ray-casting for unstructured-grid data on distributed-memory architectures, https://archive.org/details/NASA_NTRS_Archive_19960003512, fetched 2026-10-26 (unofficial)
