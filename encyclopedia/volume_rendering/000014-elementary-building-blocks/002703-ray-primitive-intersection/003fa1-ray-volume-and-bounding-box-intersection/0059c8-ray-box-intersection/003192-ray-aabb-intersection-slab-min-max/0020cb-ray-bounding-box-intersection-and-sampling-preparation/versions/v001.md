# Ray-Bounding Box Intersection and Sampling Preparation

Also called: Coarse Geometry Hit Check, OccupancyGrid Sample Occupied Regions, Voxel Count Views (Training Ray Sampling)

## Overview
This method computes the intersection interval of camera rays with a 3D axis-aligned bounding box (AABB), determines the number of sample points required along each ray based on a fixed step distance, and generates the corresponding 3D query coordinates for volume rendering. It serves as the bridge between ray generation and the actual volume sampling, ensuring that points are generated only within the valid scene bounds and at the correct spatial resolution. It differs from its parent [Ray-box intersection](776-ray-box-intersection.md) by adding the discretization logic and point generation, and from [Ray Point Sampling](2382-ray-point-sampling.md) by handling the bounding box culling and ray indexing logic explicitly.

## How it works
**Inputs:**
- `rays_o`: Ray origins, shape $(N, 3)$ or flattened batch.
- `rays_d`: Ray directions, shape $(N, 3)$ or flattened batch.
- `xyz_min`: Bounding box minimum corner, shape $(3,)$.
- `xyz_max`: Bounding box maximum corner, shape $(3,)$.
- `near`: Near clipping plane distance (scalar or tensor).
- `far`: Far clipping plane distance (scalar or tensor).
- `stepsize`: Base step size parameter.
- `voxel_size`: Physical size of a voxel (scalar).
- `rng`: Random number generator or jitter flag (optional).

**Outputs:**
- `ray_pts`: Query points, shape $(TotalPoints, 3)$.
- `mask_outbbox`: Boolean mask for points outside the box, shape $(TotalPoints,)$.
- `ray_id`: Array mapping each point to its source ray index, shape $(TotalPoints,)$.
- `hit`: Boolean array indicating which rays had valid intersections, shape $(N,)$.

**Data Structures:**
- `N_steps`: Array of length $N$ storing the number of samples per ray.
- `N_steps_cumsum`: Cumulative sum of `N_steps`, used for indexing into the flattened point array.
- `ray_id`: Array of length $TotalPoints$ storing the ray index for each point.
- `step_id`: Array of length $TotalPoints$ storing the step index (0 to $N_{samples}-1$) for each point.

**Steps:**

1.  **Batching and Preprocessing:**
    Split rays into batches (e.g., size 10,000) to manage memory. If downsampling (`downrate`) is applied, subsample the rays before flattening. Flatten `rays_o` and `rays_d` to $(-1, 3)$ and ensure they are contiguous in memory [S2, unofficial].

2.  **Handle Zero Direction Components:**
    To avoid division by zero during intersection calculation, replace any zero components in `rays_d` with a small epsilon $\epsilon = 10^{-6}$ [S2, unofficial].
    $$
    v_x = \begin{cases} 
    10^{-6} & \text{if } rays\_d[i] == 0 \\
    rays\_d[i] & \text{otherwise}
    \end{cases}
    $$
    This is applied element-wise to the direction vector [S2, unofficial].

3.  **Compute Intersection Bounds ($t_{min}, t_{max}$):**
    Calculate the intersection parameters for the AABB. For each ray, compute the intersection with the six planes of the box.
    Let $vec$ be the clamped direction vector.
    $$
    rate\_a = \frac{xyz_{max} - rays\_o}{vec}, \quad rate\_b = \frac{xyz_{min} - rays\_o}{vec}
    $$
    Compute the entry and exit distances for each axis:
    $$
    t_{near, axis} = \min(rate\_a_{axis}, rate\_b_{axis})
    $$
    $$
    t_{far, axis} = \max(rate\_a_{axis}, rate\_b_{axis})
    $$
    The global $t_{min}$ and $t_{max}$ for the ray are:
    $$
    t_{min} = \max(t_{near, x}, t_{near, y}, t_{near, z})
    $$
    $$
    t_{max} = \min(t_{far, x}, t_{far, y}, t_{far, z})
    $$
    Clamp these values to the camera's near and far planes:
    $$
    t_{min} = \text{clamp}(t_{min}, near, far)
    $$
    $$
    t_{max} = \text{clamp}(t_{max}, near, far)
    $$
    If $t_{min} > t_{max}$, the ray does not intersect the box [S2, unofficial].

