# Volumetric Clouds from Base Shape (Nubis/Frostbite Hybrid)

Also called: Hillaire 2016 light, Nubis density, Wrenninge multiple scattering

## Overview
This method renders volumetric clouds as a spherical shell defined by two altitudes above a base shape (e.g., a planet), ensuring clouds conform to any base geometry without 2D weather map tearing [S1, unofficial]. It combines Schneider’s density model (from Horizon Zero Dawn’s Nubis) with Hillaire 2016 light transport (from Frostbite) [S1, unofficial]. It is distinguished from sibling methods by its camera-centred direction grid for sky integration and its explicit lifting of the cloud layer above terrain relief to avoid depth-compositing artifacts [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Base shape geometry (implicit or explicit).
*   Two 3D noise textures: Shape noise ($128^3$ Perlin-Worley) and Detail noise ($32^3$ Worley) [S1, unofficial].
*   Camera position and orientation.
*   Sun direction and color.
*   Atmosphere parameters (for air transmittance and sky light) [S1, unofficial].

**Outputs:**
*   Cloud color and opacity per fragment, composited over the sky and into the FOG buffer [S1, unofficial].

### Data Structures
1.  **Noise Textures:**
    *   `shape_noise`: $128 \times 128 \times 128$ 3D texture, Perlin-Worley [S1, unofficial].
    *   `detail_noise`: $32 \times 32 \times 32$ 3D texture, Worley [S1, unofficial].
2.  **Camera-Centred Direction Grid:**
    *   Resolution: $1536 \times 768$ texels [S1, unofficial].
    *   Layout: Covers the hemisphere around the base's up vector.
    *   Texel Content:
        *   `cloud_light`: Radiance from clouds (including air attenuation) [S1, unofficial].
        *   `transmittance`: Cloud transmittance [S1, unofficial].
        *   `front_dist`: Distance to cloud front [S1, unofficial].
        *   `mean_dist`: Mean distance through cloud [S1, unofficial].
        *   `opacity`: Cloud opacity [S1, unofficial].
3.  **Shadow Map:**
    *   Resolution: $512 \times 512$ [S1, unofficial].
    *   Alignment: Aligned along the sun vector, centered on the camera [S1, unofficial].

### Steps

1.  **Layer Definition:**
    Define the cloud layer as a shell above the base shape. The layer height is scaled relative to the planet's radius. For Earth, the layer spans 1.5 to 4 km altitude [S1, unofficial]. The layer is lifted so its bottom is at least $1.25 \times$ the maximum ground relief to ensure the marcher ignores terrain [S1, unofficial].

2.  **Density Calculation (Schneider/Nubis):**
    For a shading point $P$ in 3D space:
    a. Sample `shape_noise` at $P$ using a "weather scale" [S1, unofficial].
    b. Determine cloud cover and type based on thresholds measured directly on the noise values [S1, unofficial].
    c. Sample `detail_noise` at $P$ for fine-scale variation [S1, unofficial].

3.  **Sky Integration (Camera-Centred Grid):**
    a. Initialize a $1536 \times 768$ grid [S1, unofficial].
    b. For each $2 \times 2$ block of texels, one invocation marches one texel's direction. The other three are reprojected based on the cloud's distance [S1, unofficial].
    c. For each marched direction:
        i. **Sun Transmittance (Air):** March through the atmosphere to calculate sun transmittance through air [S1, unofficial].
        ii. **Sun Transmittance (Cloud):** March through the cloud layer using progressive shadow samples to calculate sun transmittance through the cloud [S1, unofficial].
        iii. **Multiple Scattering:** Apply Wrenninge's multiple-scattering octaves to the cloud interior [S1, unofficial].
        iv. **Sky Light:** Sample the sky's light contribution from the atmosphere's multiple-scattering table [S1, unofficial].
        v. Compute `cloud_light`, `transmittance`, `front_dist`, `mean_dist`, and `opacity` for the texel [S1, unofficial].

4.  **Shadow Map Generation:**
    a. Render a $512 \times 512$ shadow map aligned with the sun vector around the camera [S1, unofficial].
    b. This map stores shadow information for the cloud layer [S1, unofficial].

5.  **Shading Integration:**
    a. **Sky Shader:** Place clouds over the atmosphere's sky using the precomputed grid data [S1, unofficial].
    b. **Page Shader:** Add clouds in front of the fragment to the FOG buffer [S1, unofficial].
    c. **Light Function:** Read the shadow map (one fetch) to determine cloud shadows for the fragment [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
SHAPE_NOISE_RES = 128
DETAIL_NOISE_RES = 32
GRID_WIDTH = 1536
GRID_HEIGHT = 768
SHADOW_MAP_RES = 512

# Data Structures
class CloudTexel:
    cloud_light: float  # Radiance with air attenuation
    transmittance: float
    front_dist: float
    mean_dist: float
    opacity: float

def compute_cloud_layer(base_shape, camera, sun_dir, atmosphere):
    """
    Computes the cloud layer parameters and grid.
    """
    # 1. Define Layer Bounds
    # Lifted so bottom is at least 1.25x max ground relief
    max_relief = base_shape.max_relief()
    layer_bottom = base_shape.radius() + 1.25 * max_relief
    layer_top = layer_bottom + cloud_layer_height(base_shape) # e.g., 1.5-4km for Earth
    
    # 2. Initialize Grid
    grid = [[CloudTexel() for _ in range(GRID_WIDTH)] for _ in range(GRID_HEIGHT)]
    
    # 3. March Grid Directions
    # One invocation per 2x2 block marches one texel, reprojects others
    for y in range(0, GRID_HEIGHT, 2):
        for x in range(0, GRID_WIDTH, 2):
            # Determine which of the 4 texels to march
            march_idx = (x + y) % 4
            for dy in range(2):
                for dx in range(2):
                    if (dy * 2 + dx) == march_idx:
                        # March this texel
                        dir_vec = grid_direction(x + dx, y + dy, camera)
                        texel = march_cloud_direction(dir_vec, layer_bottom, layer_top, sun_dir, atmosphere)
                        grid[y + dy][x + dx] = texel
                    else:
                        # Reproject from the marched texel
                        # (Simplified: assume reprojection logic uses cloud distance)
                        grid[y + dy][x + dx] = reproject_texel(grid[y + dy][x + dx], grid[march_y][march_x])
                        
    return grid

def march_cloud_direction(dir_vec, layer_bottom, layer_top, sun_dir, atmosphere):
    """
    Marches a single ray through the cloud layer.
    """
    # Ray origin and step
    ray_origin = camera.position
    ray_dir = dir_vec
    step_size = ... # Not given in sources [G]
    
    # Sun Transmittance through Air
    air_transmittance = atmosphere.march_sun_transmittance(ray_origin, ray_dir, step_size)
    
    # Sun Transmittance through Cloud
    cloud_transmittance = 1.0
    cloud_light = vec3(0.0)
    front_dist = 0.0
    mean_dist = 0.0
    opacity = 0.0
    inside_cloud = False
    
    # Progressive shadow samples
    for t in range(0, layer_top - layer_bottom, step_size):
        point = ray_origin + ray_dir * t
        if point.z > layer_bottom and point.z < layer_top: # Assuming Z is up
            if not inside_cloud:
                inside_cloud = true
                front_dist = t
            
            # Density Calculation
            shape_val = sample_noise(point, SHAPE_NOISE_RES, "Perlin-Worley")
            detail_val = sample_noise(point, DETAIL_NOISE_RES, "Worley")
            density = schneider_density(shape_val, detail_val)
            
            # Light Transport (Hillaire 2016)
            # Dual-lobe Henyey-Greenstein phase function
            phase = henyey_greenstein_phase(sun_dir, ray_dir, density)
            
            # Multiple Scattering (Wrenninge)
            scattering = wrenninge_multiple_scattering(density, phase)
            
            # Sky Light
            sky_light = atmosphere.multiple_scattering_table.sample(dir_vec)
            
            # Accumulate
            # Cloud light includes air attenuation
            cloud_light += scattering * air_transmittance * sun_dir.color + sky_light * density * step_size
            cloud_transmittance *= exp(-density * step_size)
            mean_dist = (mean_dist * (t - step_size) + t * step_size) / t if t > 0 else 0
            opacity = 1.0 - cloud_transmittance
            
        else:
            if inside_cloud:
                break # Exited cloud layer
                
    return CloudTexel(cloud_light, cloud_transmittance, front_dist, mean_dist, opacity)

def shade_fragment(fragment_pos, grid, shadow_map, sun_dir, atmosphere):
    """
    Shading integration for a fragment.
    """
    # 1. Get Grid Data
    # Map fragment direction to grid coordinates
    grid_uv = fragment_to_grid_uv(fragment_pos, camera)
    texel = grid[grid_uv.y][grid_uv.x]
    
    # 2. Shadow Map Fetch
    shadow_sample = shadow_map.fetch(fragment_pos, sun_dir)
    
    # 3. Final Color
    # Clouds take scene's warm sun color, reddened by air
    final_color = texel.cloud_light * shadow_sample + atmosphere.sky_color * (1.0 - texel.opacity)
    
    return final_color
```

### Parameters
*   **Noise Resolution:** Shape noise $128^3$, Detail noise $32^3$ [S1, unofficial].
*   **Layer Scale:** Earth layer 1.5 to 4 km. Lifted $1.25 \times$ max relief [S1, unofficial].
*   **Grid Resolution:** $1536 \times 768$ [S1, unofficial].
*   **Shadow Map Resolution:** $512 \times 512$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Terrain Occlusion:** The layer is lifted, not composited. The march knows nothing of terrain, so clouds sit above the largest relief and do not wrap peaks. A depth-aware march is required to fix this [S1, unofficial].
*   **Jagged Edges:** The grid resolution ($0.23^\circ$ per texel) causes jagged edges on small planets seen whole [S1, unofficial].
*   **Shadow Reach:** Shadows do not extend past the shadow map boundary (64 km on Earth, or 4 camera altitudes) [S1, unofficial].
*   **Color Accuracy:** Clouds take the scene's warm sun color, reddened by air, resulting in a pinkish-tan appearance. No white balance is applied. Fixing this requires setting the scene's light to white, which is a scene change [S1, unofficial].
*   **Wind Direction:** Wind is a render-space vector. On a planet, noise drifts through the shell away from the pole. It is now aligned with the wind; the old layer drifted against it [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The grid march is $O(W \times H)$ where $W=1536, H=768$. Shadow map generation is $O(S^2)$ where $S=512$. Per-fragment shading is $O(1)$ with one shadow map fetch [S1, unofficial].
*   **Memory Complexity:** Grid storage $1536 \times 768 \times 5 \times 4$ bytes (assuming float32 for 5 fields) $\approx 18.75$ MB. Shadow map $512 \times 512 \times 4$ bytes $\approx 1$ MB. Noise textures $128^3 \times 4$ bytes $\approx 256$ MB, $32^3 \times 4$ bytes $\approx 128$ KB [S1, unofficial].
*   **Measured Performance (RTX 3070 Laptop, power-saver mode):**
    *   Forest5cm, steady, 300 frames: Clouds off 19.05 ms, Clouds on 21.21 ms (+2.2 ms) [S1, unofficial].
    *   Single 20-frame shots: Earth's pole 4.8 → 8.1 ms (sky full of clouds). Earth from 100 km 5.2 → 11 ms [S1, unofficial].
*   **Strengths:** No 2D weather map tearing on curved bases. Efficient shadowing via shadow map (removed +33 ms cost of per-fragment shadow reads) [S1, unofficial].
*   **Limitations:** Jagged edges on small planets. Shadow reach limited by map size. Color accuracy issues without scene light changes. No light shafts, lightning, or precipitation [S1, unofficial].
*   **When to Use:** Real-time rendering of planetary clouds where performance is critical and terrain occlusion is not a primary concern.
*   **When Not to Use:** When high-fidelity cloud shadows extending far from the camera are required, or when accurate color balance is critical without scene modifications [S1, unofficial].

## Background
This method was developed for the opti-voxel engine, integrating with Godot sky and page shaders [S1, unofficial]. It combines Schneider's density model from Horizon Zero Dawn's Nubis (ported from clayjohn's godot-volumetric-cloud-demo) with Hillaire 2016 light transport from Frostbite [S1, unofficial]. It was introduced in a pull request to the opti-voxel repository in October 2026 [S1, unofficial].

## Sources
[S1] Volumetric clouds from the base's shape, https://github.com/aobayama-gaming/opti-voxel/pull/145, fetched 2026-10-28 (unofficial)
