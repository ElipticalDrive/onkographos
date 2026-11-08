# Voxel Splat Rendering

Also called: Interactive Splat Rendering with Alpha/Gamma Control, Interactive value selection rendering, Splatting (Object Order)

## Overview
Voxel Splat Rendering is an object-order volume rendering technique that projects 3D voxels onto the 2D image plane as geometric primitives (splats), typically quads or circles, which are then rasterized and blended [S2, unofficial][S3, unofficial]. It solves the problem of interactive visualization of large volumetric datasets by leveraging hardware-accelerated rasterization pipelines, allowing for real-time interaction such as slice-based value selection and parameter tuning [S2, unofficial]. Unlike ray-casting methods that compute contributions per pixel, this method computes contributions per voxel, making it distinct from siblings like Circle Splat Rendering which may use different primitive shapes or kernel approximations [S2, unofficial][S3, unofficial].

## How it works
**Inputs:** A 3D volumetric dataset (voxels with position, color, and optionally normal/density attributes); a camera view (eye position, projection matrix); and rendering parameters (point size, alpha, gamma) [S1][S2, unofficial][S3, unofficial].
**Outputs:** A 2D color image [G].

**Data Structures:**
*   **Voxel:** Contains position $(x, y, z)$, color $(r, g, b)$, and optionally a normal vector $\mathbf{n}$ [S2, unofficial]. In out-of-core implementations, voxels are stored with relative positions to reduce disk I/O [S2, unofficial].
*   **Normal Storage:** Normals can be quantized to one of 240 pre-defined directions distributed nearly uniformly over a sphere (derived from a dodecahedron triangulation), stored in 1 byte [S2, unofficial].
*   **Bounding Box:** A pre-defined region used to filter voxels during interactive selection to reduce decoding overhead [S2, unofficial].

**Steps:**

1.  **Front-to-Back Traversal:** Traverse the 3D dataset in front-to-back order relative to the current camera view [S3, unofficial][S4, unofficial].
2.  **Voxel Selection:** If interactive value selection is active, retrieve and decode only voxels corresponding to selected colors within a pre-defined bounding box [S2, unofficial]. This handles cases where different tissues share similar colors by limiting the rendered subset [S2, unofficial].
3.  **Projection and Splat Generation:** For each voxel $k$, project its center to screen space [S3, unofficial]. Apply a filtering function (e.g., Gaussian) to the voxel's contribution, proportional to its distance from the projection plane [S3, unofficial]. Render the voxel as a square or circular splat using GL commands/extensions [S2, unofficial]. The diameter can be fixed or distance-dependent [S2, unofficial].
4.  **Weighting:** If weighting contributions based on camera visibility, compute weights for each camera $i$ [S1]. Let $\mathbf{V}_{toEye}$ be the unit vector from the voxel center to the eye point [S1]. Let $\mathbf{N}$ be the unit normal to the surface at the voxel center [S1]. Let $\mathbf{V}_{toCamera}[i]$ be the unit vector from the voxel center to camera $i$'s position [S1]. Calculate view weight: $w_V = \max(0, \mathbf{V}_{toEye} \cdot \mathbf{V}_{toCamera}[i])^4$ [S1]. Calculate normal weight: $w_N = \max(0, \mathbf{N} \cdot \mathbf{V}_{toCamera}[i])^4$ [S1]. Total weight: $w[i] = w_V \cdot w_N$ [S1]. Normalize weights by dividing by the sum of all weights [S1].
5.  **Rasterization and Blending:** If alpha-blending is enabled, blend the splat's color into the frame buffer using the computed weights and alpha/gamma parameters [S2, unofficial][S3, unofficial]. If alpha-blending is disabled, use the stored normals to shade the points, creating an illusion of a shaded surface if point density is high [S2, unofficial]. Apply gamma correction to reduce the contribution of points from tissues with differing color value densities [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_voxel_splats(voxels, camera, params):
    # 1. Sort voxels back-to-front relative to camera
    def sort_by_depth(v_list, eye_pos, forward):
        return sorted(v_list, key=lambda v: dot(v.position - eye_pos, forward))
    sorted_voxels = sort_by_depth(voxels, camera.eye_position, camera.forward)
    
    # 2. Initialize frame buffer
    def zeros(res):
        return [[0.0, 0.0, 0.0] for _ in range(res[0] * res[1])]
    frame_buffer = zeros(camera.resolution)
    
    for voxel in sorted_voxels:
        # 3. Project voxel to screen space
        def project(pos, mvp):
            return mvp @ pos
        screen_pos = project(voxel.position, camera.mvp_matrix)
        
        # 4. Determine splat footprint (quad/circle)
        splat_size = params.point_size * (1.0 + params.distance_factor * distance(voxel.position, camera.eye_position))
        
        # 5. Compute weight if multi-view weighting is active
        weight = 1.0
        if params.use_weighting:
            v_to_eye = normalize(camera.eye_position - voxel.position)
            v_to_cam = normalize(camera.camera_positions - voxel.position)
            n = voxel.normal
            
            dot_v = max(0.0, dot(v_to_eye, v_to_cam))
            dot_n = max(0.0, dot(n, v_to_cam))
            
            w_v = pow(dot_v, 4)
            w_n = pow(dot_n, 4)
            weight = w_v * w_n
            
        # 6. Apply Alpha and Gamma
        alpha = params.alpha * weight
        color = apply_gamma(voxel.color, params.gamma)
        
        # 7. Rasterize Splat
        blend_pixel(screen_pos, color, alpha, frame_buffer, splat_size)
        
    return frame_buffer

