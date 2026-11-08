# Space Leaping

Also called: Fast Traversal, Skip Traversal, Space Leaping using Ray Coherence, Vikon Space Leaping System

## Overview
Space Leaping is a ray-coherence-based acceleration technique for volume rendering that skips empty regions by leveraging the spatial correlation between adjacent rays [S7]. Unlike hierarchical data structures (e.g., octrees) that require complex traversal logic and significant memory overhead, this method uses a lightweight image-plane leap buffer to record the sample count to the first non-empty voxel for each pixel, allowing neighboring rays to skip empty space based on that information [S7]. It cannot achieve as much empty space skipping as techniques using data structures for empty/non-empty voxel info because it only leaps one out of every 4 rays, getting 75% of possible acceleration [S7]. It is primarily suited for applications where the camera is close to the object surface, such as virtual endoscopy, where large volumes of empty space exist in front of the camera [S7].

## How it works
Space Leaping exploits the fact that rays casting through empty space will hit the same object boundary at similar depths [S7]. By storing the depth of the first non-empty sample for each pixel, rays that are spatially close can "leap" over empty regions if their neighbors have already identified the hit point [S7].

### Inputs and Outputs
*   **Input:** Volume dataset, camera parameters (position, orientation, projection), sampling distance $d_s$.
*   **Output:** Rendered image.
*   **Auxiliary Data:** A "Leap Buffer" of size $W \times H$ (image plane dimensions), storing the number of samples from the image plane to the first non-empty voxel for each pixel [S7].

### Data Structures
**Leap Buffer:**
*   Dimensions: $W \times H$ (width and height of the image plane).
*   Type: Integer array.
*   Content: For pixel $(u, v)$, `LeapBuffer[u][v]` stores the count of samples the detector ray took to reach the first non-empty cell [S7].
*   Memory: $O(N^2)$ where $N \times N$ is the image size (e.g., 256x256 bytes for 256x256 resolution) [S7].

### Algorithm Steps

1.  **Initialization:**
    *   Initialize `LeapBuffer` with zeros [S7].
    *   In the Vikon implementation, the buffer is filled by casting rays for a subset of pixels (e.g., every 4th ray) [S7].

2.  **Detector Ray Casting (Filling the Buffer):**
    *   Cast rays for a subset of pixels (e.g., checkerboard or stride-4 pattern) [S7].
    *   For each detector ray, march through the volume [S7].
    *   Identify the first sample point that falls within a non-empty (active) voxel [S7].
    *   Store the number of samples taken to reach this point in `LeapBuffer` for that pixel [S7].
    *   If the ray hits nothing (transparency), store a value indicating "empty" or handle via boundary conditions [S7].

3.  **Space Leap Ray Casting (Rendering):**
    *   For each pixel $(u, v)$ in the image:
        *   Retrieve `L = LeapBuffer[u][v]` [S7].
        *   **Boundary Condition Check:** If `L == 0`, do not leap. Cast a standard detector ray from the image plane [S7].
        *   **Leap Calculation:**
            *   Calculate the starting position for the leap ray. The leap ray starts at the image plane and advances by $L$ samples [S7].
            *   However, the leap buffer stores the count to the *first non-empty cell* [S7].
            *   The sample *before* that is the last empty sample [S7].
            *   Let $P_{start}$ be the position after $L$ samples. Backtrack the ray by one sample distance $d_s$ to land in the empty cell preceding the hit [S7].
            *   **Backtracking Limit:** Do not backtrack beyond the leap buffer value to avoid going behind the image plane. The maximum backtracking samples is $L$ [S7].
        *   **Verification:**
            *   Check if the backtracked position is inside an empty cell [S7].
            *   If the first sample of the leap ray is inside a non-empty cell, backtrack further until an empty cell is found [S7].
            *   If the leap ray starts in an empty cell, it is definitely before the object group [S7].
        *   **Rendering:**
            *   From the verified start position, perform standard volume rendering (marching and compositing) until the object is fully rendered or opacity threshold is reached [S7].

### Formulas
*   **Leap Start Position:**
    $$ P_{leap} = P_{image} + (L - 1) \cdot d_s \cdot \vec{d} $$
    Where $P_{image}$ is the image plane origin, $L$ is the leap buffer value, $d_s$ is the sampling distance, and $\vec{d}$ is the ray direction unit vector [S7].
    *Note: The sources describe backtracking to the "last sample in an empty cell". If $L$ is the count to the first non-empty, the index of the last empty is $L-1$ (0-indexed) or similar depending on implementation. The key is to ensure the ray starts in empty space.*

