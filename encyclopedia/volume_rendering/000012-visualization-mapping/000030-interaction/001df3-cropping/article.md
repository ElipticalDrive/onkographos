# Cropping
Also called: Axis-Aligned Bounding Box Cropping, Axis-Aligned Cropping, SetCroppingRegionPlanes, Slab Cropping, Slab-based cropping

## Overview
Cropping is an interaction technique that restricts volume rendering to a specific region of interest (ROI) by defining axis-aligned cut-planes along each spatial dimension [S1][S3, S4, unofficial]. It solves the problem of visual clutter and performance overhead by rendering only the voxels within a defined rectilinear slab, thereby revealing internal structures without altering the underlying data [S1][S3, unofficial]. Unlike [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md), which uses arbitrarily oriented planes to cut away portions of the volume, cropping is strictly axis-aligned relative to the volume's object coordinates [S3, S4, unofficial]. It differs from [Clipping](1444-clipping.md) in that it defines a sub-volume for rendering rather than just a near/far boundary for projection correctness, and from [Dynamic Clipping in Volume Rendering](7498-dynamic-clipping-in-volume-rendering.md) by being a static geometric constraint that can be updated interactively but is fundamentally defined by the intersection of axis-aligned slabs [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The volume dataset $D$ with dimensions $(N_x, N_y, N_z)$ and origin $O$. User-defined cropping parameters: six coordinates $(x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max})$ defining the region of visibility [S2, S3, S4, unofficial].
*   **Output:** A rendered image containing only the voxels satisfying the cropping constraints.

**Data Structures:**
The cropping region is stored as six floating-point or integer values representing the bounds in the volume's local coordinate system (object space) [S3, S4, unofficial]. In implementations like VTK, this is managed via a `vtkVolume` mapper with a cropping flag and a set of plane coordinates [S2, unofficial].

**Steps:**
1.  **Define Cropping Region:** The user or application sets the six bounding coordinates. These define three slabs:
    *   $S_x = \{ (x, y, z) \mid x_{min} \le x \le x_{max} \}$
    *   $S_y = \{ (x, y, z) \mid y_{min} \le y \le y_{max} \}$
    *   $S_z = \{ (x, y, z) \mid z_{min} \le z \le z_{max} \}$
    [S3, unofficial]
2.  **Combine Slabs:** The region of visibility $R$ is defined by the intersection of these slabs (standard cropping):
    $$ R = S_x \cap S_y \cap S_z $$
    Alternatively, unions or inverses can be used to create complex visibility masks, but the standard ROI is the intersection [S3, unofficial].
3.  **Transform to World/Texture Space:** If the volume is partitioned or transformed, the cropping planes must be transformed from object coordinates to the rendering space [S4, unofficial]. For a subvolume with index $(i, j, k)$ and dimensions $(N_x, N_y, N_z)$ with overlap, the subvolume origin $Origin_{Sub}$ is calculated as:
    $$ Origin_{Sub} = (i(N_x - overlap), j(N_y - overlap), k(N_z - overlap)) $$
    The distance vector $D_{subj}$ from the subvolume origin to the cut-plane position $D$ with normal $N$ is:
    $$ D_{subj} = (D - Origin_{Sub}) \cdot N $$
    [S4, unofficial]
4.  **Enable Cropping:** The rendering pipeline enables the cropping mode. In VTK, this is done by setting the cropping flag and the region planes:
    ```python
    mapper->SetCropping(1)
    mapper->SetCroppingRegionPlanes(xmin, xmax, ymin, ymax, zmin, zmax)
    mapper->SetCroppingMode(VTK_CROP_SUBVOLUME)
    ```
    [S2, unofficial]
5.  **Ray Casting with Clipping:** During ray casting, for each sample point $P$ along the ray:
    *   Check if $P$ lies within $R$.
    *   If $P \in R$, compute color and opacity as usual.
    *   If $P \notin R$, set opacity to 0 (or skip sampling entirely if the renderer supports early exit) [S3, S4, unofficial].
    *   Note: Cropping planes may fall at arbitrary voxel positions, not just integer grid indices [S3, unofficial].
6.  **Depth Handling:** When using cutting planes, the depth texture from the DVR prepass is used to maintain correct depth order, ensuring that the cut surfaces are rendered correctly relative to the remaining volume [S1].

