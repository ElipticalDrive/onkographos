# Raymarched SDF Volume Rendering

Also called: Ray Marching with Signed Distance Fields, SDF Raycasting

## Overview
Raymarched SDF Volume Rendering renders 3D geometry defined by analytic Signed Distance Functions (SDFs) by casting rays from the camera and stepping along them until they intersect the surface [S1, unofficial][S2, unofficial]. Unlike traditional rasterization or mesh-based raycasting, this method constructs the image directly from equations, allowing for complex, continuous geometry without explicit polygonal meshes [S1, unofficial][S2, unofficial]. It differs from its parent [Sphere Tracing with SDFs](6223-sphere-tracing-with-sdfs.md) by often incorporating specific shading, lighting, and fog calculations within the raymarching loop to produce the final pixel color, rather than just returning a binary hit/miss or surface normal [S2, unofficial].

## How it works
**Inputs:**
*   Ray origin $o$ (camera position) and normalized direction $\hat{r}$ for each pixel [S1, unofficial][S2, unofficial].
*   Scene SDF function $S(p)$ that returns the signed distance to the nearest surface at point $p$ [S1, unofficial].
*   Parameters: `MAX_STEPS` (maximum iterations), `HIT_THRESHOLD` (epsilon for surface detection), `MAX_RANGE` (maximum travel distance), and `STEP_SCALE` (optional multiplier for step size safety) [S1, unofficial][S2, unofficial].

**Outputs:**
*   Final pixel color (and optionally alpha) [S2, unofficial].

**Data Structures:**
*   The scene is defined by a single callable function $S(p)$ [S1, unofficial]. In complex scenes, this function may return a pair $(d, \text{material\_id})$ where $d$ is the distance and $\text{material\_id}$ identifies the object for shading [S1, unofficial].
*   Camera parameters: `_CamPos`, `_CamRight`, `_CamUp`, `_CamForward` [S2, unofficial].
*   Lighting parameters: `_LightColor0`, `_WorldSpaceLightPos0` [S2, unofficial].

**Numbered Steps:**
1.  **Ray Construction:** For each pixel with UV coordinates $uv \in [0,1]$, transform to $[-1,1]$ and construct the ray direction $\hat{r}$ using camera basis vectors and a focal length $f$:
    $$ \hat{r} = \text{normalize}(\text{_CamUp} \cdot uv_y + \text{_CamRight} \cdot uv_x + \text{_CamForward} \cdot f) $$
    [S2, unofficial]
    The ray origin $o$ is the camera position `_CamPos` [S2, unofficial].

2.  **Initialization:** Set current position $p = o$ and iteration counter $i = 0$ [S1, unofficial][S2, unofficial].

3.  **Marching Loop:** While $i < \text{MAX\_STEPS}$:
    a.  **Evaluate SDF:** Compute $d = S(p)$ [S1, unofficial][S2, unofficial].
    b.  **Hit Test:** If $d \le \text{HIT\_THRESHOLD}$ (or $d < 0$ indicating inside the surface), the ray has hit the object [S1, unofficial][S2, unofficial].
        i.  Compute the surface normal $n$ at $p$ using numerical differentiation (gradient approximation) with epsilon $\epsilon$:
            $$ n = \text{normalize}\left( \begin{bmatrix} S(p + \epsilon_{yxx}) - S(p - \epsilon_{yxx}) \\ S(p + \epsilon_{xyx}) - S(p - \epsilon_{xyx}) \\ S(p + \epsilon_{xxy}) - S(p - \epsilon_{xxy}) \end{bmatrix} \right) $$
            where $\epsilon_{yxx} = (0.0, \epsilon, 0.0)$, etc. [S2, unofficial]
        ii. Compute lighting:
            $$ \text{diffuse} = \text{_LightColor0.rgb} \cdot \max(\text{dot}(\text{lightDir}, n), 0.0) $$
            $$ \text{ambient} = \text{UNITY\_LIGHTMODEL\_AMBIENT.xyz} \cdot 2.0 $$
            $$ \text{final\_light} = \text{diffuse} + \text{ambient} $$
            [S2, unofficial]
        iii. Compute fog:
            $$ \text{dist} = \| p - \text{_CamPos} \| $$
            $$ f_{\text{fog}} = \exp(-\text{dist} \cdot \text{fogDensity}) $$
            $$ \text{fog\_color} = \text{fogColor} \cdot (1.0 - f_{\text{fog}}) + \text{color} \cdot f_{\text{fog}} $$
            [S2, unofficial]
        iv. Return the computed color and break the loop [S2, unofficial].
    c.  **Miss/Far Test:** If $d > \text{MAX\_RANGE}$, the ray has missed the scene bounds. Break the loop and return background color [S2, unofficial].
    d.  **Advance:** Update position:
        $$ p \leftarrow p + \hat{r} \cdot d \cdot \text{STEP\_SCALE} $$
        [S2, unofficial]
        Note: Standard sphere tracing uses $\text{STEP\_SCALE}=1.0$. Some implementations use a conservative scale (e.g., 0.5) to prevent tunnelling through thin structures or due to non-rigid transforms [S1, unofficial][S2, unofficial].
    e.  Increment $i$.

