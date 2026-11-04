# 3D Texture Proxy Geometry

Also called: 3D Texture Proxy, 3D Texture-Based Volume Rendering, VR Basado en Texturas 3D, VR con texturas tridimensionales, texturas 3D con planos orientados al plano de proyección

## Overview
3D Texture Proxy Geometry is a GPU-accelerated volume rendering technique where the volumetric dataset is uploaded as a 3D texture object, and polygonal slices are rasterized and blended back-to-front to form the final image [S1, S2, unofficial]. It differs from ray casting by relying on hardware-accelerated texture sampling and interpolation rather than explicit ray marching, and from 2D texture mapping by using a single 3D texture to eliminate popping artifacts and support arbitrary viewing orientations without pre-computing multiple 2D slices [S5, unofficial]. Unlike ray casting, it cannot skip unseen voxels, rendering all voxels within the view frustum [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** Volumetric scalar field data, camera position and orientation, transfer function (mapping scalar values to color and opacity) [G].
- **Output:** 2D color buffer (image) [G].

**Data Structures:**
1.  **3D Texture Object:** A GPU texture object storing the volume data [S2, unofficial].
    -   **Format:** Depends on data type. For `unsigned short` data, the sampler must be `usampler3D` [S3, unofficial]. Using `sampler3D` results in all values being 0 [S3, unofficial].
    -   **Dimensions:** Width, Height, Depth. In `glTexImage3D`, the `depth` parameter corresponds to the first dimension of the array. Passing the wrong dimension causes index confusion [S3, unofficial].
    -   **Filters:** `GL_TEXTURE_MIN_FILTER` and `GL_TEXTURE_MAG_FILTER` must be chosen carefully as they affect the interpolation performed by the GPU texture mapping hardware [S2, unofficial]. Linear filtering is used for trilinear interpolation [S3, unofficial].
    -   **Wrapping:** Typically `GL_CLAMP_TO_EDGE` to prevent artifacts at boundaries [S3, unofficial].
2.  **View-Port Aligned Slices:** Polygonal meshes representing the intersection of the 3D texture cube with arbitrarily oriented planes [S2, unofficial].
3.  **Minimum Texture (Optional/Advanced):** A luminance-alpha texture map used in specific dependent texture lookup setups, with the alpha channel set to 1 to allow for the addition of scalar values [S4].

**Numbered Steps:**

1.  **Pre-classification (Optional):** Pre-classify the volume data to assign optical properties (color/opacity) before upload, or perform post-classification in the fragment shader [S1, unofficial].
2.  **Texture Upload:** Upload the pre-classified (or raw) volume data as a 3D texture to the GPU [S1, S2, unofficial].
    -   *Pitfall:* If the volume is too large for texture memory, it must be partitioned into subvolumes swapped in and out during rendering [S1, S2, unofficial].
3.  **Compute View-Port Aligned Slices:**
    -   Calculate polygons formed by the intersection of the 3D texture cube and arbitrarily oriented planes [S2, unofficial].
    -   Traditionally computed by the CPU [S2, unofficial].
    -   Can be offloaded to a vertex shader program [S2, unofficial]. The vertex shader recognizes all 12 edges of a cube as 6 different sets of edges. For each set, it finds at most one coordinate representing the intersecting point of the edge to an arbitrary plane [S2, unofficial].
4.  **Rasterization:** Rasterize the polygonal slices through the volume parallel to the viewpoint [S1, unofficial].
5.  **Fragment Processing (Texture Lookup):**
    -   The fragment program accesses the 3D texture object [S2, unofficial].
    -   Perform texture fetching and transfer function lookup [S2, unofficial].
    -   *Advanced Dependent Lookup (nVidia Texture Shader Extension):*
        -   **Stage 1:** Perform lookup in the minimum texture based on computed ray parameters, normalized using the fourth texture coordinate [S4].
        -   **Stage 2:** Reconstruct the scalar value on the back face ($s_b$) using the scalar product with the gradient in view direction ($g_d$) and the scalar on the front face ($s_f$) [S4]. If only a 2D texture lookup is required, replace this stage with `DOT_PRODUCT_TEXTURE_2D_NV` [S4].
        -   **Stage 3:** Supply $s_f$ as a texture coordinate [S4].
        -   **Stage 4:** Look up color and opacity using the denormalized ray parameter (scaled length $l/w$ multiplied by scaling factor $w$), $s_f$, and $s_b$ [S4].
        -   Vertex program provides suitable texture coordinates for stages 2-4 to combine necessary parameters [S4].
6.  **Blending:** Blend the slices in back-to-front order to form the final image [S1, unofficial].
    -   *Optimization:* Empty space skipping can be performed in the vertex or fragment program to improve performance [S2, unofficial]. Occlusion clipping can also be applied, reporting an average of 2.8 speedup [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume(volume_data, camera, transfer_function, width, height, num_slices):
    # 1. Setup 3D Texture
    # Upload volume_data to GPU as a 3D texture object.
    # Dimensions must be the smallest powers of two that can contain the volume data [S5, unofficial].
    # In glTexImage3D, the depth parameter corresponds to the first dimension of the array [S3, unofficial].
    texture = create_3d_texture(volume_data)
    set_texture_parameters(texture, min_filter="linear", mag_filter="linear") # Linear filtering for trilinear interpolation [S2, S3, unofficial]

    # 2. Compute View-Port Aligned Slices
    # Intersect volume bounding box with arbitrarily oriented planes [S2, unofficial]
    slices = compute_view_port_aligned_slices(camera, volume_data.bounds, num_slices)

    # Initialize output buffers
    out_color = [[0.0, 0.0, 0.0] for _ in range(height * width)]
    out_alpha = [0.0] * (height * width)

    # 3. Render Loop (Back-to-Front)
    for slice in reversed(slices):
        # 4. Rasterize Slice
        screen_coords = project_to_screen(slice["vertices"], camera)
        pixels = pixels_covered_by(screen_coords, width, height)

        # 5. Fragment Processing
        for y, x in pixels:
            tex_coords = compute_texture_coords(x, y, slice, camera)
            if tex_coords is None:
                continue
            scalar = texture_sample(texture, tex_coords)
            color, opacity = transfer_function(scalar)

            # 6. Back-to-Front Blending
            idx = y * width + x
            out_alpha[idx] = out_alpha[idx] + opacity * (1.0 - out_alpha[idx])
            out_color[idx] = [
                out_color[idx][0] * (1.0 - opacity) + color[0] * opacity,
                out_color[idx][1] * (1.0 - opacity) + color[1] * opacity,
                out_color[idx][2] * (1.0 - opacity) + color[2] * opacity
            ]

    return out_color, out_alpha

def create_3d_texture(data):
    # Allocate texture memory with smallest power-of-two dimensions containing data [S5, unofficial]
    # Fill with zeros, copy data, upload [S5, unofficial]
    # Note: Original data is copied without resizing; resizing is handled by the GPU [S5, unofficial]
    return {"type": "3D", "data": data}

def set_texture_parameters(texture, min_filter, mag_filter):
    # Set interpolation mode: nearest neighbor or linear (trilinear) [S1, S2, unofficial]
    pass

def compute_view_port_aligned_slices(camera, bounds, num_slices):
    # Generate planes arbitrarily oriented to intersect the volume bounding box [S2, unofficial]
    # Vertices are computed by intersecting the plane with the 12 cube edges grouped into 6 sets [S2, unofficial]
    planes = []
    for i in range(num_slices):
        t = i / num_slices
        planes.append({
            "vertices": [], # Computed by plane-cube intersection [S2, unofficial]
            "normal": camera.view_dir,
            "dist": bounds.min + t * (bounds.max - bounds.min)
        })
    return planes

def project_to_screen(vertices, camera):
    # Transform vertices to normalized device coordinates
    return [v.project(camera) for v in vertices]

def pixels_covered_by(screen_coords, width, height):
    # Bounding box rasterization
    min_x, max_x = int(min(v.x for v in screen_coords)), int(max(v.x for v in screen_coords))
    min_y, max_y = int(min(v.y for v in screen_coords)), int(max(v.y for v in screen_coords))
    return [(y, x) for y in range(min_y, max_y + 1) for x in range(min_x, max_x + 1)]

def compute_texture_coords(x, y, slice, camera):
    # Compute 3D texture coordinates from ray intersection with slice
    ray = camera.ray_at(x, y)
    denom = dot(ray.direction, slice["normal"])
    if abs(denom) < 1e-6:
        return None # Ray parallel to slice
    t = (slice["dist"] - dot(ray.origin, slice["normal"])) / denom
    point = ray.origin + ray.direction * t
    # The GPU handles scaling to [0,1] via texture matrices; we return the raw intersection point [S5, unofficial]
    return point

def texture_sample(texture, coords):
    # Trilinear interpolation at coords [S3, unofficial]
    # Note: GPU texture mapping hardware performs the interpolation [S2, unofficial]
    return texture["data"].trilinear(coords)

def transfer_function(scalar):
    # Map scalar to color and opacity
    return [1.0, 1.0, 1.0], 0.5

def dot(a, b):
    return sum(x * y for x, y in zip(a, b))
```

**Parameters:**
-   **Interpolation Method:** Nearest neighbor or trilinear [S1, unofficial]. Trilinear is preferred for better image quality [S3, unofficial].
-   **Filters:** `GL_TEXTURE_MIN_FILTER` and `GL_TEXTURE_MAG_FILTER` [S2, unofficial].
-   **Texture Memory:** A single 3D texture suffices if memory allows [S3, unofficial].
-   **Texture Dimension:** $256^3$ due to OpenGL texture dimension limitations [S5, unofficial]. The original data volume is copied without resizing; resizing is handled by the GPU [S5, unofficial].
-   **Memory Usage:** 16 MiB in main memory for the textures in this implementation [S5, unofficial].

**Edge Cases and Pitfalls:**
-   **Memory Limit:** Large volumes exceed texture memory capacity [S1, unofficial]. Must be partitioned and swapped [S1, S2, unofficial].
-   **Data Type Mismatch:** Using `sampler3D` for `unsigned short` data results in all values being 0 [S3, unofficial].
-   **Dimension Confusion:** In `glTexImage3D`, the `depth` parameter corresponds to the first dimension of the array. Passing the wrong dimension causes index confusion [S3, unofficial].
-   **Unseen Voxels:** All voxels within the slice are rendered; unseen voxels cannot be skipped [S1, unofficial].
-   **Center Misalignment:** If the volume does not occupy the entire 3D texture, the volume center does not coincide with the texture center. Translations must be added to the texture matrix to align the volume center with the polygon set center [S5, unofficial].
-   **Primitive Size:** Primitives must be large enough to contain the entire volume during any rotation. The minimum size is the diagonal of the volume [S5, unofficial].
-   **Performance Degradation:** Increasing primitive size to fit the volume can negatively affect rendering performance unless alpha testing is applied to avoid drawing transparent fragments to the framebuffer [S5, unofficial].
-   **Banding Artifacts:** Projection-oriented polygons introduce horizontal or vertical bands depending on the number of polygons used [S5, unofficial].

## Subtopics

### [Tri-linear Interpolation via LRP](913-tri-linear-interpolation-via-lrp.md)
A GPU-based tri-linear interpolation method using the LRP (linear interpolation) instruction, fetching 8 nearest neighbors via pre-computed texture coordinates. It sets itself apart by leveraging specific GPU instructions for efficient interpolation, suitable when hardware supports LRP. Choose it when maximizing interpolation performance on compatible GPUs.

### [Multi-Volume Rendering with Multiple Textures](1568-multi-volume-rendering-with-multiple-textures.md)
A technique that utilizes the GPU's capability to handle multiple texture units simultaneously, assigning one 3D texture to each data class or volume. It sets itself apart by enabling simultaneous visualization of multiple classes without blending them into a single texture, avoiding texture memory limits for single large volumes. Choose it when rendering multi-class datasets or when hardware supports multiple active texture units.

### [Proxy Geometry Compression](1917-proxy-geometry-compression.md)
A scheme to store volume data and proxy geometry locally in graphics hardware to avoid CPU-GPU bandwidth bottlenecks, using compression to fit within limited memory. It sets itself apart by reducing memory footprint and transfer overhead, making it suitable for constrained hardware. Choose it when texture memory is insufficient or bandwidth is a bottleneck.

## Operating characteristics
-   **Time Complexity:** Not explicitly stated in the sources, but performance scales with pixel count and slice count, with optimizations like occlusion clipping yielding a 2.8× speedup [S2, unofficial].
-   **Memory Complexity:** Linear with respect to volume size, constrained by available texture memory [S1, S3, unofficial].
-   **Measured Performance:** Li et al. reported an average of 2.8 speedup using optimization techniques like occlusion clipping in fragment programs [S2, unofficial].
-   **Scaling:** Scaling to higher resolutions is limited by texture memory, requiring paging or out-of-core techniques for large datasets [S2, unofficial].
-   **Strengths:** Hardware-accelerated trilinear interpolation eliminates bands from 2D texturization and supports arbitrary viewing orientations [S3, S5, unofficial].
-   **Limitations:** Cannot skip unseen voxels, limited by texture memory size, and projection-oriented polygons may introduce horizontal or vertical bands depending on polygon count [S1, S5, unofficial].
-   **When to Use:** Moderate-sized volumes fitting in texture memory, when hardware interpolation is preferred [S2, S3, unofficial].
-   **When Not to Use:** Volumes exceeding texture memory capacity, or when skipping unseen voxels is critical for performance [S1, unofficial].

## Background
-   **Origin:** Popularized by Cabral et al. in 1994 for nonshaded volume rendering [S1, unofficial].
-   **History:** Examined by Cabral et al. on the SGI RealityEngine, the first hardware to implement 3D texture mapping [S2, unofficial]. The algorithm was initially inaccessible for common usage due to the high cost of the RealityEngine [S2, unofficial].
-   **Development:** Rezk-Salama et al. proposed GPU vertex shader optimization to offload computation of view-port aligned polygons from CPU to GPU, balancing workload and improving performance [S2, unofficial].
-   **Usage:** Became a mainstream algorithm after the introduction of affordable and powerful GPUs with 3D texture mapping support [S2, unofficial].

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S2] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S3] upupming/volume-rendering, https://github.com/upupming/volume-rendering, fetched 2026-10-26 (unofficial)
[S4] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S5] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
