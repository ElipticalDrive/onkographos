# 3DGStream

Also called: Neural Transformation Cache for 3DGS

## Overview
3DGStream is a method for online training of 3D Gaussian Splatting (3DGS) in dynamic scene reconstruction [S1][S2, unofficial]. It models the transformation between frames as a neural transformation cache and adaptively adds 3D Gaussians to handle emerging objects [S1][S2, unofficial]. It is distinguished from static 3DGS by its ability to incrementally update the scene representation as new video frames arrive, rather than requiring offline batch optimization of a fixed set of Gaussians.

## How it works
**Inputs and Outputs:**
*   **Input:** A sequence of images (video frames) with known camera poses, arriving online.
*   **Output:** A dynamic 3DGS representation consisting of a set of 3D Gaussians whose attributes (position, rotation, scale, opacity, color) vary with time, managed via a neural transformation cache.

**Data Structures:**
*   **3D Gaussians:** Standard 3DGS attributes: mean position $\mathbf{\mu}$, covariance (defined by rotation $\mathbf{q}$ and scale $\mathbf{s}$), opacity $\alpha$, and spherical harmonic coefficients for color $\mathbf{c}$ [S1][S2, unofficial].
*   **Neural Transformation Cache:** A neural network that predicts the transformation (displacement and/or rotation) required to map a Gaussian's canonical state to its state at a specific time $t$ [S1][S2, unofficial]. The exact architecture and input/output dimensions of this cache are not given in the sources.

**Steps:**
1.  **Initialization:** The scene is initialized with a point cloud. For scenes with large static parts, the point cloud is randomly initialized into dynamic and static subsets, which are optimized separately to decouple dynamic and static components [S1][S2, unofficial].
2.  **Neural Transformation Cache Modeling:** The transformation between frames is modeled using a neural transformation cache [S1][S2, unofficial]. This cache likely takes the canonical Gaussian attributes and time $t$ as input to predict the dynamic attributes [G].
3.  **Online Training:** The 3DGS is trained online as new frames arrive [S1][S2, unofficial]. The rendering process uses the transformed Gaussians for the current frame's time $t$ [G].
4.  **Adaptive Gaussian Addition:** New 3D Gaussians are adaptively added to the scene to handle emerging objects that were not present in previous frames [S1][S2, unofficial]. This likely involves splitting or cloning existing Gaussians based on rendering error or depth uncertainty, similar to standard 3DGS densification, but applied dynamically over time [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for 3DGStream online training loop
# Note: Specific implementation details of the Neural Transformation Cache
# and adaptive addition logic are not provided in the sources [S1][S2, unofficial].

def initialize_scene(images):
    # Randomly initialize point cloud into dynamic and static subsets
    # if large static parts are detected [S1][S2, unofficial].
    gaussians = random_initialize_point_cloud(images[0])
    cache = NeuralTransformationCache() # Architecture not specified [S1][S2, unofficial]
    return gaussians, cache

def online_training_step(gaussians, cache, new_image, camera_pose, time_t):
    # 1. Transform Gaussians using the neural transformation cache [S1][S2, unofficial]
    # The cache predicts transformation based on time t and canonical attributes
    transformed_gaussians = cache.transform(gaussians, time_t)
    
    # 2. Render the image using transformed Gaussians
    # Standard 3DGS rasterization is used with the time-specific attributes
    rendered_image = render_3dgs(transformed_gaussians, camera_pose)
    
    # 3. Compute loss (e.g., L1 or LPIPS) between rendered and ground truth
    loss = compute_loss(rendered_image, new_image)
    
    # 4. Backpropagate to update Gaussians and Cache
    # Gradients flow through the cache transformation to update canonical attributes
    update_parameters(gaussians, cache, loss)
    
    # 5. Adaptively add new Gaussians for emerging objects [S1][S2, unofficial]
    # This step likely involves densification based on error or depth
    gaussians = adaptive_add_gaussians(gaussians, rendered_image, new_image)
    
    return gaussians, cache

# Main loop
gaussians, cache = initialize_scene(video_frames)
for t, (image, pose) in enumerate(video_frames):
    gaussians, cache = online_training_step(gaussians, cache, image, pose, t)
```

**Parameters:**
*   Learning rate for Gaussians and Cache: Not given in the sources.
*   Time frequency $\omega$ for Fourier-based temporal modeling (if used): Not given in the sources. 3DGStream uses a neural cache, not explicitly Fourier series, though the parent family mentions Fourier as a common form [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Emerging Objects:** If new objects appear rapidly, the adaptive addition mechanism must be robust to avoid artifacts or slow convergence [S1][S2, unofficial].
*   **Static/Dynamic Decoupling:** Incorrect initialization of static vs. dynamic subsets may lead to poor reconstruction of static parts [S1][S2, unofficial].
*   **Precision:** Standard floating-point precision issues in 3DGS rasterization and neural network training apply [G].

## Subtopics

### Neural Transformation Cache
A method for modeling temporal transformations using a neural network. It sets itself apart from polynomial or Fourier-based parameterization by learning the transformation directly from data. Choose it when the motion is complex and not easily modeled by simple functions. See [Deformation Field Modeling](13852-deformation-field-modeling.md).

### Adaptive Gaussian Addition
A technique for dynamically adding Gaussians during training. It sets itself apart from fixed-set methods by allowing the scene complexity to grow. Choose it for scenes with occlusions or emerging objects. See [Dynamic and Temporal Modeling](9265-dynamic-and-temporal-modeling.md).

## Operating characteristics
*   **Time Complexity:** Online training implies per-frame processing time. The complexity depends on the number of Gaussians and the cost of the neural transformation cache. Not given in the sources.
*   **Memory Complexity:** Depends on the number of Gaussians and the size of the neural transformation cache. Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enables online reconstruction of dynamic scenes; handles emerging objects; decouples static and dynamic parts.
*   **Limitations:** Requires careful initialization; performance depends on the neural cache's capacity and training stability.
*   **When to use:** When reconstructing dynamic scenes from video in an online or incremental manner.
*   **When not to use:** For static scenes or when offline batch optimization is feasible and preferred for higher quality.

## Background
3DGStream was introduced as a method for online training of 3DGS in dynamic scene reconstruction [S1][S2, unofficial]. It builds on the principles of 3D Gaussian Splatting and extends them to handle temporal dynamics through a neural transformation cache and adaptive Gaussian addition [S1][S2, unofficial]. It is part of the broader family of dynamic 3DGS methods, which also includes approaches like 4D Gaussian Splatting and GaussianFlow [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
