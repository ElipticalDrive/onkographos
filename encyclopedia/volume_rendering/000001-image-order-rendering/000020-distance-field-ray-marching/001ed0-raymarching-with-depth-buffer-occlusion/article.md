# Raymarching with Depth Buffer Occlusion

Also called: Depth-buffered Raymarching, Raymarching with Mesh Interaction, Z-Buffer Occlusion

## Overview
Raymarching with Depth Buffer Occlusion is a technique that allows raymarched Signed Distance Field (SDF) objects to correctly occlude and be occluded by standard mesh-based geometry [S2, unofficial]. It solves the problem where raymarched objects typically float on top of all other scene elements because the raymarching loop ignores the depth of pre-rendered meshes [S2, unofficial]. By querying the scene's depth buffer to determine the distance to the nearest mesh surface, the raymarcher can terminate early if the mesh is closer, ensuring proper visual layering [S2, unofficial]. This method is distinct from pure SDF rendering as it hybridizes rasterization (for meshes) with raymarching (for SDFs), whereas siblings like [SDF Depth Buffer Integration](1955-sdf-depth-buffer-integration.md) may use different integration strategies or acceleration structures.

## How it works
The method integrates the standard sphere tracing loop [Parent](21-distance-field-ray-marching.md) with a depth buffer lookup. The core modification is calculating the maximum allowable travel distance for each ray based on the nearest mesh surface and enforcing this limit within the marching loop.

### Inputs and Outputs
*   **Inputs**:
    *   Ray origin $ro$ (camera position in world space).
    *   Ray direction $rd$ (normalized direction in world space).
    *   Scene depth texture `_CameraDepthTexture` [S2, unofficial].
    *   UV coordinates $uv$ for depth sampling.
    *   SDF scene definition (implicit function).
*   **Outputs**:
    *   Final pixel color, blending the raymarched SDF result with the background (mesh) color.

### Data Structures
*   **Depth Buffer**: A texture (e.g., `sampler2D`) storing linear or non-linear depth values [S2, unofficial].
*   **Vertex Shader Output (`v2f`)**:
    *   `ray`: A `float3` representing the world-space ray direction. Crucially, this vector is normalized such that its z-component in eyespace is 1.0 (magnitude in z-direction) before transformation to world space [S2, unofficial].

### Steps
1.  **Vertex Shader Preparation**:
    *   Compute the ray direction $rd$ in eyespace (view space).
    *   Normalize the ray in the z-axis to create $r_n$. In eyespace, $z < 0$ is the forward direction. To avoid flipping the ray direction when dividing by a negative z-value, divide by the absolute value of the z-component [S2, unofficial]:
        $$ r_n = \frac{r_d}{|r_d.z|} $$
    *   Transform $r_n$ from eyespace to world space using the inverse view matrix:
        $$ o.ray = \text{mul}(\text{_CameraInvViewMatrix}, r_n) $$
    *   Pass $o.ray$ to the fragment shader [S2, unofficial].

2.  **Fragment Shader Depth Sampling**:
    *   Sample the depth buffer texture at the current UV coordinates to get the non-linear depth value [S2, unofficial].
    *   Convert the sampled depth to linear eyespace depth $d$ using a function like `LinearEyeDepth()` [S2, unofficial]. This $d$ represents the distance along the z-axis from the camera plane to the mesh surface.
    *   Calculate the true Euclidean distance $s$ from the camera origin to the mesh surface along the ray. This leverages similar triangles, using the length of the z-normalized ray vector $r_n$ (passed as `i.ray.xyz` in world space, but its length is invariant to rotation if we consider the projection geometry, or more precisely, the length of the vector that has unit z-component in view space):
        $$ s = d \times |r_n| $$
        Where $|r_n|$ is the magnitude of the vector $r_n$ computed in step 1. Note: In the source code, `length(i.ray.xyz)` is used, which corresponds to the length of the transformed vector. Since the transformation is a rotation (inverse view matrix), the length is preserved from the eyespace $r_n$ [S2, unofficial].

3.  **Raymarching Loop**:
    *   Initialize current distance traveled $t = 0$ [S2, unofficial].
    *   Iterate up to `maxstep` (e.g., 64) [S2, unofficial]:
        *   **Occlusion Check**: If $t \ge s$, the ray has reached or passed the mesh surface. Terminate the loop and return a transparent color (or the background color) to allow the mesh to show through [S2, unofficial].
        *   **SDF Evaluation**: Evaluate the SDF at the current position $p = ro + t \cdot \hat{rd}$ (where $\hat{rd}$ is the normalized world-space ray direction).
        *   **Hit Check**: If the SDF value $d_{sdf} \le \text{HIT\_THRESHOLD}$, a hit is found. Compute shading and return the color.
        *   **Step**: Advance $t$ by $d_{sdf}$.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader Logic (Conceptual)
def vertex_shader(appdata):
    # Compute ray direction in eyespace
    rd_eyespace = normalize(appdata.position - camera_position_eyespace)
    
    # Normalize in Z-axis to create rn
    # Use abs() to prevent direction flip in eyespace where z < 0 is forward
    rn_eyespace = rd_eyespace / abs(rd_eyespace.z)
    
    # Transform to world space
    ray_world = mul(_CameraInvViewMatrix, rn_eyespace)
    
    return {
        'uv': appdata.uv,
        'ray': ray_world,
        'position': appdata.position
    }

