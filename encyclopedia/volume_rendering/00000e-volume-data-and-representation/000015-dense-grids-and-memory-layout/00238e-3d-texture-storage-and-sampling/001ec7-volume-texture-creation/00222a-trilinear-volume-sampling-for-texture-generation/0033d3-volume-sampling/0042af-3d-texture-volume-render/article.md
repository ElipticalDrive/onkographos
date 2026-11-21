# 3D Texture Volume Render

Also called: 3D Texture Volume Render node

## Overview
The 3D Texture Volume Render node renders volumetric textures from 3D data to create cloud and fog effects by integrating light through a volume defined by a 3D texture and its corresponding signed distance field (SDF) [S1]. It differs from standard ray marching on raw voxel grids by operating on a specific compressed texture layout (16x16 grid of 256 slices) and providing built-in lighting models (directional + hemispherical skylight) and post-processing features like dithering and ground planes [S1]. It builds on [Volume Sampling](7111-volume-sampling.md) for density retrieval [G].

## How it works
### Inputs and Outputs
**Inputs:**
1.  **3D Signed Distance Field Grayscale**: A 4096x4096 grayscale texture representing the SDF of the shape [S1].
2.  **Density Grayscale**: A 4096x4096 grayscale texture representing the density of the shape [S1].

**Output:**
*   **Rendered Image**: An image of resolution `Output Resolution` x `Output Resolution` [S1].

### Data Structures
The input textures follow a specific layout defined by Adobe Substance 3D Designer [S1]:
*   **Dimensions**: 4096 pixels width x 4096 pixels height [S1].
*   **Logical Structure**: Represents 256 slices of 3D data [S1].
*   **Grid Layout**: The 256 slices are arranged in a 16x16 grid within the 2D texture [S1]. Each slice is a 256x256 pixel square ($4096 / 16 = 256$) [S1]. Slices are indexed $i \in [0, 255]$. Slice $i$ is located at grid coordinates $(gx, gy)$ where $gx = i \pmod{16}$ and $gy = \lfloor i / 16 \rfloor$ [S1]. Pixel coordinates $(u, v)$ for a specific slice $i$ and local coordinate $(sx, sy)$ within the slice are $u = gx \times 256 + sx$ and $v = gy \times 256 + sy$ [S1].
*   **Value Range**: SDF contains signed distance values [S1]. Density is mapped from 0 (entirely transparent) to 1 (entirely opaque) [S1].

### Parameters
*   **Output Resolution** (Integer): Resolution of the output image in X and Y. Must be a power of two [S1].
*   **Camera Position** (Float2): Position of the camera around the shape [S1].
*   **Light Position** (Float2): Position of the directional light around the shape [S1].
*   **Camera Distance** (Float): Distance from the camera to the shape [S1].
*   **Camera FOV** (Float): Field of view of the camera in degrees [S1].
*   **Absorption** (Float): Adjusts how much light is absorbed as it passes through the volume [S1].
*   **Feather** (Float): Multiplies the Density input with the inner distance field value to adjust the fading gradient width from the volume's outer limit inwards [S1].
*   **Light Color Mode** (Integer): `0` for Temperature (Kelvin), `1` for RGB Color [S1].
*   **Light Temperature** (Float): Light temperature in Kelvin. Useful values: 1800 K (Candle), 2800 K (Incandescent), 5500 K (Daylight), 6200 K (Natural white), 7000 K (Overcast sky) [S1].
*   **Light Color** (Float3): RGB color of the directional light [S1].
*   **Light Intensity** (Float): Intensity of the directional light [S1].
*   **Ambient Color** (Float3): Color of the ambient skylight [S1].
*   **Ambient Intensity** (Float): Intensity of the ambient skylight [S1].
*   **Albedo** (Float3): Albedo color of the volume [S1].
*   **Background Mode** (Integer): `0` for Shaded, `1` for Constant Color [S1].
*   **Background Color** (Float4): Color for the background [S1].
*   **Dithering** (Float): Intensity of blue noise dithering to smooth shading [S1].
*   **Enable Ground Plane** (Boolean): Renders an infinite ground plane if True [S1].
*   **Infinite Plane** (Boolean): Extends ground plane to horizon if True (requires Enable Ground Plane) [S1].
*   **Ground Plane Size** (Float2): Size of the ground plane if Infinite Plane is False (requires Enable Ground Plane) [S1].

