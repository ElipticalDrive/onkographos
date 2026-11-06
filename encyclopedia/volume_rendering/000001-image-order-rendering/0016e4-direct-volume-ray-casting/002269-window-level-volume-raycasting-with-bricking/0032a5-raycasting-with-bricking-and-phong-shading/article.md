# Raycasting with Bricking and Phong Shading

Also called: parallel-volume-renderer raycasting, skip's raycaster

## Overview
A CPU-based direct volume ray casting algorithm that accelerates traversal using a brick-based data layout and computes illumination via Phong shading with gradient estimation. It differs from its parent [Window/Level Volume Raycasting with Bricking](2775-window-level-volume-raycasting-with-bricking.md) by explicitly integrating Phong shading and gradient calculation into the ray loop, and from its sibling [Octree-based Void Culling](3828-octree-based-void-culling.md) by relying on fixed-size bricks and distance volume tables rather than hierarchical void culling for acceleration.

## How it works
### Inputs and Outputs
**Inputs:**
- Volume data stored in a brick-based layout (`data.data2`) or linear layout (`data.data`).
- Camera parameters: eye position (`from`), target (`to`), up vector (`up`), zoom, near plane (`n`), far plane (`f`).
- Screen dimensions: `scr_x`, `scr_y`.
- Opacity transfer function: Lookup Table (`edt_lut[4096]`).
- Shading parameters: Light direction (`L`), view direction (`V`), ambient/diffuse/specular coefficients (via `CShader`).
- Acceleration structures: Distance volume tables (`incrTable`, `incr`) with `MAX_DISTANCE = 255`.

**Outputs:**
- Rendered image texture (`textura.data`), typically RGB or RGBA.

### Data Structures
**Image Struct:**
- `sirka` (unsigned long): Width.
- `vyska` (unsigned long): Height.
- `data` (UINT8*): Pixel data. Size is `sirka * vyska * 4` for RGBA or `sirka * vyska * 3` for RGB (Linux version).

**CRaycaster Class Fields:**
- `scr_x`, `scr_y` (UINT16): Window dimensions.
- `kamera` (CKamera): Camera state.
- `data` (CData): Volume data container.
- `cam_u`, `cam_v` (float): Ray direction components in camera space.
- `ray_start` (CVektor3): Ray origin.
- `delta` (CVektor3): Step vector.
- `samplingRate` (float): Default 1.0.
- `poc_vz`, `max_vz` (UINT16): Sample counters.
- `typ_dat` (UINT8): Bit depth of volume data.
- `r` (CRay): Current ray state.
- `tmin`, `tmax` (float): Entry/exit distances.
- `mi`, `mj`, `mk` (UINT16): Current brick indices.
- `offset` (CVektor3): Brick offset.
- `vystup` (CVektor3): Exit point.
- `v[8]` (UINT32): Indices of the 8 corners of the current brick.
- `svetlo` (CSvetlo): Light source.
- `intenzita` (float): Accumulated intensity.
- `pozadie` (float): Background intensity.
- `farba` (float): Interpolated color.
- `acc_color` (float): Accumulated color.
- `shader` (CShader): Shading model parameters.
- `L`, `V`, `N` (CVektor3): Light, View, Normal vectors.
- `c_out` (CFarba): Output color from shading.
- `oa`, `ob` (UINT16): Opacity thresholds.
- `foa`, `fob` (float): Opacity coverage.
- `acc_opacity` (float): Accumulated opacity.
- `edt_lut[4096]` (float): Opacity LUT.
- `incrTable[MAX_DISTANCE]` (CVektor3): Distance volume table for positions.
- `incr[MAX_DISTANCE]` (int): Distance volume table for steps.
- `textura` (Image): Output texture.

### Steps
1. **Initialization:**
   - Initialize raycaster with screen dimensions `scr_x`, `scr_y`.
   - Set camera using `nastavKameru` with parameters `from`, `to`, `up`, `zoom`, `n`, `f`.
   - Initialize texture with `initTexture`.
   - Initialize data with `CData`.
   - Initialize opacity editor with `initOpacEdt` and LUT with `vytvorEdtLut`.
   - Initialize distance volume tables with `initRadcTables`.

