# Object Space Ray Setup — Camera Space Transformation for Ray Origin, Local Space Ray Setup, Object Space Ray Casting

Also called: Ray Origin Calculation for Near Plane Quad, RayCasterShaderCodeTester::NearPlaneVertexShader

## Overview
This method calculates the ray origin in volume texture space for a given vertex on the near plane quad, used for testing ray casting shaders [S1, unofficial]. It transforms a vertex from view space to world space and then maps it into the local volume coordinate system. It differs from camera-space ray setup [4179-object-space-ray-setup-camera-space-transformation-for-ray-o.md] by operating directly on object-space vertices and using hardware interpolation of 3D UVs, rather than computing ray origins and directions in camera space.

## How it works
**Inputs:**
- `glVertex`: The vertex position in view space (passed from the vertex shader input).
- `InverseModelViewMatrix`: The inverse of the model-view matrix, transforming view space to world space.
- `VolTranslation`: The translation vector of the volume in world space.
- `VolScale`: The scale factor of the volume in world space.
- `ProjectionMatrix`: The projection matrix for clipping (used for `gl_Position`).

**Outputs:**
- `gl_Position`: The clip-space position for rasterization.
- `RayPosition`: The ray origin in volume texture space (normalized 0-1 range).

**Data Structures:**
- `vec4`: 4-component floating-point vector.
- `vec3`: 3-component floating-point vector.
- `mat4`: 4x4 floating-point matrix.

**Steps:**
1. Compute the clip-space position: `gl_Position = ProjectionMatrix * glVertex`.
2. Transform the vertex from view space to world space: `worldPos = InverseModelViewMatrix * glVertex`.
3. Extract the 3D position: `rayPosition = worldPos.xyz`.
4. Translate the position to volume-local space: `rayPosition = rayPosition - VolTranslation`.
5. Scale the position to volume-local space: `rayPosition = rayPosition / VolScale`.
6. Offset the position to center it in texture coordinates: `rayPosition = rayPosition + 0.5f`.
7. Pack the result into a 4D vector: `RayPosition = vec4(rayPosition, 1.0f)`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def NearPlaneVertexShader(glVertex: vec4, InverseModelViewMatrix: mat4, VolTranslation: vec3, VolScale: float, ProjectionMatrix: mat4) -> (vec4, vec4):
    # Step 1: Compute clip-space position
    gl_Position = ProjectionMatrix * glVertex
    
    # Step 2: Transform vertex from view space to world space
    worldPos = InverseModelViewMatrix * glVertex
    
    # Step 3: Extract 3D position
    rayPosition = worldPos.xyz
    
    # Step 4: Translate to volume-local space
    rayPosition = rayPosition - VolTranslation
    
    # Step 5: Scale to volume-local space
    rayPosition = rayPosition / VolScale
    
    # Step 6: Offset to center in texture coordinates (0.5 to 1.5 range, assuming unit cube centered at 0)
    rayPosition = rayPosition + 0.5f
    
    # Step 7: Pack result
    RayPosition = vec4(rayPosition, 1.0f)
    
    return gl_Position, RayPosition
```

**Parameters:**
- `VolTranslation`: Not given in the sources.
- `VolScale`: Not given in the sources.

**Edge Cases and Pitfalls:**
- Precision: Floating-point precision issues may arise if the volume is very large or very small.
- Boundaries: The method assumes the volume is a unit cube centered at the origin in texture space. If the volume is not a unit cube, the offset and scale may need adjustment.
- Degenerate Input: If `VolScale` is zero, division by zero will occur.

## Subtopics

### [Object Space Ray Setup via Vertex Shader UV Computation](13156-object-space-ray-setup-via-vertex-shader-uv-computation.md)
This method establishes the geometric foundation for GPU-based volume raycasting by mapping a 3D bounding box (typically a unit cube) to screen space while simultaneously computing normalized 3D texture coordinates for each fragment. It solves the problem of translating object-space vertex positions into both clip-space positions for rasterization and valid texture lookup coordinates for the fragment shader. It differs from camera-space ray setup [4179-object-space-ray-setup-camera-space-transformation-for-ray-o.md] by operating directly on object-space vertices and using hardware interpolation of 3D UVs, rather than computing ray origins and directions in camera space.

## Operating characteristics
- Time Complexity: Not given in the sources.
- Memory Complexity: Not given in the sources.
- Measured Performance: Not given in the sources.
- Scaling: Not given in the sources.
- Strengths: Simple to implement, works well for testing ray casting shaders.
- Limitations: Limited to near plane quad, may not be suitable for complex volume rendering.
- When to use: When testing ray casting shaders or for simple volume rendering.
- When not to use: For complex volume rendering or when performance is critical.

## Background
This method is described in the thesis "VoxVizThesis" by apecoraro [S1, unofficial]. It is used in the context of volume rendering, specifically for testing ray casting shaders.

## Sources
[S1] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
