# Point Cloud Volumetric Rendering via 2D Atlas

Also called: Point Cloud Volume, Volumetric Viewer

## Overview
A real-time GPU-accelerated technique for rendering 3D volumetric data (specifically point clouds or scalar fields) by storing the volume in a 2D texture atlas and rendering it as a point cloud where each point represents a voxel/sample [S1, unofficial]. The depth of each sample is encoded in the atlas layout to allow correct sorting and rendering order without traditional depth buffering for the volume itself [S1, unofficial]. It achieves 60Hz on GTX1080 for a 4k atlas, while Intel 630 graphics render a 2k atlas at ~20fps [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D scalar field (volume) represented as a sequence of 2D slices (images) or a mathematical function [S1, unofficial].
- **Output:** A rendered point cloud visualization of the volume, where points are colored based on scalar value and positioned in 3D space [S1, unofficial].

**Data Structures:**
- **Atlas Texture:** A 2D texture (`GL_TEXTURE_2D`) with `GL_RGBA` format and `GL_NEAREST` filtering [S1, unofficial].
- **Atlas Layout:** The 3D volume is tiled into a 2D grid [S1, unofficial]. The atlas dimensions are defined by constants: `SAMPLEWIDTH` (2048) and `WIDTH` (128) [S1, unofficial]. The atlas is tiled into a grid of `TILING` x `TILING` blocks, where `TILING` is calculated as `int(math.sqrt(len(volume)))` or hardcoded as 16 [S1, unofficial].
- **Depth LUT:** A lookup table (`int depthLUT[TILING][TILING]`) mapping tile coordinates to their linear depth index in the volume [S1, unofficial].

**Parameters:**
- `cutoffThreshold`: A uniform float used as the minimum value for the RGB channels of the atlas sample to be considered part of the volume [S1, unofficial]. Default initial value is `0.05 * 7 = 0.35` [S1, unofficial]. Can be adjusted with keys 'R' and 'T' [S1, unofficial].
- `pointSize`: A uniform float controlling the size of the rendered points [S1, unofficial]. Default initial value is `1.0` [S1, unofficial]. Can be adjusted with keys 'Y' and 'H' [S1, unofficial].
- `minVal` and `maxVal`: Uniform floats defining the range for color remapping [S1, unofficial]. `maxVal` is `np.max(img) / 256.0` [S1, unofficial]. `minVal` is either `cutoff` (if `minValAbsolute` is false) or `np.min(img) / 256.0` (if `minValAbsolute` is true) [S1, unofficial].

**Steps:**
1. **Generate the Atlas:** Convert the 3D volume data into a 2D texture atlas [S1, unofficial]. If using an image sequence, read files sorted by name, create a tiling grid, and save the result [S1, unofficial]. The viewer reads the data from bottom to up, left to right [S1, unofficial]. The generated atlas must be rotated -90 degrees before use [S1, unofficial].
2. **Load Atlas:** Load the atlas image (e.g., `brainatlas2k.png`) and upload it to a 2D texture (`GL_TEXTURE_2D`) with `GL_RGBA` format and `GL_NEAREST` filtering [S1, unofficial].
3. **Vertex Shader Processing:** For each vertex (representing a pixel in the atlas), compute the 3D position [S1, unofficial].
    - Calculate sample coordinate `sCoord` from `gl_VertexID` [S1, unofficial].
    - Calculate tile coordinate `tCoord` and volume coordinate `vCoord` [S1, unofficial].
    - Sample the atlas texture at `sCoord` [S1, unofficial].
    - If the sample values (RGB) exceed the `THRESHOLD` (cutoffThreshold), return a 3D position [S1, unofficial]:
      $$x = (u \mod 128 - 64) \times 2$$
      $$y = (v \mod 128 - 64) \times 2$$
      $$z = \text{depth}(\lfloor u/128 \rfloor, \lfloor v/128 \rfloor) - 128$$
      where $\text{depth}(x, y)$ is the linear index of the tile $(x, y)$ in the atlas grid [S1, unofficial].
4. **Color Remapping:** In the vertex shader, if the sample is above threshold, remap its color [S1, unofficial]. The grayscale value `s` (from `sColor.x`) is divided into three thirds based on `minVal` and `maxVal` [S1, unofficial]. If `s` is in the first third, color is `(s, 0, 0, alpha)` [S1, unofficial]. If in the second third, `(0, s, 0, alpha)` [S1, unofficial]. If in the third third, `(0, 0, s, alpha)` [S1, unofficial]. Otherwise, white [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
SAMPLEWIDTH = 2048
WIDTH = 128
TILING = 16

# Uniforms (computed from source image data `img_data`)
maxVal = max(img_data) / 256.0
minVal = min(img_data) / 256.0  # or cutoffThreshold if minValAbsolute is false
cutoffThreshold = 0.35
pointSize = 1.0

# Depth LUT Initialization
depthLUT = [[0] * TILING for _ in range(TILING)]
current_depth = 0
for x in range(TILING):
    for y in range(TILING):
        depthLUT[x][y] = current_depth
        current_depth += 1

def get_depth(tile_x, tile_y):
    return depthLUT[tile_x][tile_y]

def get_sample_coordinate(vertex_id):
    x = vertex_id % SAMPLEWIDTH
    y = vertex_id // SAMPLEWIDTH
    return (float(x), float(y))

def get_volume_coordinate(s_coord):
    x = int(s_coord[0]) % WIDTH
    y = int(s_coord[1]) % WIDTH
    return (float(x), float(y))

def get_tile(sample_coord):
    x = int(sample_coord[0])
    y = int(sample_coord[1])
    tile_x = x // WIDTH
    tile_y = y // WIDTH
    return (tile_x, tile_y)

def remap_color(s_color, min_val, max_val):
    s = s_color[0]
    alpha = s_color[3]
    third = (max_val - min_val) / 3.0
    fst_third = min_val + third
    snd_third = min_val + 2 * third
    if s < fst_third:
        return (s, 0.0, 0.0, alpha)
    elif s < snd_third:
        return (0.0, s, 0.0, alpha)
    elif s <= max_val:
        return (0.0, 0.0, s, alpha)
    else:
        return (1.0, 1.0, 1.0, alpha)  # white

def main_render_loop(atlas_array):
    # For each vertex in the atlas (total pixels = SAMPLEWIDTH * SAMPLEWIDTH)
    for vertex_id in range(SAMPLEWIDTH * SAMPLEWIDTH):
        s_coord = get_sample_coordinate(vertex_id)
        v_coord = get_volume_coordinate(s_coord)
        t_coord = get_tile(s_coord)

        # Sample the atlas texture (direct array access for pseudocode)
        s_color = atlas_array[int(s_coord[1])][int(s_coord[0])]

        # Check threshold
        if s_color[0] > cutoffThreshold and s_color[1] > cutoffThreshold and s_color[2] > cutoffThreshold:
            # Compute 3D position
            x = (v_coord[0] - 64.0) * 2.0
            y = (v_coord[1] - 64.0) * 2.0
            z = float(get_depth(t_coord[0], t_coord[1])) - 128.0

            # Remap color
            final_color = remap_color(s_color, minVal, maxVal)

            # Emit point primitive
            output_point(x, y, z, final_color, pointSize)
```

**Edge Cases and Pitfalls:**
- On Windows, depth rendering might fail due to a known bug in the implementation [S1, unofficial]. Linux with supported hardware is recommended [S1, unofficial].
- Changing between 2k and 4k atlases doesn't change the quality drastically, but reduces the framerate significantly [S1, unofficial]. Use what you can get away with [S1, unofficial].
- The program requires GLSL 3.30 ES support [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Real-time rendering, achieving 60Hz on GTX1080 for a 4k atlas [S1, unofficial].
- **Memory Complexity:** Depends on the atlas size. 2k atlas renders at ~20fps on Intel 630 graphics [S1, unofficial].
- **Strengths:** Real-time performance, simple implementation using point clouds, no need for complex ray marching or volume accumulation [S1, unofficial].
- **Limitations:** Quality does not improve significantly with larger atlases, but performance degrades. Windows compatibility issues with depth rendering [S1, unofficial].
- **When to use:** When real-time visualization of volumetric data is required and hardware supports GLSL 3.30 ES [S1, unofficial].
- **When not to use:** When high-quality volume rendering (e.g., ray marching) is needed, or on Windows systems with known depth rendering bugs [S1, unofficial].

## Background
This method is part of "A Real-Time GPU Accelerated 3D Point Cloud Based Volumetric Viewer" done with python and OpenGL [S1, unofficial]. It was tested on Ubuntu 19.04, Manjaro and OpenSuse Tumbleweed with the latest (43x) NVidia drivers [S1, unofficial].

## Sources
[S1] teo3n/pointcloudvolumetric, https://github.com/teo3n/pointcloudvolumetric, fetched 2026-10-27 (unofficial)
