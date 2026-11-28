# Neural Assets Capture and Rendering Pipeline

Also called: Neural Assets, Neural Volumetric Object Capture

## Overview
A method for capturing real-world objects using consumer hardware (mobile phone) and rendering them as volumetric assets in interactive environments, separating foreground object from background using two radiance field models [S1, unofficial]. It sets itself apart by transpiling the neural model into GLSL/HLSL shader code for real-time volumetric raycasting and fusing the asset with a marching-cubes mesh for physics integration, enabling seamless use in game engines [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ~90-second video of the object recorded with fixed exposure, white balance, and sensitivity [S1, unofficial].
*   **Output:** A transpiled shader (GLSL/HLSL) for real-time rendering and a low-fidelity collision mesh [S1, unofficial].

**Data Structures:**
*   **Radiance Field Models:** Two separate models (foreground and background) using a grid of features and a small MLP decoder [S1, unofficial].
*   **Camera Parameters:** Intrinsics $K$ and poses $R_k, t_k$ for each frame $k$ [S1, unofficial].
*   **Bounding Box:** A 3D bounding box defining the foreground object region [S1, unofficial].

**Steps:**
1.  **Capture:** Record video with fixed camera settings [S1, unofficial].
2.  **Filtering:** Apply a gradient-based motion blur detector to filter blurry frames [S1, unofficial].
3.  **Sampling:** Select a subset of frames sampled at an interval of 6 FPS [S1, unofficial].
4.  **Pose Estimation:** Process selected frames with off-the-shelf photogrammetry software to estimate approximate camera intrinsic parameters $K$ and poses $R_k, t_k$ for each frame $k$, resulting in a point cloud [S1, unofficial].
5.  **Segmentation:** Ask the user to provide a rough 3D bounding box around the foreground object using the initial point cloud [S1, unofficial].
6.  **Reconstruction:** Reconstruct the scene using two separate radiance field models: one for the background and one for the foreground, focusing model capacity on the object of interest [S1, unofficial].
7.  **Optimization:** Optimize the model using a grid of features and a small MLP decoder, refining camera parameters via backpropagated gradients from the 3D representation [S1, unofficial].
8.  **Transpilation:** Transpile the model architecture into GLSL or HLSL shader code for a pixel shader [S1, unofficial].
9.  **Rendering:** Render the asset using volumetric raycasting in the shader, aggregating sample points to produce a result that blends correctly with the background [S1, unofficial].
10. **Integration:** Fuse the neural asset with a marching-cubes generated low-fidelity mesh for collision detection and shadowing integration [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def capture_neural_asset(video_path, output_shader_path, output_mesh_path):
    # Step 1: Capture is assumed done by user with fixed settings
    video = load_video(video_path)
    
    # Step 2: Filter blurry frames
    frames = video.get_frames()
    good_frames = []
    for frame in frames:
        if not is_blurry_gradient_based(frame):
            good_frames.append(frame)
            
    # Step 3: Sample at 6 FPS
    sampled_frames = sample_interval(good_frames, fps=6)
    
    # Step 4: Photogrammetry for poses
    # Note: Uses off-the-shelf software, not implemented here
    point_cloud, K, poses = run_photogrammetry(sampled_frames)
    
    # Step 5: User provides bounding box
    # Note: Interactive step, assumed provided
    bbox = get_user_bounding_box(point_cloud)
    
    # Step 6 & 7: Reconstruct and Optimize two radiance fields
    # Foreground model focuses on object inside bbox
    # Background model handles outside
    fg_model = build_radiance_field(point_cloud, K, poses, region=bbox)
    bg_model = build_radiance_field(point_cloud, K, poses, region=outside(bbox))
    
    fg_model.optimize(grid_features=True, mlp_decoder=True, refine_cameras=True)
    bg_model.optimize(grid_features=True, mlp_decoder=True, refine_cameras=True)
    
    # Step 8: Transpile to shader
    shader_code = transpile_to_glsl(fg_model)
    save_shader(shader_code, output_shader_path)
    
    # Step 10: Generate collision mesh
    collision_mesh = marching_cubes(fg_model, resolution=low)
    save_mesh(collision_mesh, output_mesh_path)
    
    return fg_model, bg_model

def is_blurry_gradient_based(frame):
    # Simple gradient-based motion blur detector
    # Returns True if blurry
    gradients = compute_gradients(frame)
    blur_score = sum(gradients)
    return blur_score > threshold # [G] Threshold not specified in sources
```

**Parameters:**
*   **Capture Duration:** ~90 seconds [S1, unofficial].
*   **Sampling Rate:** 6 FPS [S1, unofficial].
*   **Camera Settings:** Fixed exposure, white balance, and sensitivity [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Blurry Frames:** Must be filtered to hinder detailed reconstruction [S1, unofficial].
*   **Camera Tracking:** Does not use on-device tracking or SLAM; relies on fixed settings and photogrammetry [S1, unofficial].
*   **Integration:** Standard shader code ensures portability across hardware and software systems [S1, unofficial].

## Subtopics

### Inverted Sphere Parameterization
Uses an inverted sphere for background modeling. Choose when background is complex and distant. [Inverted Sphere Parameterization](5573-inverted-sphere-parameterization.md)

### DoubleNeRF
Uses two NeRFs for foreground and background. Choose when explicit separation is needed without shader transpilation. [DoubleNeRF](7391-doublenerf.md)

### Physically Based Inverse Rendering with Split-Sum Approximation
Focuses on physically based material capture. Choose when material properties are critical. [Physically Based Inverse Rendering with Split-Sum Approximation](11361-physically-based-inverse-rendering-with-split-sum-approximat.md)

### Background Inversion for Unit Sphere
Optimizes background on a unit sphere. Choose when background is simple and spherical. [Background Inversion for Unit Sphere](11463-background-inversion-for-unit-sphere.md)

### SNeRF
Uses spherical harmonics for view-dependent effects. Choose when view-dependent appearance is key. [SNeRF](14885-snerf.md)

### NDJIR
Neural Differentiable Joint Inverse Rendering. Choose when joint optimization of geometry and lighting is needed. [NDJIR](15962-ndjir.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Renders at interactive framerates, including stereoscopic settings for VR [S1, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Real-time rendering via shader transpilation; integration with game engine subsystems (color blending, shadowing, LoD, physics); portable across hardware/software; captures translucent effects [S1, unofficial].
*   **Limitations:** Requires user-provided bounding box; relies on off-the-shelf photogrammetry for poses; fixed camera settings required during capture [S1, unofficial].
*   **When to Use:** When real-time integration of captured objects into interactive environments (games, VR) is required.
*   **When Not to Use:** When high-fidelity offline rendering is sufficient and real-time constraints are not present.

## Background
Originated by the authors of "Neural Assets: Volumetric Object Capture and Rendering for Interactive Environments" [S1, unofficial]. It builds on recent advances in neural volumetric rendering based on seminal works on neural volumes [15] and radiance fields (NeRFs) [20] [S1, unofficial]. It addresses the problem of integrating neural volumetric models with existing interactive graphics systems, which typically use different rendering primitives and lack support for game engine subsystems [S1, unofficial].

## Sources
[S1] Neural Assets: Volumetric Object Capture and Rendering for Interactive Environments, https://core.ac.uk/download/552466290.pdf, fetched 2026-10-30 (unofficial)
