# Multiresolution Atlas Buffer (MAB)

Also called: VR DVR acceleration wrapper, bracket approach for VR raycasting, gaze-dependent resolution DVR

## Overview
The Multiresolution Atlas Buffer (MAB) is an acceleration wrapper for Direct Volume Rendering (DVR) in Virtual Reality (VR) that reduces ray density adaptively based on peripheral vision and Head-Mounted Display (HMD) lens distortion, then reconstructs the full-resolution image via compositing [S1]. It differs from standard foveated rendering by not requiring high-speed eye-tracking hardware; instead, it leverages the fixed nature of HMD lenses and the non-uniform resolution of the human visual system to create a static, gaze-centered resolution map [S1]. Unlike sibling methods that modify the ray traversal itself (e.g., [Level-of-detail (LOD) methods](1007-level-of-detail-lod-methods.md)) or dynamic volume resolution ([Dynamic Volume Resolution & Alpha Mask Update](2539-dynamic-volume-resolution-alpha-mask-update.md)), MAB operates as a "bracket" around existing raycasters, modifying ray construction and post-traversal compositing [S1].

## How it works
The method consists of two primary steps injected into the rendering pipeline: (1) adaptive ray density reduction during ray construction, and (2) compositing computation after ray traversal to reconstruct the final image [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Current frame buffer dimensions (typically matching HMD resolution, e.g., $2880 \times 1600$) [S1].
    *   Gaze center coordinates in screen space (assumed to be the center of the viewport or a fixed point due to lack of eye-tracking) [S1].
    *   Target refresh rate (e.g., $\ge 90$ Hz, tending toward $120$ Hz or more) [S1].
    *   Current frame timing metrics [S1].
*   **Outputs:**
    *   A full-resolution, consistent framebuffer image submitted to the HMD [S1].
    *   A set of rays with reduced density sent to the underlying raycaster [S1].

### Data Structures
**Multiresolution Atlas Buffer (MAB):**
*   **Layout:** The MAB is a texture or buffer partitioned into multiple regions. Each region corresponds to the entire frame buffer at a specific resolution level [S1].
*   **Resolution Levels:** The system supports discrete levels with progressively lower resolutions. The resolution scaling values between subsequent levels are arbitrary and not necessarily a power of two [S1].
*   **Ordering:** Levels are ordered to minimize the overall framebuffer size [S1].
*   **Discarding:** Black areas (empty regions) in each level are discarded before ray traversal to save computation [S1].

**Mapping Functions:**
A bidirectional mapping between the framebuffer space and MAB space is mandatory [S1].
*   **Forward Mapping ($f_{map}$):** Assigns every MAB fragment a unique position in the framebuffer and a blending factor [S1].
*   **Inverse Mapping:** Assigns each framebuffer fragment several MAB fragments and blending factors for compositing [S1].

### Numbered Steps

1.  **MAB Construction:**
    *   Define the set of resolution levels. Unlike Mip-Maps, ratios between levels are not fixed (e.g., not strictly 4:1) [S1].
    *   Allocate the MAB structure with regions for each resolution level [S1].

2.  **Ray Construction (Opening Brace):**
    *   For each pixel in the target framebuffer, determine the required resolution level based on the distance to the gaze center [S1].
    *   Compute the ray origin and direction. The ray density is adaptively reduced in peripheral regions [S1].
    *   Feed the reduced set of rays to the underlying raycaster [S1].

3.  **Ray Traversal:**
    *   The underlying raycaster traverses the volume data for the provided rays and stores results into the corresponding regions of the MAB [S1].

4.  **Compositing (Closing Brace):**
    *   Render a full-screen primitive covering the entire output framebuffer [S1].
    *   For each generated fragment in the framebuffer, compute the mapping to the MAB using two functions:
        *   **Function $f_a$:** Takes the distance to the center of gaze (based on screen-space coordinates) and outputs a set of blending factors for the MAB [S1].
        *   **Function $f_b$:** Takes the screen-space coordinates of the fragment and outputs a set of corresponding coordinates in the MAB [S1].
    *   For each non-zero entry in the blending factor set from $f_a$, perform a lookup into the MAB using the coordinates from $f_b$ to obtain stored color values [S1].
    *   Blend these values together into a single color for display [S1].

5.  **Adaptive Timing Adjustment:**
    *   Evaluate frame timings for the current frame [S1].
    *   Adjust rendering parameters (such as the number of resolution levels or the resolution scaling factors) for the next subsampling stage to maintain a consistent refresh rate [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
HMD_WIDTH = 2880
HMD_HEIGHT = 1600
TARGET_FPS = 90.0

# Data Structures
class MABLevel:
    width: int
    height: int
    buffer: Texture2D # Stores raycasting results

class MAB:
    levels: List[MABLevel]
    # Ordered to minimize overall framebuffer size

# Mapping Functions
def f_a(screen_x: float, screen_y: float, gaze_x: float, gaze_y: float) -> List[float]:
    """
    Computes blending factors based on distance to gaze center.
    Returns a list of weights corresponding to MAB levels.
    """
    dist = sqrt((screen_x - gaze_x)**2 + (screen_y - gaze_y)**2)
    # Implementation details for blending factor calculation are not specified in sources.
    # [G] Typically involves a falloff function (e.g., Gaussian or linear) 
    # mapping distance to a set of weights for available resolution levels.
    return compute_blending_factors(dist)

def f_b(screen_x: float, screen_y: float, level_index: int) -> Tuple[float, float]:
    """
    Computes MAB coordinates for a specific framebuffer fragment and level.
    """
    level = mab.levels[level_index]
    # Map screen coordinates to the resolution of the specific MAB level
    scale_x = level.width / HMD_WIDTH
    scale_y = level.height / HMD_HEIGHT
    return (screen_x * scale_x, screen_y * scale_y)

def render_frame(gaze_x: float, gaze_y: float, frame_time: float):
    # Step 1: Adaptive Parameter Adjustment
    # Adjust resolution levels or scaling factors based on frame_time to maintain TARGET_FPS
    # [G] Logic to compare frame_time with 1/TARGET_FPS and adjust MAB configuration.
    adjust_mab_parameters(frame_time)

    # Step 2: Ray Construction (Opening Brace)
    rays = []
    for y in range(HMD_HEIGHT):
        for x in range(HMD_WIDTH):
            # Determine which MAB level(s) to use for this pixel
            # [G] Logic to select primary level based on distance to gaze
            level_idx = select_level_for_pixel(x, y, gaze_x, gaze_y)
            
            # Compute ray for the selected resolution
            # The ray density is reduced in peripheral regions
            ray = compute_ray_for_resolution(x, y, mab.levels[level_idx])
            rays.append(ray)

    # Step 3: Ray Traversal
    # Render rays into the MAB
    for ray in rays:
        result = raycaster.traverse(ray)
        # Store result in the appropriate MAB level region
        store_in_mab(ray, result)

    # Step 4: Compositing (Closing Brace)
    output_buffer = Texture2D(HMD_WIDTH, HMD_HEIGHT)
    
    # Render full-screen primitive
    for y in range(HMD_HEIGHT):
        for x in range(HMD_WIDTH):
            # Get blending factors
            factors = f_a(x, y, gaze_x, gaze_y)
            
            final_color = (0.0, 0.0, 0.0, 0.0)
            
            # Composite from multiple MAB levels
            for i, weight in enumerate(factors):
                if weight > 0:
                    # Get MAB coordinates
                    mab_x, mab_y = f_b(x, y, i)
                    
                    # Lookup color from MAB
                    color = mab.levels[i].buffer.sample(mab_x, mab_y)
                    
                    # Blend
                    final_color += color * weight
            
            output_buffer.set_pixel(x, y, final_color)

    return output_buffer
```

### Parameters
*   **Resolution Scaling:** Arbitrary values between subsequent MAB levels; not necessarily powers of two [S1].
*   **HMD Update Rate:** At least 90 Hz, tending toward 120 Hz or more [S1].
*   **HMD Resolution:** Around $2880 \times 1600$ pixels [S1].

### Edge Cases and Pitfalls
*   **Eye Tracking:** The method does not consider rapid eye movements because HMD lenses are fixed and provide only a slight angle with the highest projected resolution [S1].
*   **Precision:** Not given in the sources.
*   **Boundaries:** The mapping functions $f_a$ and $f_b$ must handle boundary conditions where gaze is near the edge of the viewport [S1].

## Subtopics

### FAVR (Foveated Adaptive Volume Rendering)
The predecessor method by Waschk and Krüger (2020) that considers human perception and HMD lens distortion but is the conceptual basis for MAB [S1].

### Mip-Maps
A related texture filtering technique. MAB differs by supporting arbitrary, non-power-of-two resolution ratios between levels to minimize noticeable differences [S1].

## Operating characteristics
*   **Time Complexity:** The method reduces the number of primary rays traversed by the raycaster, thereby reducing the computational load of the volume rendering step. The compositing step adds overhead proportional to the number of framebuffer pixels and the number of MAB levels sampled per pixel [S1].
*   **Memory Complexity:** Requires storage for the MAB, which contains multiple resolution levels of the frame buffer. The levels are ordered to minimize the overall framebuffer size [S1].
*   **Measured Performance:** Designed to make DVR feasible in VR setups by guaranteeing stable high refresh rates (90 Hz+) to avoid cyber-sickness [S1].
*   **Strengths:** Works as a wrapper around existing raycasters with minimal code changes; does not require eye-tracking hardware; adapts to hardware characteristics [S1].
*   **Limitations:** Does not account for rapid eye movements; relies on fixed gaze assumption (typically center) [S1].
*   **When to use:** VR setups with HMDs where DVR is performance-bound by fragment processing and high refresh rates are critical [S1].
*   **When not to use:** Desktop rendering without VR constraints; scenarios requiring precise foveated rendering based on actual eye position [S1].

## Background
The Multiresolution Atlas Buffer (MAB) method was introduced by Jens Krüger and Andre Waschk in the WSCG 2022 Proceedings (CSRN 3201) [S1]. It builds upon the FAVR (Foveated Adaptive Volume Rendering) approach by Waschk and Krüger (2020) and earlier gaze-dependent rendering works [S1]. The method addresses the challenge of rendering high-resolution volume data in VR, where HMD resolutions (e.g., $2880 \times 1600$) and high refresh rates (90-120 Hz) create significant performance bottlenecks for DVR raycasters [S1]. By leveraging the non-uniform resolution of the human visual system and the fixed nature of HMD lenses, MAB reduces ray density in peripheral regions while maintaining visual fidelity in the center [S1].

## Sources
[S1] Embracing Raycasting for Virtual Reality, https://doi.org/10.24132/csrn.3201.16, fetched 2026-10-29
