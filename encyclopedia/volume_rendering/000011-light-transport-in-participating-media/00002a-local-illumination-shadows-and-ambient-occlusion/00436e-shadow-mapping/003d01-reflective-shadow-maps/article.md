# Reflective Shadow Maps

Also called: Virtual Point Lights (VPL) via Shadow Maps

## Overview
Reflective Shadow Maps (RSM) approximate single-bounce indirect illumination in real-time rendering by reusing data from a shadow map pass [S1, unofficial]. Unlike standard shadow mapping, which only determines visibility for direct lighting, RSM stores surface properties alongside depth to treat texels as virtual point lights during the final shading pass [S1, unofficial]. This distinguishes it from Light Propagation Volumes, which discretize the scene into 3D cells encoded with spherical harmonics, by operating directly on screen-space texels [S1, unofficial].

## How it works
**Inputs:** Scene geometry and light source(s) [G].
**Outputs:** Final shaded image with direct and single-bounce indirect lighting [S1, unofficial].
**Data Structures:** Reflective Shadow Map (RSM) textures containing depth, albedo, normal, and direct illumination per texel [S1, unofficial].
**Steps:**
1. **Direct Lighting Pass:** Render the scene from the light's point of view, storing depth, albedo, normal, and direct illumination into the RSM textures [S1, unofficial].
2. **Virtual Point Light (VPL) Generation:** Treat texels of the RSM as point lights to provide single-bounce indirect illumination [S1, unofficial].
3. **Screen-Space Splatting (Optimization):** Reverse the processing by creating lights based on the entire map and splatting them in screen-space [S1, unofficial].
4. **Final Shading:** Combine the direct lighting with the accumulated indirect lighting from the splatted VPLs to produce the final image [S1, unofficial].
**Formulas:** The exact irradiance calculation is not given in the sources. The contribution of each VPL is derived from its stored direct illumination and albedo, projected onto the screen [S1, unofficial].
**Parameters:** RSM resolution determines the number of potential VPLs [S1, unofficial]. Heuristic thresholds are used to select which texels become VPLs to balance performance and quality [S1, unofficial].
**Edge Cases and Pitfalls:** Using all texels as VPLs leads to excessive performance cost, requiring heuristics to choose regions of the map [S1, unofficial]. An insufficient number of selected lights causes temporal stability issues and flickering when lights or cameras move [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_reflective_shadow_maps(scene, light, camera):
    # Step 1: Render RSM from light's perspective
    rsm_depth = render_shadow_map_depth(scene, light)
    rsm_albedo = render_shadow_map_albedo(scene, light)
    rsm_normal = render_shadow_map_normal(scene, light)
    rsm_direct_light = render_shadow_map_direct_light(scene, light)

    # Step 2: Select VPLs from RSM
    # Heuristic: Select texels with direct illumination above a threshold to avoid excessive cost [S1, unofficial]
    vpl_list = []
    threshold = 0.0  # [G] Implementation-dependent threshold
    for y in range(rsm_depth.height):
        for x in range(rsm_depth.width):
            if rsm_direct_light[y, x] > threshold:
                world_pos = texel_to_world_position(x, y, rsm_depth[y, x], light.view_matrix)
                vpl_list.append({
                    'position': world_pos,
                    'normal': rsm_normal[y, x],
                    'albedo': rsm_albedo[y, x],
                    'intensity': rsm_direct_light[y, x]
                })

    # Step 3: Splat VPLs to screen space
    # Reverse processing: project VPLs and splat contribution to covered pixels [S1, unofficial]
    indirect_buffer = zero_buffer(camera.resolution)
    for vpl in vpl_list:
        screen_pos = world_to_screen(vpl.position, camera.view_matrix, camera.projection_matrix)
        radius = calculate_influence_radius(vpl.intensity)
        for dy in range(-radius, radius + 1):
            for dx in range(-radius, radius + 1):
                px, py = screen_pos.x + dx, screen_pos.y + dy
                if 0 <= px < camera.width and 0 <= py < camera.height:
                    dist_sq = distance_squared(vpl.position, pixel_world_position(px, py, camera))
                    if dist_sq > 0:
                        contribution = vpl.intensity * vpl.albedo / dist_sq
                        indirect_buffer[py, px] += contribution

    # Step 4: Final composite
    direct_image = render_direct_lighting(scene, light, camera)
    final_image = direct_image + indirect_buffer
    return final_image

# Helper definitions (standard operations)
def render_shadow_map_depth(scene, light): ... # [G] Standard depth pass
def render_shadow_map_albedo(scene, light): ... # [G] Standard albedo pass
def render_shadow_map_normal(scene, light): ... # [G] Standard normal pass
def render_shadow_map_direct_light(scene, light): ... # [G] Standard direct lighting pass
def texel_to_world_position(x, y, depth, view_matrix): ... # [G] Inverse view-projection transform
def world_to_screen(pos, view_matrix, proj_matrix): ... # [G] View-projection transform
def calculate_influence_radius(intensity): ... # [G] Intensity-based falloff
def distance_squared(p1, p2): return sum((a-b)**2 for a, b in zip(p1, p2))
def pixel_world_position(px, py, camera): ... # [G] Ray origin reconstruction
def zero_buffer(resolution): return [[0.0]*resolution.width for _ in range(resolution.height)]
def render_direct_lighting(scene, light, camera): ... # [G] Standard direct lighting pass
```

## Subtopics

### Beer Shadow Maps
Beer Shadow Maps are a variant used primarily for volumetric cloud shadowing in Unreal Engine [S2]. They use cascaded shadow maps to support far shadow distances for clouds and ground casting [S2]. Compared to ray-marched shadows, they are faster and support far distances but lack volumetric self-shadow color and accuracy [S2]. They are recommended for console platforms like Xbox One and PlayStation 4 [S2]. Parameters include `Cloud Shadow Ray Sample Count Scale` (drives sample count), `Cloud Shadow Map Resolution Scale` (drives resolution and sunlight shaft cost), and `Cloud Shadow Extent` (reduces map radius for sharper results) [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Performance scales with the number of selected VPLs and the splatting resolution [S1, unofficial].
*   **Memory Complexity:** Requires storing additional textures (albedo, normal, direct light) in the RSM, increasing memory usage compared to standard shadow mapping [S1, unofficial].
*   **Strengths:** Efficient approximation of single-bounce indirect illumination; compatible with existing shadow map infrastructure; real-time performance with heuristics [S1, unofficial].
*   **Limitations:** Flickering due to temporal instability; limited to single-bounce; performance sensitive to VPL count [S1, unofficial].
*   **When to Use:** Real-time applications requiring diffuse interreflection where full global illumination is too expensive [G].
*   **When Not to Use:** Scenarios requiring multi-bounce GI or accurate volumetric shadows [G].

## Background
Reflective Shadow Maps were inspired by the Virtual Point Lights technique used in the production of *Shrek 2* [S1, unofficial]. Tabellion and Lamorlette utilized a direct lighting pass and cached lighting to create one-bounce indirect lighting [S1, unofficial]. Dachsbacher and Stamminger formalized this into the Reflective Shadow Maps technique, rendering from the light's perspective and storing surface properties to treat texels as point lights [S1, unofficial]. The method was optimized by reversing the processing to splat lights in screen-space [S1, unofficial]. Kaplanyan's Light Propagation Volumes also aim for first-bounce GI but use 3D cells and spherical harmonics, contrasting with RSM's texel-based VPLs [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Volumetric Cloud Component in Unreal Engine | Unreal Engine 5.8 Documentation | Epic Developer Community, https://dev.epicgames.com/documentation/en-us/unreal-engine/volumetric-cloud-component-in-unreal-engine, fetched 2026-10-30
