# Raymarching with Depth Buffer Occlusion

Also called: Depth-buffered Raymarching, Raymarching with Mesh Interaction, Z-Buffer Occlusion

## Overview
Raymarching with Depth Buffer Occlusion is a technique that allows raymarched Signed Distance Field (SDF) objects to correctly occlude and be occluded by standard mesh-based geometry [S2, unofficial]. It solves the problem where raymarched objects typically float on top of all other scene elements because the raymarching loop ignores the depth of pre-rendered meshes [S2, unofficial]. By querying the scene's depth buffer to determine the distance to the nearest mesh surface, the raymarcher can terminate early if the mesh is closer, ensuring proper visual layering [S2, unofficial].

## How it works
The method integrates the standard sphere tracing loop [Parent](21-distance-field-ray-marching.md) with a depth buffer lookup. The core modification is calculating the maximum allowable travel distance for each ray based on the nearest mesh surface and enforcing this limit within the marching loop [S2, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   Ray origin $ro$ (camera position in world space) [S2, unofficial].
    *   Ray direction $rd$ (normalized direction in world space) [S2, unofficial].
    *   UV coordinates $uv$ for depth sampling [S2, unofficial].
    *   SDF scene definition (implicit function) [Parent, S2, unofficial].
    *   Scene depth texture `_CameraDepthTexture` [S2, unofficial].
*   **Outputs**:
    *   Final pixel color, blending the raymarched SDF result with the background (mesh) color [S2, unofficial].

### Data Structures
*   **Depth Buffer**: A texture (e.g., `sampler2D`) storing linear eyespace depth values [S2, unofficial].
*   **Vertex Shader Output (`v2f`)**:
    *   `ray`: A `float3` representing the worldspace ray direction normalized in z [S2, unofficial].

### Steps
1.  **Vertex Shader Preparation**:
    *   Compute the ray direction $rd$ in eyespace (view space) [S2, unofficial].
    *   Normalize the ray in the z-axis to create $r_n$ by dividing by `abs(rd.z)`, ensuring the z-component is 1.0 [S2, unofficial]. In eyespace, $z < 0$ is the forward direction [S2, unofficial].
        $$ r_n = \frac{r_d}{|r_d.z|} $$
    *   Transform $r_n$ from eyespace to worldspace using the inverse view matrix:
        $$ o.ray = \text{mul}(\text{_CameraInvViewMatrix}, r_n) $$
    *   Pass $o.ray$ to the fragment shader [S2, unofficial].

2.  **Fragment Shader Depth Sampling**:
    *   Sample the depth buffer texture at the current UV coordinates to get the depth value [S2, unofficial].
    *   Convert the sampled depth to linear eyespace depth $d$ using a function like `LinearEyeDepth()` [S2, unofficial]. This $d$ represents the distance along the z-axis from the camera plane to the mesh surface [S2, unofficial].
    *   Calculate the true Euclidean distance $s$ from the camera origin to the mesh surface along the ray. This leverages similar triangles, using the length of the z-normalized ray vector $r_n$ (passed as `i.ray.xyz`):
        $$ s = d \times |r_n| $$
        Where $|r_n|$ is the magnitude of the vector $r_n$ computed in step 1 [S2, unofficial].

3.  **Raymarching Loop**:
    *   Initialize current distance traveled $t = 0$ [S2, unofficial].
    *   Iterate up to `maxstep` (e.g., 64) [S2, unofficial]:
        *   **Occlusion Check**: If $t \ge s$, the ray has reached or passed the mesh surface. Terminate the loop and return a transparent color to allow the mesh to show through [S2, unofficial].
        *   **SDF Evaluation**: Evaluate the SDF at the current position $p = ro + t \cdot \hat{rd}$ (where $\hat{rd}$ is the normalized world-space ray direction) [Parent, S2, unofficial].
        *   **Hit Check**: If the SDF value $d_{sdf} \le \text{HIT\_THRESHOLD}$, a hit is found [Parent]. Compute shading and return the color.
        *   **Step**: Advance $t$ by $d_{sdf}$ [Parent].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def normalize(v):
    l = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    return (v[0]/l, v[1]/l, v[2]/l)

def length(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def mat_mul_vec(M, v):
    # M is a 4x4 inverse view matrix, v is a direction vector (w=0)
    return (M[0][0]*v[0] + M[0][1]*v[1] + M[0][2]*v[2],
            M[1][0]*v[0] + M[1][1]*v[1] + M[1][2]*v[2],
            M[2][0]*v[0] + M[2][1]*v[1] + M[2][2]*v[2])

def vertex_shader(appdata, inv_view_matrix):
    # Compute ray direction in eyespace
    rd_eyespace = normalize(appdata['position'])
    # Normalize in Z-axis to create rn
    # Use abs() to prevent direction flip in eyespace where z < 0 is forward
    rn_eyespace = (rd_eyespace[0] / abs(rd_eyespace[2]), rd_eyespace[1] / abs(rd_eyespace[2]), 1.0)
    # Transform to world space
    ray_world = mat_mul_vec(inv_view_matrix, rn_eyespace)
    return {'uv': appdata['uv'], 'ray': ray_world}

def fragment_shader(v2f, depth_tex, camera_pos_ws, sdf_func):
    # 1. Get Depth Buffer Value
    depth_val = depth_tex[v2f['uv']]
    # 2. Convert to Linear Eyespace Depth
    # LinearEyeDepth converts the non-linear depth buffer value to linear distance [G]
    d = depth_val 
    # 3. Calculate True Distance to Mesh Surface
    # length(v2f['ray']) is the magnitude of the z-normalized ray vector
    s = d * length(v2f['ray'])
    # 4. Raymarch
    ro = camera_pos_ws
    rd = normalize(v2f['ray'])
    t = 0.0
    maxstep = 64
    hit_threshold = 0.001
    
    for i in range(maxstep):
        # Check against mesh depth
        if t >= s:
            # Ray hit the mesh first, return transparent to let mesh show
            return (0.0, 0.0, 0.0, 0.0)
        
        # Evaluate SDF
        p = (ro[0] + rd[0]*t, ro[1] + rd[1]*t, ro[2] + rd[2]*t)
        dist = sdf_func(p)
        
        # Check for SDF hit
        if dist <= hit_threshold:
            # Compute normal and shading
            normal = compute_normal(p, sdf_func)
            color = shade(p, normal, rd)
            return (color[0], color[1], color[2], 1.0)
        
        # Step forward
        t += dist
    
    # If loop finishes without hit or mesh occlusion, return transparent
    return (0.0, 0.0, 0.0, 0.0)

def compute_normal(p, sdf_func, eps=0.001):
    dx = sdf_func((p[0]+eps, p[1], p[2])) - sdf_func((p[0]-eps, p[1], p[2]))
    dy = sdf_func((p[0], p[1]+eps, p[2])) - sdf_func((p[0], p[1]-eps, p[2]))
    dz = sdf_func((p[0], p[1], p[2]+eps)) - sdf_func((p[0], p[1], p[2]-eps))
    return normalize((dx, dy, dz))

def shade(p, normal, rd):
    # Standard Lambertian shading [G]
    light_dir = normalize((1.0, 1.0, -1.0))
    diff = max(0.0, normal[0]*light_dir[0] + normal[1]*light_dir[1] + normal[2]*light_dir[2])
    return (diff, diff, diff)
```

### Parameters
*   `_CameraDepthTexture`: Sampler2D for the scene depth buffer [S2, unofficial].
*   `maxstep`: Maximum iterations for the raymarch loop (e.g., 64) [S2, unofficial].
*   `hit_threshold`: Epsilon for surface hit detection (e.g., 0.001) [Parent].

### Pitfalls
*   **Ray Direction Flip**: In eyespace, $z < 0$ is forward [S2, unofficial]. Dividing the ray direction by $r_d.z$ without taking the absolute value (`abs()`) will flip the ray direction if $r_d.z$ is negative, causing the raymarched scene to appear inverted [S2, unofficial].
*   **Depth Precision**: The accuracy of occlusion depends on the depth buffer precision [G]. Low precision may cause z-fighting or incorrect occlusion boundaries between SDF and mesh objects [G].
*   **Transparency**: The method assumes opaque meshes for the depth buffer. Transparent meshes may require additional handling or separate depth passes [G].

## Subtopics

### Radii Scaling for Sphere-Cone Occlusion
This variant addresses the specific case of scale-invariant integral surfaces (like skeleton-based sphere-cones) to guarantee correct occlusion in the depth buffer [S3]. It computes a scaling factor for the radii of sphere-cone primitives to ensure the SDF does not penetrate the occluding depth buffer [S3]. The worst-case configuration for this scaling is identified as two segment primitives aligned with the viewing direction $-\mathbf{v}$, both having radius $\tau$ at point $q$ [S3]. Due to surface scale invariance, the scaling factor is computed numerically for a unit radius case, representing the distance from $q$ to the isosurface in direction $\mathbf{v}$ [S3]. This worst-case configuration is independent of the actual skeleton layout, requiring no additional skeleton processing [S3].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the number of raymarch steps. The depth buffer lookup is $O(1)$ per pixel.
*   **Memory Complexity**: $O(W \cdot H)$ for the depth buffer texture, where $W$ and $H$ are the screen dimensions.
*   **Performance**: Adds minimal overhead (one depth texture sample and a few arithmetic operations) to the raymarching loop [S2, unofficial]. The primary cost remains the SDF evaluations.
*   **Strengths**: Enables seamless integration of SDF objects with standard rasterized meshes without complex hybrid rendering pipelines [S2, unofficial]. Simple to implement in shader languages like HLSL/GLSL [S2, unofficial].
*   **Limitations**: Only works for opaque meshes that write to the depth buffer. Does not handle transparent meshes or SDF objects occluding each other (unless the SDF objects are also rendered to a depth buffer first, which is not described in the primary source). Precision issues may arise with very large or very small scene scales [G].

## Background
The technique addresses a common issue in real-time raymarching where SDF objects fail to interact with the rest of the scene's geometry [S2, unofficial]. It leverages standard graphics API features (depth buffers) to solve this interaction problem [S2, unofficial]. The radii scaling variant for sphere-cones was introduced in research on fast ray tracing of scale-invariant integral surfaces [S3].

## Sources
[S1] Как Я Заставил Esp8266 Рисовать 3d, Стрелять И Летать: Полный ..., https://habr.com/ru/articles/1097380/, fetched 2026-10-26 (unofficial)
[S2] Raymarching Distance Fields: Concepts and Implementation ..., https://adrianb.io/2016/10/01/raymarching.html, fetched 2026-10-27 (unofficial)
[S3] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
