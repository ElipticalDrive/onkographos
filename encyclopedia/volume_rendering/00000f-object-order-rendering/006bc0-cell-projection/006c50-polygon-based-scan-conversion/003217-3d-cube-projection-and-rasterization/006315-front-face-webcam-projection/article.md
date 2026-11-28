# Front-Face Webcam Projection

Also called: 8x8x8 Cube Front Face Projection, Webcam stream projection on volumetric display

## Overview
A method to project a 2D webcam video feed onto the front face of an 8x8x8 volumetric LED cube by averaging pixel colors in grid squares [S1, unofficial]. It differs from sibling methods like [2D Raycasting with Z-Buffering](8233-2d-raycasting-with-z-buffering.md) and [Cube Map Rendering](16527-cube-map-rendering.md) by restricting output to a single 2D plane of voxels, ignoring depth information for the primary image [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A video stream from a webcam.
*   Output: RGB intensity values for the 64 voxels on the front face of the cube.

**Data Structures:**
*   `Frame`: A 2D array of pixels, size $512 \times 512$. Each pixel contains RGB values.
*   `Grid`: A 2D array of voxels, size $8 \times 8$. Each voxel contains an RGB value.

**Steps:**
1.  Capture a single frame from the webcam feed [S1, unofficial].
2.  Crop the frame to a resolution of $512 \times 512$ pixels. This ensures the width and height are multiples of 8 [S1, unofficial].
3.  Divide the $512 \times 512$ frame area into an $8 \times 8$ grid of squares. Each square has a width and height of $512 / 8 = 64$ pixels [S1, unofficial].
4.  For each square $(i, j)$ in the grid, where $i, j \in [0, 7]$:
    a. Identify the pixel coordinates covered by the square.
    b. Calculate the average RGB value of all pixels within that square.
    c. Assign this averaged RGB value to the voxel at position $(i, j)$ in the output grid [S1, unofficial].
5.  Light up the voxels on the front face of the cube using the values from the $8 \times 8$ grid [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def process_webcam_frame(webcam_frame):
    # Step 2: Crop to 512x512
    # Assuming webcam_frame is larger or different aspect ratio
    cropped_frame = webcam_frame.crop(0, 0, 512, 512)
    
    # Step 3: Define grid parameters
    grid_size = 8
    square_size = 512 // grid_size  # 64
    
    output_grid = [[None for _ in range(grid_size)] for _ in range(grid_size)]
    
    # Step 4: Average pixels in each square
    for y in range(grid_size):
        for x in range(grid_size):
            # Define square boundaries
            x_start = x * square_size
            y_start = y * square_size
            x_end = x_start + square_size
            y_end = y_start + square_size
            
            r_sum = 0
            g_sum = 0
            b_sum = 0
            pixel_count = 0
            
            # Iterate over all pixels in the square
            for py in range(y_start, y_end):
                for px in range(x_start, x_end):
                    pixel = cropped_frame.get_pixel(px, py)
                    r_sum += pixel.r
                    g_sum += pixel.g
                    b_sum += pixel.b
                    pixel_count += 1
            
            # Calculate average
            if pixel_count > 0:
                avg_r = r_sum // pixel_count
                avg_g = g_sum // pixel_count
                avg_b = b_sum // pixel_count
                output_grid[y][x] = (avg_r, avg_g, avg_b)
            else:
                output_grid[y][x] = (0, 0, 0)
                
    # Step 5: Render to cube front face
    render_front_face(output_grid)
```

**Parameters:**
*   Frame resolution: $512 \times 512$ pixels [S1, unofficial].
*   Grid subdivision: $8 \times 8$ squares per frame [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Using a 3D display for a 2D image results in lost resolution and limited utility [S1, unofficial].
*   If the input frame is not square or not divisible by 8, cropping or scaling is required to maintain the $512 \times 512$ constraint [S1, unofficial].

## Subtopics

### Depth and Color Projection with Kinect
A method to project both color and depth information from a Kinect sensor onto an 8x8x8 volumetric LED cube, mapping depth to the Z-axis position of voxels [S1, unofficial]. It differs from standard webcam projection by utilizing depth data to vary voxel Z-position, whereas webcam projection only uses color on a fixed 2D plane [S1, unofficial].
*   **Steps:**
    1.  Divide the Kinect video image into 64 areas ($8 \times 8$ grid) and calculate the average pixel values (color) for each area [S1, unofficial].
    2.  Recompose an $8 \times 8$ image representing the depth using the averaged depth values from the Kinect for every pixel in each area [S1, unofficial].
    3.  Use the averaged depth values to determine the Z-axis position of each voxel on the cube [S1, unofficial].
    4.  Map the averaged color values to the voxels at their determined Z-positions [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels in the cropped frame ($O(W \times H)$), where $W=H=512$. The grid aggregation is constant relative to the output size ($8 \times 8$).
*   **Memory Complexity:** $O(1)$ for the output grid ($8 \times 8$ voxels). Input frame storage is $O(512 \times 512)$.
*   **Strengths:** Simple implementation; low computational overhead for the aggregation step; suitable for real-time video feeds on modest hardware like a Raspberry Pi [S1, unofficial].
*   **Limitations:** Significant loss of spatial resolution (from 512x512 to 8x8); no depth information in the standard webcam variant; limited utility for complex 3D visualization [S1, unofficial].
*   **When to use:** When a simple, real-time 2D representation of a video feed is needed on a volumetric display.
*   **When not to use:** When high-resolution detail or 3D spatial information is required.

## Background
Originated from tutorials on volumetric displays, specifically the L3DCube, using Processing for implementation [S1, unofficial]. The method was developed to demonstrate basic video projection on voxel-based hardware, with variants extending to depth sensors like Kinect [S1, unofficial]. It is often used in educational or hobbyist contexts for data visualization and interactive art [S1, unofficial].

## Sources
[S1] Volumetric display and data visualization: 4 animations for the L3DCube, http://digitaljunky.io/volumetric-display-and-data-visualization-4-animations-for-the-l3dcube/, fetched 2026-10-26 (unofficial)
