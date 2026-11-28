# Voxel-Based Virtual Object Relighting

Also called: Image-based hybrid global illumination, Voxel-based approach for virtual objects relighting

## Overview
Voxel-Based Virtual Object Relighting is an image-based method for inserting virtual objects into real environments and estimating light transport between real and virtual parts without manual geometric modeling [S1, unofficial]. It extracts data from multiple 2.5D high dynamic range (HDR) captures stored in a voxel grid to determine light interactions [S1, unofficial]. Unlike existing methods, it requires no offline computation, geometric modeling of the scene, light probes, or hemispherical cameras [S1, unofficial]. It builds on [Data Augmentation with Virtual Object Insertion](19387-data-augmentation-with-virtual-object-insertion.md) by providing the specific rendering and light transport estimation pipeline for the inserted assets.

## How it works
**Inputs:**
*   A set of registered HDR-Z images (HDR color + depth) of the real scene.
*   Camera intrinsic and extrinsic parameters.
*   Virtual object geometry (mesh).
*   A fixed radiance threshold for segmentation.
*   Voxel grid resolution (typically $64 \times 64 \times 64$) [S1, unofficial].

**Outputs:**
*   An augmented image containing the virtual object, its shadows, and the modified lighting of the real environment.

**Data Structures:**
1.  **Voxel Grid:** A regular 3D grid of size $N \times N \times N$ (default $N=64$) [S1, unofficial]. Each voxel stores:
    *   `radiance`: Mean radiance of backprojected pixels [S1, unofficial].
    *   `is_environment`: Boolean flag indicating if the voxel contains real scene geometry [S1, unofficial].
    *   `is_virtual`: Boolean flag indicating if the voxel contains virtual object geometry [S1, unofficial].
    *   `light_intensity`: Accumulated light intensity from traced rays [S1, unofficial].
    *   *Note:* The sources state that "Major part of the voxels contain data about the scene, so we do not need to deal with sparse matrix or compression schemes" [S1, unofficial].
2.  **Ray List:** A list of rays attached to the scene, where each ray contains:
    *   `origin`: 3D point in voxel space (from backprojection) [S1, unofficial].
    *   `direction`: Vector pointing towards the camera [S1, unofficial].
    *   `intensity`: Radiance value of the source pixel [S1, unofficial].
3.  **Ambient Energy:** A scalar value representing averaged low-radiance pixel values [S1, unofficial].

**Steps:**

1.  **Input Data Preparation:**
    Acquire HDR-Z images using a calibrated Time of Flight (TOF) camera (e.g., Mesa SR4000) for depth and a coupled webcam (e.g., Logitech Pro 9000) for color [S1, unofficial]. Combine multiple LDR exposures with varying shutter times to create HDR images [S1, unofficial]. Register cameras using fiducial patterns to ensure rigid localization [S1, unofficial]. Median filter depth maps to eliminate pixels with low confidence scores [S1, unofficial].

2.  **Geometry and Light Sampling (Backprojection):**
    For each HDR-Z image, backproject all pixels into the regular voxel grid in the scene reference frame using the inverse of intrinsic and extrinsic parameter matrices [S1, unofficial]. If multiple pixels reproject into the same voxel, store the mean value of their radiance in that voxel [S1, unofficial]. Repeat for all input images to create a voxelized surface of observed objects [S1, unofficial]. Mark these voxels as `is_environment = true` [S1, unofficial].

3.  **Light Sampling (Ray Generation):**
    Segment the HDR input image using a fixed threshold applied uniformly to all images in the set [S1, unofficial]. Separate pixels into two groups:
    *   **High radiance:** Pixels with radiance above the threshold.
    *   **Low radiance:** Pixels with radiance below the threshold [S1, unofficial].

4.  **Ray Creation:**
    For each high radiance pixel:
    *   Calculate the visible 3D point by backprojecting the pixel using its depth value [S1, unofficial].
    *   Create a ray in voxel grid space starting at this 3D point and pointing towards the camera [S1, unofficial].
    *   Store the pixel's radiance as the ray's `intensity` value [S1, unofficial].
    Gather all such rays into the scene-attached ray list [S1, unofficial].

5.  **Ambient Estimation:**
    Average the radiances of all low radiance pixels across all images in the set [S1, unofficial]. This computed value represents the ambient energy not represented by any specific ray [S1, unofficial].

