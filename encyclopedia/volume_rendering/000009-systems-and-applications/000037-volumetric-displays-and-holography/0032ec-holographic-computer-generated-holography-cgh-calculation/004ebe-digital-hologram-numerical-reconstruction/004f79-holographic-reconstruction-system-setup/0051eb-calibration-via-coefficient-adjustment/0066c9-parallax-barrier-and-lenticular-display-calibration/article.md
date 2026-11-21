# Parallax-barrier and Lenticular Display Calibration

Also called: Automatic calibration of parallax-barrier

## Overview
A procedure to automatically calibrate the position of a parallax-barrier with respect to the underlying rear-projection image using a digital camera and homography [S1]. It differs from manual alignment by using a computer-guided laser-cut barrier and automated image processing to map projector sub-pixels to specific viewing angles [S1].

## How it works
**Inputs:**
- A rear-projection screen with a parallax-barrier mounted ~10 mm in front [S1].
- A high-resolution digital camera (3K × 2K) positioned ~3 meters from the screen, facing the center [S1].
- Multiple projectors covering the screen area [S1].
- The physical barrier has slits 1 pixel wide, spaced 16 pixels apart, supporting 16 views [S1].

**Outputs:**
- A 3 × 3 homography matrix for each projector mapping display coordinates to camera pixels [S1].
- A view assignment for every sub-pixel in each projector's image space [S1].

**Data Structures:**
- Homography matrix $H$: 3 × 3 matrix representing the projective transformation between the projector plane and the camera image plane [S1].
- Slit line equation: $ax + by + c = 0$, derived via linear regression [S1].
- View map: A 2D array matching the projector resolution, where each element stores an integer view index (0–15) [S1].

**Steps:**
1. **Homography Computation:**
   - Display checkerboard images on each projector [S1].
   - Capture corresponding images with the camera [S1].
   - Fit horizontal and vertical lines to the observed checkerboard patterns [S1].
   - Compute the intersection of each horizontal line with each vertical line analytically to obtain 48 coordinates of checkerboard corners in the camera image [S1].
   - Compute the 3 × 3 homography matrix $H$ that maps points on the display to camera pixels using these corner correspondences [S1].

2. **Slit Detection in Camera Space:**
   - Project a white image on each projector to create a continuous bright surface [S1].
   - Capture a single image with the camera [S1].
   - Apply a Laplacian edge detector to the captured image [S1].
   - Threshold the result to isolate pixels belonging to the slits (high intensity areas) [S1].
   - Perform a flood-fill operation to group pixels corresponding to each slit [S1].
   - Use linear regression on the pixels of each slit group to compute an analytical line equation for the slit [S1].

3. **Mapping to Projector Space:**
   - Transform the slit lines from camera space to the image space of each projector using the computed homographies $H$ [S1].
   - The resulting slit lines in the projector images are roughly 16 pixels apart [S1].

4. **Interpolation and View Assignment:**
   - Interpolate the line equations for neighboring original slit lines to obtain slit lines for the remaining 15 views [S1].
   - Rasterize all lines for all 16 views [S1].
   - Entirely fill the projector image space so that each sub-pixel has a view assignment [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calibrate_parallax_barrier(projectors, camera, barrier_slit_count=16):
    """
    Calibrate parallax barrier position relative to projectors.
    """
    homographies = {}
    
    # Step 1: Compute Homographies
    for projector in projectors:
        # Display checkerboard on projector
        projector.display_checkerboard()
        
        # Capture image
        camera_image = camera.capture()
        
        # Detect lines
        horizontal_lines = fit_lines(camera_image, orientation='horizontal')
        vertical_lines = fit_lines(camera_image, orientation='vertical')
        
        # Compute corners
        corners = []
        for h_line in horizontal_lines:
            for v_line in vertical_lines:
                corner = intersect(h_line, v_line)
                corners.append(corner)
        
        # Compute homography H mapping display points to camera pixels
        # Note: Display points are known from checkerboard pattern definition
        display_points = get_checkerboard_pattern_points()
        H = compute_homography(display_points, corners)
        homographies[projector.id] = H

    # Step 2: Detect Slits in Camera Space
    # Project white image
    for projector in projectors:
        projector.display_white()
    
    # Capture single image
    white_image = camera.capture()
    
    # Edge detection and thresholding
    edges = laplacian_edge_detector(white_image)
    thresholded = threshold(edges, value=0) # Threshold to isolate slits
    
    # Flood fill to group slit pixels
    slit_groups = flood_fill(thresholded)
    
    # Linear regression for slit lines
    slit_lines_camera = []
    for group in slit_groups:
        line = linear_regression(group) # Returns ax + by + c = 0
        slit_lines_camera.append(line)
    
    # Step 3: Transform Slits to Projector Space
    slit_lines_projector = {}
    for proj_id, H in homographies.items():
        # Inverse homography to map from camera to projector
        H_inv = inverse(H)
        lines_proj = []
        for line_cam in slit_lines_camera:
            # Transform line equation from camera space to projector space
            line_proj = transform_line(line_cam, H_inv)
            lines_proj.append(line_proj)
        slit_lines_projector[proj_id] = lines_proj

    # Step 4: Interpolation and Rasterization
    view_maps = {}
    for proj_id, lines_proj in slit_lines_projector.items():
        # Interpolate lines for all 16 views
        # Assuming lines_proj contains the primary visible slits
        all_lines = interpolate_lines(lines_proj, count=barrier_slit_count)
        
        # Rasterize and assign views
        width, height = projector.resolution
        view_map = empty_array(width, height, dtype=int)
        
        for view_idx, line in enumerate(all_lines):
            # Rasterize line and fill regions between lines with view_idx
            # This step assigns each sub-pixel a view
            assign_view_to_pixels(view_map, line, view_idx, all_lines)
        
        view_maps[proj_id] = view_map

    return homographies, view_maps
```

**Parameters:**
- Camera resolution: 3K × 2K [S1].
- Camera distance: ~3 meters [S1].
- Barrier slit width: 1 pixel [S1].
- Barrier slit spacing: 16 pixels [S1].
- Number of views: 16 [S1].

**Edge Cases:**
- If the camera resolution is insufficient to detect slits, the Laplacian edge detection may fail [S1].
- Degenerate checkerboard patterns (e.g., collinear points) will fail homography computation [G].

## Operating characteristics
- **Time Complexity:** Dominated by line fitting and homography computation per projector. Linear in the number of pixels for edge detection and flood fill.
- **Memory Complexity:** Proportional to the camera and projector resolution for storing images and view maps.
- **Strengths:** Automated, high precision due to high-resolution camera and analytical line fitting [S1].
- **Limitations:** Requires precise camera placement and high-resolution imaging [S1].

## Background
Originated in the context of distributed rendering for multiview parallax displays [S1]. The method uses a computer-guided laser to cut the parallax-barrier slits into a sheet of black Mylar polyester film, which is glued to a sheet of glass mounted ~10 mm in front of the rear-projection screen [S1].

## Sources
[S1] Distributed rendering for multiview parallax displays, https://doi.org/10.1117/12.650171, fetched 2026-11-01
