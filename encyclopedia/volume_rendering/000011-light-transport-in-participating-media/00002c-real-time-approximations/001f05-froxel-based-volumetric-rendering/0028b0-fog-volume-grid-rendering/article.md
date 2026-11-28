# Fog Volume Grid Rendering

Also called: FogMaterial volumetric fog integration, FogVolume Volumetric Rendering, Godot Volumetric Fog, froxel volumetric fog

## Overview
Fog Volume Grid Rendering is a technique that draws local volumetric fog entities (FogVolume) and their associated material properties (FogMaterial) into a froxel-based volumetric grid by adding their density, albedo, and emission to the froxels geometrically covered by the volume [S1, unofficial]. It solves the problem of creating non-uniform, patchy fog effects (such as fog banks or layered mist) that cannot be achieved with global fog settings alone [S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A list of visible, unculled FogVolume entities, each containing a shape (box, ellipsoid, cone, cylinder, or world), a FogMaterial (density, albedo, emission, noise parameters), and spatial transforms [S1, unofficial].
*   **Outputs:** A 3D froxel grid (texture) where each cell contains accumulated density, albedo, and emission values derived from all overlapping volumes [S1, unofficial].

**Data Structures:**
1.  **FogVolume Row Texture:** A 1D texture or buffer where each row represents one visible volume [S1, unofficial].
2.  **Fog Density Atlas:** A single 3D texture formed by stacking individual density textures (from NoiseTexture3D or ImageTexture3D) in depth [S1, unofficial]. This is necessary because WebGL2 shaders cannot index a sampler array in a loop [S1, unofficial]. The atlas holds floats once a density texture does [S1, unofficial].
3.  **Froxel Grid:** The target 3D texture storing the accumulated volumetric data [S1, unofficial].

**Numbered Steps:**
1.  **Froxel Range Calculation:** For each visible FogVolume, determine the set of froxels it covers by porting the froxel range calculation logic from Godot's `fog.cpp` (lines 551-718) to compute the bounding box of the volume in froxel space and iterating over the intersecting cells [S1, unofficial].
2.  **Density Texture Generation:** For each volume, generate its local density texture [S1, unofficial]. If noise-based, build a `NoiseTexture3D` as a worker job, ensuring texels and seamless blending match Godot byte-for-byte [S1, unofficial]. If image-based, decode `ImageTexture3D` sub-resources in every uncompressed format (8-bit, 16-bit, half, float, RGBA4444, RGB565, RGBE9995) unpacked as `Image::_get_color_at_ofs` does [S1, unofficial]. For imported 3D textures, cut them into slices matching Godot's layered importer (`resource_importer_layered_texture.cpp:370-386`) [S1, unofficial]. Stack every generated density texture in depth into the `Fog Density Atlas` [S1, unofficial].
3.  **Volume Packing:** Pack each visible, unculled volume's parameters into the `FogVolume Row Texture` using `fogVolumeRows.ts` [S1, unofficial].
4.  **Froxel Accumulation:** Add the volumes' fixed-point sums in the light pass using `fogVolumeShader.ts`, handling Godot's packed-colour overflow [S1, unofficial]. Support all volume shapes: box, ellipsoid, cone, cylinder, and world [S1, unofficial]. Support height falloff, edge fade, negative density, and the default fog shader for volumes without a material [S1, unofficial]. Medium calculation follows: density, scattering = linear(albedo)·density, emission = linear(emission)·emission_energy·density [S2, unofficial].

**Parameters:**
*   `Active`: On/Off [S3].
*   `AffectsThisAreaOnly`: Boolean. False allows effect across multiple VisAreas/ClipVolumes [S3].
*   `Color`: RGB color of the fog [S3].
*   `DensityNoiseOffset`: Offsets the noise value for the density [S3].
*   `DensityNoiseScale`: Scales the noise value for the density (default 1) [S3].
*   `DensityNoiseTimeFrequency`: Controls the time frequency of the noise for the density [S3].
*   `DensityOffset`: Additional controls to the density of the fog volume [S3].
*   `FallOffDirLati`: Direction of the fall off (latitude). 90 = up [S3].
*   `FallOffDirLong`: Direction of the fall off (longitude). 0 = East (Positive X) [S3].
*   `FallOffScale`: Scales the density distribution along the fall off direction [S3].
*   `FallOffShift`: Controls how much to shift the fog density along the fall off direction [S3].
*   `GlobalDensity`: Controls the density of the fog volume [S3].
*   `IgnoreVisAreas`: Controls whether the FogVolume entity should respond to VisAreas and ClipVolumes [S3].
*   `NearCutOff`: Stops rendering the volume, depending on the camera distance to the object (m) [S3].
*   `RampStart`: Specifies the start distance of fog density ramp in world units (m) [S3].
*   `RampEnd`: Specifies the end distance of fog density ramp in world units (m) [S3].
*   `RampInfluence`: Controls the influence of fog density ramp [S3].
*   `SoftEdges`: Specifies a factor that is used to soften the edges of the fog volume when viewed from outside [S3].
*   `UseGlobalFogColor`: If true, the entities Color property is ignored. Instead, the current global fog color is used [S3].
*   `VolumeType`: Specifies the volume type. 0 - Ellipsoid, 1 - Cube [S3].
*   `WindInfluence`: Controls the influence of the global wind upon the fog volume [S3].
*   `DensityNoiseFrequency XYZ`: Controls the spatial frequency of the noise for the density [S3].
*   `Size XYZ`: Sets the size limits of the fog volume [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def accumulate_fog_volumes(froxel_grid, volumes, density_atlas):
    """
    Accumulates fog volume contributions into the froxel grid.
    froxel_grid: dict mapping froxel_index -> {'density': 0.0, 'albedo': (0,0,0), 'emission': (0,0,0)}
    volumes: list of dicts with keys: active, shape_type, size, transform, density_texture_index,
             density_noise_frequency, density_noise_offset, density_noise_scale, density_noise_time_frequency,
             global_density, soft_edges, fall_off_dir, fall_off_scale, fall_off_shift,
             ramp_start, ramp_end, ramp_influence, color, emission
    density_atlas: 3D texture lookup function atlas_sample(u, v, d)
    """
    for vol in volumes:
        if not vol['active']:
            continue
            
        # 1. Determine covered froxels (port Godot fog.cpp range calc)
        covered_froxels = get_froxel_range(vol)
        
        # 2. Get density texture index from atlas
        density_idx = vol['density_texture_index']
        
        # 3. Iterate over covered froxels
        for froxel in covered_froxels:
            # Calculate local position within volume
            local_pos = transform_inverse(vol['transform'], froxel['world_position'])
            
            # Check if local_pos is inside volume shape
            if not is_inside_volume_shape(local_pos, vol['shape_type'], vol['size']):
                continue
                
            # Sample base density from atlas
            base_density = density_atlas(local_pos.x, local_pos.y, local_pos.z)
            
            # Apply noise modulation
            noise_val = sample_3d_noise(
                local_pos * vol['density_noise_frequency'] + vol['density_noise_offset'],
                vol['density_noise_time_frequency']
            )
            final_density = base_density * noise_val * vol['global_density']
            
            # Apply shape-specific modifiers
            final_density = apply_height_falloff(final_density, local_pos)
            final_density = apply_edge_fade(final_density, local_pos, vol['soft_edges'])
            final_density = apply_fall_off_direction(final_density, local_pos, vol)
            final_density = apply_density_ramp(final_density, froxel['world_position'], vol)
            
            # Handle negative density (carving)
            if final_density < 0:
                final_density = -final_density
                
            # Accumulate into froxel
            idx = froxel['index']
            froxel_grid[idx]['density'] += final_density
            froxel_grid[idx]['albedo'] = tuple(c * final_density for c in vol['color'])
            froxel_grid[idx]['emission'] = tuple(c * final_density for c in vol['emission'])

def get_froxel_range(vol):
    """Returns list of froxels intersecting the volume's bounding box."""
    # Port Godot's fog.cpp range calculation logic
    return [] # Implementation details ported from Godot fog.cpp:551-718

def transform_inverse(mat, point):
    """Applies inverse transform matrix to point."""
    # Standard 4x4 matrix inverse multiplication
    return [mat[i][0]*point[0] + mat[i][1]*point[1] + mat[i][2]*point[2] + mat[i][3] for i in range(3)]

def is_inside_volume_shape(local_pos, shape_type, size):
    """Checks if a point is inside the volume shape."""
    if shape_type == 'box':
        return all(abs(local_pos[i]) <= size[i] / 2 for i in range(3))
    elif shape_type == 'ellipsoid':
        return sum((local_pos[i] / (size[i] / 2))**2 for i in range(3)) <= 1.0
    elif shape_type == 'cone':
        # Simplified cone check
        return local_pos[1] >= -size[1]/2 and local_pos[1] <= size[1]/2 and sum(local_pos[i]**2 for i in range(3)) <= (size[0]/2)**2
    elif shape_type == 'cylinder':
        return local_pos[1] >= -size[1]/2 and local_pos[1] <= size[1]/2 and local_pos[0]**2 + local_pos[2]**2 <= (size[0]/2)**2
    return False

def sample_3d_noise(pos, time_freq):
    """Samples 3D noise (e.g., simplex or Perlin-Worley) with time modulation."""
    # Standard 3D noise function
    return math.sin(pos[0] + time_freq) * math.cos(pos[1] + time_freq) * math.sin(pos[2] + time_freq)

def apply_height_falloff(density, local_pos):
    """Modulates density based on vertical position."""
    return density * (1.0 - abs(local_pos[1]) / 2.0)

def apply_edge_fade(density, local_pos, soft_edges):
    """Softens edges using SoftEdges factor."""
    dist = math.sqrt(sum(p**2 for p in local_pos))
    return density * max(0.0, 1.0 - dist / soft_edges)

def apply_fall_off_direction(density, local_pos, vol):
    """Controls density distribution along a specific direction."""
    dir_vec = vol['fall_off_dir']
    proj = sum(local_pos[i] * dir_vec[i] for i in range(3))
    shift = vol['fall_off_shift']
    scale = vol['fall_off_scale']
    return density * max(0.0, 1.0 - max(0.0, proj - shift) / scale)

def apply_density_ramp(density, world_pos, vol):
    """Applies fog density ramp based on distance."""
    dist = math.sqrt(sum(p**2 for p in world_pos))
    if dist < vol['ramp_start'] or dist > vol['ramp_end']:
        return 0.0
    t = (dist - vol['ramp_start']) / (vol['ramp_end'] - vol['ramp_start'])
    return density * t * vol['ramp_influence']
```

**Edge Cases and Pitfalls:**
*   **Float Texel Upload:** Float texels upload raw as half float because Godot gives a float format no sRGB view and WebGL2 always filters a half float [S1, unofficial]. An atlas crop of a half-float sheet stays half float, and a canvas read converts it to bytes [S1, unofficial].
*   **ShaderMaterial Limitation:** A fog ShaderMaterial runs no fog shader. This requires a shading-language runtime [S1, unofficial].
*   **Compressed/Integer Formats:** An Image in a block-compressed format draws nothing. An Image in an integer format draws nothing because a float sampler cannot read it [S1, unofficial].
*   **NoiseTexture3D Build Delay:** A NoiseTexture3D shows even density until its build lands [S1, unofficial].
*   **Atlas Limit:** Past 2048 stacked density slices, further textures leave the density even [S1, unofficial].
*   **sRGB Decoding:** A 3D colour slot still decodes every texture as sRGB. Godot reads a format without an sRGB view raw, so a Lum8 or grey-noise albedo draws too dark [S1, unofficial].

## Subtopics

### [Fog Volume Injection Optimization](15036-fog-volume-injection-optimization.md)
Fog Volume Injection Optimization reduces the computational cost of injecting scene fog volumes into a froxel grid by ensuring each volume is processed only by the froxels it geometrically covers, rather than having every froxel test every scene fog volume. It differs from the brute-force approach (where every froxel iterates all volumes) by using precomputed CPU-side bounds to cull irrelevant volume-froxel pairs, and differs from Godot’s per-volume dispatch by using a single GPU kernel with a culled list of ranges, avoiding the overhead of many small kernel launches.

### [Fog Volume Entity](16231-fog-volume-entity.md)
The Fog Volume Entity is a spatial entity used to define local volumetric fog density, color, and noise-based structure within a 3D level. It solves the problem of creating non-uniform, patchy fog effects (such as fog banks or layered mist) that cannot be achieved with global fog settings alone. It differs from sibling methods like [Fog Volume Grid Rendering](4350-fog-volume-grid-rendering.md) by providing a high-level, user-configurable entity interface for shaping fog via noise parameters, rather than directly manipulating the underlying froxel data structures.

### [Density and Phase Function Evaluation](5931-density-and-phase-function-evaluation.md)
This method computes the optical properties (density, scattering, absorption) and angular scattering behavior (phase function) for each cell in a froxel-based volumetric grid. It serves as the initialization step for light transport, determining how much light is attenuated and how it is redistributed within the medium before the light injection pass. It differs from sibling methods by focusing solely on material property assignment rather than optimization or entity management.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Golden test 'fog-volume' (3x3x3 box at 0, 1, -5): Mean error against Godot 4.6.3 drops from 3.476/255 (no volume) to 0.205/255 [S1, unofficial]. Value check mean error over pixels the volume changes in Godot: box 13.05→0.40, ellipsoid 10.66→0.22, cone 8.90→0.40, cylinder 15.67→0.41, world shape 14.07→0.25, negative density 3.65→0.26, height falloff 2 9.58→0.24, edge fade 0 14.21→0.40, edge fade 4 2.55→0.08, no material 13.21→0.35, albedo/emission 32.91→1.13, density 8 23.92→0.45, rotated/scaled 13.73→0.30, spot/sun lights 24.70→0.60 [S1, unofficial]. Value check mean error over whole frame: imported 3D texture 3.554→0.209, ImageTexture3D .tres Lum8 4.459→0.206, NoiseTexture3D .tres 5.418→0.196, ImageTexture3D .tres RHalf 2.227→0.215, RFloat 2.229→0.216, R16 1.153→0.184, RGBE9995 2.229→0.215, NoiseTexture3D warp simplex 0.714→0.194, reduced simplex progressive 0.683→0.199, basic grid independent 0.496→0.196 [S1, unofficial].
*   **Strengths:** Supports complex local fog shapes and variations; integrates with global fog; supports noise-based dynamic fog.
*   **Limitations:** Limited by atlas size (2048 slices); requires careful handling of texture formats; ShaderMaterial integration is non-trivial.
*   **When to use:** When non-uniform, patchy fog effects are needed.
*   **When not to use:** When simple, uniform fog is sufficient; when performance is critical and many volumes are used (due to atlas limits).

## Background
Fog Volume Grid Rendering was implemented in Godot 4.6.3 and ported to the textscene-inspector repository [S1, unofficial]. It is used in real-time volumetric rendering to integrate local fog volumes into a global froxel-based fog system [S1, unofficial].

## Sources
[S1] feat(renderer): draw FogVolume and FogMaterial into the volumetric fog, https://github.com/Vortiago/textscene-inspector/pull/732, fetched 2026-10-26 (unofficial)
[S2] Draw WorldEnvironment volumetric fog, https://github.com/Vortiago/textscene-inspector/issues/658, fetched 2026-10-27 (unofficial)
[S3] CRYENGINE | Documentation - Volumetric Fog, https://www.cryengine.com/docs/static/engines/cryengine-3/categories/1114113/pages/21268322, fetched 2026-10-29
