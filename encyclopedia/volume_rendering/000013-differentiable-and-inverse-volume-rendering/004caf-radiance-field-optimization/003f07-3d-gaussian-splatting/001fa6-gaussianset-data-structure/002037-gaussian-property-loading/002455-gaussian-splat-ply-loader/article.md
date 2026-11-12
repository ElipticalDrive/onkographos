# Gaussian Splat PLY Loader

Also called: Gaussian Splatting PLY Parser, PLYData structure loader, PLYLoader::load, Specialized Binary PLY Loader

## Overview
The Gaussian Splat PLY Loader parses 3D Gaussian Splatting scene data from a Polygon File Format (PLY) file, decoding raw vertex properties into a structured Gaussian representation suitable for rendering [S1, unofficial][S2, unofficial]. It solves the problem of converting the standardized, often compressed or logarithmically encoded PLY storage format into the explicit mathematical parameters (position, scale, rotation, color, opacity) required by the rasterization pipeline [S1, unofficial][S2, unofficial]. It differs from its siblings by focusing on the initial ingestion and decoding of the PLY file into the `GaussianData` structure, whereas [Coordinate System Conversion for Gaussian Splats](3935-coordinate-system-conversion-for-gaussian-splats.md) handles spatial alignment and [INRIA V1 PLY Splat Parsing](6822-inria-v1-ply-splat-parsing.md) addresses specific legacy format variations [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A binary little-endian PLY file containing vertex properties (`x`, `y`, `z`, `scale_0`, `scale_1`, `scale_2`, `rot_0`, `rot_1`, `rot_2`, `rot_3`, `f_dc_0`, `f_dc_1`, `f_dc_2`, `red`, `green`, `blue`, `opacity`, plus optional `f_rest_*`, `octlevel`, `octpath`, and `grid_*_value` properties) [S1, unofficial][S2, unofficial].
- Output: A populated `GaussianData` structure containing arrays for positions, scales, rotations, and colors, followed by a packed GPU-ready buffer [S1, unofficial][S2, unofficial].

**Data Structures:**
The loader populates the following arrays, each of size `vertexCount`:
- `positions`: `glm::vec3` (3 floats) [S2, unofficial].
- `scales`: `glm::vec3` (3 floats, decoded from log-space) [S1, unofficial][S2, unofficial].
- `rotations`: `glm::quat` (4 floats, normalized) [S1, unofficial][S2, unofficial].
- `colors`: `glm::u8vec4` (4 unsigned chars: R, G, B, A) [S2, unofficial].
- Optional arrays: `shRestValues` (sized by `vertexCount × count(f_rest)`), `gridValues` (sized by `vertexCount × count(grid)`), `sceneCenter` (3 floats), `sceneExtent` (1 float) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_ply(file_path):
    # 1. Fetch and validate file
    raw_data = fetch_chunks(file_path)
    header = decode_text(raw_data[:10000] or raw_data)
    assert header.startswith('ply') and 'format binary_little_endian' in header
    assert 'element vertex' in header
    vertex_count = parse_integer(header, 'element vertex count')
    
    # 2. Parse metadata
    scene_center = parse_floats(header, 'scene_center')  # 3 floats
    scene_extent = parse_float(header, 'scene_extent')   # 1 float
    
    # 3. Request properties
    has_rgb = request_property('red') and request_property('green') and request_property('blue')
    opacity_ptr = request_property('opacity')
    has_sh = request_property('f_dc_0') and request_property('f_dc_1') and request_property('f_dc_2')
    
    # 4. Allocate arrays
    positions = array(vertex_count, vec3)
    scales = array(vertex_count, vec3)
    rotations = array(vertex_count, quat)
    colors = array(vertex_count, u8vec4)
    sh_rest = array(vertex_count * count_f_rest, float) if request_property('f_rest_*') else None
    grid_values = array(vertex_count * count_grid, float) if request_property('grid_*_value') else None
    
    # 5. Reinterpret buffers
    x_data = reinterpret_float(request_property('x'))
    y_data = reinterpret_float(request_property('y'))
    z_data = reinterpret_float(request_property('z'))
    scale_0_data = reinterpret_float(request_property('scale_0'))
    scale_1_data = reinterpret_float(request_property('scale_1'))
    scale_2_data = reinterpret_float(request_property('scale_2'))
    rot_0_data = reinterpret_float(request_property('rot_0'))
    rot_1_data = reinterpret_float(request_property('rot_1'))
    rot_2_data = reinterpret_float(request_property('rot_2'))
    rot_3_data = reinterpret_float(request_property('rot_3'))
    f_dc_0_data = reinterpret_float(request_property('f_dc_0')) if has_sh else None
    f_dc_1_data = reinterpret_float(request_property('f_dc_1')) if has_sh else None
    f_dc_2_data = reinterpret_float(request_property('f_dc_2')) if has_sh else None
    red_data = reinterpret_int(request_property('red')) if has_rgb else None
    green_data = reinterpret_int(request_property('green')) if has_rgb else None
    blue_data = reinterpret_int(request_property('blue')) if has_rgb else None
    opacity_data = reinterpret_float(opacity_ptr) if opacity_ptr else None
    
    # 6. Decode loop
    for i in range(vertex_count):
        positions[i] = vec3(x_data[i], y_data[i], z_data[i])
        scales[i] = vec3(exp(scale_0_data[i]), exp(scale_1_data[i]), exp(scale_2_data[i]))
        q = quat(rot_0_data[i], rot_1_data[i], rot_2_data[i], rot_3_data[i])
        rotations[i] = normalize(q)
        
        if has_sh:
            r = clamp((0.5f + SH_C0 * f_dc_0_data[i]) * 255.0f, 0.0f, 255.0f)
            g = clamp((0.5f + SH_C0 * f_dc_1_data[i]) * 255.0f, 0.0f, 255.0f)
            b = clamp((0.5f + SH_C0 * f_dc_2_data[i]) * 255.0f, 0.0f, 255.0f)
        elif has_rgb:
            r = clamp(red_data[i], 0, 255)
            g = clamp(green_data[i], 0, 255)
            b = clamp(blue_data[i], 0, 255)
        else:
            r = g = b = 255
            
        if opacity_data is not None:
            a = 1.0f / (1.0f + exp(-opacity_data[i]))
            a = clamp(a * 255.0f, 0.0f, 255.0f)
        else:
            a = 255.0f
            
        colors[i] = u8vec4(r, g, b, a)
        
    # 7. Pack for GPU
    data.pack(positions, scales, rotations, colors)
    return GaussianData(positions, scales, rotations, colors, sh_rest, grid_values, scene_center, scene_extent)
```

**Formulas:**
- Scale decoding: $s_i = \exp(s_{\text{raw}, i})$ [S1, unofficial][S2, unofficial].
- Spherical Harmonics DC color: $c = \text{clamp}((0.5 + SH\_C0 \times f_{\text{dc}}) \times 255.0, 0.0, 255.0)$ [S1, unofficial][S2, unofficial].
- Opacity decoding: $a = \text{sigmoid}(o) = \frac{1.0}{1.0 + \exp(-o)}$ [S1, unofficial][S2, unofficial].

**Parameters:**
- $SH\_C0$: First spherical harmonics basis coefficient. Value is $\frac{1}{\sqrt{4\pi}}$ [G].

**Edge Cases and Pitfalls:**
- Missing properties: If RGB or Opacity properties are absent, the loader defaults to white color and full opacity without crashing [S1, unofficial][S2, unofficial].
- Strict aliasing: Direct casting of float pointers to uint32 pointers for GPU packing violates strict aliasing rules; `memcpy` must be used for position copying [S2, unofficial].
- Normalization: Rotations must be normalized to unit quaternions to ensure valid rotation matrices [S1, unofficial][S2, unofficial].
- Log-space scales: Scales are stored in log-space to prevent negative values during optimization; they must be exponentiated upon loading [S1, unofficial][S2, unofficial].
- Missing octree properties: If `octlevel` or `octpath` properties are missing, the loader throws an error [S1, unofficial].
- Missing SH rest coefficients: If `f_rest` properties are missing, `shRestValues` remains undefined, resulting in no directional lighting visibility [S1, unofficial].

## Subtopics

### [PLY Splat Publishing](10283-ply-splat-publishing.md)
PLY Splat Publishing is a ROS 2 node that loads 3D Gaussian Splat data from a PLY file, reconstructs the full Gaussian parameters (including covariance matrices and spherical harmonics), and publishes them as a custom `SplatArray` message for visualization in tools like RViz. It builds on [Gaussian Splat PLY Loader](3254-gaussian-splat-ply-loader.md) by handling the specific ROS message serialization and QoS configuration required for real-time rendering pipelines.

### Scene Storage Format Modification
This variant modifies the standard 3DGS PLY format to enable direct streaming to GPU memory, eliminating decoding overhead and optimizing for cache locality [S3]. It removes obsolete normal properties, reorders spherical harmonics data for better cache-locality, decodes opacity and scale properties during storage/streaming, and normalizes the rotation quaternion [S3]. It also limits the scene to a central cube, excluding Gaussians with means outside this volume [S3]. The mesh is stored alongside the modified scene file, including a list of page IDs (one entry per mesh face), a buffer containing page links, and metadata (page size, number of LOD levels) [S3].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of Gaussians $N$ for parsing and decoding $O(N)$ [S1, unofficial][S2, unofficial]. The packing step involves matrix multiplications per Gaussian, also $O(N)$ [S2, unofficial].
- **Memory Complexity:** $O(N)$ for storing the decoded properties and the packed GPU buffer [S1, unofficial][S2, unofficial]. The packed buffer uses 8 uint32s (32 bytes) per Gaussian [S2, unofficial].
- **Strengths:** Handles standard and modified PLY formats; decodes log-space scales and sigmoid opacity correctly; prepares data for efficient GPU rasterization [S1, unofficial][S2, unofficial].
- **Limitations:** Requires the PLY file to contain specific property names; does not handle coordinate system conversion (delegated to sibling method); does not handle compression (deferred to future work in modified format) [S2, unofficial][S3].
- **When to use:** When loading a pre-trained 3D Gaussian Splatting model from a PLY file for real-time rendering [S1, unofficial][S2, unofficial].

## Background
The method is based on the implementation in the `svraster-webgl` repository, an unofficial WebGL-based 3D Gaussian Splatting renderer [S1, unofficial]. The C++ packing and data structure layout originate from the `gsplat-opengl` repository [S2, unofficial]. The modifications for virtual memory streaming and cache locality are described in "Virtual memory for 3D Gaussian Splatting" [S3].

## Sources
[S1] Gaussian Splatting Alternative: WebGL Implementation of Nvidia's SVRaster, https://github.com/samuelm2/svraster-webgl, fetched 2026-10-26 (unofficial)
[S2] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
[S3] Virtual memory for 3D Gaussian Splatting, https://zenodo.org/records/19690968, fetched 2026-10-26