**Parameters:**
*   **Cropping Bounds:** $(x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max})$. Typical values depend on the dataset size and the desired ROI [G].
*   **Cropping Mode:** Flag `VTK_CROP_SUBVOLUME` restricts rendering to the defined region [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth Order:** When cutting planes are active, standard depth buffering may fail if the entry/exit points are not correctly tracked. A front-face depth texture is often required to maintain correct depth order [S1].
*   **Arbitrary Positions:** Cropping planes can be placed at non-integer coordinates. The renderer must handle fractional voxel sampling correctly via interpolation or nearest-neighbor lookup [S3, unofficial].
*   **Invalid Regions:** If $x_{min} > x_{max}$, the region is empty. Implementations should clamp or skip such regions [G].
*   **Boundary Artifacts:** At the cropping boundaries, discontinuities may occur. Standard boundary handling techniques are required to mitigate artifacts [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_cropped_volume(volume_data, volume_origin, volume_spacing, crop_bounds, camera_origin, camera_directions, image_width, image_height, step_size):
    """
    Render a volume with axis-aligned cropping.
    
    Args:
        volume_data: 3D array of voxel values.
        volume_origin: (ox, oy, oz) in object coordinates.
        volume_spacing: (sx, sy, sz) in object coordinates per voxel.
        crop_bounds: (xmin, xmax, ymin, ymax, zmin, zmax) in object coordinates.
        camera_origin: (cx, cy, cz) in world coordinates.
        camera_directions: 2D array of direction vectors in world coordinates.
        image_width, image_height: Output resolution.
        step_size: Distance between samples in world coordinates.
    
    Returns:
        2D array of RGBA tuples.
    """
    # 1. Transform crop bounds to world coordinates
    wx_min = volume_origin[0] + crop_bounds[0] * volume_spacing[0]
    wx_max = volume_origin[0] + crop_bounds[1] * volume_spacing[0]
    wy_min = volume_origin[1] + crop_bounds[2] * volume_spacing[1]
    wy_max = volume_origin[1] + crop_bounds[3] * volume_spacing[1]
    wz_min = volume_origin[2] + crop_bounds[4] * volume_spacing[2]
    wz_max = volume_origin[2] + crop_bounds[5] * volume_spacing[2]

    image = [[(0.0, 0.0, 0.0, 0.0) for _ in range(image_width)] for _ in range(image_height)]

    for y in range(image_height):
        for x in range(image_width):
            ray_o = camera_origin
            ray_d = camera_directions[y][x]
            # Normalize direction
            len_d = (ray_d[0]**2 + ray_d[1]**2 + ray_d[2]**2)**0.5
            ray_d = (ray_d[0]/len_d, ray_d[1]/len_d, ray_d[2]/len_d)

            # 2. Intersect ray with volume bounding box (world space)
            vol_min = volume_origin
            vol_max = (volume_origin[0] + volume_data.shape[0]*volume_spacing[0],
                       volume_origin[1] + volume_data.shape[1]*volume_spacing[1],
                       volume_origin[2] + volume_data.shape[2]*volume_spacing[2])
            t_min, t_max = intersect_ray_aabb(ray_o, ray_d, vol_min, vol_max)
            if t_min is None or t_max is None:
                continue

            # 3. Intersect ray with crop bounds (world space)
            t_crop_min, t_crop_max = intersect_ray_aabb(ray_o, ray_d, (wx_min, wy_min, wz_min), (wx_max, wy_max, wz_max))
            if t_crop_min is None:
                continue # Ray misses crop region entirely
            # Clip ray segment to crop region
            t_start = max(t_min, t_crop_min)
            t_end = min(t_max, t_crop_max)
            if t_start >= t_end:
                continue

            # 4. Ray casting loop within crop region
            t = t_start
            acc_color = (0.0, 0.0, 0.0, 0.0)
            while t < t_end:
                pos = (ray_o[0] + t*ray_d[0], ray_o[1] + t*ray_d[1], ray_o[2] + t*ray_d[2])
                # Convert pos to voxel indices
                ix = int((pos[0] - volume_origin[0]) / volume_spacing[0])
                iy = int((pos[1] - volume_origin[1]) / volume_spacing[1])
                iz = int((pos[2] - volume_origin[2]) / volume_spacing[2])
                # Clamp indices to volume bounds
                ix = max(0, min(volume_data.shape[0]-1, ix))
                iy = max(0, min(volume_data.shape[1]-1, iy))
                iz = max(0, min(volume_data.shape[2]-1, iz))
                value = volume_data[ix][iy][iz]
                color, opacity = apply_transfer_function(value)
                acc_color = composite_rgba(acc_color, color, opacity)
                if acc_color[3] >= 1.0:
                    break
                t += step_size
            image[y][x] = acc_color
    return image

def intersect_ray_aabb(ray_o, ray_d, aabb_min, aabb_max):
    t1 = (aabb_min[0] - ray_o[0]) / ray_d[0] if ray_d[0] != 0 else float('-inf')
    t2 = (aabb_max[0] - ray_o[0]) / ray_d[0] if ray_d[0] != 0 else float('inf')
    t3 = (aabb_min[1] - ray_o[1]) / ray_d[1] if ray_d[1] != 0 else float('-inf')
    t4 = (aabb_max[1] - ray_o[1]) / ray_d[1] if ray_d[1] != 0 else float('inf')
    t5 = (aabb_min[2] - ray_o[2]) / ray_d[2] if ray_d[2] != 0 else float('-inf')
    t6 = (aabb_max[2] - ray_o[2]) / ray_d[2] if ray_d[2] != 0 else float('inf')
    t_enter = max(min(t1, t2), min(t3, t4), min(t5, t6))
    t_exit = min(max(t1, t2), max(t3, t4), max(t5, t6))
    if t_enter > t_exit:
        return None, None
    return t_enter, t_exit

def apply_transfer_function(value):
    # Standard linear mapping for demonstration
    return (value, value, value, min(value, 1.0))

def composite_rgba(acc, color, opacity):
    r = acc[0] + color[0] * opacity * (1.0 - acc[3])
    g = acc[1] + color[1] * opacity * (1.0 - acc[3])
    b = acc[2] + color[2] * opacity * (1.0 - acc[3])
    a = acc[3] + opacity * (1.0 - acc[3])
    return (r, g, b, a)
```

## Subtopics

### [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md)
A clipping plane for volume suppression uses one or more arbitrarily oriented planes to cut away selected portions of a volume dataset, uncovering internal details. Unlike [Cropping](1648-cropping.md), which restricts visibility to a rectilinear slab defined by planes parallel to the volume axes, clipping planes operate at arbitrary angles and can define a planar slice of finite thickness. It is distinct from [Clipping](1444-clipping.md) in its specific application to volumetric data nodes where the interior of the clipped region retains data, often requiring capping of intersected geometry in medical contexts.

### [Clipping](1444-clipping.md)
Clipping is a technique to enable correct perspective projection in sliced-based volume rendering by defining a near clipping boundary relative to the viewpoint. It solves the problem of incorrect depth impression in inside views, where parallel projection fails to provide accurate spatial cues. Unlike general cropping [Cropping](1648-cropping.md) which cuts along volume faces, this method clips against the camera's near/far planes and active clipping planes to ensure only relevant slabs are processed.

### [Variance Clipping](1970-variance-clipping.md)
Variance clipping is a history rectification technique used in Temporal Anti-Aliasing (TAA) for volumetric rendering that constructs a tight bounding box for valid color values based on the statistical mean and variance of a neighborhood of samples. It solves the problem of ghosting artifacts during rapid illumination changes by adapting the clipping region to the local signal distribution, allowing it to reconstruct signals faster than fixed-bound methods. It sets itself apart from [AABB Clamping](2019-aabb-clamping.md) and [AABB Clipping](1444-clipping.md) by using a statistically derived, typically smaller, clipping region that reacts more precisely to changes, though at the cost of higher temporal instability.

### [Clipped Cropping Region Planes](2931-clipped-cropping-region-planes.md)
Clipped Cropping Region Planes is a preprocessing step that clamps user-defined axis-aligned cropping planes to the volume's bounding box, ensuring the resulting clipping region is valid and contained within the data bounds. It sets apart from its parent method, [Cropping](1648-cropping.md), by enforcing geometric validity before rendering, preventing undefined behavior when planes fall outside the dataset.

### [Dynamic Clipping in Volume Rendering](7498-dynamic-clipping-in-volume-rendering.md)
Dynamic Clipping in Volume Rendering restricts the visualization to a specific region of interest (ROI) by applying six axis-aligned clipping planes (X min/max, Y min/max, Z min/max) that are updated in response to user interaction. Unlike static cropping masks which are pre-defined, this method allows the clipping boundaries to be adjusted interactively via a widget, converting index-space constraints into world-space planes for the renderer. It sets itself apart from sibling methods like [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md) by explicitly managing the geometric transformation of clipping boundaries relative to the volume's coordinate system, and from [Variance Clipping](1970-variance-clipping.md) by operating on spatial extent rather than statistical data properties.

### [Pre-integrated Clipping Boundary Handling](7867-pre-integrated-clipping-boundary-handling.md)
A method to handle artifacts at clip boundaries when using pre-integrated volume rendering with low sampling rates, by mapping an opaque texture onto the clip geometry. It sets apart from [Clipping](1444-clipping.md) and [Clipped Cropping Region Planes](2931-clipped-cropping-region-planes.md) by explicitly computing the optical contribution of the gap between the clipping surface and the first volume slice, rather than relying solely on depth buffering or simple suppression.

### [Clipping Geometry Illumination](6620-clipping-geometry-illumination.md)
Clipping geometry makes parts of the volume transparent. At the boundary of this geometry, the surface normal of the clipping geometry is used for illumination instead of the volume's gradient. This technique, described in Weiskopf et al., TVCG 2003, enhances the visual clarity of cutaway views by providing consistent lighting cues at the newly exposed surfaces.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the cropped region. Cropping reduces the effective $N$ by excluding voxels outside the ROI, thus improving performance proportionally to the volume of the ROI relative to the full volume [G].
*   **Memory Complexity:** $O(1)$ additional memory for storing the six plane coordinates [S3, S4, unofficial]. Specific depth texture memory requirements are not given in the sources [G].
*   **Measured Performance:** In VolumePro, cropping is described as an "easy way of specifying a rectilinear region of interest for rendering" [S3, unofficial]. In VTK, enabling `VTK_CROP_SUBVOLUME` allows the mapper to skip sampling outside the region, significantly speeding up rendering for large datasets with small ROIs [S2, unofficial].
*   **Strengths:** Simple to implement; effective for focusing on specific features; reduces computational load; axis-aligned nature allows for efficient spatial partitioning and culling [G].
*   **Limitations:** Cannot handle arbitrarily oriented cuts (use [Clipping plane for volume suppression](887-clipping-plane-for-volume-suppression.md) for that); may introduce artifacts at boundaries if not handled carefully; requires depth texture management for correct ordering when planes cut through the volume [S1].
*   **When to use:** When the user needs to inspect internal structures by removing outer layers; when performance is critical and the ROI is small; when the ROI is naturally axis-aligned (e.g., a specific organ in a medical scan aligned with the scanner axes) [G].
*   **When not to use:** When the ROI is oriented at an arbitrary angle; when the user needs to see the relationship between the cropped region and the rest of the volume in a single view without deformation (use [Focus+Context Visualization via Deformation](3001-focus-context-visualization-via-deformation.md) instead) [G].

## Background
Cropping originated as a fundamental interaction technique in volume visualization to allow users to "see inside" volumetric datasets. The VolumePro real-time ray-casting system popularized the concept of defining cropping regions using six registers for axis-aligned slabs, allowing for complex visibility masks through set operations (intersection, union, inverse) [S3, unofficial]. The technique is widely implemented in visualization libraries such as VTK (via `SetCroppingRegionPlanes` and `VTK_CROP_SUBVOLUME`) [S2, unofficial] and is a core feature in educational tools like VolumeVisual, where it is synchronized with transfer function editing to help users understand volume rendering concepts [S1]. The patent US 6,407,737 B1 describes a method for rendering shear-warped partitioned volumes where cropping is specified by object coordinates and characterized by "slab" thickness [S4, unofficial]. Clipping geometry illumination, a related technique for improving the visual quality of cropped views, was introduced by Weiskopf et al. in TVCG 2003 .

## Sources
[S1] VolumeVisual: Design and Evaluation of an Educational Software Tool for Teaching and Learning Volume Visualization, https://doi.org/10.18260/1-2--38034, fetched 2026-10-26
[S2] kpengk/VolumeClipping, https://github.com/kpengk/VolumeClipping, fetched 2026-10-26 (unofficial)
[S3] [PDF] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S4] (12) United States Patent (10) Patent N0.: US 6,407,737 B1, https://vcg.seas.harvard.edu/publications/rendering-a-shear-warped-partitioned-volume-data-set/paper, fetched 2026-10-29 (unofficial)
[S5] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
