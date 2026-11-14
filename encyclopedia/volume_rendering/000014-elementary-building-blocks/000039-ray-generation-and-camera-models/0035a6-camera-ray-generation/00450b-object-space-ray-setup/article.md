# Object Space Ray Setup

Also called: Camera Space Transformation for Ray Origin, Local Space Ray Setup, Object Space Ray Casting, Ray Origin Calculation for Near Plane Quad, Ray Casting Vertex Shader, RayCasterShaderCodeTester::NearPlaneVertexShader

## Overview
Object Space Ray Setup constructs ray origins and directions within the local coordinate system of a mesh, ensuring that the raytraced volume moves, scales, and rotates rigidly with the object's transform [S2, unofficial]. Unlike world-space ray generation, which treats the volume as a static window into the world, this method embeds the object's transformation directly into the ray parameters, allowing the raytraced geometry to behave like a physical part of the mesh [S2, unofficial]. It serves as the foundational step for [Camera Ray Generation](7456-camera-ray-generation.md) when the rendering target is a specific object rather than the entire scene.

## How it works
The method computes the ray origin and direction in the vertex shader and passes them to the fragment shader for normalization and use in ray marching.

**Inputs:**
- `v.vertex`: The vertex position in local/object space (from `appdata`).
- `u_camera` or `_WorldSpaceCameraPos`: The camera position in world space.
- `unity_WorldToObject` or `inverse(modelMatrix)`: The matrix transforming world coordinates to object space.

**Outputs:**
- `rayOrigin`: The starting point of the ray in object space.
- `rayDirection`: The direction of the ray in object space.

**Data Structures:**
The vertex shader output struct (`v2f`) contains:
- `position` (`SV_POSITION`): Clip-space position for rasterization.
- `localPosition` (`TEXCOORD0`, `float4`): The vertex position in local space, used as the ray origin.
- `viewDirection` (`TEXCOORD1`, `float4`): The unnormalized view vector in local space.

**Steps:**
1. **Transform Camera to Object Space:** Multiply the world-space camera position by the world-to-object matrix. The w-component of the camera position vector must be `1.0` to include translation [S2, unofficial].
   $$ \mathbf{c}_{obj} = \mathbf{M}_{world \to obj} \times \begin{bmatrix} \mathbf{c}_{world} \\ 1.0 \end{bmatrix} $$
   Where $\mathbf{c}_{obj}$ is the camera position in object space and $\mathbf{M}_{world \to obj}$ is the inverse model matrix [S1, unofficial][S2, unofficial].

2. **Compute Ray Origin:** The ray origin is the vertex's local position.
   $$ \mathbf{o} = \mathbf{v}_{local} $$
   Where $\mathbf{v}_{local}$ is `v.vertex` [S2, unofficial].

3. **Compute Unnormalized View Direction:** Subtract the object-space camera position from the local vertex position.
   $$ \mathbf{d}_{raw} = \mathbf{v}_{local} - \mathbf{c}_{obj} $$
   Where $\mathbf{d}_{raw}$ is the raw view direction vector [S2, unofficial].

4. **Pass to Fragment Shader:** Interpolate `localPosition` and `viewDirection` across the primitive.

5. **Normalize Direction:** In the fragment shader, normalize the view direction. Normalization must occur here, not in the vertex shader, because linear interpolation of vectors distorts their length, particularly on low-poly meshes or at close camera distances [S2, unofficial].
   $$ \mathbf{d} = \frac{\mathbf{d}_{raw}}{\|\mathbf{d}_{raw}\|} $$

**Pitfalls:**
- **W-Component Error:** If the camera position is transformed with a w-component of `0`, the translation component of the inverse matrix is ignored, resulting in incorrect ray origins that only account for rotation and scale [S2, unofficial].
- **Interpolation Distortion:** Normalizing the view direction in the vertex shader leads to incorrect angles in the fragment shader due to perspective-correct interpolation artifacts [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
struct v2f {
    float4 position : SV_POSITION;
    float4 localPosition : TEXCOORD0;
    float4 viewDirection : TEXCOORD1;
};

v2f vert(appdata v) {
    v2f o;
    
    // 1. Compute clip-space position for rasterization
    o.position = UnityObjectToClipPos(v.vertex);
    
    // 2. Store local position as ray origin
    o.localPosition = v.vertex;
    
    // 3. Transform camera world position to object space
    // Note: w-component must be 1.0 to include translation [S2, unofficial]
    float4 objectSpaceCameraPos = mul(unity_WorldToObject, float4(_WorldSpaceCameraPos, 1.0));
    
    // 4. Compute unnormalized view direction in local space
    o.viewDirection = v.vertex - objectSpaceCameraPos;
    
    return o;
}

# Fragment Shader
float4 frag(v2f i) : SV_Target {
    // 5. Normalize direction in fragment shader to avoid interpolation distortion [S2, unofficial]
    float3 dir = normalize(i.viewDirection.xyz);
    
    // 6. Use local position as ray origin
    float3 rayOrigin = i.localPosition.xyz;
    
    // Ray marching logic would follow here, using rayOrigin and dir
    // ...
    
    return float4(1.0, 1.0, 1.0, 1.0);
}
```

## Subtopics

### [Object Space Ray Setup — Camera Space Transformation for Ray Origin, Local Space Ray Setup, Object Space Ray Casting](4179-object-space-ray-setup-camera-space-transformation-for-ray-o.md)
This variant explicitly details the construction of the ray origin in volume texture space by transforming a vertex from the near plane quad through inverse view and model transformations, followed by scaling and translation to align with the volume grid. It differs from world-space ray generation by embedding the object's transform directly into the ray origin calculation, allowing the raytraced object to move, scale, and rotate with the mesh transform [1642-object-space-ray-setup.md].

## Operating characteristics
- **Time Complexity:** $O(1)$ per vertex for setup; $O(1)$ per fragment for normalization. The cost is negligible compared to the ray marching loop.
- **Memory Complexity:** $O(1)$ additional memory per vertex for the `v2f` struct.
- **Strengths:** Ensures the raytraced volume is rigidly attached to the mesh, simplifying animation and transformation logic. Intuitive for single-object rendering [S2, unofficial].
- **Limitations:** Requires the camera position to be available in the shader. Not suitable for global illumination or complex scene traversal where world-space consistency is required.
- **When to use:** When rendering a single object or a set of objects that move together as a unit, and the volume is defined in local space.
- **When not to use:** When the volume is static in world space and the camera moves through it, or when multiple independent objects need to be raytraced in a single pass with different transforms.

## Background
The method originates from basic sphere tracing and ray marching tutorials, where object-space rendering is preferred for its simplicity and intuitive handling of object transformations [S2, unofficial]. It is commonly used in real-time volume rendering demos and educational resources, such as those by Ronja's Tutorials and various GitHub repositories demonstrating GLSL-based volume rendering [S1, unofficial][S2, unofficial][S3, unofficial].

## Sources
[S1] Show HN: Volume rendering 3D data in Three.js and GLSL, https://github.com/SuboptimalEng/volume-rendering, fetched 2026-10-26 (unofficial)
[S2] Spheretracing Basics | Ronja's tutorials, https://www.ronja-tutorials.com/post/043-spheretracing-basics/, fetched 2026-10-27 (unofficial)
[S3] Volumetric Render Engine - GitHub, https://github.com/mikejernil/volumetric-render-engine, fetched 2026-10-27 (unofficial)