4.  **Fallback:** If the loop terminates without a hit, return the background color or transparent color [S2, unofficial].

**Formulas:**
*   Ray: $p(s) = o + s\,\hat{r}$ [S1, unofficial].
*   Sphere SDF: $d(p) = \|p - c\| - r$ [S2, unofficial].
*   Box SDF: $d(p) = \| \max(|p - c| - b, 0.0) \|$ [S2, unofficial].
*   Union: $d_{union}(p) = \min(d_1(p), d_2(p))$ [S1, unofficial][S2, unofficial].
*   Intersection: $d_{intersect}(p) = \max(d_1(p), d_2(p))$ [S1, unofficial][S2, unofficial].
*   Subtraction: $d_{sub}(p) = \max(d_1(p), -d_2(p))$ [S1, unofficial][S2, unofficial].

**Parameters:**
*   `MAX_STEPS`: Typically 64 to 130 [S1, unofficial][S2, unofficial].
*   `HIT_THRESHOLD`: Typically 0.001 to 0.005 [S1, unofficial][S2, unofficial].
*   `MAX_RANGE`: Typically 400.0 or similar scene-bound dependent value [S2, unofficial].
*   `focalLength`: Experimentally determined, e.g., 0.62 [S2, unofficial].
*   `fogDensity`: e.g., 0.1 [S2, unofficial].
*   `STEP_SCALE`: 1.0 for standard sphere tracing; <1.0 (e.g., 0.5) for conservative stepping [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Tunnelling:** In non-rigid transforms (e.g., bending thin wings), distances may be overstated. Using a conservative step scale or increasing `MAX_STEPS` prevents rays from passing through thin geometry [S1, unofficial].
*   **Clipping:** The rendering quad must be positioned beyond the camera's near clipping plane (e.g., $z=0.32$ if near clip is $0.3$) [S2, unofficial].
*   **Layer Culling:** To render SDF volumes in the background, use a separate camera with "depth only" clear flags and exclude the volume layer from the main camera's culling mask [S2, unofficial].
*   **Normal Accuracy:** Numerical differentiation requires a small epsilon. Too large an epsilon causes jagged normals; too small causes noise [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raymarch_sdf_volume_rendering(cam_pos, cam_right, cam_up, cam_forward, focal_length, scene_sdf, max_steps=64, hit_threshold=0.005, max_range=400.0, step_scale=0.5, fog_density=0.1, fog_color=(0.8, 0.8, 0.8), light_color=(1.0, 1.0, 1.0), light_dir=(0.0, 1.0, 0.0)):
    """
    Renders a single ray using SDF volume rendering techniques.
    Note: This is a simplified procedural representation. In practice, 
    scene_sdf, lighting, and fog parameters are often shader uniforms.
    """
    # 1. Ray Construction (Assumed pre-computed per pixel)
    # uv is in [-1, 1]
    # ray_dir = normalize(cam_up * uv.y + cam_right * uv.x + cam_forward * focal_length)
    # ray_orig = cam_pos
    
    # For the sake of this self-contained function, we assume ray_orig and ray_dir are inputs
    # or we construct a default ray for demonstration. 
    # Let's assume the caller passes ray_orig and ray_dir.
    pass

def render_pixel(uv, cam_pos, cam_right, cam_up, cam_forward, focal_length, scene_sdf, max_steps=64, hit_threshold=0.005, max_range=400.0, step_scale=0.5, fog_density=0.1, fog_color=(0.8, 0.8, 0.8), light_color=(1.0, 1.0, 1.0), light_dir=(0.0, 1.0, 0.0)):
    # Transform UV from [0,1] to [-1,1]
    uv = uv * 2.0 - 1.0
    
    # Construct ray
    ray_dir = normalize(cam_up * uv.y + cam_right * uv.x + cam_forward * focal_length)
    ray_orig = cam_pos
    
    p = ray_orig
    color = (0.0, 0.0, 0.0, 0.0) # Default transparent/background
    
    for i in range(max_steps):
        # Evaluate SDF
        d = scene_sdf(p)
        
        # Hit test
        if d <= hit_threshold:
            # Compute Normal via numerical differentiation
            eps = 0.02
            eps_yxx = (0.0, eps, 0.0)
            eps_xyx = (0.0, 0.0, eps)
            eps_xxy = (eps, 0.0, 0.0)
            
            nx = scene_sdf(p + eps_yxx) - scene_sdf(p - eps_yxx)
            ny = scene_sdf(p + eps_xyx) - scene_sdf(p - eps_xyx)
            nz = scene_sdf(p + eps_xxy) - scene_sdf(p - eps_xxy)
            normal = normalize((nx, ny, nz))
            
            # Lighting
            diffuse = light_color * max(dot(light_dir, normal), 0.0)
            ambient = (0.5, 0.5, 0.5) # Placeholder for UNITY_LIGHTMODEL_AMBIENT * 2.0
            final_light = diffuse + ambient
            
            # Base color (e.g., gray)
            base_color = (0.7, 0.7, 0.7)
            lit_color = base_color * final_light
            
            # Fog
            dist = length(p - ray_orig)
            fog_factor = exp(-dist * fog_density)
            fog_result = fog_color * (1.0 - fog_factor) + lit_color * fog_factor
            
            # Alpha (optional, based on lighting)
            alpha = max(dot(light_dir, normal), 0.0)
            
            color = (fog_result[0], fog_result[1], fog_result[2], alpha)
            break
            
        # Far test
        if d > max_range:
            break
            
        # Advance
        p = p + ray_dir * d * step_scale
        
    return color
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is the average number of raymarch steps per pixel. $M$ depends on scene complexity and `MAX_STEPS` [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per ray, as the SDF is analytic and does not require storing a 3D texture or voxel grid [S1, unofficial].
*   **Performance:** Performance is heavily dependent on the cost of the SDF evaluation. Complex SDFs (e.g., with many boolean operations or deformations) slow down the loop [S1, unofficial][S2, unofficial].
*   **Strengths:** Infinite resolution geometry, no mesh artifacts, easy to create complex shapes via boolean operations, compact code [S1, unofficial][S2, unofficial].
*   **Limitations:** No inherent support for transparency/translucency without complex accumulation (this article focuses on opaque surface rendering via SDFs), aliasing on thin structures if step size is not conservative, no direct support for shadows from other objects without additional raycasts [S1, unofficial][S2, unofficial].
*   **When to use:** Stylized rendering, procedural geometry, real-time effects where mesh generation is prohibitive [S1, unofficial][S2, unofficial].
*   **When not to use:** Photorealistic rendering requiring complex light transport (global illumination), scenes with highly detailed static geometry better represented by meshes [S1, unofficial][S2, unofficial].

## Background
*   **Origin:** The technique is rooted in the work of Inigo Quilez and others on raymarching with SDFs in fragment shaders [S1, unofficial][S2, unofficial].
*   **History:** Popularized in the demoscene and real-time graphics communities for creating procedural art and effects without traditional modeling tools [S1, unofficial][S2, unofficial].
*   **Usage:** Used in real-time shaders for visual effects, procedural generation, and educational demonstrations of ray tracing principles [S1, unofficial][S2, unofficial].

## Sources
[S1] el-bakkali/manim-glsl-butterflies, https://github.com/el-bakkali/manim-glsl-butterflies, fetched 2026-10-26 (unofficial)
[S2] Advanced shaders in Unity - Volume render a death star | Habrador, https://www.habrador.com/tutorials/shaders/1-volume-rendering-death-star/, fetched 2026-10-28 (unofficial)
