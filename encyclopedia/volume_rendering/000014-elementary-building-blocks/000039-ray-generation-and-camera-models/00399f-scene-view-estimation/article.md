# Scene View Estimation

Also called: Estimate Scene View, Make Render Input

## Overview
Scene View Estimation computes the camera position, orientation, and projection parameters required to render a 3D scene into a target viewport [S1, unofficial]. It solves the problem of automatically framing the scene content within the camera's view frustum by analyzing the spatial extent of the scene data [S1, unofficial]. It differs from sibling methods like [Arcball Camera](1639-arcball-camera.md) or [First-Person Camera Control](2721-first-person-camera-control.md) by being fully automatic and data-driven, rather than user-controlled or trajectory-based [S1, unofficial]. It builds on [Camera Matrix and Coordinate Transformation](6231-camera-matrix-and-coordinate-transformation.md) for the final matrix assembly [S1, unofficial].

## How it works
**Inputs:**
- `scene`: A scene object containing `sceneBounds` (global bounds) and `splatSets` (a list of splat sets, each with its own bounds) [S1, unofficial].
- `width`, `height`: Target viewport dimensions in pixels [S1, unofficial].
- `frameIndex`: Current frame identifier [S1, unofficial].

**Outputs:**
- `input`: A render input structure containing:
  - `view`: View matrix (look-at) [S1, unofficial].
  - `proj`: Projection matrix (perspective) [S1, unofficial].
  - `cameraPosition`: Eye position vector [S1, unofficial].
  - `viewportWidth`, `viewportHeight`: Viewport dimensions [S1, unofficial].
  - `nearPlane`, `farPlane`: Clipping plane distances [S1, unofficial].
  - `frameIndex`: Frame ID [S1, unofficial].
  - `settings`: Configuration flags (`antialiasing`, `fastCulling`) [S1, unofficial].

**Data Structures:**
- `bounds`: Axis-Aligned Bounding Box (AABB) defined by minimum and maximum 3D vectors [S1, unofficial].
- `view`: A struct containing `center` (3D vector) and `radius` (float) [S1, unofficial].

**Steps:**
1. Initialize `bounds` with `scene.sceneBounds` [S1, unofficial].
2. Iterate over each set in `scene.splatSets`, merging each set's bounds into `bounds` using `MergeBounds` [S1, unofficial].
3. If `bounds` is invalid, return a default view with `center = {0, 0, 2.5}` and `radius = 1.0` [S1, unofficial].
4. Otherwise, compute `center = ComputeAabbCenter(bounds)` and `radius = max(ComputeAabbRadius(bounds), 0.5)` [S1, unofficial].
5. Calculate `aspect = width / height` [S1, unofficial].
6. Calculate `distance = max(view.radius * 2.5, 1.5)` [S1, unofficial].
7. Set `eye = {view.center.x, view.center.y, view.center.z - distance}` [S1, unofficial].
8. Set `input.view = LookAt(eye, view.center, {0.0, 1.0, 0.0})` [S1, unofficial].
9. Set `input.proj = Perspective(1.0, aspect, 0.01, max(100.0, distance + view.radius * 4.0))` [S1, unofficial].
10. Set `input.cameraPosition = eye`, `input.viewportWidth = width`, `input.viewportHeight = height`, `input.nearPlane = 0.01`, `input.farPlane = max(100.0, distance + view.radius * 4.0)`, `input.frameIndex = frameIndex` [S1, unofficial].
11. Set `input.settings.antialiasing = True`, `input.settings.fastCulling = True` [S1, unofficial].

**Formulas:**
- $aspect = \frac{width}{height}$
- $distance = \max(view.radius \times 2.5, 1.5)$
- $eye = (view.center.x, view.center.y, view.center.z - distance)$
- $farPlane = \max(100.0, distance + view.radius \times 4.0)$

**Parameters and Typical Values:**
- `aspect`: Derived from viewport dimensions.
- `distance`: Scaled by `view.radius * 2.5`, minimum 1.5.
- `nearPlane`: Fixed at 0.01.
- `farPlane`: Scaled by `distance + view.radius * 4.0`, minimum 100.0.
- `FOV`: Fixed at 1.0 radian.

