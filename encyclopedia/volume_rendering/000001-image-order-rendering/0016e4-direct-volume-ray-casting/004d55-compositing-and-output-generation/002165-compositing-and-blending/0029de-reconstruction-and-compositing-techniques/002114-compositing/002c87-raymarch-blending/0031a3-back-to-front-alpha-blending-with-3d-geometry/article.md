# Back-to-Front Alpha Blending with 3D Geometry

Also called: WebGL blending for volume-geometry intersection

## Overview
This method composites a ray-marched volume over pre-rendered opaque 3D polygonal geometry by leveraging standard graphics pipeline alpha blending [S1]. It solves the occlusion problem where volumetric data must correctly interact with solid meshes, ensuring that the volume is blended against the background color already present in the framebuffer from previous rendering passes [S1]. Unlike standard ray casting which only evaluates volume density, this approach ensures correct occlusion of the volume by solid geometry, solving the problem of rendering transparent volumes over opaque meshes in a specific pipeline configuration [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Volume texture (3D scalar field) [S1].
    *   Depth textures (RGBA packed) from previous passes rendering the 3D geometry [S1].
    *   Color buffer containing the rendered 3D geometry from previous passes [S1].
    *   Ray origin and direction (derived from camera and screen coordinates) [G].
*   **Outputs:**
    *   Final color and alpha for each pixel, composited into the screen buffer [S1].

**Data Structures:**
*   **Depth Texture:** Since WebGL 1.0 lacks native float depth texture support on all devices, depth information from passes 1 and 3 is packed into RGBA textures [S1]. The fragment shader must unpack this depth value to determine if a ray intersects a 3D object at a specific step [S1].
*   **Frame Buffer Objects (FBO):** Used to store the intermediate depth and color data from the geometry passes [S1].

**Parameters:**
*   **Blend Function:** `gl.SRC_ALPHA` for source and `gl.ONE_MINUS_SRC_ALPHA` for destination [S1].
*   **Early Ray Termination Threshold:** Accumulated alpha equal to 1.0 (opaque) [S1].

**Steps:**
1.  **Geometry Passes (Passes 1 and 3):** Render the opaque 3D polygonal objects and store the depth information in RGBA textures [S1].
2.  **Volume Ray Casting (Pass 5):** Execute the ray-casting fragment shader [S1].
3.  **Depth Unpacking:** In the ray-casting shader, unpack the depth value from the RGBA texture sampled at the current pixel coordinate [S1].
4.  **Ray Traversal:** Traverse the volume along the ray [S1].
5.  **Accumulation:** Accumulate volume color and alpha along the ray [S1].
6.  **Early Ray Termination:** If the accumulated alpha reaches 1.0, terminate the ray traversal early [S1].
7.  **Blending:** Output the accumulated volume color and alpha [S1]. The graphics pipeline applies the blend function to composite this result with the existing color in the framebuffer (which contains the 3D geometry color) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard vector constructors
def vec3(x, y, z):
    return (x, y, z)
def vec4(x, y, z, w):
    return (x, y, z, w)

# Helper: Unpack RGBA depth to float (standard packing scheme)
def unpack_depth(rgba):
    return rgba[0] + rgba[1] * 256.0 + rgba[2] * 65536.0 + rgba[3] * 16777216.0

# Helper: Sample 3D volume texture
def sample_volume(tex, pos):
    return tex.trilinear_sample(pos)

# Helper: Sample depth texture
def sample_depth(tex, uv):
    return tex.sample(uv)

def ray_cast_fragment_shader(depth_texture, volume_texture, ray_origin, ray_direction, step_size, max_distance, resolution):
    # 1. Unpack depth from previous geometry passes
    uv = (gl_FragCoord[0] / resolution[0], gl_FragCoord[1] / resolution[1])
    depth_rgba = sample_depth(depth_texture, uv)
    depth_value = unpack_depth(depth_rgba)
    
    # 2. Initialize accumulation
    acc_color = vec3(0.0, 0.0, 0.0)
    acc_alpha = 0.0
    
    # 3. Ray traversal
    t = 0.0
    while t < max_distance:
        sample_pos = vec3(ray_origin[0] + ray_direction[0] * t,
                          ray_origin[1] + ray_direction[1] * t,
                          ray_origin[2] + ray_direction[2] * t)
        density = sample_volume(volume_texture, sample_pos)
        
        # Convert density to step alpha
        step_alpha = density * step_size
        
        # 4. Early Ray Termination
        if acc_alpha >= 1.0:
            break
            
        # 5. Accumulate color and alpha (Front-to-Back)
        weight = step_alpha * (1.0 - acc_alpha)
        sample_color = vec3(density, density, density) # Simplified transfer function
        acc_color = vec3(acc_color[0] + weight * sample_color[0],
                         acc_color[1] + weight * sample_color[1],
                         acc_color[2] + weight * sample_color[2])
        acc_alpha += weight
        
        t += step_size
        
    # 6. Output for blending
    # The pipeline applies: gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)
    return vec4(acc_color[0], acc_color[1], acc_color[2], acc_alpha)
```

**Edge Cases and Pitfalls:**
*   **Transparent Geometry:** The base implementation assumes opaque 3D objects. Transparent 3D polygonal objects are not supported [S1].
*   **Depth Precision:** Packing depth into RGBA may introduce precision loss compared to native float depth textures [S1].
*   **WebGL Compatibility:** Native depth texture support is not universal in WebGL 1.0, necessitating the RGBA packing fallback [S1].

## Subtopics

### [Ray-Casting with 3D Geometry Intersection](6576-ray-casting-with-3d-geometry-intersection.md)
This method extends standard direct volume ray casting by integrating pre-rendered 3D polygonal geometry depth data, allowing volumetric rays to detect intersections with opaque solid objects and terminate or blend correctly. Unlike standard ray casting which only evaluates volume density, this approach ensures correct occlusion of the volume by solid geometry, solving the problem of rendering transparent volumes over opaque meshes in a single pass or specific pipeline configuration.

## Operating characteristics
*   **Memory Complexity:** Requires storage for the volume texture and the depth textures from previous passes [S1].
*   **Strengths:** Correctly handles occlusion of volume by opaque geometry; leverages standard hardware blending [S1].
*   **Limitations:** Does not support transparent 3D geometry in the base implementation; dependent on WebGL version and extensions for depth texture support [S1].
*   **When to use:** When rendering volumetric data over opaque 3D meshes in a WebGL environment where standard blending is desired [S1].
*   **When not to use:** When transparent 3D geometry is present in the scene [S1].

## Background
This method was described in the context of deploying volume rendering interactive visualizations in web platforms with intersected 3D geometry [S1]. It utilizes WebGL directives to enable blending and perform back-to-front composition [S1]. The approach was tested with opaque 3D objects [S1].

## Sources
[S1] Deployment of Volume Rendering Interactive Visualizations in Web Platforms With Intersected 3D Geometry, https://doi.org/10.2312/ceig.20161312, fetched 2026-10-26