4.  **Compute Number of Samples ($n_{samples}$):**
    Calculate the number of samples required for each ray based on the interval length and the desired step distance.
    First, compute the physical step distance:
    $$
    stepdist = stepsize \times voxel\_size
    $$
    (Note: Some implementations include a random jitter factor `rng` in the step size calculation [S2, unofficial]).
    Calculate the norm of the ray direction:
    $$
    r_{norm} = \sqrt{rays\_d[0]^2 + rays\_d[1]^2 + rays\_d[2]^2}
    $$
    The number of samples is:
    $$
    n_{samples} = \max\left(\left\lceil \frac{(t_{max} - t_{min}) \times r_{norm}}{stepdist} \right\rceil, 1.0\right)
    $$
    The $\max(\dots, 1.0)$ ensures at least one sample per ray to simplify subsequent indexing [S2, unofficial]. Store these in `N_steps`.

5.  **Generate Sample Positions ($t_{samples}$):**
    Create normalized positions in $[0, 1]$ using `linspace` with length $n_{samples}$. Map these to the clamped interval:
    $$
    t_{samples} = t_{min} + t_{normalized} \times (t_{max} - t_{min})
    $$
    If randomization is enabled, add uniform jitter within each bin:
    $$
    jitter = \text{torch.rand\_like}(t_{samples}) \times bin\_size
    $$
    $$
    bin\_size = \frac{t_{max} - t_{min}}{n_{samples}}
    $$
    $$
    t_{samples} = t_{samples} + jitter
    $$
    [S1, unofficial].

6.  **Compute Ray Start and Normalized Direction:**
    Calculate the starting position of the ray segment inside the box and the normalized direction vector:
    $$
    rays_{start} = rays\_o + rays\_d \times t_{min}
    $$
    $$
    rays_{dir} = \frac{rays\_d}{r_{norm}}
    $$
    [S2, unofficial].

7.  **Flatten and Index Points:**
    Calculate the total number of points across all rays:
    $$
    total\_len = \sum N_{steps}
    $$
    Compute the cumulative sum of `N_steps` to determine the start index of each ray's points in the flattened array:
    $$
    N_{steps\_cumsum}[i] = \sum_{j=0}^{i} N_{steps}[j]
    $$
    Initialize `ray_id` array of size $total\_len$ with zeros. Mark the start of each ray's segment:
    $$
    ray\_id[N_{steps\_cumsum}[i-1]] = 1 \quad \text{for } i > 0
    $$
    Perform an in-place cumulative sum on `ray_id` to assign the correct ray index to each point:
    $$
    ray\_id[idx] = \sum_{k=0}^{idx} ray\_id[k]
    $$
    [S2, unofficial].

    Compute `step_id` for each point. For index $idx$, let $rid = ray\_id[idx]$. The starting index for ray $rid$ is $start\_idx = N_{steps\_cumsum}[rid-1]$ if $rid > 0$, else $0$.
    $$
    step\_id[idx] = idx - start\_idx
    $$
    [S2, unofficial].

8.  **Generate Query Points:**
    For each index $idx$, retrieve the ray index $i_{ray} = ray\_id[idx]$ and step index $i_{step} = step\_id[idx]$.
    Calculate the distance along the normalized direction:
    $$
    dist = stepdist \times i_{step}
    $$
    Compute the 3D point:
    $$
    p = rays_{start}[i_{ray}] + rays_{dir}[i_{ray}] \times dist
    $$
    [S2, unofficial].

