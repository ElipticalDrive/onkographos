# VR-NeRF

Also called: Virtualized Walkable Spaces

## Overview
VR-NeRF is a holistic pipeline for the capture, reconstruction, and rendering of high-fidelity walkable spaces in virtual reality, designed to achieve higher resolution, frame rate, and visual fidelity than standard NeRF [S1]. It distinguishes itself by targeting native rendering on headsets like the Meta Quest Pro with strict latency and resolution constraints, utilizing specific training configurations and pruning strategies to maintain smoothness via Asynchronous Spacewarp (ASW) [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Thousands of uniformly distributed HDR photos of a scene captured via a custom multi-camera rig [S1].
*   **Output:** A reconstructed neural radiance field optimized for VR rendering, capable of generating stereoscopic eyebuffers at high frame rates [S1].

**Data Structures:**
*   **Voxel Grid:** A 3D grid storing features and density. The sources mention grid resolutions such as $1024^3$ for aggressive pruning scenarios [S1].
*   **Pruning Mask:** A binary or thresholded structure determining occupied vs. free voxels based on density [S1].

**Method Steps:**
1.  **Capture:** Acquire HDR images using a one-of-a-kind multi-camera rig that captures thousands of uniformly distributed photos [S1].
2.  **Optimization:** Optimize the neural radiance field using a novel perceptual color space for HDR model optimization [S1].
3.  **Feature Weighting:** Apply level-of-detail feature weighting to allow the model to reproduce distance-dependent appearance (e.g., varying level of detail, rig shadows) [S1].
4.  **Pruning:** Threshold density to determine if a voxel is occupied. For bounded scenes with solid surfaces, a higher threshold (e.g., $\alpha = 0.3$) and finer grid resolution (e.g., $1024^3$) can be applied to accelerate rendering [S1].
5.  **Free-Space Management:** To mitigate extrapolation failures caused by distance-aware features, prune as much free-space as possible. This ensures density does not suddenly appear when the user moves through previously unseen free-space regions [S1].

**Parameters:**
*   **Density Threshold ($\alpha$):** $0.3$ for aggressive pruning in bounded scenes [S1].
*   **Grid Resolution:** $1024^3$ for aggressive pruning [S1].
*   **Native VR Resolution:** Two $2064 \times 2096$ eyebuffers for Meta Quest Pro [S1].
*   **Target Frame Rate:** 72 FPS [S1].
*   **ASW Critical Threshold:** 36 FPS (half of native 72 FPS) [S1].

**Pitfalls and Edge Cases:**
*   **Aggressive Pruning Artifacts:** Using a high density threshold ($> 0.3$) and fine grid resolution ($1024^3$) causes box-like artifacts in VR for complex scenes (reflective/transparent surfaces, unbounded scenes). These artifacts are less visible in 2D renders [S1].
*   **Extrapolation Failure:** Distance-aware features reduce the model's ability to extrapolate to unseen viewpoints or distances because feature vectors with unseen weighting may be used, causing density to vary with distance [S1]. This is mitigated by extensive free-space pruning [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# VR-NeRF Rendering Configuration and Pruning Logic
# Note: Core optimization and capture steps are described in text due to lack of algorithmic pseudocode in sources.

def configure_vr_rendering(headset="Meta Quest Pro"):
    """
    Configure rendering parameters for VR-NeRF.
    """
    if headset == "Meta Quest Pro":
        eye_buffer_width = 2064
        eye_buffer_height = 2096
        target_fps = 72
        asw_threshold_fps = 36  # Half of native 72 FPS
        return {
            "resolution": (eye_buffer_width, eye_buffer_height),
            "stereo": True,
            "target_fps": target_fps,
            "asw_threshold": asw_threshold_fps
        }
    else:
        raise ValueError("Unsupported headset configuration")

def apply_pruning(voxel_grid, density_threshold=0.3, grid_res=1024**3):
    """
    Prune free-space voxels to prevent density extrapolation artifacts.
    
    Args:
        voxel_grid: The 3D feature/density grid.
        density_threshold: Alpha value for pruning. 0.3 is aggressive.
        grid_res: Total number of voxels (e.g., 1024^3).
        
    Returns:
        Pruned voxel grid.
    """
    # Prune voxels where density is below threshold
    # This prevents density from suddenly appearing in free-space
    # when moving through unseen viewpoints.
    occupied_mask = voxel_grid.density > density_threshold
    
    # Apply mask to grid
    pruned_grid = voxel_grid.copy()
    pruned_grid.density[~occupied_mask] = 0.0
    
    return pruned_grid

def evaluate_vr_quality(frame_times):
    """
    Evaluate VR experience quality using p99 FPS.
    
    Args:
        frame_times: List of frame times in milliseconds or FPS samples.
        
    Returns:
        p99_fps: The 99th percentile FPS.
    """
    # p99 FPS is preferred over mean FPS to ensure smoothness via ASW
    # If p99 > 36 FPS, ASW can deliver a smooth experience.
    # If p99 < 36 FPS, user may notice stuttering and motion sickness.
    p99_fps = percentile(frame_times, 99)
    return p99_fps

def check_pruning_artifacts(scene_type, threshold, resolution):
    """
    Warn about potential artifacts based on scene complexity.
    """
    if threshold > 0.3 and resolution == 1024**3:
        if scene_type in ["reflective", "transparent", "unbounded"]:
            return "Warning: Aggressive pruning may cause box-like artifacts in VR."
    return "Pruning configuration appears safe for scene type."
```

## Operating characteristics
*   **Time Complexity:** Not explicitly defined in terms of Big-O, but performance is measured in FPS. Halving the resolution (from full to half) only approximately doubles the FPS, despite rendering 1/4 the pixels. This indicates substantial per-frame overhead (e.g., kernel launches, VR compositor, OpenGL display pipeline) or insufficient parallelism at lower resolutions [S1].
*   **Memory Complexity:** Not explicitly defined, but grid resolution affects memory usage (e.g., $1024^3$ grid) [S1].
*   **Measured Performance:**
    *   **Hardware:** Tested on an off-the-shelf 3-GPU workstation and a custom 20-GPU rendering workstation [S1].
    *   **Half-Resolution (2064x2096):**
        *   3-GPU workstation: Achieved 36 FPS for half the scenes [S1].
        *   20-GPU workstation: All scenes but one achieved p99 of 72 FPS, providing a smooth experience even without ASW [S1].
    *   **Full-Resolution:**
        *   20-GPU workstation: 4 out of 6 scenes exceeded the critical ASW threshold of 36 FPS p99 [S1].
*   **Strengths:** Achieves higher resolution, frame rate, and visual fidelity than standard NeRF; suitable for comfortable VR viewing of neural radiance fields [S1].
*   **Limitations:** Aggressive pruning causes artifacts in complex scenes; distance-aware features hinder extrapolation; significant per-frame overhead limits scaling benefits from resolution reduction [S1].
*   **When to Use:** When high-fidelity, walkable space reconstruction is required for VR headsets like Meta Quest Pro, and sufficient GPU resources (e.g., 20-GPU workstation) are available [S1].
*   **When Not to Use:** For complex scenes with reflective/transparent surfaces or unbounded environments if aggressive pruning is necessary for performance, as this leads to visible artifacts [S1].

## Background
VR-NeRF was presented as the first holistic approach for capture, reconstruction, and rendering of high-fidelity walkable spaces in virtual reality [S1]. It integrates a custom multi-camera rig, a novel perceptual color space for HDR optimization, and efficient feature mip-mapping [S1]. The work addresses the specific challenges of VR rendering, such as the need for high frame rates to avoid motion sickness and the limitations of standard NeRF in this context [S1].

## Sources
[S1] VR-NeRF: High-Fidelity Virtualized Walkable Spaces, https://doi.org/10.1145/3610548.3618139, fetched 2026-10-28