*   **Backtracking Limit:**
    $$ N_{backtrack} \le L $$
    The number of samples to backtrack is limited by $L$ to prevent the ray from moving behind the image plane [S7].

### Parameters
*   **Sampling Distance ($d_s$):** Determines the granularity of the leap. Smaller $d_s$ increases accuracy but reduces leap efficiency [S7].
*   **Leap Stride:** The Vikon system leaps one out of every 4 rays, achieving 75% of the possible acceleration [S7].

### Edge Cases and Pitfalls
*   **Zero Leap Buffer Value:** If `LeapBuffer[u][v] == 0`, the space leap ray should not be leaped at all. A standard ray must be cast [S7].
*   **Backtracking Beyond Image Plane:** Care must be taken not to backtrack the leap ray beyond the value stored in the leap buffer, as this would place the ray behind the image plane, causing incorrect rendering [S7].
*   **Non-Empty Start:** If the leap ray's first sample falls inside a non-empty cell, it must be backtracked until it is inside an empty cell [S7].
*   **Limitation:** Space leaping is only possible between the image plane and the first object hit. Empty space beyond the first hit cannot be leaped using this method [S7].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_space_leaping(volume, camera, leap_buffer, sampling_distance, opacity_threshold=0.99):
    """
    Renders the volume using Space Leaping.
    Assumes leap_buffer is pre-filled by detector rays.
    """
    height = camera.height
    width = camera.width
    image = [[0.0, 0.0, 0.0, 0.0] for _ in range(height)]
    
    for y in range(height):
        for x in range(width):
            L = leap_buffer[y][x]
            ray_dir = camera.get_ray_direction(x, y)
            ray_origin = camera.get_ray_origin(x, y)
            
            # Boundary Condition: If L is 0, do not leap
            if L == 0:
                color, alpha = standard_ray_march(volume, ray_origin, ray_dir, sampling_distance, opacity_threshold)
                image[y][x] = [color[0], color[1], color[2], alpha]
                continue
            
            # Leap buffer stores samples to first non-empty cell.
            # We want to start at the last empty cell (L-1 samples from origin)
            leap_samples = L - 1
            
            # Guard against negative leap samples
            if leap_samples < 0:
                leap_samples = 0
                
            # Calculate leap start position
            leap_origin = ray_origin + leap_samples * sampling_distance * ray_dir
            
            # Backtracking: If the first sample at leap_origin is non-empty,
            # backtrack until empty. Limit backtracking to L samples.
            current_pos = leap_origin
            backtracked = 0
            
            while not is_empty(volume, current_pos) and backtracked < L:
                current_pos -= sampling_distance * ray_dir
                backtracked += 1
            
            # If we backtracked all the way to or behind image plane, fallback to full march
            if backtracked >= L:
                color, alpha = standard_ray_march(volume, ray_origin, ray_dir, sampling_distance, opacity_threshold)
            else:
                # Render from the verified empty start position
                color, alpha = standard_ray_march(volume, current_pos, ray_dir, sampling_distance, opacity_threshold)
            
            image[y][x] = [color[0], color[1], color[2], alpha]

    return image

def standard_ray_march(volume, origin, direction, step_size, opacity_threshold):
    """
    Standard front-to-back volume compositing loop.
    """
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_alpha = 0.0
    pos = origin
    max_samples = int(volume.depth / step_size)
    
    for _ in range(max_samples):
        density = volume.sample(pos)
        opacity = density * 0.5  # Simplified transfer function [G]
        
        if opacity > 0.0:
            weight = (1.0 - accumulated_alpha) * opacity
            color_sample = volume.color_sample(pos)
            accumulated_color[0] += weight * color_sample[0]
            accumulated_color[1] += weight * color_sample[1]
            accumulated_color[2] += weight * color_sample[2]
            accumulated_alpha += weight
            
            if accumulated_alpha >= opacity_threshold:
                break
        
        pos += step_size * direction
        
    return accumulated_color, accumulated_alpha

def is_empty(volume, pos):
    """
    Checks if a position falls in an empty/transparent region.
    """
    return volume.sample(pos) <= 0.0

