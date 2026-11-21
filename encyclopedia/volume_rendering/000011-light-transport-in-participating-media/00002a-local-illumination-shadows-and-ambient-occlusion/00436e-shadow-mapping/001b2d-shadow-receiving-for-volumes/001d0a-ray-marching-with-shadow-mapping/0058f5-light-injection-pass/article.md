# Light Injection Pass

Also called: Light-Space Rendering Pass, Light/Volume two-pass, Main Rendering Pass, Precomputed Lighting Pass, Volumetric Light (Sun Shafts and Lamp Glow), Wave D volumetric light

## Overview
The Light Injection Pass is a GPU fragment-stage algorithm that computes volume illumination by traversing light rays from the light source's perspective, determining visibility via a binary search on an opacity shadow map (A-Buffer), and compositing light samples into camera space [S3, unofficial]. It is specifically implemented as a post-processing pass that marches rays from the eye to the depth buffer to add light scattered by air (sun shafts and lamp glow), executed only when `r_realtimeLighting` is set to 2 [S1, unofficial]. It builds on [Ray Marching with Shadow Mapping](1415-ray-marching-with-shadow-mapping.md) by precomputing in-scattered radiance into a 3D texture using shadow maps, which is then integrated during the view-ray march [S2, unofficial]. Unlike [Shadow Volume Rendering](1556-shadow-volume-rendering.md) which uses geometric volumes, or [Deep Shadow Maps](2166-deep-shadow-maps.md) which store depth distributions, this method relies on a two-pass approach: an injection pass to populate lighting data and a rendering pass to composite it [S3, unofficial].

## How it works
The method consists of two distinct phases: the Light Injection Pass (populating the 3D texture with lighting data) and the Main Rendering Pass (compositing light into the final image).

### Inputs and Outputs
*   **Inputs:**
    *   3D texture (voxel grid) storing extinction coefficient and in-scattered radiance.
    *   Shadow maps (cascaded for directional lights, cubemaps for point lights).
    *   Camera view and projection matrices.
    *   Light source parameters (position, color, range, type).
    *   Volume parameters: density, anisotropy ($g$), range.
*   **Outputs:**
    *   Updated 3D texture with in-scattered radiance.
    *   Final image with volumetric lighting effects (sun shafts, lamp glow).

### Data Structures
*   **3D Texture:** Stores extinction coefficient ($\sigma_t$) and in-scattered radiance ($L_{in}$). Accessed via image load/store (`glBindImageTexture`) [S2, unofficial].
*   **A-Buffer (Visibility Buffer):** Stores visibility ($v$) and distance ($d$) to the camera for each sample along the ray. The number of layers is $2^k - 1$, where $k$ is manually specified or set via data variation analysis [S3, unofficial].
*   **Shadow Maps:** Cascaded shadow maps (4 cascades) for directional lights; shadow cubemaps for point lights [S1, unofficial][S2, unofficial].

### Step 1: Light Injection Pass (Precomputation)
This pass calculates the incoming radiance for each voxel based on light sources.

1.  **World Position Reconstruction:** For each voxel at 3D grid index $(x, y, z)$, map to clip space, apply the inverse view-projection matrix, and divide by $w$ to determine the world-space center of the cell [S2, unofficial].
2.  **Shadow Map Sampling:** Sample standard shadow maps (cascaded for directional, cubemaps for point lights) to check for occlusion [S2, unofficial].
3.  **In-Scattering Calculation:** For all unshadowed lights, evaluate light color, attenuation, and the phase function $p(\theta)$. Store the resulting in-scattered radiance alongside the extinction coefficient in the 3D texture using image load/store [S2, unofficial].

### Step 2: Main Rendering Pass (Compositing)
This pass traverses light rays from the light source, determines visibility, and projects light into camera space.

