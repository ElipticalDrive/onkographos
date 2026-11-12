# Scene View Estimation

Also called: Estimate Scene View, Make Render Input

## Overview
Scene View Estimation computes the camera position, orientation, and projection parameters required to render a 3D scene (specifically Gaussian Splatting scenes) into a target viewport. It solves the problem of automatically framing the scene content within the camera's view frustum by analyzing the spatial extent of the scene data. It differs from sibling methods like [Arcball Camera](1639-arcball-camera.md) or [First-Person Camera Control](2721-first-person-camera-control.md) by being fully automatic and data-driven, rather than user-controlled or trajectory-based. It builds on [Camera Matrix and Coordinate Transformation](6231-camera-matrix-and-coordinate-transformation.md) for the final matrix assembly.

## How it works
**Inputs:**
- `scene`: A scene object containing `sceneBounds` (global bounds) and `splatSets` (a list of splat sets, each with its own bounds).
- `width`, `height`: Target viewport dimensions in pixels.
- `frameIndex`: Current frame identifier.

**Outputs:**
- `input`: A render input structure containing:
  - `view`: View matrix (look-at).
  - `proj`: Projection matrix (perspective).
  - `cameraPosition`: Eye position vector.
  - `viewportWidth`, `viewportHeight`: Viewport dimensions.
  - `nearPlane`, `farPlane`: Clipping plane distances.
  - `frameIndex`: Frame ID.
  - `settings`: Configuration flags (`antialiasing`, `fastCulling`).

**Data Structures:**
- `bounds`: Axis-Aligned Bounding Box (AABB) defined by minimum and maximum 3D vectors.
- `view`: A struct containing `center` (3D vector) and `radius` (float).

**Steps:**
1. **Compute Scene Bounds:**
   - Initialize `bounds` with `scene.sceneBounds`.
   - Iterate through each set in `scene.splatSets`.
   - Merge each set's bounds into `bounds` using `MergeBounds`.
   - *Note:* `MergeBounds` updates the min/max corners to encompass both the existing `bounds` and the new set's bounds.

2. **Validate Bounds:**
   - Check if `bounds` is invalid (e.g., empty or uninitialized).
   - If invalid, return a default view:
     - `center = {0, 0, 2.5}`
     - `radius = 1.0`
   - If valid, compute the view parameters:
     - `center = ComputeAabbCenter(bounds)`
     - `radius = max(ComputeAabbRadius(bounds), 0.5f)`
     - *Note:* `ComputeAabbCenter` returns the midpoint of the AABB. `ComputeAabbRadius` returns the distance from the center to the farthest corner (half-diagonal length).

3. **Calculate Camera Distance:**
   - Calculate aspect ratio: `aspect = width / height`.
   - Calculate camera distance from center: `distance = max(view.radius * 2.5f, 1.5f)`.
   - *Note:* The distance scales with the scene size but has a minimum floor of 1.5 units.

4. **Set Eye Position:**
   - Set `eye = {view.center.x, view.center.y, view.center.z - distance}`.
   - *Note:* The camera is positioned directly behind the scene center along the negative Z-axis (assuming a right-handed coordinate system where the camera looks down -Z, or positive Z is up/back).

5. **Construct View Matrix:**
   - Set `input.view = LookAt(eye, view.center, {0.0f, 1.0f, 0.0f})`.
   - *Note:* `LookAt` creates a view matrix looking from `eye` at `view.center` with up vector `{0, 1, 0}`.

6. **Construct Projection Matrix:**
   - Calculate `farPlane = max(100.0f, distance + view.radius * 4.0f)`.
   - Set `input.proj = Perspective(1.0f, aspect, 0.01f, farPlane)`.
   - *Note:* `Perspective` creates a perspective projection matrix with field of view 1.0 radian, aspect ratio, near plane 0.01, and calculated far plane. The far plane ensures the entire scene (plus margin) is visible.

