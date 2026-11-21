# Template-based Ray-Casting

Also called: VolumePro Rendering Pipeline Stages

## Overview
Template-based Ray-Casting is a hardware-accelerated direct volume rendering technique that optimizes memory access patterns by processing voxels in planar slices parallel to the image plane [S1, unofficial]. It distinguishes itself from shear-warp methods by performing full tri-linear interpolation and supporting sub-pixel ray origins, thereby eliminating view-dependent artifacts during base-plane switches [S1, unofficial].

## How it works
**Inputs:** 3D volume dataset, camera position, camera orientation, image plane resolution [S1, unofficial].
**Outputs:** Final rendered image on the display plane [S1, unofficial].

**Data Structures:**
*   **Volume Memory:** Stores voxel data. The architecture ensures each voxel is read exactly once per frame [S1, unofficial].
*   **Rendering Pipelines:** Parallel processing units (e.g., four pipelines in the vg500 ASIC) [S1, unofficial].
*   **Intermediate Buffer:** Stores the rendered image on the base plane before warping [S1, unofficial].

**Steps:**
1.  **Base Plane Identification:** Identify the volume face (plane of voxels) that is most parallel and nearest to the current image plane. This plane becomes the base plane [S1, unofficial].
2.  **Ray Generation:** Cast rays from each pixel on the base plane into the volume dataset. Rays may start at sub-pixel locations to support supersampling and prevent artifacts [S1, unofficial].
3.  **Voxel Access:** Read and process voxels in planes parallel to the base plane. Within each slice, voxels are accessed in top-to-bottom scanline order to maximize memory bandwidth efficiency [S1, unofficial].
4.  **Interpolation:** Perform tri-linear interpolation on the sampled voxel values [S1, unofficial].
5.  **Compositing & Warping:** Accumulate color and opacity along the ray path within the base plane buffer, then warp the resulting base-plane image onto the final image plane using 2D texture mapping [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def template_based_ray_casting(volume, volume_dims, camera_origin, camera_u, camera_v, image_width, image_height, base_plane_origin, base_plane_u, base_plane_v, base_plane_z_step, warp_matrix):
    # 1. Identify base plane (parameters passed for self-containment)
    # 2. Initialize base plane buffer
    base_image = [[0.0, 0.0] for _ in range(image_width * image_height)]

    # 3. Process rays for each pixel in the base plane
    for py in range(image_height):
        for px in range(image_width):
            # Ray origin on base plane (supports sub-pixel locations)
            ray_o = base_plane_origin + base_plane_u * px + base_plane_v * py
            # Ray direction towards camera origin
            ray_d = camera_origin - ray_o
            ray_d = normalize(ray_d)

            color = 0.0
            alpha = 0.0

            # 4. Traverse volume in slice-parallel order
            # Voxels are read in top-to-bottom scanline order within slices
            t = 0.0
            while t < 1.0:
                pos = ray_o + ray_d * t
                # Map position to volume coordinates
                vx, vy, vz = map_to_volume_coords(pos, volume_dims)

                # Read voxel (each read once per frame in hardware)
                sample = volume[vx][vy][vz]

                # Tri-linear interpolation
                interp_sample = tri_linear_interpolate(volume, pos, volume_dims)

                # Compositing step (front-to-back)
                color, alpha = composite_step(interp_sample, color, alpha)

                t += base_plane_z_step

            base_image[py * image_width + px] = (color, alpha)

    # 5. Warp base plane to image plane
    final_image = warp_image(base_image, image_width, image_height, warp_matrix)
    return final_image

def normalize(v):
    mag = (v[0]**2 + v[1]**2 + v[2]**2)**0.5
    return (v[0]/mag, v[1]/mag, v[2]/mag)

def map_to_volume_coords(pos, dims):
    return (int(pos[0]), int(pos[1]), int(pos[2]))

def tri_linear_interpolate(volume, pos, dims):
    x, y, z = pos
    x0, y0, z0 = int(x), int(y), int(z)
    x1, y1, z1 = x0+1, y0+1, z0+1
    fx, fy, fz = x-x0, y-y0, z-z0
    v000 = volume[x0][y0][z0]
    v100 = volume[x1][y0][z0]
    v010 = volume[x0][y1][z0]
    v110 = volume[x1][y1][z0]
    v001 = volume[x0][y0][z1]
    v101 = volume[x1][y0][z1]
    v011 = volume[x0][y1][z1]
    v111 = volume[x1][y1][z1]
    c00 = lerp(v000, v100, fx)
    c10 = lerp(v010, v110, fx)
    c01 = lerp(v001, v101, fx)
    c11 = lerp(v011, v111, fx)
    c0 = lerp(c00, c10, fy)
    c1 = lerp(c01, c11, fy)
    return lerp(c0, c1, fz)

def lerp(a, b, t):
    return a + (b - a) * t

def composite_step(sample, color, alpha):
    new_alpha = alpha + sample * (1 - alpha)
    new_color = color + sample * (1 - alpha)
    return new_color, new_alpha

def warp_image(base_image, width, height, matrix):
    final = [[0.0, 0.0] for _ in range(width * height)]
    for y in range(height):
        for x in range(width):
            bx, by = x, y
            fx = matrix[0]*bx + matrix[1]*by + matrix[2]
            fy = matrix[3]*bx + matrix[4]*by + matrix[5]
            if 0 <= fx < width and 0 <= fy < height:
                final[y*width+x] = base_image[int(fy)*width+int(fx)]
    return final
```

**Parameters:**
*   **Pipeline Clock Speed:** 125 MHz (each pipeline accepts a new voxel every cycle) [S1, unofficial].
*   **Interpolation:** Tri-linear [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **View-dependent artifacts:** Occur when switching base planes if sub-pixel ray starting locations are not supported. VolumePro prevents this by allowing sub-pixel ray origins and tri-linear interpolation [S1, unofficial].

## Subtopics

### [Cube-4](14515-cube-4.md)
A parallel and scalable architecture implementing slice-parallel ray-casting with tri-linear interpolation and surface normal estimation. It uses local, fixed-bandwidth interconnections and avoids precomputations.

## Operating characteristics
*   **Memory Access:** Optimized for regular, object-order data access. Each voxel is read exactly once per frame [S1, unofficial].
*   **Performance:** Hardware pipelines run at 125 MHz, processing one voxel per cycle per pipeline [S1, unofficial].
*   **Strengths:** High memory bandwidth efficiency; high-quality 3D interpolation; no view-dependent artifacts during base-plane switching [S1, unofficial].
*   **Limitations:** Requires specific hardware support (e.g., vg500 ASIC) for optimal performance; warping step requires off-screen 2D texture mapping capabilities [S1, unofficial].
*   **When to use:** When hardware acceleration is available and high-quality interpolation is required without precomputed data structures [S1, unofficial].

## Background
Template-based Ray-Casting is the core algorithm implemented in the VolumePro system, a PCI-based volume rendering accelerator [S1, unofficial]. The system uses the vg500 ASIC, which contains four rendering pipelines [S1, unofficial]. It was developed as an improvement over shear-warp methods, specifically addressing their limitations in interpolation quality and artifact generation [S1, unofficial].

## Sources
[S1] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
