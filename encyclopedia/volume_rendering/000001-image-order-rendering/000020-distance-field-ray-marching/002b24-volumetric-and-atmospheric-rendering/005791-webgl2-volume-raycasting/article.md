# WebGL2 Volume Raycasting

Also called: VolRen, Volume Raycasting Fragment Shader (Colormap/Segmentation/Scalar), Web-based volumetric data explorer

## Overview
WebGL2 Volume Raycasting is a lightweight, web-based volumetric data explorer that renders 3D scalar fields using WebGL2 raymarching within a fragment shader [S1, unofficial]. It supports 8-bit, 16-bit, and float data types, with optional segmentation masks and clipping planes [S1, unofficial]. Unlike general-purpose SDF-based sphere tracing which targets implicit surfaces, this method performs front-to-back alpha blending over a discrete 3D texture grid to visualize scalar fields [S1, unofficial]. It is distinguished by its specific WebGL2 implementation details, including handling of integer texture formats for 16-bit data and a fixed 180x1 colormap texture [S1, unofficial].

## How it works
The method operates by casting rays from a camera through a 3D volume texture, sampling scalar values, and compositing them into a 2D image using alpha blending [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   **Volume Texture:** A 3D texture (`gl.TEXTURE0`) containing the scalar field. Dimensions are `volDims[0]`, `volDims[1]`, `volDims[2]` [S1, unofficial].
    *   **8-bit:** Format `R8`, Type `UNSIGNED_BYTE`, Filter `LINEAR` [S1, unofficial].
    *   **16-bit:** Format `R16UI`, Type `UNSIGNED_SHORT`, Filter `NEAREST` [S1, unofficial].
    *   **Float:** Format `R32F`, Type `FLOAT`, Filter `LINEAR` [S1, unofficial].
*   **Colormap Texture:** A 2D texture (`gl.TEXTURE1`) of size 180x1 pixels, format `RGBA8` [S1, unofficial].
*   **Segmentation Texture (Optional):** A 3D texture (`gl.TEXTURE2`) of dimensions `volDims`, format `R8`, Type `UNSIGNED_BYTE`, Filter `NEAREST`, Wrapping `CLAMP_TO_EDGE` [S1, unofficial].
*   **Uniforms:**
    *   `highp usampler3D volume`: The volume texture sampler [S1, unofficial].
    *   `highp sampler2D colormap`: The colormap texture sampler [S1, unofficial].
    *   `bool use_colormap`: Enables colormap mode [S1, unofficial].
    *   `bool use_seg`: Enables segmentation mode [S1, unofficial].
    *   `vec3 volume_scale`: Scaling factor for the volume [S1, unofficial].
    *   `vec3 volume_dims`: Dimensions of the volume [S1, unofficial].
    *   `float dt_scale`: Step size scaling factor (Default: 1.0) [S1, unofficial].
    *   `ivec2 comp`: Comparison coordinates for inversion logic [S1, unofficial].
    *   `vec3 colors[25]`: Array of colors for segmentation IDs (Note: Source code implies this is a global constant or defined elsewhere, as it is not declared as a uniform in the provided snippet) [S1, unofficial].
*   **Vertex Shader Inputs:**
    *   `pos`: Vertex position [S1, unofficial].
    *   `eye_pos`: Camera eye position in world space [S1, unofficial].

**Outputs:**
*   `out vec4 color`: The final fragment color [S1, unofficial].

### Data Structures
The volume is stored as a 3D texture. The specific internal format depends on the bit depth of the source data [S1, unofficial]. The colormap is a 1D lookup table represented as a 180x1 2D texture [S1, unofficial].

### Algorithm Steps

1.  **Vertex Shader Transformation:**
    *   Compute volume translation: `volume_translation = vec3(0.5) - volume_scale * 0.5` [S1, unofficial].
    *   Transform vertex position: `gl_Position = proj_view * vec4(pos * volume_scale + volume_translation, 1.0)` [S1, unofficial].
    *   Compute transformed eye position: `transformed_eye = (eye_pos - volume_translation) / volume_scale` [S1, unofficial].
    *   Compute ray direction in object space: `vray_dir = pos - transformed_eye` [S1, unofficial].

2.  **Fragment Shader Initialization:**
    *   Normalize ray direction: `ray_dir = normalize(vray_dir)` [S1, unofficial].
    *   Intersect ray with the volume bounding box defined by `box_min` and `box_max` using the slab method to obtain entry and exit distances `t_hit.x` (t0) and `t_hit.y` (t1) [S1, unofficial].
    *   If `t_hit.x > t_hit.y`, discard the fragment (ray misses the volume) [S1, unofficial].
    *   Clamp start distance: `t_hit.x = max(t_hit.x, 0.0)` [S1, unofficial].

3.  **Step Size Calculation:**
    *   Compute step vector components: `dt_vec = 1.0 / (vec3(volume_dims) * abs(ray_dir))` [S1, unofficial].
    *   Compute step size: `dt = dt_scale * min(dt_vec.x, min(dt_vec.y, dt_vec.z))` [S1, unofficial].

4.  **Ray Marching Loop:**
    *   Initialize accumulated color: `color = vec4(0.0, 0.0, 0.0, 0.0)` [G].
    *   Initialize current position: `p = eye_pos_in_volume_space + t_hit.x * ray_dir` [G].
    *   Loop `t` from `t_hit.x` to `t_hit.y` with step `dt`:
        *   **Sample Volume:**
            *   Sample the volume texture at `p`: `val_raw = texture(volume, p).r` [S1, unofficial].
            *   **Normalization:**
                *   If data type is 8-bit or float: `val = val_raw` (already float or normalized) [S1, unofficial].
                *   If data type is 16-bit: `val = float(val_raw) / 65536.0` [S1, unofficial].
                *   If `p` is outside `[0.0, 1.0]` in any dimension, set `val = 0.0` [S1, unofficial].
        *   **Color/Alpha Computation:**
            *   Determine `val_color` based on mode:
                *   **Colormap Mode (`use_colormap == true`):**
                    *   `trans_sample = texture(colormap, vec2(val, 0.5))` [S1, unofficial].
                    *   `val_color.rgb = trans_sample.rgb` [S1, unofficial].
                    *   `val_color.a = val * trans_sample.a` [S1, unofficial].
                *   **Segmentation Mode (`use_seg == true`):**
                    *   Convert segmentation value to integer: `seg = int(texture(seg_volume, p).r)` [S1, unofficial].
                    *   Loop `k` from 0 to `min(seg, 24)`:
                        *   If `k == seg`, `val_color = vec4(colors[k].rgb, 1.0)` [S1, unofficial].
                    *   [G] If `seg` is out of bounds, default to transparent or black.
                *   **Scalar Mode (Default):**
                    *   `val_color.rgb = vec3(1.0)` (White) [S1, unofficial].
                    *   `val_color.a = val` [S1, unofficial].
            *   **Inversion Logic:**
                *   Check inversion condition: `invert = (comp.x == int(gl_FragCoord.x)) || (comp.y == int(gl_FragCoord.y))` [S1, unofficial].
                *   If `invert` is true:
                    *   Colormap Mode: `val_color = vec4(1.0, 1.0, 1.0, 1.0)` [S1, unofficial].
                    *   Segmentation Mode: `val_color = vec4(1.0, 1.0, 1.0, 1.0)` [S1, unofficial].
                    *   Scalar Mode: `val_color.a = 1.0 - val` [S1, unofficial].
        *   **Opacity Correction:**
            *   Apply Beer-Lambert law approximation for step size: `val_color.a = 1.0 - pow(1.0 - val_color.a, dt_scale)` [S1, unofficial].
        *   **Alpha Blending:**
            *   `color.rgb += (1.0 - color.a) * val_color.a * val_color.rgb` [S1, unofficial].
            *   `color.a += (1.0 - color.a) * val_color.a` [S1, unofficial].
        *   **Early Termination:**
            *   If `color.a >= 0.95`, break loop [S1, unofficial].
        *   **Advance Ray:**
            *   `p += ray_dir * dt` [S1, unofficial].

5.  **Final Output:**
    *   Set `gl_FragColor = color` [G].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
uniform mat4 proj_view;
uniform vec3 volume_scale;
uniform vec3 eye_pos;
in vec3 pos;

void main() {
    vec3 volume_translation = vec3(0.5) - volume_scale * 0.5;
    gl_Position = proj_view * vec4(pos * volume_scale + volume_translation, 1.0);
    
    // Pass transformed eye and ray dir to fragment shader
    vec3 transformed_eye = (eye_pos - volume_translation) / volume_scale;
    vec3 vray_dir = pos - transformed_eye;
    gl_FragColor = vec4(0.0); // Placeholder, actual color computed in FS
    // Note: In WebGL, varyings are used. 
    // out vec3 v_ray_dir;
    // out vec3 v_transformed_eye;
    // v_ray_dir = vray_dir;
    // v_transformed_eye = transformed_eye;
}

# Fragment Shader
uniform highp usampler3D volume;
uniform highp sampler2D colormap;
uniform bool use_colormap;
uniform bool use_seg;
uniform vec3 volume_dims;
uniform float dt_scale;
uniform ivec2 comp;
uniform vec3 colors[25]; // Assumed global or uniform

in vec3 v_ray_dir;
in vec3 v_transformed_eye;
out vec4 color;

void main() {
    vec3 ray_dir = normalize(v_ray_dir);
    
    // Slab intersection with box [0,1]^3 (assuming normalized coords)
    // Simplified slab method for unit cube
    vec3 t0 = (vec3(0.0) - v_transformed_eye) / ray_dir;
    vec3 t1 = (vec3(1.0) - v_transformed_eye) / ray_dir;
    vec3 t_min = min(t0, t1);
    vec3 t_max = max(t0, t1);
    float t_hit_x = max(max(t_min.x, t_min.y), t_min.z);
    float t_hit_y = min(min(t_max.x, t_max.y), t_max.z);
    
    if (t_hit_x > t_hit_y) {
        discard;
    }
    
    t_hit_x = max(t_hit_x, 0.0);
    
    vec3 dt_vec = 1.0 / (volume_dims * abs(ray_dir));
    float dt = dt_scale * min(dt_vec.x, min(dt_vec.y, dt_vec.z));
    
    vec3 p = v_transformed_eye + t_hit_x * ray_dir;
    vec4 acc_color = vec4(0.0, 0.0, 0.0, 0.0);
    
    // Inversion check
    bool invert = (comp.x == int(gl_FragCoord.x)) || (comp.y == int(gl_FragCoord.y));
    
    for (float t = t_hit_x; t < t_hit_y; t += dt) {
        // Sample volume
        float val_raw = texture(volume, p).r;
        float val;
        
        // Normalization based on texture type (handled by sampler type in GLSL)
        // For R16UI, val_raw is uint, needs conversion
        // For R8/Float, val_raw is float in [0,1] or normalized
        // Assuming GLSL handles basic type conversion, but explicit division for 16-bit:
        // Note: In actual GLSL, usampler3D returns uint. 
        // val = float(val_raw) / 65536.0; // Only if 16-bit
        
        // Boundary check
        if (p.x < 0.0 || p.x > 1.0 || p.y < 0.0 || p.y > 1.0 || p.z < 0.0 || p.z > 1.0) {
            val = 0.0;
        } else {
            val = val_raw; // Simplified, actual normalization depends on texType
        }
        
        vec4 val_color;
        
        if (use_colormap) {
            vec4 trans_sample = texture(colormap, vec2(val, 0.5));
            val_color.rgb = trans_sample.rgb;
            val_color.a = val * trans_sample.a;
        } else if (use_seg) {
            // Segmentation logic
            // int seg = int(texture(seg_volume, p).r); // seg_volume not passed in uniform list
            // Loop k from 0 to min(seg, 24)
            // if (k == seg) val_color = vec4(colors[k], 1.0);
            // Placeholder for missing seg_volume uniform
            val_color = vec4(0.0, 0.0, 0.0, 0.0); 
        } else {
            val_color.rgb = vec3(1.0);
            val_color.a = val;
        }
        
        if (invert) {
            if (use_colormap || use_seg) {
                val_color = vec4(1.0, 1.0, 1.0, 1.0);
            } else {
                val_color.a = 1.0 - val;
            }
        }
        
        // Opacity correction
        val_color.a = 1.0 - pow(1.0 - val_color.a, dt_scale);
        
        // Alpha blending
        acc_color.rgb += (1.0 - acc_color.a) * val_color.a * val_color.rgb;
        acc_color.a += (1.0 - acc_color.a) * val_color.a;
        
        if (acc_color.a >= 0.95) {
            break;
        }
        
        p += ray_dir * dt;
    }
    
    color = acc_color;
}
```

### Parameters
*   `dt_scale`: Controls step size scaling. Default is 1.0. Lower values increase sampling density and quality but reduce performance [S1, unofficial].
*   `targetFrameTime`: JavaScript variable set to 32 ms to manage frame rate [S1, unofficial].

### Edge Cases and Pitfalls
*   **Non-uniform Volumes:** The vertex shader transformation assumes a specific centering logic. Non-uniform size volumes may require different transformation to center properly, as noted in the source's TODO comment [S1, unofficial].
*   **Offset Initialization:** The initialization of the ray start position affects visual quality. "With offset is bad for flat surfaces, without has wood grain" [S1, unofficial].
*   **Missing Uniforms:** The `colors` array used in segmentation mode is not declared as a uniform in the provided shader code, implying it is a global constant or defined elsewhere [S1, unofficial]. The segmentation texture sampler is also not explicitly passed as a uniform in the provided list, though logic exists [S1, unofficial].
*   **Filtering:** LINEAR filtering is used for 8-bit and float volumes, while NEAREST filtering is used for 16-bit volumes, likely due to integer texture limitations [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of steps along the ray. The number of steps is determined by the volume dimensions, ray direction, and `dt_scale` [S1, unofficial].
*   **Memory Complexity:** Dominated by the storage of the 3D volume texture and the 2D colormap texture [S1, unofficial].
*   **Performance:** Performance is heavily dependent on the volume resolution and the `dt_scale` parameter. WebGL2 allows for efficient texture sampling, but fragment shader execution time can be high for large volumes or small step sizes [S1, unofficial].
*   **Strengths:** Lightweight, web-based, supports multiple data types (8-bit, 16-bit, float), supports segmentation masks [S1, unofficial].
*   **Limitations:** Limited to scalar field visualization, no advanced lighting models (default is white scalar), potential precision issues with 16-bit integer textures, missing uniform declarations in source code [S1, unofficial].

## Background
The method is implemented in "VolRen: A lightweight web-based volumetric data explorer" by johanna-b [S1, unofficial]. It is designed to be a simple, accessible tool for visualizing 3D scalar fields in a web browser using WebGL2 [S1, unofficial].

## Sources
[S1] johanna-b/VisWeb, https://github.com/johanna-b/VisWeb, fetched 2026-10-27 (unofficial)
