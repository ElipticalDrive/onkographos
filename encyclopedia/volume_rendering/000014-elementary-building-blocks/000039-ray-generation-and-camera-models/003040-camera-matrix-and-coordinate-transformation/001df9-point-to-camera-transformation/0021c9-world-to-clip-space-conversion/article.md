# World to Clip Space Conversion

Also called: WorldToClipSpace

## Overview
Converts a 3D world position into 2D screen-space pixel coordinates, enabling texture sampling and pixel location determination in camera space [S1, unofficial]. It differs from [3D Point Projection to 2D](3148-3d-point-projection-to-2d.md) by explicitly constructing the view basis from camera orientation vectors and applying a manual perspective division based on field-of-view and aspect ratio, rather than relying on a pre-computed projection matrix [S1, unofficial].

## How it works
**Inputs:**
- `worldPosition`: 3D position in world coordinates ($vec3<f32>$).
- `camPos`: Camera position in world coordinates ($vec3<f32>$).
- `camForward`: Camera forward vector in world coordinates ($vec3<f32>$).
- `cameraData.fov`: Field of view in radians ($f32$).
- `cameraData.aspectRatio`: Aspect ratio (width/height) ($f32$).
- `sceneData.screenWidth`: Screen width in pixels ($u32$).
- `sceneData.screenHeight`: Screen height in pixels ($u32$).

**Outputs:**
- `pixelPos`: 2D pixel coordinates ($vec2<f32>$).

**Data Structures:**
- `cameraData`: Contains `fov` and `aspectRatio`.
- `sceneData`: Contains `screenWidth` and `screenHeight`.

**Steps:**
1.  **Construct View Basis:** Calculate the right and up vectors to form an orthonormal basis.
    -   $camRight = \text{normalize}(\text{cross}(camForward, (0.0, 1.0, 0.0)))$
    -   $camUp = \text{normalize}(\text{cross}(camRight, camForward))$
    -   *Note:* This assumes the camera's "up" is generally aligned with the world Y-axis. If `camForward` is parallel to $(0,1,0)$, this cross product becomes degenerate [G].
2.  **Transform to View Space:** Compute the vector from the camera to the world position and project it onto the basis vectors.
    -   $camToWorldPos = worldPosition - camPos$
    -   $viewX = \text{dot}(camRight, camToWorldPos)$
    -   $viewY = \text{dot}(camUp, camToWorldPos)$
    -   $viewZ = \text{dot}(camForward, camToWorldPos)$
3.  **Perspective Projection:** Apply perspective division using the tangent of half the FOV.
    -   $scale = \tan(\frac{fov}{2.0})$
    -   $clipX = \frac{viewX}{viewZ \cdot scale \cdot aspectRatio}$
    -   $clipY = \frac{viewY}{viewZ \cdot scale}$
4.  **Normalize to [0, 1]:** Offset and scale the clip coordinates to the range $[0, 1]$.
    -   $clipX = clipX \cdot (1.0 - \frac{1.0}{screenWidth})$
    -   $clipY = clipY \cdot (1.0 - \frac{1.0}{screenHeight})$
    -   $clipX = \frac{clipX}{2.0} + 0.5$
    -   $clipY = \frac{clipY}{2.0} + 0.5$
5.  **Map to Pixel Coordinates:** Scale to the actual screen resolution.
    -   $pixelX = clipX \cdot (screenWidth - 1)$
    -   $pixelY = clipY \cdot (screenHeight - 1)$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def world_to_clip_space(worldPosition: vec3, camPos: vec3, camForward: vec3, 
                        fov: f32, aspectRatio: f32, screenWidth: u32, screenHeight: u32) -> vec2:
    # Step 1: Construct View Basis
    # [G] Guard against degenerate forward vector parallel to world up (0,1,0)
    worldUp = vec3(0.0, 1.0, 0.0)
    camRight = normalize(cross(camForward, worldUp))
    camUp = normalize(cross(camRight, camForward))
    
    # Step 2: Transform to View Space
    camToWorldPos = worldPosition - camPos
    viewX = dot(camRight, camToWorldPos)
    viewY = dot(camUp, camToWorldPos)
    viewZ = dot(camForward, camToWorldPos)
    
    # Step 3: Perspective Projection
    # [G] Handle viewZ <= 0 (behind camera) to avoid division by zero or negative depth artifacts
    if viewZ <= 0.0:
        return vec2(-1.0, -1.0)
        
    scale = tan(fov / 2.0)
    clipX = viewX / (viewZ * scale * aspectRatio)
    clipY = viewY / (viewZ * scale)
    
    # Step 4: Normalize to [0, 1]
    # The source applies a slight offset based on screen dimensions before normalizing
    clipX = clipX * (1.0 - (1.0 / f32(screenWidth)))
    clipY = clipY * (1.0 - (1.0 / f32(screenHeight)))
    
    clipX = clipX / 2.0 + 0.5
    clipY = clipY / 2.0 + 0.5
    
    # Step 5: Map to Pixel Coordinates
    pixelX = clipX * f32(screenWidth - 1)
    pixelY = clipY * f32(screenHeight - 1)
    
    return vec2(pixelX, pixelY)