### Algorithm
The volume is represented within the bounds of a unit cube [S1]. The rendering process involves casting rays from the camera through the unit cube, sampling the density and SDF, computing transmittance and color, and applying lighting [G].

1.  **Initialize Camera**: Construct the camera projection matrix using `Camera FOV`, `Camera Distance`, and `Output Resolution` [G]. Define the camera origin and direction vectors based on `Camera Position` and `Camera Distance` [S1].
2.  **Light Color Calculation**: If `Light Color Mode` is Temperature: Convert `Light Temperature` to RGB color $C_{light}$ using a blackbody radiation approximation [S1]. If `Light Color Mode` is RGB: Set $C_{light} = \text{Light Color}$ [S1]. Calculate directional light vector $L$ from `Light Position` [S1].
3.  **Background Calculation**: If `Background Mode` is Constant: $C_{bg} = \text{Background Color}$ [S1]. If `Background Mode` is Shaded: Calculate $C_{bg}$ based on `Background Color`, `Light Color`, and `Light Intensity` [S1].
4.  **Ray Casting Loop**: For each pixel $(x, y)$ in the output image: Compute ray origin $O$ and direction $D$ in world space [G]. Intersect ray with the unit cube $[-0.5, 0.5]^3$ [G]. Let the entry point be $t_{min}$ and exit point be $t_{max}$ [G]. Initialize accumulated color $C_{acc} = (0, 0, 0)$ and transmittance $T = 1.0$ [G]. Define step size $\Delta t$ based on `Output Resolution` and `Camera Distance` [G]. Iterate $t$ from $t_{min}$ to $t_{max}$ with step $\Delta t$: Compute world position $P = O + t \cdot D$ [G]. Normalize $P$ to texture coordinates assuming unit cube bounds $[0,1]$ [G]. Determine slice index $i$ from the normalized Z-coordinate [G]. Sample the **Density Grayscale** texture at $(u, v)$ corresponding to slice $i$ and XY coordinates. Let raw density be $d_{raw}$ [S1]. Sample the **SDF** texture at $(u, v)$ for slice $i$. Let SDF value be $s$ [S1]. Apply **Feather**: $d = d_{raw} \times \max(0, s) \times \text{Feather}$ [S1]. Calculate optical depth increment: $\Delta \tau = d \cdot \Delta t \cdot \text{Absorption}$ [G]. Update transmittance: $T = T \cdot e^{-\Delta \tau}$ [G]. Calculate diffuse contribution from directional light: $C_{dir} = C_{light} \cdot \text{Light Intensity} \cdot \text{Albedo} \cdot \max(0, N \cdot L)$ [G]. Calculate ambient contribution: $C_{amb} = \text{Ambient Color} \cdot \text{Ambient Intensity} \cdot \text{Albedo}$ [S1]. Total light contribution $C_{light\_total} = C_{dir} + C_{amb}$ [G]. Calculate emitted color $C_{emit} = C_{light\_total} \cdot d$ [G]. Update accumulated color: $C_{acc} = C_{acc} + C_{emit} \cdot T \cdot \Delta t$ [G]. If $T < \epsilon$ (e.g., $10^{-4}$), break loop [G]. Final Color: $C_{final} = C_{acc} + C_{bg} \cdot T$ [G]. Apply **Dithering**: Add blue noise scaled by `Dithering` to $C_{final}$ to smooth shading [S1]. Set pixel $(x, y)$ to $C_{final}$ [S1].
5.  **Ground Plane**: If `Enable Ground Plane` is True: Intersect ray with the ground plane (z = -0.5 or similar, depending on unit cube placement) [G]. If intersection exists and is closer than the volume exit, replace $C_{final}$ with the ground plane color (shaded or constant based on `Background Mode` and `Ground Plane Size`/`Infinite Plane`) [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def add_vectors(a, b):
    return (a[0] + b[0], a[1] + b[1], a[2] + b[2])

def scale_vector(v, s):
    return (v[0] * s, v[1] * s, v[2] * s)

def normalize(v):
    length = math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)
    if length == 0:
        return (0.0, 0.0, 0.0)
    return (v[0] / length, v[1] / length, v[2] / length)