9.  **Bounding Box Culling Mask:**
    Check if the generated point lies outside the bounding box.
    $$
    mask_{outbbox}[idx] = (xyz_{min}[0] > p[0]) \lor (xyz_{min}[1] > p[1]) \lor (xyz_{min}[2] > p[2]) \lor (xyz_{max}[0] < p[0]) \lor (xyz_{max}[1] < p[1]) \lor (xyz_{max}[2] < p[2])
    $$
    [S2, unofficial].

10. **Hit Detection:**
    Initialize a `hit` tensor of zeros (bool) with length $N$ (number of rays). A ray is considered to have "hit" if $t_{min} \le t_{max}$ after clamping [S2, unofficial].

**Pitfalls:**
- **Division by Zero:** Ray direction components can be zero. Clamping to $10^{-6}$ is required [S2, unofficial].
- **Empty Intersections:** If $t_{min} > t_{max}$, the ray misses the box. The logic must handle this to avoid generating invalid samples or negative sample counts.
- **Single Sample Guarantee:** Ensuring $n_{samples} \ge 1$ prevents empty arrays for rays that barely intersect the box [S2, unofficial].
- **Precision:** Using `float` precision for large scenes may lead to jittering artifacts; double precision may be required for very large coordinate spaces [G].

## Subtopics

### NDC Bounding Box Computation
Computes the bounding box in Normalized Device Coordinates (NDC) space from a set of rays. It calculates the minimum and maximum of ray origins and endpoints in NDC space and returns the stacked min/max bounds. This is useful for scenes using NDC projection where the bounding box is defined in clip space rather than world space [S3, unofficial].

### Clipping Distance Computation via Volume Corners
Computes the near and far clipping distances for ray marching by projecting the 8 corners of the volume bounding box onto the camera view direction. It extracts the camera position and view direction from the camera matrix (handling world-to-camera vs camera-to-world inversion via a boolean mask to avoid branching). It generates the 8 corners of the volume in IJK space, transforms them to RAS space using the `ijk2ras` matrix, and projects the vector from the camera to each corner onto the normalized view direction. The near distance is set to the minimum projected distance minus a margin, and the far distance to the maximum plus the margin. A default margin of 30.0 mm is used to expand the clipping range [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of rays and $S$ is the average number of samples per ray. The intersection calculation is $O(N)$, and point generation is $O(N \cdot S)$.
- **Memory Complexity:** $O(N \cdot S)$ to store the generated points, masks, and indices.
- **Performance:** The method is highly parallelizable and is typically implemented in CUDA for performance. Batching rays (e.g., 10,000 per batch) helps manage GPU memory usage [S2, unofficial].
- **Strengths:** Ensures samples are only generated within the valid scene volume, reducing unnecessary computations for empty space. Handles edge cases like zero-direction rays and near-far clipping robustly.
- **Limitations:** The fixed step size may lead to aliasing if the step size is too large relative to the scene features. The method assumes an axis-aligned bounding box, which may not tightly fit rotated or irregular geometries [G].
- **When to Use:** When rendering volumetric data within a known bounding box, especially in coarse geometry search phases or when using fixed-step ray marching.
- **When Not to Use:** When the scene geometry is highly irregular and a tight bounding box is not available, or when adaptive step sizes are required for high-frequency details.

## Background
This method is a core component of differentiable volume rendering pipelines, particularly those based on Direct Volume Rendering (DVR) or Neural Radiance Fields (NeRF) with explicit bounding boxes. It is used in frameworks like DirectVoxGO [S2, unofficial] and FLASH [S1, unofficial] to efficiently sample the volume grid. The approach of computing ray-box intersections and then discretizing the intersection interval is a standard technique in computer graphics for ray tracing and volume rendering. The specific implementation details, such as handling zero-direction components and generating ray IDs for flattened arrays, are derived from practical implementations in PyTorch-based rendering engines.

## Sources
[S1] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S2] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S3] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S4] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
