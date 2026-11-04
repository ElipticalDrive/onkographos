# Simple Reflections

Also called: Iterative Specular Reflections

## Overview
Simple Reflections is a local illumination technique that approximates specular highlights by raymarching along the geometric reflection vector at a surface hit point and blending the resulting color with the base surface color [S1, S2, unofficial]. It calculates a simple reflection by raymarching off the surface normal and blending the result with the surface color [S1, unofficial]. Unlike [Specular Reflections via Polynomial Expansion](1172-specular-reflections-via-polynomial-expansion.md), which uses polynomial expansion of the Phong term for analytical evaluation, Simple Reflections relies on iterative numerical raymarching [S1, S2, unofficial].

## How it works
**Inputs:**
*   `color`: The base surface color (RGB) at the hit point [S2, unofficial].
*   `point`: The 3D coordinates of the surface hit point [S2, unofficial].
*   `direction`: The incoming view direction vector (normalized) [S2, unofficial].
*   `scene`: The signed distance field or scene representation providing normal and raymarching queries [S2, unofficial].
*   `light`: The light source definition [S2, unofficial].
*   `reflection-count`: Integer, maximum number of bounces allowed (default 3) [S2, unofficial].
*   `DRAW-DISTANCE`: Float, maximum raymarch distance for a valid hit [S2, unofficial].

**Outputs:**
*   `final_color`: The blended RGB color including reflections [S2, unofficial].

**Data Structures:**
*   Vectors are 3-component floats ($\mathbf{v} = [x, y, z]$) [G].
*   Colors are 3-component floats in range $[0, 1]$ [G].

**Steps:**
1. Initialize state variables:
    *   $p \leftarrow \text{point}$
    *   $d \leftarrow \text{direction}$
    *   $n \leftarrow \text{get-normal}(p, \text{scene})$ [S2, unofficial]
    *   $i \leftarrow 0$
    *   $\text{not-done?} \leftarrow \text{true}$ [S2, unofficial]
    *   $C_{\text{out}} \leftarrow \text{color}$ [S2, unofficial]
2. Loop while $i < \text{reflection-count}$ AND $\text{not-done?}$:
    1. Compute the reflection vector $\mathbf{r}$ using the surface normal $\mathbf{n}$ and incoming direction $\mathbf{d}$:
        $$ \mathbf{r} = \mathbf{d} - 2(\mathbf{d} \cdot \mathbf{n})\mathbf{n} $$ [S1, S2, unofficial]
    2. Raymarch from a point slightly above the surface to avoid self-intersection. Start position $p_{\text{start}} = p + \mathbf{n} \cdot \epsilon$ (implemented as `above-surface-point` in the source) [S2, unofficial].
    3. March ray in direction $\mathbf{r}$ through the scene. Let the hit distance be `distance` and the hit color be `hit_color` [S2, unofficial].
    4. If `distance` < `DRAW-DISTANCE`:
        1. Update position: $p \leftarrow p + \mathbf{r} \cdot \text{distance}$ [S2, unofficial].
        2. Update normal: $n \leftarrow \text{get-normal}(p, \text{scene})$ [S2, unofficial].
        3. Update direction: $d \leftarrow \mathbf{r}$ [S2, unofficial].
        4. Compute lightness $L_{\text{hit}}$ at the new point $p$ [S2, unofficial].
        5. Attenuate lightness by factor 0.5: $l \leftarrow L_{\text{hit}} / 2$ [S2, unofficial].
        6. Blend colors. For each channel $c \in \{R, G, B\}$:
            $$ c_{\text{new}} = (c_{\text{old}} + c_{\text{hit}} \cdot l) \cdot 0.66 $$ [S2, unofficial]
            Update $C_{\text{out}}$ with these new channel values [S2, unofficial].
        7. Increment bounce counter: $i \leftarrow i + 1$ [S2, unofficial].
    5. Else (ray did not hit within distance):
        1. Set $\text{not-done?} \leftarrow \text{false}$ [S2, unofficial].