7. **Populate Render Input:**
   - Set `input.cameraPosition = eye`.
   - Set `input.viewportWidth = width`.
   - Set `input.viewportHeight = height`.
   - Set `input.nearPlane = 0.01f`.
   - Set `input.farPlane = farPlane`.
   - Set `input.frameIndex = frameIndex`.
   - Set `input.settings.antialiasing = true`.
   - Set `input.settings.fastCulling = true`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def EstimateSceneView(scene, width, height, frameIndex):
    # Step 1: Compute Scene Bounds
    bounds = scene.sceneBounds
    for splat_set in scene.splatSets:
        bounds = MergeBounds(bounds, splat_set.bounds)

    # Step 2: Validate Bounds and Compute View Parameters
    if bounds is invalid:
        view_center = Vector3(0, 0, 2.5)
        view_radius = 1.0
    else:
        view_center = ComputeAabbCenter(bounds)
        view_radius = max(ComputeAabbRadius(bounds), 0.5f)

    # Step 3: Calculate Camera Distance
    aspect = width / height
    distance = max(view_radius * 2.5f, 1.5f)

    # Step 4: Set Eye Position
    eye = Vector3(view_center.x, view_center.y, view_center.z - distance)

    # Step 5: Construct View Matrix
    up_vector = Vector3(0.0f, 1.0f, 0.0f)
    view_matrix = LookAt(eye, view_center, up_vector)

    # Step 6: Construct Projection Matrix
    far_plane = max(100.0f, distance + view_radius * 4.0f)
    proj_matrix = Perspective(1.0f, aspect, 0.01f, far_plane)

    # Step 7: Populate Render Input
    input = RenderInput()
    input.view = view_matrix
    input.proj = proj_matrix
    input.cameraPosition = eye
    input.viewportWidth = width
    input.viewportHeight = height
    input.nearPlane = 0.01f
    input.farPlane = far_plane
    input.frameIndex = frameIndex
    input.settings.antialiasing = true
    input.settings.fastCulling = true

    return input
```

**Parameters and Typical Values:**
- `aspect`: Derived from viewport.
- `distance`: Scaled by `view_radius * 2.5`, min 1.5.
- `nearPlane`: Fixed at 0.01.
- `farPlane`: Scaled by `distance + view_radius * 4.0`, min 100.0.
- `FOV`: Fixed at 1.0 radian.

**Edge Cases and Pitfalls:**
- **Invalid Bounds:** If the scene has no splats or invalid bounds, a default view is returned. This prevents crashes but may result in an empty or poorly framed render.
- **Small Scenes:** The `max(..., 0.5f)` and `max(..., 1.5f)` guards prevent the camera from getting too close to small or zero-sized scenes, which could cause numerical instability or clipping issues.
- **Large Scenes:** The `max(..., 100.0f)` guard for the far plane ensures that very large scenes do not result in excessively large far planes that could reduce depth buffer precision.

## Subtopics

### [Depth Network](13169-depth-network.md)
Estimates depth from a single image using a hybrid design to resolve scaling/shifting ambiguity and inconsistent estimations across views.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of splat sets, $O(N)$, where $N$ is the number of sets in `scene.splatSets`. The bounding box merge and center/radius computations are constant time per set.
- **Memory Complexity:** Constant, $O(1)$, as it only stores temporary bounds and view parameters.
- **Strengths:** Simple, fast, and automatic. Ensures the entire scene is visible within the frustum.
- **Limitations:** Does not account for camera orientation preferences (always looks from -Z). May not frame aesthetically pleasing views for complex scenes with sparse distributions.
- **When to Use:** When an automatic, data-driven camera setup is required for rendering 3D Gaussian Splatting scenes.
- **When Not to Use:** When user-controlled camera movement or specific artistic framing is required.

## Background
This method is implemented in the DirectXSplat project, a DirectX 12 implementation of 3D Gaussian Splatting. It provides a standardized way to initialize the camera for rendering splat sets.

## Sources
[S1] pbkx/DirectXSplat, https://github.com/pbkx/DirectXSplat, fetched 2026-10-27 (unofficial)