6.  **Virtual Object Insertion:**
    Insert the virtual object geometry into the voxel space [S1, unofficial]. Mark voxels intersecting the virtual geometry as `is_virtual = true` [S1, unofficial].
    For each ray in the ray list:
    *   Trace the ray into the voxel space using a voxel traversal method [S1, unofficial].
    *   If the ray intersects virtual geometry:
        *   Store the first virtual voxel intersected [S1, unofficial].
        *   Shade this voxel using the ray's `intensity` [S1, unofficial].
        *   Store the first environment voxel intersected by the ray *after* the virtual object (or note that the ray's energy does not reach it) [S1, unofficial]. *Note: The text states "store the first environment voxel intersecting the ray, as the energy of the intersected ray do not reach it anymore," implying the environment voxel is occluded or shadowed by the virtual object.* [S1, unofficial].

7.  **Rendering Augmented Images:**
    Compute two images:
    *   **Primal Image:** Render only the virtual objects. Set vertex colors using the lighting information (stored in voxels) and normals of the corresponding voxels [S1, unofficial].
    *   **Light Difference Coefficients:** Backproject voxel coefficients (representing light occlusion/modification) to the image plane [S1, unofficial]. Multiply these coefficients with the original input image to get the environment influenced by the virtual object [S1, unofficial].
    Combine both images using their depth maps to form the final augmented image [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_based_virtual_object_relighting(hdr_z_images, camera_params, virtual_mesh, voxel_size=64, threshold=0.5):
    """
    hdr_z_images: List of tuples (hdr_color, depth_map, extrinsic_matrix, intrinsic_matrix)
    camera_params: Intrinsic and extrinsic parameters for all cameras
    virtual_mesh: 3D mesh of the virtual object
    voxel_size: Resolution of the voxel grid (N x N x N)
    threshold: Fixed radiance threshold for segmentation
    """
    
    # Initialize voxel grid
    # Voxel structure: {radiance: float, is_environment: bool, is_virtual: bool, light_intensity: float}
    voxel_grid = initialize_voxel_grid(voxel_size)
    
    # Step 1 & 2: Backprojection of Geometry
    for hdr_color, depth_map, extrinsic, intrinsic in hdr_z_images:
        # Median filter depth map to remove low confidence pixels [S1, unofficial]
        depth_map = median_filter(depth_map)
        
        for y in range(height):
            for x in range(width):
                depth = depth_map[y, x]
                if depth is valid:
                    # Backproject pixel to 3D point in camera space
                    point_cam = backproject_pixel(x, y, depth, intrinsic)
                    # Transform to scene reference frame
                    point_scene = transform(point_cam, extrinsic)
                    
                    # Map to voxel coordinates
                    voxel_coord = world_to_voxel(point_scene, voxel_size)
                    
                    # Get radiance (e.g., luminance or specific channel)
                    radiance = get_radiance(hdr_color, x, y)
                    
                    # Store mean radiance if multiple pixels map to same voxel [S1, unofficial]
                    voxel_grid[voxel_coord].radiance = mean(voxel_grid[voxel_coord].radiance, radiance)
                    voxel_grid[voxel_coord].is_environment = True

    # Step 3 & 4: Ray Generation
    ray_list = []
    for hdr_color, depth_map, extrinsic, intrinsic in hdr_z_images:
        for y in range(height):
            for x in range(width):
                depth = depth_map[y, x]
                if depth is valid:
                    point_cam = backproject_pixel(x, y, depth, intrinsic)
                    point_scene = transform(point_cam, extrinsic)
                    radiance = get_radiance(hdr_color, x, y)
                    
                    if radiance > threshold:
                        # Create ray from 3D point towards camera [S1, unofficial]
                        direction = normalize(camera_center - point_scene)
                        ray = Ray(origin=point_scene, direction=direction, intensity=radiance)
                        ray_list.append(ray)
                    else:
                        # Accumulate for ambient estimation [S1, unofficial]
                        ambient_sum += radiance
                        ambient_count += 1

    # Step 5: Ambient Estimation
    ambient_energy = ambient_sum / ambient_count if ambient_count > 0 else 0.0

    # Step 6: Virtual Object Insertion
    # Insert virtual mesh into voxel grid [S1, unofficial]
    for voxel in voxel_grid:
        if voxel_contains_geometry(voxel.world_position, virtual_mesh):
            voxel.is_virtual = True
            
    # Trace rays and update lighting [S1, unofficial]
    for ray in ray_list:
        # Voxel traversal [S1, unofficial]
        hit_virtual = False
        hit_environment = False
        first_virtual_voxel = None
        first_env_voxel = None
        
        current_voxel = voxel_at(ray.origin)
        step = voxel_step(ray.direction)
        
        while not out_of_bounds(current_voxel):
            if voxel_grid[current_voxel].is_virtual and not hit_virtual:
                hit_virtual = True
                first_virtual_voxel = current_voxel
                # Shade virtual voxel with ray intensity [S1, unofficial]
                voxel_grid[current_voxel].light_intensity = ray.intensity
            elif voxel_grid[current_voxel].is_environment and hit_virtual:
                # Store first environment voxel after virtual hit [S1, unofficial]
                # Energy does not reach it, so it is shadowed
                first_env_voxel = current_voxel
                break # Or continue to find further interactions? Text implies storing the first one.
            
            current_voxel = next_voxel(current_voxel, step)
            
        # Note: The text says "store the first environment voxel intersecting the ray... energy... do not reach it".
        # This implies marking the environment voxel as shadowed or storing the occlusion info.
        if first_env_voxel:
            voxel_grid[first_env_voxel].is_shadowed = True

    # Step 7: Rendering
    # Primal Image: Render virtual objects using voxel lighting [S1, unofficial]
    primal_image = render_virtual_objects(virtual_mesh, voxel_grid)
    
    # Light Difference Coefficients: Backproject voxel modifications [S1, unofficial]
    # This likely involves projecting the shadowed/modified voxels back to image space
    light_diff_image = backproject_voxel_modifications(voxel_grid, camera_params)
    
    # Combine with original image [S1, unofficial]
    final_image = combine_images(primal_image, light_diff_image, original_image)
    
    return final_image
```

**Parameters:**
*   **Voxel Grid Resolution:** $64 \times 64 \times 64$ [S1, unofficial].
*   **Radiance Threshold:** Fixed value applied to all images for segmentation [S1, unofficial]. Exact value not specified, but must be consistent across the image set [S1, unofficial].
*   **Assumptions:**
    *   Light sources must be visible during acquisition [S1, unofficial].
    *   Only Lambertian virtual objects and point sources are handled [S1, unofficial].
    *   Inter-reflections and color bleeding are not taken into account [S1, unofficial].
    *   A small number of light samples are enough for plausible lighting [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision Loss:** The voxel approximation causes a lack of precision and square shadows in rendered results [S1, unofficial]. This can be corrected with a 3D smoothing function [S1, unofficial].
*   **Noise:** Depth maps may contain low-confidence pixels. Median filtering is required to eliminate these [S1, unofficial].
*   **Threshold Consistency:** The same threshold must be used for all images in the set to ensure coherent classification [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of rays traced and the voxel grid size. Tracing and updating rays takes 220ms for 120,476 rays in a $64^3$ grid on a 2.83 GHz Core 2 quad PC with nVidia Geforce 9600 GT [S1, unofficial].
*   **Memory Complexity:** Uses a regular voxel grid. Since "Major part of the voxels contain data about the scene," sparse matrix or compression schemes are not needed [S1, unofficial]. Memory usage is proportional to $N^3$ where $N$ is the voxel resolution.
*   **Performance:**
    *   Image acquisition: ~300ms [S1, unofficial].
    *   Constructing/updating octree (or voxel grid): 40ms [S1, unofficial].
    *   Tracing/updating rays: 220ms (for all rays) [S1, unofficial].
    *   Cleaning noisy voxels: 20ms [S1, unofficial].
    *   Rendering final image: 120ms [S1, unofficial].
    *   Total processing time per frame is approximately 480ms on the specified hardware [S1, unofficial].
*   **Strengths:**
    *   No offline computation or geometric modeling of the scene required [S1, unofficial].
    *   No light probes or hemispherical cameras needed [S1, unofficial].
    *   Automatically deals with long and near field illumination and occlusions between real and virtual parts [S1, unofficial].
    *   Seamless insertion of virtual objects [S1, unofficial].
*   **Limitations:**
    *   Square shadows due to voxel approximation [S1, unofficial].
    *   Limited to Lambertian objects and point sources [S1, unofficial].
    *   No inter-reflections or color bleeding [S1, unofficial].
    *   Requires visible light sources during acquisition [S1, unofficial].
    *   Output resolution may be low, leading to blurriness [S1, unofficial].

## Background
*   **Origin:** The method was presented by François Fouquet, Jean-Philippe Farrugia, and Sylvain Brandel from Université de Lyon, CNRS, Université Lyon 1, LIRIS in 2011 [S1, unofficial].
*   **Paper:** "A Voxel-Based Approach for Virtual Objects Relighting" published in Computer Graphics International 2011 [S1, unofficial].
*   **History:** The work proposes an intermediate approach between rendering and image-based techniques, similar to global illumination acceleration methods [S1, unofficial]. It aims to automatically insert virtual objects into acquired views of real scenes [S1, unofficial].
*   **Usage:** Demonstrated by inserting a virtual David statue into a classroom scene and a virtual object into a virtual museum room [S1, unofficial].

## Sources
[S1] A Voxel-Based Approach for Virtual Objects Relighting, https://hal.science/hal-00655769v1, fetched 2026-10-26 (unofficial)