3. Return $C_{\text{out}}$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simple_reflections(color, point, direction, scene, light, reflection_count=3, draw_distance=100.0):
    """
    Computes simple iterative specular reflections.
    
    Args:
        color: tuple (r, g, b) base surface color.
        point: vec3 hit point.
        direction: vec3 incoming view direction.
        scene: object with get_normal(p) and march_ray(p, d) methods.
        light: object with point_lightness(p, scene) method.
        reflection_count: int max bounces.
        draw_distance: float max raymarch distance.
        
    Returns:
        tuple (r, g, b) final color.
    """
    # Initialize state
    p = list(point)
    d = list(direction)
    n = scene.get_normal(p)
    i = 0
    not_done = True
    out_color = list(color)
    
    while i < reflection_count and not_done:
        # 1. Compute reflection vector r = d - 2(d . n)n
        dot_dn = sum(d[j] * n[j] for j in range(3))
        r = [d[j] - 2 * dot_dn * n[j] for j in range(3)]
        
        # 2. Raymarch from above surface point
        epsilon = 1e-4
        start_point = [p[j] + n[j] * epsilon for j in range(3)]
        
        # 3. March ray
        distance, hit_color = scene.march_ray(start_point, r)
        
        if distance < draw_distance:
            # 4. Hit found
            p = [p[j] + r[j] * distance for j in range(3)]
            n = scene.get_normal(p)
            d = r
            
            # Compute lightness and attenuate
            lightness = light.point_lightness(p, scene)
            l = lightness / 2.0
            
            # Blend colors
            new_color = []
            for j in range(3):
                c_old = out_color[j]
                c_hit = hit_color[j]
                c_new = (c_old + c_hit * l) * 0.66
                new_color.append(c_new)
            out_color = new_color
            
            i += 1
        else:
            not_done = False
            
    return tuple(out_color)
```

**Parameters:**
*   `reflection-count`: Maximum number of reflection bounces allowed. Default value in code is 3 [S2, unofficial].
*   `0.66`: Color blending factor applied after adding hit color to prevent overflow and maintain brightness across bounces [S2, unofficial].
*   `0.5`: Light attenuation factor applied to reflection bounces to make them appear darker [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Infinite Recursion:** Prevented by checking `i < reflection-count` and `not-done?` [S2, unofficial].
*   **Self-Intersection:** The raymarch starts from `above-surface-point` (offset by normal) to prevent the ray from immediately hitting the surface it just reflected off [S2, unofficial].
*   **Language Limitations:** In languages without a `break` statement (e.g., Fennel), the loop counter must be incremented manually at the end of each iteration, and `not-done?` must be used to exit early [S2, unofficial].
*   **Color Overflow:** The multiplication by 0.66 ensures that the sum of base and reflected colors does not exceed 1.0, preventing clipping artifacts [S2, unofficial].

## Subtopics

### Cube Map Mapping
Maps reflected directions to cube map texture coordinates following OpenGL specifications [S3, unofficial]. It computes reflected directions using `reflected_directions = 2 * surface_normals * sum(view_directions * surface_normals, dim=-1, keepdim=True) - view_directions` [S3, unofficial]. It finds the max absolute component index and value to determine the cube face, then computes texture coordinates by dividing the reflected direction by the max absolute value [S3, unofficial]. Linear mapping is used for the third coordinate, and grid coordinates are constructed for each face based on the max index and sign, stacking (u, v, face_index) where face_index is mapped from 0-5 to -1 to 1 [S3, unofficial]. The cube map is sampled using `nn.functional.grid_sample` and the output is permuted [S3, unofficial]. This variant is chosen when environment mapping is preferred over iterative raymarching for reflections [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(k \cdot m)$, where $k$ is the number of reflection bounces (default 3) and $m$ is the number of steps per raymarch [S2, unofficial].
*   **Memory Complexity:** $O(1)$ per pixel, as it only stores current state variables [S2, unofficial].
*   **Performance:** Fast for low bounce counts. Performance degrades linearly with increased `reflection-count` [S2, unofficial].
*   **Strengths:** Simple to implement, avoids environment maps by computing reflections via direct raymarching [S1, S2, unofficial].
*   **Limitations:** Approximate, limited by `DRAW-DISTANCE` [S2, unofficial]. Not suitable for complex inter-reflections beyond the fixed bounce count [S2, unofficial].
*   **When to use:** Real-time raymarched scenes with mirror-like surfaces [S2, unofficial].
*   **When not to use:** High-fidelity rendering requiring accurate global illumination or infinite bounces [S2, unofficial].

## Background
Originated in the context of raymarching signed distance fields (SDFs) for real-time graphics [S1, S2, unofficial]. The technique was popularized in blog posts and tutorials demonstrating SDF rendering with simple lighting models [S1, S2, unofficial]. It is a foundational technique for adding specular highlights to raymarched scenes without resorting to recursive ray tracing [S1, S2, unofficial].

## Sources
[S1] Raymarching Distance Fields (2013), http://9bitscience.blogspot.com/2013/07/raymarching-distance-fields_14.html, fetched 2026-10-26 (unofficial)
[S2] Raymarching with Fennel and LÖVE · Andrey Listopadov, https://andreyor.st/posts/2020-10-15-raymarching-with-fennel-and-love/, fetched 2026-10-30 (unofficial)
[S3] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