```

## Subtopics

### Screen-Space Voxel Center Conversion
Converts a voxel grid coordinate ($ivec3$) directly to screen-space fragment coordinates using the inverse projection-view matrix [S2, unofficial]. Unlike the standard method which builds a view basis, this variant assumes a standard OpenGL/Direct3D pipeline matrix stack [S2, unofficial].
- **Definition:** Transforms voxel center coordinates to fragment coordinates [S2, unofficial].
- **Steps:**
    1.  Construct homogeneous position: $pos = vec4(vc.x, vc.y, vc.z, 1)$ [S2, unofficial].
    2.  Transform to clip space: $transformed\_vc = \text{inverse}(invVP) \cdot pos$ [S2, unofficial].
    3.  Convert to NDC: $NDC\_vc = (\frac{transformed\_vc}{transformed\_vc.w}).xyz$ [S2, unofficial].
    4.  Convert to screen space: $screenspace\_vc = vec3(NDC\_vc.x \cdot 2 - 1, NDC\_vc.y \cdot 2 - 1, NDC\_vc.z \cdot 2 - 1)$ [S2, unofficial].
    5.  Convert to fragment coordinates: $fragCoord\_vc = vec3(screenspace\_vc.x \cdot windowWidth, screenspace\_vc.y \cdot windowHeight, screenspace\_vc.z)$ [S2, unofficial].
- **Parameters:** `windowWidth`, `windowHeight` (viewport dimensions in pixels); `invVP` (inverse projection-view matrix) [S2, unofficial].
- **Code:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
vec4 pos = vec4(vc.x, vc.y, vc.z, 1);
vec4 transformed_vc = inverse(invVP) * pos;
vec3 NDC_vc = (transformed_vc/transformed_vc.w).xyz;
vec3 screenspace_vc = vec3(NDC_vc.x*2 - 1, NDC_vc.y*2 -1, NDC_vc.z*2 - 1);
vec3 fragCoord_vc = vec3(screenspace_vc.x * windowWidth, screenspace_vc.y * windowHeight, screenspace_vc.z);
```

### Position Vertex Shader
A vertex shader implementation that transforms input positions to clip space using a combined World-View-Projection matrix [S3, unofficial].
- **Steps:**
    1.  Transform input position: $output.Position = \text{mul}(float4(input.Position, 1.0) \cdot ScaleFactor, WorldViewProj)$ [S3, unofficial].
    2.  Pass through texture coordinates: $output.texC = input.texC.xyz$ [S3, unofficial].
    3.  Pass through clip space position: $output.pos = output.Position$ [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per point. Involves a few vector operations, a tangent, and divisions.
- **Memory Complexity:** $O(1)$.
- **Strengths:** Explicit control over the projection geometry; useful for custom camera models or when the projection matrix is not available or needs to be constructed dynamically from orientation vectors [S1, unofficial].
- **Limitations:** Requires manual construction of the view basis, which can be numerically unstable if the forward vector is parallel to the world up vector [G]. The specific normalization step (offsetting by $1 - 1/N$) is non-standard and may cause off-by-one errors if the screen dimensions change or if the method is compared against standard OpenGL/Direct3D clip space definitions [S1, unofficial].
- **When to use:** When generating rays from pixel centers or sampling textures based on world positions in a custom rendering loop where standard pipeline matrices are not used [S1, unofficial].
- **When not to use:** When standard graphics pipeline APIs are available, as they handle clip space conversion more robustly and consistently [S2, S3, unofficial].

## Background
This method is derived from WebGPU voxel raymarching implementations, specifically used for converting world positions to screen coordinates for texture sampling and anti-smearing effects [S1, unofficial]. It represents a manual implementation of the view-projection pipeline, often found in compute shaders or custom rendering passes where fine-grained control over coordinate transformation is required [S1, unofficial].

## Sources
[S1] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
[S2] nilspin/VoxelHashing_demo, https://github.com/nilspin/VoxelHashing_demo, fetched 2026-10-28 (unofficial)
[S3] 3D Texture Only interpolates on Y-Axis, https://gamedev.stackexchange.com/questions/166205, fetched 2026-10-29 (unofficial)
