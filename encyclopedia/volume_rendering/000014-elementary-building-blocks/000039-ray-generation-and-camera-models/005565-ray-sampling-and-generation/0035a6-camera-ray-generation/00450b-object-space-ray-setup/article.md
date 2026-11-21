# Object Space Ray Setup

Also called: Camera Space Transformation for Ray Origin, Local Space Ray Setup, Object Space Ray Casting, Ray Origin Calculation for Near Plane Quad, Ray Casting Vertex Shader, RayCasterShaderCodeTester::NearPlaneVertexShader

## Overview
Object Space Ray Setup constructs ray origins and directions within the local coordinate system of a mesh, ensuring that the raytraced volume moves, scales, and rotates rigidly with the object's transform [S2, unofficial]. Unlike world-space ray generation, which treats the volume as a static window into the world, this method embeds the object's transformation directly into the ray parameters, allowing the raytraced geometry to behave like a physical part of the mesh [S2, unofficial].

## How it works
The method computes the ray origin and direction in the vertex shader and passes them to the fragment shader for normalization and use in ray marching [S2, unofficial].

**Inputs:**
- `v.vertex`: The vertex position in local/object space (from `appdata`) [S2, unofficial].
- `_WorldSpaceCameraPos` or `u_camera`: The camera position in world space [S2, unofficial].
- `unity_WorldToObject` or `inverse(modelMatrix)`: The matrix transforming world coordinates to object space [S2, unofficial].

**Outputs:**
- `rayOrigin`: The starting point of the ray in object space [S2, unofficial].
- `rayDirection`: The direction of the ray in object space [S2, unofficial].

**Data Structures:**
The vertex shader output struct (`v2f`) contains:
- `position` (`SV_POSITION`): Clip-space position for rasterization [S2, unofficial].
- `localPosition` (`TEXCOORD0`, `float4`): The vertex position in local space, used as the ray origin [S2, unofficial].
- `viewDirection` (`TEXCOORD1`, `float4`): The unnormalized view vector in local space [S2, unofficial].

**Steps:**
1. **Transform Camera to Object Space:** Multiply the world-space camera position by the world-to-object matrix. The w-component of the camera position vector must be `1.0` to include translation [S2, unofficial].
   $$ \mathbf{c}_{obj} = \mathbf{M}_{world \to obj} \times \begin{bmatrix} \mathbf{c}_{world} \\ 1.0 \end{bmatrix} $$
   Where $\mathbf{c}_{obj}$ is the camera position in object space and $\mathbf{M}_{world \to obj}$ is the inverse model matrix [S1, unofficial][S2, unofficial].

2. **Compute Ray Origin:** The ray origin is the vertex's local position [S2, unofficial].
   $$ \mathbf{o} = \mathbf{v}_{local} $$
   Where $\mathbf{v}_{local}$ is `v.vertex` [S2, unofficial].

3. **Compute Unnormalized View Direction:** Subtract the object-space camera position from the local vertex position [S2, unofficial].
   $$ \mathbf{d}_{raw} = \mathbf{v}_{local} - \mathbf{c}_{obj} $$
   Where $\mathbf{d}_{raw}$ is the raw view direction vector [S2, unofficial].

4. **Pass to Fragment Shader:** Interpolate `localPosition` and `viewDirection` across the primitive [G].

5. **Normalize Direction:** In the fragment shader, normalize the view direction. Normalization must occur here, not in the vertex shader, because linear interpolation of vectors distorts their length, particularly on low-poly meshes or at close camera distances [S2, unofficial].
   $$ \mathbf{d} = \frac{\mathbf{d}_{raw}}{\|\mathbf{d}_{raw}\|} $$

**Pitfalls:**
- **W-Component Error:** If the camera position is transformed with a w-component of `0`, the translation component of the inverse matrix is ignored, resulting in incorrect ray origins that only account for rotation and scale [S2, unofficial].
- **Interpolation Distortion:** Normalizing the view direction in the vertex shader leads to incorrect angles in the fragment shader due to perspective-correct interpolation artifacts [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def vertex_setup(vertex_local, camera_world, world_to_object_matrix):
    # 1. Compute clip-space position for rasterization (standard rasterization step)
    clip_position = compute_clip_space(vertex_local) # [G] Standard rasterization step

    # 2. Store local position as ray origin
    local_position = vertex_local

    # 3. Transform camera world position to object space
    # w-component must be 1.0 to include translation [S2, unofficial]
    camera_obj = world_to_object_matrix.transform(camera_world, w=1.0)

    # 4. Compute unnormalized view direction in local space
    view_direction = vertex_local - camera_obj

    return {
        'clip_position': clip_position,
        'local_position': local_position,
        'view_direction': view_direction
    }

def fragment_setup(vertex_output):
    # 5. Normalize direction in fragment shader to avoid interpolation distortion [S2, unofficial]
    raw_dir = vertex_output['view_direction']
    length = math.sqrt(sum(c * c for c in raw_dir))
    dir = [c / length for c in raw_dir]

    # 6. Use local position as ray origin
    ray_origin = vertex_output['local_position']

    # Ray marching logic would follow here, using ray_origin and dir
    # [G] Implementation of ray marching is outside the scope of this setup method.
    return ray_origin, dir
```

## Subtopics

### [Object Space Ray Setup — Camera Space Transformation for Ray Origin, Local Space Ray Setup, Object Space Ray Casting](4179-object-space-ray-setup-camera-space-transformation-for-ray-o.md)
This variant explicitly details the construction of the ray origin in volume texture space by transforming a vertex from the near plane quad through inverse view and model transformations, followed by scaling and translation to align with the volume grid. It differs from camera-space ray setup by operating directly on object-space vertices and using hardware interpolation of 3D UVs, rather than computing ray origins and directions in camera space.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Ensures the raytraced volume is rigidly attached to the mesh, simplifying animation and transformation logic [S2, unofficial]. Intuitive for single-object rendering [S2, unofficial].
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
The method originates from basic sphere tracing and ray marching tutorials, where object-space rendering is preferred for its simplicity and intuitive handling of object transformations [S2, unofficial].

## Sources
[S1] Show HN: Volume rendering 3D data in Three.js and GLSL, https://github.com/SuboptimalEng/volume-rendering, fetched 2026-10-26 (unofficial)
[S2] Spheretracing Basics | Ronja's tutorials, https://www.ronja-tutorials.com/post/043-spheretracing-basics/, fetched 2026-10-27 (unofficial)
[S3] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