def intersect_unit_cube(origin, direction):
    t_min = -math.inf
    t_max = math.inf
    for i in range(3):
        if direction[i] == 0:
            if origin[i] < -0.5 or origin[i] > 0.5:
                return (-math.inf, -math.inf)
        else:
            t1 = (-0.5 - origin[i]) / direction[i]
            t2 = (0.5 - origin[i]) / direction[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    return (t_min, t_max)

def blue_noise(x, y):
    h = x * 374761393.0 + y * 668265263.0
    h = (h & 0x7fffffff) / 2147483647.0
    return h - 0.5

def sample_3d_texture(texture, slice_idx, u, v):
    slice_size = 256
    grid_size = 16
    gx = slice_idx % grid_size
    gy = slice_idx // grid_size
    px = gx * slice_size + u
    py = gy * slice_size + v
    px = max(0, min(px, 4095))
    py = max(0, min(py, 4095))
    return texture[py][px]

def blackbody_to_rgb(temp_k):
    temp_k = max(1000, min(40000, temp_k))
    temp_c = temp_k / 100.0
    if temp_c <= 66:
        r = 255.0
        g = 99.4708025861 * math.log(temp_c) - 161.1195681661
        b = 138.5177312231 * math.log(temp_c - 10.0) - 305.0447927307
    else:
        r = 329.698727446 * math.pow(temp_c - 60.0, -0.1332047592)
        g = 288.1221695283 * math.pow(temp_c - 60.0, -0.0755146169)
        b = 255.0
    return (r / 255.0, g / 255.0, b / 255.0)

def render_3d_texture_volume(sdf_texture, density_texture, params):
    output_res = params['output_resolution']
    camera_pos = params['camera_position']
    camera_dist = params['camera_distance']
    camera_fov = params['camera_fov']
    absorption = params['absorption']
    feather = params['feather']
    light_color_mode = params['light_color_mode']
    light_temp = params['light_temperature']
    light_color_rgb = params['light_color']
    light_intensity = params['light_intensity']
    light_pos = params['light_position']
    ambient_color = params['ambient_color']
    ambient_intensity = params['ambient_intensity']
    albedo = params['albedo']
    bg_mode = params['background_mode']
    bg_color = params['background_color']
    dithering = params['dithering']
    enable_ground = params['enable_ground_plane']
    infinite_plane = params['infinite_plane']
    ground_size = params['ground_plane_size']

    if light_color_mode == 0:
        light_color = blackbody_to_rgb(light_temp)
    else:
        light_color = light_color_rgb

    light_dir = normalize(add_vectors(light_pos, scale_vector(camera_pos, -1.0)))

    if bg_mode == 1:
        bg_color_final = bg_color
    else:
        bg_color_final = (bg_color[0] * light_color[0] * light_intensity,
                          bg_color[1] * light_color[1] * light_intensity,
                          bg_color[2] * light_color[2] * light_intensity,
                          bg_color[3])

    output_image = [[0.0] * 4 for _ in range(output_res)]

    aspect = 1.0
    fov_rad = math.radians(camera_fov)
    h = math.tan(fov_rad / 2.0) * camera_dist
    w = h * aspect

    for y in range(output_res):
        for x in range(output_res):
            ndc_x = (x / output_res - 0.5) * 2.0
            ndc_y = (y / output_res - 0.5) * 2.0
            ray_dir = normalize((ndc_x * w, ndc_y * h, -camera_dist))
            ray_origin = camera_pos

            t_min, t_max = intersect_unit_cube(ray_origin, ray_dir)

            if t_max < 0:
                output_image[y][x] = list(bg_color_final)
                continue

            t = t_min
            acc_color = [0.0, 0.0, 0.0]
            transmittance = 1.0
            step_size = (t_max - t_min) / output_res

            while t < t_max and transmittance > 1e-4:
                pos = add_vectors(ray_origin, scale_vector(ray_dir, t))
                norm_pos = add_vectors(pos, (0.5, 0.5, 0.5))

                slice_idx = int(norm_pos[2] * 256)
                slice_idx = max(0, min(255, slice_idx))

                u = int(norm_pos[0] * 256)
                v = int(norm_pos[1] * 256)
                u = max(0, min(255, u))
                v = max(0, min(255, v))

                density_raw = sample_3d_texture(density_texture, slice_idx, u, v)
                sdf_val = sample_3d_texture(sdf_texture, slice_idx, u, v)

                density = density_raw * sdf_val * feather
                if density < 0:
                    density = 0.0

                delta_tau = density * absorption * step_size
                transmittance *= math.exp(-delta_tau)

                light_contrib = scale_vector(light_color, light_intensity)
                ambient_contrib = scale_vector(ambient_color, ambient_intensity)
                emitted = scale_vector(add_vectors(light_contrib, ambient_contrib), density)

                acc_color = add_vectors(acc_color, scale_vector(emitted, transmittance * step_size))

                t += step_size

            final_color = add_vectors(acc_color, scale_vector(bg_color_final[:3], transmittance))

            noise = blue_noise(x, y) * dithering
            final_color = add_vectors(final_color, (noise, noise, noise))

            output_image[y][x] = [final_color[0], final_color[1], final_color[2], 1.0]

    return output_image
```

### Edge Cases and Pitfalls
*   **SDF Input Format**: The SDF must be a 4096x4096 texture with 256 slices in a 16x16 grid. Using a different layout will result in incorrect sampling [S1].
*   **Density Range**: Density values are expected to be in [0, 1]. Values outside this range may cause unexpected transparency or opacity [S1].
*   **Feather Negative Values**: If the SDF value is negative (outside the shape) and Feather is positive, the density becomes negative. The algorithm must clamp the density to 0 to prevent negative absorption/emission [G].
*   **Camera Position**: The camera position is relative to the unit cube. Moving the camera too close or too far may result in the volume being out of frame or clipped [S1].
*   **Ground Plane**: The ground plane is only rendered if `Enable Ground Plane` is True. If `Infinite Plane` is False, `Ground Plane Size` must be set appropriately [S1].

## Operating characteristics
*   **Time Complexity**: $O(R^2 \cdot N)$, where $R$ is the output resolution and $N$ is the number of steps per ray. $N$ is determined by the volume depth and step size [G].
*   **Performance**: Performance depends heavily on the output resolution and the number of steps. Higher `Output Resolution` and smaller `Absorption` (requiring more steps for transmittance to drop) will increase render time [S1].
*   **Strengths**: Simple setup for volumetric effects like clouds and fog. Built-in lighting and post-processing (dithering) [S1].
*   **Limitations**: Fixed input texture format (4096x4096, 256 slices). Limited control over the volume integration algorithm compared to custom GLSL injection [S1].

## Background
The 3D Texture Volume Render node is part of the Adobe Substance 3D Designer node library, specifically in the Filter > Effect section [S1]. It is designed for creating volumetric textures from 3D data, such as cloud and fog effects, using a 3D texture and its corresponding signed distance field [S1]. The node was last updated on September 9, 2026 [S1].

## Sources
[S1] 3D Texture Volume Render | Adobe Substance 3D Designer, https://experienceleague.adobe.com/en/docs/substance-3d-designer/using/substance-graphs/nodes-reference-for-substance-graphs/node-library/filters/effects/3d-texture-volume-render, fetched 2026-10-27