1.  **Define Ray:** Define a ray from front to back (from the light source) [S3, unofficial].
2.  **Loop for Samples:** For all samples $s$ along the ray:
    a.  **Read Voxel:** Read/classify voxel at $s$ to get color/opacity $c$ [S3, unofficial].
    b.  **Check Opacity:** If opacity in $c > 0$:
        i.  **Project to Camera:** Project $s$ to camera space and get texture coordinate $t$ [S3, unofficial].
        ii. **Calculate Distance:** Calculate distance $d$ to camera (in volume space) [S3, unofficial].
        iii. **Binary Search:** Retrieve closest layers to $s$ based on $t$ and $d$ using binary search in the A-Buffer. Optimize by checking against first and last layer before initiating search [S3, unofficial].
        iv. **Calculate Visibility:** Calculate visibility $v$ through linear interpolation between closest layers with respect to $d$ [S3, unofficial].
        v.  **Compositing:** Perform front-to-back compositing along the ray into $r$ [S3, unofficial].
        vi. **Compute Color/Opacity:** Compute color/opacity as seen from the camera [S3, unofficial].
        vii. **Project to Covered Pixels:** Project $s$ onto camera to determine covered pixels $c$ [S3, unofficial].
        viii. **Atomic Add:** For all pixels $p$ in $c$, add contribution to pixels in the resulting image using atomic add operations (requires OpenGL 4.0 or `GL_NV_shader_atomic_float` extension for floats) [S3, unofficial].
        ix.  **Determine Next Position:** Determine next position or if done (early ray termination enabled) [S3, unofficial].
    c.  **Compensate Perspective:** Compensate for perspective projection by changing light sample footprint (similar to splat footprint or cone tracing). Oversampling light rays is used as a simpler alternative [S3, unofficial].

### Specific Light Types
*   **Sun (Directional Light):**
    *   March 16 steps along the ray from the eye to the depth buffer [S1, unofficial].
    *   At each step, sample the shadow from the finest of the four cascades that contains the sun [S1, unofficial].
    *   Stop the sun ray march when the distance reaches `r_volumetricRange` (3000 units) [S1, unofficial].
*   **Lamps (Point Lights):**
    *   Determine the segment of the ray that lies inside the lamp's reach using a ray-sphere test [S1, unofficial].
    *   March 4 steps along the selected lamp ray segment [S1, unofficial].
    *   Use equi-angular sampling (Kulla & Fajardo 2012) to spread the steps by the angle at the lamp, which cancels the $1/d^2$ light falloff against the sampling density [S1, unofficial].
    *   Apply the same shade, cone, and fade terms used by `RtLightParticle`, plus one sample from the lamp's cube shadow map [S1, unofficial].
    *   **Lamp Culling:** Lamps whose upper-bound contribution is below the visible threshold are skipped. The upper-bound contribution is calculated as $C = \pi / h$, where $h$ is the distance from the ray to the lamp center [S1, unofficial].

### Scattering Model
*   Use the Henyey-Greenstein phase function for forward scattering with an anisotropy parameter $g$ of 0.5 [S1, unofficial].
*   The phase function $p(\theta)$ is defined as:
    $$ p(\theta) = \frac{1 - g^2}{4\pi (1 + g^2 - 2g \cos \theta)^{3/2}} $$
    where $\theta$ is the angle between the view vector and the light direction [S2, unofficial].

