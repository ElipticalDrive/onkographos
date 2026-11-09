# Progressive Ray Tracing

Also called: Progressive Refinement Ray Casting

## Overview
Progressive Ray Tracing is an adaptive sampling technique that produces lower-quality images which are then improved over time [S1, unofficial]. It solves the problem of maintaining interactive frame rates during user interaction by computing only a subset of rays, then progressively filling in the remaining pixels with actual values once interaction ceases [S2, book]. It sets itself apart from its sibling [Checkerboard Progressive Rendering](4836-checkerboard-progressive-rendering.md) by using interpolation to fill gaps during interaction rather than temporal blending of partial frames, and by refining the image spatially (pixel-by-pixel) rather than just reducing the active pixel count per frame.

## How it works
**Inputs and Outputs:**
- Input: User interaction state (active/inactive), scene geometry/volume data, camera parameters, subsampling factor $n$.
- Output: A rendered image that transitions from an interpolated approximation to a fully ray-traced result.

**Data Structures:**
- Image buffer: 2D array of color values $(C_{x,y})$.
- Interaction flag: Boolean indicating if the user is currently manipulating the view.

**Steps:**
1. **Interaction Phase:**
   - If the user is interacting with the scene:
     - Compute ray casting results only for every $n$-th pixel along each image dimension (e.g., $n=2$ for a checkerboard-like pattern, resulting in a 4x speedup for 2D) [S2, book].
     - Let $P_{sampled}$ be the set of pixels where rays were actually traced.
     - For all other pixels $P_{missing}$, calculate their color values using interpolation based on the surrounding sampled pixels [S2, book].
     - Display the interpolated image.

2. **Refinement Phase:**
   - If the user stops interacting:
     - Identify all pixels in $P_{missing}$ that still hold interpolated values.
     - Progressively replace these interpolated values with actual ray-traced results. The order of replacement can be arbitrary or based on a heuristic (e.g., distance from sampled pixels), but the source specifies "progressively filled in" [S2, book].
     - As each pixel is updated, the image quality improves until all pixels are fully ray-traced.

**Parameters:**
- $n$: The subsampling stride. Typical value $n=2$ is mentioned, yielding a four-times speed-up [S2, book]. Larger $n$ values increase speed but result in blurrier, less detailed images [S2, book].

**Edge Cases and Pitfalls:**
- **High-Frequency Details:** The method tends to lose high-frequency details during the interpolated phase [S1, unofficial].
- **Bias vs. Noise:** While progressive refinement reduces bias compared to some other low-quality methods, it may still introduce noise if not combined with techniques like Russian roulette termination [S1, unofficial].
- **Interpolation Artifacts:** The quality of the interpolated image depends heavily on the interpolation method used. Poor interpolation can lead to noticeable artifacts before refinement completes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_ray_tracing(scene, camera, n=2):
    # Initialize image buffer with zeros or previous frame
    image = initialize_image_buffer()
    user_interacting = True
    
    while True:
        if user_interacting:
            # Compute only every n-th ray
            for y in range(0, height, n):
                for x in range(0, width, n):
                    ray = camera.generate_ray(x, y)
                    color = trace_ray(scene, ray)
                    image[x, y] = color
            
            # Interpolate missing pixels
            image = interpolate_missing_pixels(image, n)
            
            # Check for interaction stop
            if not check_user_input():
                user_interacting = False
        else:
            # Refine the image
            # Identify pixels that are not yet fully traced
            # This could be tracked via a mask or by checking against a 'traced' flag
            pixels_to_trace = get_untraced_pixels(image, n)
            
            if not pixels_to_trace:
                break # Rendering complete
            
            # Progressively trace remaining pixels
            # In a real implementation, this would be done incrementally per frame
            for (x, y) in pixels_to_trace:
                ray = camera.generate_ray(x, y)
                color = trace_ray(scene, ray)
                image[x, y] = color
                
            # Update display
            display_image(image)
            
            # Check if all pixels are traced
            if is_fully_traced(image, n):
                break
        
        display_image(image)
```

## Subtopics

### Checkerboard Progressive Rendering
Reduces per-frame compute by tracing only half the pixels each frame while blending results over time. [Link](4836-checkerboard-progressive-rendering.md)

## Operating characteristics
- **Time Complexity:** During interaction, the complexity is reduced by a factor of $n^2$ for 2D subsampling (e.g., 4x speedup for $n=2$) [S2, book]. Refinement phase complexity is proportional to the number of missing pixels.
- **Memory Complexity:** Requires storage for the image buffer and potentially a mask to track traced pixels.
- **Performance:** Provides a four-times speed-up with $n=2$ without a noticeable decrease in image quality during interaction [S2, book]. Larger reductions in sampling rate increase speed but degrade image quality [S2, book].
- **Strengths:** Maintains interactivity during view manipulation; improves image quality progressively after interaction stops.
- **Limitations:** Loses high-frequency details during the interpolated phase; requires interpolation which may introduce artifacts; refinement phase may still be slow for large scenes.
- **When to Use:** Interactive volume visualization where user interaction is frequent but final high-quality rendering can be deferred.
- **When Not to Use:** When high-frequency details are critical during interaction; when real-time feedback is required without any interpolation artifacts.

## Background
Progressive ray tracing was introduced as a technique to produce lower quality images which are then improved with time [S1, unofficial]. It is part of the broader field of adaptive sampling and resolution techniques in volume rendering. The method is discussed in the context of advanced computer graphics, specifically within the VTK (Visualization Toolkit) framework [S2, book]. It is often compared with object-order techniques like splatting, which may use precomputed resolution levels or octree representations [S2, book].

## Sources
[S1] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