def fill_leap_buffer(volume, camera, leap_buffer, sampling_distance, stride=4):
    """
    Fills the leap buffer by casting rays for a subset of pixels.
    """
    height = camera.height
    width = camera.width
    max_samples = int(volume.depth / sampling_distance)
    
    for y in range(height):
        for x in range(width):
            # Only cast for subset of rays (e.g., every 4th)
            if (x % stride != 0) or (y % stride != 0):
                continue
                
            ray_dir = camera.get_ray_direction(x, y)
            ray_origin = camera.get_ray_origin(x, y)
            
            samples = 0
            pos = ray_origin
            
            # March until non-empty or end of volume
            while samples < max_samples:
                if is_empty(volume, pos):
                    samples += 1
                    pos += sampling_distance * ray_dir
                else:
                    # Found first non-empty
                    leap_buffer[y][x] = samples
                    break
            else:
                # Hit nothing
                leap_buffer[y][x] = 0
```

## Subtopics

### [Boundary Pixel Detector Ray Casting](5870-boundary-pixel-detector-ray-casting.md)
Boundary Pixel Detector Ray Casting is a safety mechanism within Space Leaping that ensures volumetric objects projecting exclusively onto the edges of the image buffer are not erroneously skipped. It distinguishes itself from standard space leaping by treating all image boundary pixels as mandatory "detector pixels," preventing the algorithm from leaping over objects located at the periphery of the view frustum.

## Operating characteristics
*   **Time Complexity:** $O(N^2)$, where $N \times N$ is the image plane size. The primary computational cost is filling the leap buffer, which scans the buffer once [S7].
*   **Memory Complexity:** $O(N^2)$, requiring only one buffer (the leap buffer) of size equal to the image plane (e.g., 256x256 bytes for 256x256 resolution) [S7].
*   **Performance:** Achieved speedups of up to 165% over non-space-leap ray casting on a Pentium-IV, 2.6GHz PC with 1GB RAM [S7].
*   **Acceleration Efficiency:** By leaping one out of every 4 rays, it achieves 75% of the possible acceleration compared to techniques that use data structures for empty/non-empty voxel info [S7].
*   **Strengths:** Low memory overhead, simple implementation, suitable for interactive rendering in virtual endoscopy where the camera is close to the object surface [S7].
*   **Limitations:** Cannot achieve as much empty space skipping as hierarchical methods. Space leaping is only possible between the image plane and the first object hit; empty space beyond the first hit cannot be leaped [S7].

## Background
Space Leaping was incorporated into the Vikon system for perspective direct volume rendering, specifically for endoscopy applications like Virtual Colonoscopy [S7]. The system allows users to navigate inside 3D datasets, requiring interactive rendering capabilities [S7]. The technique is ideal for virtual endoscopy and applications requiring camera placement close to the object surface, as well as high-resolution volume rendering with orthographic and perspective projections [S7]. The method exploits temporal coherence in volume animations by storing coordinates of the first non-transparent voxel in a C-buffer, which can be used to determine where to start tracing the ray in subsequent frames [S1, unofficial].

## Sources
[S1] Direct Volume Rendering Techniques, https://www.scribd.com/document/853597780/VIS-Modules-06-Direct-Volume-Rendering, fetched 2026-10-25 (unofficial)
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S3] What Is Ray Marching?, https://computergraphics.stackexchange.com/questions/161/what-is-ray-marching-is-sphere-tracing-the-same-thing, fetched 2026-10-26 (unofficial)
[S4] Direct Volume Rendering | CGL @ ETHZ, https://cgl.ethz.ch/teaching/former/scivis_07/Notes/stuff/StuttgartCourse/VIS-Modules-06-Direct_Volume_Rendering.pdf, fetched 2026-10-26 (unofficial)
[S5] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
[S6] Empty Space Skipping, https://users.cg.tuwien.ac.at/bruckner/homepage/content/mastersthesis/html/node11.html, fetched 2026-10-28 (unofficial)
[S7] Light Weight Space Leaping using Ray Coherence, https://sci-hub.red/storage/moscow/3086/b1c8d8b37aeadf6b2ca729a1a7a2ae54/light-weight-space-leaping-using-ray-coherence.pdf, fetched 2026-10-28
[S8] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