def apply_gamma(color, gamma):
    return [c ** (1.0 / gamma) for c in color]

def blend_pixel(screen_pos, color, alpha, frame_buffer, size):
    for dy in range(-size, size + 1):
        for dx in range(-size, size + 1):
            px, py = screen_pos.x + dx, screen_pos.y + dy
            if 0 <= px < frame_buffer.width and 0 <= py < frame_buffer.height:
                idx = py * frame_buffer.width + px
                old_color = frame_buffer[idx]
                frame_buffer[idx] = [
                    old_color[0] * (1.0 - alpha) + color[0] * alpha,
                    old_color[1] * (1.0 - alpha) + color[1] * alpha,
                    old_color[2] * (1.0 - alpha) + color[2] * alpha
                ]
```

**Edge Cases and Pitfalls:**
*   **Close-range views:** Voxel splats may appear as rectangular blocks covering many pixels, causing pixelation artifacts due to the finite resolution of RGB images [S1].
*   **Single-pixel quads:** GPU rasterization is not optimized for single-pixel quads, though performance remains acceptable [S1].
*   **Aliasing:** Minor aliasing artifacts similar to non-anti-aliased triangle models may occur [S1].
*   **Color Ambiguity:** Different tissues may share similar colors, requiring bounding box filtering or alpha/gamma tuning to isolate features [S2, unofficial].

## Subtopics

### [Image-Space Sheet-Buffered Splatting Pipeline](8961-image-space-sheet-buffered-splatting-pipeline.md)
A GPU-based rendering pipeline for volume visualization that processes voxels through texture-based stages: viewing transform, GPGPU sorting, bucket size computation, and rendering. It uses 2D textures to store voxel data throughout the pipeline.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels $N$, $O(N)$, assuming constant-time projection and rasterization [S3, unofficial].
*   **Memory Complexity:** Memory usage scales with the active voxelset size; out-of-core techniques load only selected voxelsets on demand to manage disk I/O and interactive selection [S2, unofficial].
*   **Performance:** Enables real-time interactivity due to hardware acceleration [S2, unofficial]. Performance is acceptable even for giga-voxel resolutions at HD resolution, though single-pixel quads are not optimized by GPUs [S1].
*   **Strengths:** Simple implementation and efficient for large datasets via out-of-core loading and interactive value selection [S2, unofficial].
*   **Limitations:** Artifacts at close range (blockiness, pixelation), aliasing, and potential over-rendering if color selection is not precise [S1][S2, unofficial].

## Background
Voxel Splat Rendering was first published in [20] [S4, unofficial]. It is an object-space front-to-back volume rendering method [S3, unofficial]. The technique has been extended to support interactive visualization of very large datasets using out-of-core point-based approaches, allowing for real-time decoding and rendering [S2, unofficial]. It is also applied in image-based modeling contexts, such as sparse voxelization for novel view synthesis [S1].

## Sources
[S1] Real-time high-resolution sparse voxelization with application to image-based modeling, https://doi.org/10.1145/2492045.2492053, fetched 2026-11-01
[S2] Interactive Visualization of Very Large Datasets Using an Out-of-Core Point-based Approach, https://core.ac.uk/download/484268453.pdf, fetched 2026-10-27 (unofficial)
[S3] Indirect volume rendering, https://slidetodoc.com/direct-volume-rendering-what-is-volume-rendering-accumulate/, fetched 2026-10-29 (unofficial)
[S4] Fourier volume rendering, https://shiftleft.com/mirrors/www.hpl.hp.com/techreports/95/HPL-95-73.pdf, fetched 2026-10-31 (unofficial)
