# Object Space Ray Setup

Also called: Camera Space Transformation for Ray Origin, Local Space Ray Setup, Object Space Ray Casting

## Overview
Object Space Ray Setup constructs ray origins and directions in the local coordinate system of the rendered mesh rather than world space [S2, unofficial]. This ensures that the raytraced object moves, scales, and rotates rigidly with the mesh transform, treating the object as a self-contained volume [S2, unofficial]. Unlike world-space methods where the object acts as a static window into a larger world, this approach simplifies scene graph transformations for isolated volume rendering [S2, unofficial]. It builds on [Ray Origin and Direction Setup](1793-ray-origin-and-direction-setup.md) by fixing the coordinate frame to the object.

## How it works
**Inputs:**
- Vertex position in local space ($v_{local}$).
- Camera position in world space ($C_{world}$).
- World-to-Object transformation matrix ($M_{wo}$), typically the inverse of the Model matrix [S1, S2, unofficial].

**Outputs:**
- Ray origin in object space ($O_{obj}$).
- Ray direction in object space ($D_{obj}$).

**Data Structures:**
The vertex shader output struct (`v2f`) passes data to the fragment shader. The layout is:
- `position`: Clip space position (`SV_POSITION`).
- `localPosition`: Local space vertex position (`TEXCOORD0`, `float4`). The w-component is typically 1.0 [S2, unofficial].
- `viewDirection`: Unnormalized view vector in local space (`TEXCOORD1`, `float4`). The w-component is unused or 0 [S2, unofficial].

**Steps:**

1. **Transform Camera to Object Space:**
   Compute the camera position in object space ($C_{obj}$) by multiplying the world-to-object matrix with the camera's world position. The camera position must be treated as a point (w=1) to include translation [S2, unofficial].
   $$ C_{obj} = M_{wo} \cdot \begin{bmatrix} C_{world} \\ 1 \end{bmatrix} $$
   [S1, S2, unofficial]

2. **Compute View Direction (Vertex Shader):**
   Calculate the unnormalized view direction vector in local space by subtracting the object-space camera position from the vertex's local position [S2, unofficial].
   $$ V_{dir} = v_{local} - C_{obj} $$
   [S2, unofficial]

3. **Pass Data to Fragment Shader:**
   Interpolate `localPosition` and `viewDirection` across the primitive [S2, unofficial].

4. **Normalize Direction (Fragment Shader):**
   Normalize the interpolated view direction to obtain the unit ray direction [S2, unofficial].
   $$ D_{obj} = \text{normalize}(V_{dir}) $$
   [S2, unofficial]

5. **Set Ray Origin:**
   Use the interpolated local position as the ray origin [S1, S2, unofficial].
   $$ O_{obj} = v_{local} $$
   [S1, S2, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
struct v2f {
    vec4 position;      // Clip space
    vec4 localPosition; // Local space origin
    vec4 viewDirection; // Unnormalized local space view vector
};

v2f vertex_shader(vec4 vertex, vec3 camera_world_pos, mat4 world_to_object_matrix) {
    v2f o;
    
    // 1. Transform camera to object space
    // w-component must be 1.0 to include translation [S2, unofficial]
    vec4 camera_obj = world_to_object_matrix * vec4(camera_world_pos, 1.0);
    
    // 2. Compute unnormalized view direction in local space [S2, unofficial]
    o.viewDirection = vertex - camera_obj;
    
    // 3. Store local position for ray origin [S2, unofficial]
    o.localPosition = vertex;
    
    // 4. Compute clip space position for rasterization
    o.position = compute_clip_space_position(vertex);
    
    return o;
}

# Fragment Shader
vec4 fragment_shader(v2f i) {
    // 1. Normalize direction in fragment shader to avoid interpolation distortion [S2, unofficial]
    vec3 dir = normalize(i.viewDirection.xyz);
    
    // 2. Set ray origin from local position [S2, unofficial]
    vec3 origin = i.localPosition.xyz;
    
    // 3. Proceed with ray marching/spheretracing using origin and dir in object space
    // ...
}
```

**Parameters:**
- `world_to_object_matrix`: The inverse of the model matrix applied to the mesh [S1, S2, unofficial].
- `camera_world_pos`: The position of the camera in world coordinates [S2, unofficial].

**Edge Cases and Pitfalls:**
- **W-Component:** When transforming the camera position, the w-component must be 1.0. If it is 0, the translation component of the matrix is ignored, resulting in only rotation and scale being applied to the camera position [S2, unofficial].
- **Interpolation Distortion:** Normalization must occur in the fragment shader, not the vertex shader. Interpolating normalized vectors between vertices distorts their length, causing visible artifacts, especially on low-poly objects or at close camera distances [S2, unofficial].

## Subtopics

### Object Space Ray Setup — Camera Space Transformation for Ray Origin, Local Space Ray Setup, Object Space Ray Casting
This variant specifically targets volume texture space mapping for a near-plane quad [S3, unofficial]. It transforms the vertex from view space to world space using the InverseModelViewMatrix, then applies a translation (`VolTranslation`) and scale (`VolScale`) to map the position into volume texture coordinates, finally offsetting by 0.5 to center the coordinates [S3, unofficial]. This is used for testing ray casting shaders against volume data [S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per vertex for setup; normalization is $O(1)$ per fragment.
- **Memory Complexity:** $O(1)$ additional memory for storing the transformed camera position and view vector in the vertex output struct [S2, unofficial].
- **Strengths:** The raytraced object moves, scales, and rotates with the mesh transform, making it intuitive for isolated object rendering [S2, unofficial]. It avoids the need to transform the entire scene into world space for local raymarching [S2, unofficial].
- **Limitations:** It is specific to the object's local coordinate system. If the object is part of a complex scene requiring global interactions (e.g., shadows from other objects), additional transformations are needed [S2, unofficial].
- **When to use:** When rendering a single object or a self-contained volume where the camera relative to the object is the primary concern [S2, unofficial].
- **When not to use:** When the raymarching needs to interact with other world-space objects or when the object's transform is dynamic and complex, requiring world-space consistency [S2, unofficial].

## Background
The method is described in tutorials on spheretracing and volume rendering, emphasizing the distinction between world-space and object-space ray generation [S2, unofficial]. It is used in GLSL-based volume rendering implementations, such as those in Three.js, where the camera position is transformed to object space to serve as the ray origin [S1, unofficial]. The technique is also applied in voxel visualization tools for mapping near-plane vertices to volume texture space [S3, unofficial].

## Sources
[S1] Show HN: Volume rendering 3D data in Three.js and GLSL, https://github.com/SuboptimalEng/volume-rendering, fetched 2026-10-26 (unofficial)
[S2] Spheretracing Basics | Ronja's tutorials, https://www.ronja-tutorials.com/post/043-spheretracing-basics/, fetched 2026-10-27 (unofficial)
[S3] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