2. **Per-Pixel Setup:**
   - For each pixel, calculate ray direction components `cam_u`, `cam_v` and start position `ray_start`.
   - Compute step vector `delta` from camera direction and `samplingRate`.
   - Initialize accumulation variables: `acc_color = 0`, `acc_opacity = 0`.
   - Set `poc_vz = 0`.

3. **Ray Loop:**
   - Loop while `poc_vz` is less than `max_vz` (or until ray leaves volume).
   - Determine current block indices `mi`, `mj`, `mk` using `zistiBlok` or `vratBlok` with current ray position `r.pos`.
   - Calculate block offset `offset` using `vratOffsetBloku`.
   - Get data indices `v[0]` to `v[7]` for the 8 corners of the brick containing the sample.

4. **Gradient Computation:**
   - Compute gradient `N` using `getAnalGrad` with data values from `data.data2` (bricking layout) or `data.data` (linear layout) at indices `v[0]` to `v[7]` and `offset`.
   - Normalize `N` using `normalizuj`.
   - Negate `N` (`N = N * -1`).

5. **Shading and Color:**
   - Compute color `c_out` using Phong shading: `c_out = tienujPhong(CFarba(0.0f, farba/4095.0f, 0.0f, 1.0f), shader, L, V, N)`.
   - Extract green channel `farba = c_out.g`.
   - If no shading or specific condition, `farba = farba/4095.0f * shader.Fa`.

6. **Accumulation:**
   - Perform front-to-back blending:
     $$acc\_color = acc\_color + farba \cdot \alpha \cdot (1.0f - acc\_opacity)$$
     $$acc\_opacity = acc\_opacity + \alpha \cdot (1.0f - acc\_opacity)$$
     where $\alpha$ is the opacity of the current sample (from `edt_lut`), `farba` is the color contribution, `acc_color` is the accumulated color, and `acc_opacity` is the accumulated opacity.
   - Increment `poc_vz`.

7. **Finalization:**
   - After loop, set `intenzita = acc_color`.
   - If ray did not hit data, `intenzita = pozadie`.
   - Clamp `intenzita` to max 1.0f.
   - Write to texture: `textura.data[tex] = 0`, `textura.data[tex+1] = 0`, `textura.data[tex+2] = (UINT8)(intenzita * 255.0f)`.
   - Increment texture pointer `tex += 3`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raycast_with_bricking_phong(scr_x, scr_y, camera, volume_data, shader, light, background):
    # Initialization
    init_texture(scr_x, scr_y)
    setup_camera(camera)
    init_data(volume_data)
    init_opacity_lut()
    init_distance_tables()
    
    # Precompute ray directions
    cam_u, cam_v = compute_ray_directions(camera)
    ray_start = camera.eye
    delta = compute_step_vector(camera, volume_data.sampling_rate)
    
    for y in range(scr_y):
        for x in range(scr_x):
            acc_color = 0.0
            acc_opacity = 0.0
            poc_vz = 0
            max_vz = compute_max_samples(camera, volume_data)
            
            # Ray origin for this pixel
            ray_pos = compute_ray_origin(camera, x, y)
            ray_dir = compute_ray_direction(camera, x, y)
            
            while poc_vz < max_vz:
                # Check if ray left volume
                if is_ray_out_of_volume(ray_pos, camera, volume_data):
                    break
                
                # Determine brick indices
                mi, mj, mk = get_brick_indices(ray_pos, volume_data)
                offset = get_brick_offset(mi, mj, mk, volume_data)
                
                # Get 8 corner indices
                v = get_corner_indices(mi, mj, mk, volume_data)
                
                # Compute gradient
                N = compute_gradient(volume_data, v, offset)
                N = normalize(N)
                N = N * -1.0
                
                # Phong Shading
                # Note: 'farba' here refers to the scalar value from volume data interpolated at current position
                # Assuming 'farba' is obtained from volume data at ray_pos before this step
                # In the source, 'farba' is used in tienujPhong. 
                # We assume 'farba' is the scalar value at current sample point.
                # The source code snippet shows: c_out = tienujPhong(CFarba(0.0f, farba/4095.0f, 0.0f, 1.0f), shader, L, V, N)
                # This implies 'farba' is the volume value.
                
                # Compute view vector V (from eye to ray_pos)
                V = normalize(ray_pos - camera.eye)
                
                c_out = phong_shading(farba / 4095.0, shader, light.direction, V, N)
                farba = c_out.g # Extract green channel as intensity/color
                
                # If no shading, use raw value
                if not shading_enabled:
                    farba = (farba / 4095.0) * shader.Fa
                
                # Get opacity from LUT
                alpha = opacity_lut[farba]
                
                # Front-to-back accumulation
                acc_color = acc_color + farba * alpha * (1.0 - acc_opacity)
                acc_opacity = acc_opacity + alpha * (1.0 - acc_opacity)
                
                # Advance ray
                ray_pos = ray_pos + delta
                poc_vz += 1
            
            # Finalize pixel
            intensity = acc_color
            if acc_opacity < 0.001: # Ray did not hit data effectively
                intensity = background
            
            intensity = min(intensity, 1.0)
            
            # Write to texture (RGB)
            tex_idx = (y * scr_x + x) * 3
            texture.data[tex_idx] = 0
            texture.data[tex_idx + 1] = 0
            texture.data[tex_idx + 2] = int(intensity * 255.0)