### Noise Reduction
*   Apply per-pixel interleaved-gradient jitter during sampling [S1, unofficial].
*   Smooth the result using an existing depth-aware blur pass [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Main Rendering Pass (Simplified)
def main_rendering_pass(light_source, camera, volume_texture, a_buffer):
    # Define ray from front to back (from light source)
    ray = define_ray_from_light(light_source)
    
    # Loop for all samples s along the ray
    for s in ray.samples:
        # Read/classify voxel at s to color/opacity c
        c = read_voxel(volume_texture, s)
        
        if c.opacity > 0:
            # Project s to camera space and get texture coord t
            t = project_to_camera_space(s, camera)
            
            # Calculate distance d to camera (in volume space)
            d = calculate_distance_to_camera(s, camera)
            
            # Retrieve closest layers to s based on t and d (binary search)
            # Optimize: check against first and last layer before search
            closest_layers = binary_search_a_buffer(a_buffer, t, d)
            
            # Calculate visibility v through linear interpolation
            v = interpolate_visibility(closest_layers, d)
            
            # Perform compositing front-to-back along ray into r
            r = composite_front_to_back(r, c, v)
            
            # Compute color/opacity as seen from camera
            color_opacity = compute_color_opacity(r)
            
            # Project s onto camera to determine covered pixels c
            covered_pixels = project_to_covered_pixels(s, camera)
            
            # Add contribution to pixels in resulting image (atomic add)
            for p in covered_pixels:
                atomic_add(p, color_opacity)
            
            # Determine next position or if done (early ray termination)
            if is_done(r):
                break

# Light Injection Pass (Simplified)
def light_injection_pass(light_source, volume_texture, shadow_map):
    # World Position Reconstruction
    for x, y, z in volume_texture.grid:
        world_pos = reconstruct_world_position(x, y, z)
        
        # Shadow Map Sampling
        is_shadowed = sample_shadow_map(shadow_map, world_pos, light_source)
        
        if not is_shadowed:
            # In-Scattering Calculation
            radiance = calculate_in_scattered_radiance(light_source, world_pos)
            # Store in 3D texture using image load/store
            write_to_volume(volume_texture, x, y, z, radiance)
```

## Subtopics

### Inject Direct Lighting Pass
A variant that renders the shadow map from the light's point of view and reconstructs the world position of each shadow map texel to traverse down an octree data structure, calculating reflected radiance (assuming diffuse reflection) and writing it to 3D texture bricks [S4, unofficial]. It uses a cascade shadow map with 4 cascades, with the last cascade used for light injection. Slope scale depth bias is disabled to ensure the reconstructed world position is exactly inside the voxel [S4, unofficial]. Pitfalls include shadow map instability causing flickering during cone tracing when the camera moves, and low shadow map resolution causing light injection artifacts where only a subset of voxels are filled [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The pass is performed individually for each light source with no restriction on computation order and no increase in memory usage, as light is additive [S3, unofficial].
*   **Memory Complexity:** Requires storage for the 3D texture, shadow maps, and the A-Buffer (visibility buffer) with $2^k - 1$ layers [S3, unofficial].
*   **Measured Performance:** On a GTX 1080 at 2560×1440 with `r_realtimeLighting` 2 and MSAA 4×:
    *   'm3l2 house room' scene (sun through windows/door): 0.91 ms total, with 0.41 ms for sun and blur [S1, unofficial].
    *   'Tech demo lamp hall' scene (48 colored lamps): 0.93 ms total, with 0.44 ms for sun and blur [S1, unofficial].
*   **Scaling:** Performance improved from 4.4 ms (half-size march, 8 uniform steps, 4-tap shadows) to 0.9 ms (quarter resolution, equi-angular sampling, 4 steps, real culling, 1-tap shadows) [S1, unofficial].
*   **Strengths:** Efficient for sun shafts and lamp glow; supports perspective (point lights) and orthogonal (directional lights) projections; area lights can be simulated using fish-eye lenses [S3, unofficial].
*   **Limitations:** Does not support froxel or temporal accumulation (would allow more steps); does not link density to a map's fog; does not include extinction (the scene isn't dimmed by the haze) [S1, unofficial].
*   **When to Use:** When real-time volumetric lighting for sun shafts and point lights is required, and performance is critical.
*   **When Not to Use:** When global illumination or complex scattering effects are needed, or when the scene requires extinction effects.

## Background
This approach is based on the method used by Frostbite and UE4 before the introduction of froxels [S1, unofficial]. The equi-angular sampling technique is attributed to Kulla & Fajardo (2012) [S1, unofficial]. It is implemented as part of the Wave D volumetric light feature in the OpenMOHAA project [S1, unofficial].

## Sources
[S1] feat(render): the light in the air: sun shafts and the glow round lamps, in their shadows (r_volumetric), https://github.com/ssshoshi/openmohaa/pull/116, fetched 2026-10-26 (unofficial)
[S2] Volumetric Fog Using 3D Textures in OpenGL - opengl, https://salivity.github.io/opengl/article/volumetric-fog-using-3-d-textures-in-open-gl, fetched 2026-10-27 (unofficial)
[S3] Multimodal Volume Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2015/sunden15multimodal.pdf, fetched 2026-10-29 (unofficial)
[S4] Implementing Voxel Cone Tracing, http://simonstechblog.blogspot.com/2013/01/implementing-voxel-cone-tracing.html, fetched 2026-10-27 (unofficial)