# Fragment Shader Logic
def fragment_shader(v2f):
    # 1. Get Depth Buffer Value
    depth_texel = tex2D(_CameraDepthTexture, v2f.uv)
    
    # 2. Convert to Linear Eyespace Depth
    # LinearEyeDepth converts the non-linear depth buffer value to linear distance
    d = LinearEyeDepth(depth_texel.r)
    
    # 3. Calculate True Distance to Mesh Surface
    # length(v2f.ray) is the magnitude of the z-normalized ray vector
    # This vector has a z-component of 1.0 in eyespace before rotation
    s = d * length(v2f.ray)
    
    # 4. Raymarch
    ro = _CameraWS # Camera position in world space
    rd = normalize(v2f.ray) # Normalized world space direction
    t = 0.0
    maxstep = 64
    hit_threshold = 0.001
    
    for i in range(maxstep):
        # Check against mesh depth
        if t >= s:
            # Ray hit the mesh first, return transparent to let mesh show
            return fixed4(0, 0, 0, 0)
        
        # Evaluate SDF
        p = ro + rd * t
        dist = scene_sdf(p)
        
        # Check for SDF hit
        if dist <= hit_threshold:
            # Compute normal and shading
            normal = compute_normal(p)
            color = shade(p, normal, rd)
            return fixed4(color, 1.0)
        
        # Step forward
        t += dist
    
    # If loop finishes without hit or mesh occlusion, return transparent
    return fixed4(0, 0, 0, 0)

# Final Blending
def final_color(mesh_color, sdf_color):
    # Alpha blend: mesh is background, SDF is foreground
    # Assuming sdf_color has alpha 1.0 if hit, 0.0 if missed
    return mesh_color * (1.0 - sdf_color.a) + sdf_color.xyz * sdf_color.a
```

### Parameters
*   `_CameraDepthTexture`: Sampler2D for the scene depth buffer [S2, unofficial].
*   `maxstep`: Maximum iterations for the raymarch loop (e.g., 64) [S2, unofficial].
*   `hit_threshold`: Epsilon for surface hit detection (e.g., 0.001) [Parent](21-distance-field-ray-marching.md).

### Pitfalls
*   **Ray Direction Flip**: In eyespace, $z < 0$ is forward. Dividing the ray direction by $r_d.z$ without taking the absolute value (`abs()`) will flip the ray direction if $r_d.z$ is negative, causing the raymarched scene to appear inverted [S2, unofficial].
*   **Depth Precision**: The accuracy of occlusion depends on the depth buffer precision. Low precision may cause z-fighting or incorrect occlusion boundaries between SDF and mesh objects.
*   **Transparency**: The method assumes opaque meshes for the depth buffer. Transparent meshes may require additional handling or separate depth passes.

## Subtopics

### Radii Scaling for Sphere-Cone Occlusion
This variant addresses the specific case of scale-invariant integral surfaces (like skeleton-based sphere-cones) to guarantee correct occlusion in the depth buffer [S3]. It computes a scaling factor for the radii of sphere-cone primitives to ensure the SDF does not penetrate the occluding depth buffer [S3]. The worst-case configuration for this scaling is identified as two segment primitives aligned with the viewing direction $-\mathbf{v}$, both having radius $\tau$ at point $q$ [S3]. Due to surface scale invariance, the scaling factor is computed numerically for a unit radius case, representing the distance from $q$ to the isosurface in direction $\mathbf{v}$ [S3]. This worst-case configuration is independent of the actual skeleton layout, requiring no additional skeleton processing [S3].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of raymarch steps. The depth buffer lookup is $O(1)$ per pixel.
*   **Memory Complexity**: $O(W \cdot H)$ for the depth buffer texture, where $W$ and $H$ are the screen dimensions.
*   **Performance**: Adds minimal overhead (one depth texture sample and a few arithmetic operations) to the raymarching loop [S2, unofficial]. The primary cost remains the SDF evaluations.
*   **Strengths**: Enables seamless integration of SDF objects with standard rasterized meshes without complex hybrid rendering pipelines [S2, unofficial]. Simple to implement in shader languages like HLSL/GLSL [S2, unofficial].
*   **Limitations**: Only works for opaque meshes that write to the depth buffer. Does not handle transparent meshes or SDF objects occluding each other (unless the SDF objects are also rendered to a depth buffer first, which is not described in the primary source). Precision issues may arise with very large or very small scene scales.

## Background
The technique was popularized by Adrian Buehler in a blog post titled "Raymarching Distance Fields: Concepts and Implementation" [S2, unofficial]. It addresses a common issue in real-time raymarching where SDF objects fail to interact with the rest of the scene's geometry. The method leverages standard graphics API features (depth buffers) to solve this interaction problem. The concept of using a depth buffer to limit raymarching is also referenced in earlier contexts, such as the "depth buffering technique using a 1D array" for 2D raycasting in embedded systems [S1, unofficial], though the 3D mesh interaction method is distinct. The radii scaling variant for sphere-cones was introduced in research on fast ray tracing of scale-invariant integral surfaces [S3].

## Sources
[S1] Как Я Заставил Esp8266 Рисовать 3d, Стрелять И Летать: Полный ..., https://habr.com/ru/articles/1097380/, fetched 2026-10-26 (unofficial)
[S2] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S3] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
