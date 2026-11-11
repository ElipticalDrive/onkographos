# iMAP

Also called: implicit mapping

## Overview
iMAP is the first NeRF-based dense online SLAM model that jointly optimizes camera pose and an implicit scene representation [S1]. It solves the problem of simultaneous localization and mapping by treating the scene as a continuous radiance field rather than a discrete mesh or point cloud [S1]. It sets itself apart from its siblings by being the foundational method that introduced the iterative two-step approach of tracking and mapping to NeRF-based SLAM, upon which NICE-SLAM and subsequent methods build [S1].

## How it works
iMAP employs an iterative two-step approach consisting of tracking and mapping [S1].

**Inputs and Outputs:**
*   **Inputs:** A sequence of RGB images from a moving camera.
*   **Outputs:** Camera poses for each frame and a continuous implicit scene representation (NeRF) [S1].

**Data Structures:**
*   **NeRF Model:** A multi-layer perceptron (MLP) that maps 3D coordinates and viewing directions to color and density values [S1].
*   **Keyframe Set:** A sparse, incrementally selected set of images used for optimization [S1].

**Algorithm Steps:**
1.  **Keyframe Selection:** Select a sparse subset of images from the input stream to serve as keyframes for optimization [S1].
2.  **Tracking (Pose Optimization):** Optimize the camera pose with respect to the current NeRF model [S1]. This step is computationally faster than mapping.
3.  **Mapping (Bundle Adjustment):** Perform joint optimization of the camera pose and the NeRF model parameters [S1]. This step is computationally slower than tracking.
4.  **Parallel Execution:** Run the tracking step in parallel with the mapping process to maintain real-time-like tracking speed [S1].

**Parameters and Edge Cases:**
*   The method relies on the NeRF architecture's ability to generalize from sparse views.
*   Performance is limited by the computational cost of the NeRF-based mapping step [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# iMAP Algorithm
# Inputs: image_sequence (list of RGB images), initial_pose_guess
# Outputs: poses (list of camera poses), nerf_model (trained NeRF)

def imap(image_sequence):
    # Initialize NeRF model with random weights
    nerf_model = initialize_nerf()
    
    # Initialize keyframe set with first few frames
    keyframes = select_initial_keyframes(image_sequence)
    
    # Initialize poses for keyframes
    poses = {kf_id: initial_pose_guess for kf_id in keyframes}
    
    for frame_id, image in enumerate(image_sequence):
        # Step 1: Tracking (Pose Optimization)
        # Optimize pose for the current frame relative to the current NeRF
        # This is done in parallel with mapping to maintain speed
        current_pose = optimize_pose(
            image=image, 
            nerf_model=nerf_model, 
            initial_pose=poses.get(frame_id, None)
        )
        poses[frame_id] = current_pose
        
        # Step 2: Mapping (Bundle Adjustment)
        # Jointly optimize poses and NeRF parameters using keyframes
        # This step is slower and runs concurrently with tracking
        nerf_model, poses = bundle_adjustment(
            keyframes=keyframes,
            poses=poses,
            images=[image_sequence[kf] for kf in keyframes],
            nerf_model=nerf_model
        )
        
        # Incrementally select new keyframes if necessary
        if should_add_keyframe(image, poses):
            keyframes.add(frame_id)
            
    return poses, nerf_model
```

## Subtopics

### NICE-SLAM
NICE-SLAM is an improvement on iMAP that uses a hierarchical grid-based representation to fill gaps in large-scale unobserved scene features [S1]. It achieves lower pose estimation errors and better scene reconstruction results than iMAP while using approximately one-quarter of the FLOPs, one-third of the tracking time, and half of the mapping time [S1]. It is still far from achieving real-time performance [S1].

### NICER-SLAM
NICER-SLAM is an end-to-end dense SLAM system that performs simultaneous tracking and mapping using only RGB inputs [S1]. It uses a hierarchical neural implicit representation based on signed distance functions (SDFs) [S1]. It leverages monocular geometric cues, optical flow, and a warping loss to guide optimization without depth supervision [S1]. It proposes a locally adaptive SDF-to-density transformation tailored for indoor scene dynamics [S1].

## Operating characteristics
*   **Tracking Speed:** Pose tracking speed is close to camera framerate [S1].
*   **Overall Performance:** Combined with the slower NeRF-based mapping step, iMAP is far from achieving real-time performance [S1].
*   **Efficiency:** NICE-SLAM (the direct successor) uses approximately one-quarter of the FLOPs of iMAP, one-third of the tracking time, and half of the mapping time [S1].
*   **Accuracy:** NICE-SLAM achieves lower pose estimation errors and better scene reconstruction results than iMAP [S1].
*   **Limitations:** The NeRF-based mapping step is computationally expensive, preventing real-time operation [S1].

## Background
iMAP was introduced by Sucar et al. in March 2021 [S1]. It is the first NeRF-based dense online SLAM model [S1]. It was followed by NICE-SLAM in December 2021 [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