```

### Parameters
- `MAX_DISTANCE`: Constant defined as 255, representing the maximum jump distance for the ray in the distance volume acceleration structure.
- `samplingRate`: Float parameter controlling the number of samples taken along the ray. Default value is 1.

### Pitfalls
- Ray leaving the volume is checked by decrementing `poc_vzr` and breaking if zero. An alternative check using scalar product `skalarsucin(r.dir, vystup-r.pos) < 0.0f` is commented out.
- Data access violation during camera rotation was fixed in version 1.03. Camera rotation around origin (0,0,0) and distance change via mouse were added.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of pixels and the number of samples per ray. Acceleration structures (bricks, distance tables) reduce the number of samples processed by skipping empty space.
- **Memory Complexity:** Proportional to the volume data size plus the size of the brick indices and distance tables. The distance table size is fixed at `MAX_DISTANCE = 255`.
- **Performance:** Version 1.04 increased FPS by 0.3-0.4 with fast 'out of data check'. Version 1.05 added distance volume acceleration structure.
- **Strengths:** CPU-based, allows for complex shading (Phong) and gradient computation. Brick-based layout provides spatial locality.
- **Limitations:** CPU-bound, potentially slower than GPU-based ray casting for high resolutions. Fixed brick size may not be optimal for all data distributions.
- **When to use:** When CPU-based rendering is required, or when specific shading models like Phong are needed without GPU shader support.
- **When not to use:** For real-time high-resolution rendering where GPU acceleration is preferred.

## Background
- **Origin:** Developed by "skip" in 2006.
- **History:**
  - Version 1.00 (2.1.2006): Initial set and check of camera and scene, not optimized raycasting.
  - Version 1.01 (4.1.2006): OpenGL and texture support, rendering to texture, major bugs fixed.
  - Version 1.02 (5.1.2006): Homog lookup data acceleration.
  - Version 1.03: Fixed access violation by camera rotation, added camera rotation around origin and distance change.
  - Version 1.04: Fast implementation of 'out of data check', fps increased by 0.3-0.4.
  - Version 1.05 (11.4.2006): Bugs fixed, faster ray-leave-volume check, distance volume acceleration structure added.
  - Version 1.06 (Jul 2006): RGBA support.
- **Usage:** Used in the `parallel-volume-rendering` project, which also includes MPI-based parallelization components.

## Sources
[S1] parallel-volume-rendering, https://github.com/besnik/parallel-volume-rendering, fetched 2026-10-27 (unofficial)