**Edge Cases and Pitfalls:**
- **Invalid Bounds:** If the scene has no splats or invalid bounds, a default view is returned to prevent crashes [S1, unofficial].
- **Small Scenes:** The `max(..., 0.5)` and `max(..., 1.5)` guards prevent the camera from getting too close to small or zero-sized scenes, which could cause numerical instability or clipping issues [S1, unofficial].
- **Large Scenes:** The `max(..., 100.0)` guard for the far plane ensures that very large scenes do not result in excessively large far planes that could reduce depth buffer precision [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def EstimateSceneView(scene, width, height, frameIndex):
    # Step 1: Compute Scene Bounds
    bounds = scene.sceneBounds
    for splat_set in scene.splatSets:
        bounds = MergeBounds(bounds, splat_set.bounds)

    # Step 2: Validate Bounds and Compute View Parameters
    if bounds is invalid:  # Check if bounds is invalid (e.g., empty or uninitialized)
        view_center = Vector3(0, 0, 2.5)
        view_radius = 1.0
    else:
        view_center = ComputeAabbCenter(bounds)
        view_radius = max(ComputeAabbRadius(bounds), 0.5)

    # Step 3: Calculate Camera Distance
    aspect = width / height
    distance = max(view_radius * 2.5, 1.5)

    # Step 4: Set Eye Position
    eye = Vector3(view_center.x, view_center.y, view_center.z - distance)

    # Step 5: Construct View Matrix
    up_vector = Vector3(0.0, 1.0, 0.0)
    view_matrix = LookAt(eye, view_center, up_vector)

    # Step 6: Construct Projection Matrix
    far_plane = max(100.0, distance + view_radius * 4.0)
    proj_matrix = Perspective(1.0, aspect, 0.01, far_plane)

    # Step 7: Populate Render Input
    input = RenderInput()
    input.view = view_matrix
    input.proj = proj_matrix
    input.cameraPosition = eye
    input.viewportWidth = width
    input.viewportHeight = height
    input.nearPlane = 0.01
    input.farPlane = far_plane
    input.frameIndex = frameIndex
    input.settings.antialiasing = True
    input.settings.fastCulling = True

    return input
```

## Subtopics

### [Depth Network](13169-depth-network.md)
Hybrid module for estimating and refining per-pixel depth maps from image inputs, resolving scaling/shifting ambiguities and mitigating MVS inaccuracies on sparse views. Combines initial MVS estimation with U-Net refinement using multi-view stereo cues and image features for consistent depth maps.

### [Viewpoint Estimation via Sphere Classification](15364-viewpoint-estimation-via-sphere-classification.md)
Estimates camera viewpoint (azimuth and elevation) by classifying rendered volume images into discrete regions on a viewing sphere using HEALPix division, evaluating loss via geodesic distance. Provides probabilities for each interval to capture viewpoint ambiguity, unlike regression methods.

### MG + (Geodesic Bin & Delta Mode)
Predicts viewpoint label via classification, estimates viewpoint residual via regression, and combines them for the final viewpoint [S2]. Uses a K-means dictionary size $K=24$ and geodesic distance importance $\alpha=10$ for best performance [S2]. Modified to be category-specific rather than category-independent [S2].

### AlexNet (SD+GS)
Viewpoint estimation method based on AlexNet that ignores the camera-lit angle and focuses only on azimuth and elevation [S2]. Achieves lower classification accuracy than VGGNet (UD+GS) across all tested categories and angle tolerances (2° to 15°) [S2].

### Batch Active View Selection
Selects a batch of $B$ views to capture new images, avoiding similar views via a greedy optimization algorithm that is $1/e$-approximate in Fisher Information [S3]. Initializes selected set $S_0 = \emptyset$ and Fisher Information matrix $H_0 = H[w^*|D_{train}]$ [S3]. Iteratively selects view $i$ maximizing $\text{tr}(H''[y_{i,acq}|x_{acq_i}, w^*] H_{b-1}^{-1})$, adds it to $S_b$, and updates $H_b = H_{b-1} + H''[y_{i,acq}|x_{acq_i}, w^*]$ [S3].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of splat sets, $O(N)$, where $N$ is the number of sets in `scene.splatSets` [S1, unofficial]. The bounding box merge and center/radius computations are constant time per set.
- **Memory Complexity:** Constant, $O(1)$, as it only stores temporary bounds and view parameters [S1, unofficial].
- **Strengths:** Simple, fast, and automatic. Ensures the entire scene is visible within the frustum [S1, unofficial].
- **Limitations:** Does not account for camera orientation preferences (always looks from -Z). May not frame aesthetically pleasing views for complex scenes with sparse distributions [S1, unofficial].
- **When to Use:** When an automatic, data-driven camera setup is required for rendering 3D Gaussian Splatting scenes [S1, unofficial].
- **When Not to Use:** When user-controlled camera movement or specific artistic framing is required [S1, unofficial].

## Background
This method is implemented in the DirectXSplat project, a DirectX 12 implementation of 3D Gaussian Splatting [S1, unofficial]. It provides a standardized way to initialize the camera for rendering splat sets [S1, unofficial].

## Sources
[S1] pbkx/DirectXSplat, https://github.com/pbkx/DirectXSplat, fetched 2026-10-27 (unofficial)
[S2] CNNs Based Viewpoint Estimation for Volume Visualization, https://doi.org/10.1145/3309993, fetched 2026-10-29
[S3] FisherRF: Active View Selection and Mapping with Radiance Fields Using Fisher Information, https://doi.org/10.1007/978-3-031-72624-8_24, fetched 2026-10-31
