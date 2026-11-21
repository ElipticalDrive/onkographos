# Reflective Shadow Maps

Also called: Virtual Point Lights (VPL) via Shadow Maps

## Overview
Reflective Shadow Maps (RSM) are a technique for approximating single-bounce indirect illumination (global illumination) in real-time rendering by reusing data from a standard shadow map pass [S1, unofficial]. Unlike standard shadow mapping which only determines visibility for direct lighting, RSM stores surface properties (albedo, normal, direct illumination) alongside depth to treat texels as virtual point lights during the final shading pass [S1, unofficial]. This distinguishes it from its siblings by providing diffuse interreflection without the volumetric storage overhead of Light Propagation Volumes or the ray-tracing cost of path tracing [S1, unofficial].

## How it works
**Inputs:**
*   Scene geometry.
*   Light source(s) (typically directional or point lights).
*   Camera position and view matrix.

**Outputs:**
*   Final shaded image with direct and single-bounce indirect lighting.
*   Reflective Shadow Map (RSM) textures containing depth, albedo, normal, and direct illumination per texel.

**Data Structures:**
The RSM is a texture array or a set of textures rendered from the light's perspective. Each texel stores:
1.  **Depth:** Distance from the light to the surface.
2.  **Albedo:** Surface diffuse color ($C_{diff}$).
3.  **Normal:** Surface normal vector ($\mathbf{n}$) in light space or world space.
4.  **Direct Illumination:** The direct lighting contribution ($L_{dir}$) at that point, including light intensity and attenuation.

**Steps:**
1.  **Direct Lighting Pass:** Render the scene from the light's point of view. Store the depth, albedo, normal, and calculated direct illumination for each visible surface fragment into the RSM textures [S1, unofficial].
2.  **Virtual Point Light (VPL) Generation:** Treat every texel in the RSM as a potential point light source. Each texel $i$ emits light based on its stored direct illumination and albedo [S1, unofficial].
3.  **Screen-Space Splatting (Optimization):** Instead of tracing rays from the camera to every texel (which is expensive), reverse the process. Project the VPLs onto the screen and "splat" their contribution into the frame buffer or a dedicated indirect lighting buffer [S1, unofficial]. This involves calculating the irradiance contribution of each VPL to the pixels it covers.
4.  **Final Shading:** Combine the direct lighting (from the initial pass) with the accumulated indirect lighting from the splatted VPLs to produce the final image [S1, unofficial].

**Formulas:**
The indirect illumination $L_{ind}$ at a pixel is approximated by summing contributions from VPLs. For a VPL $i$ with direct illumination $L_{dir,i}$ and albedo $C_{diff,i}$, the contribution to a surface point with normal $\mathbf{n}_p$ and view direction $\mathbf{v}$ is often approximated using a simplified diffuse reflection model:
$$ L_{ind} = \sum_{i} \frac{L_{dir,i} \cdot C_{diff,i}}{d_i^2} \cdot f_r(\mathbf{n}_p, \mathbf{n}_i, \mathbf{v}) $$
Where $d_i$ is the distance from the VPL to the surface point, and $f_r$ is the BRDF (often simplified to a cosine term $\max(0, \mathbf{n}_p \cdot \mathbf{n}_i)$ for diffuse surfaces) [S1, unofficial].

**Parameters:**
*   **RSM Resolution:** Determines the number of VPLs. Higher resolution increases accuracy but also performance cost [S1, unofficial].
*   **Heuristic Thresholds:** Used to select which texels become VPLs to balance performance and quality [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Performance Cost:** Using all texels as VPLs is computationally prohibitive. Heuristics must be employed to select a subset of texels (e.g., based on illumination intensity or spatial clustering) [S1, unofficial].
*   **Temporal Stability:** If the number of selected VPLs is insufficient or changes abruptly between frames, flickering occurs, especially when lights or cameras move [S1, unofficial].
*   **Precision:** Depth precision in the RSM must be sufficient to avoid z-fighting and incorrect VPL placement, particularly for distant surfaces.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_reflective_shadow_maps(scene, light, camera):
    # Step 1: Render RSM from light's perspective
    rsm_depth = render_depth_from_light(scene, light)
    rsm_albedo = render_albedo_from_light(scene, light)
    rsm_normal = render_normal_from_light(scene, light)
    rsm_direct_light = render_direct_lighting(scene, light)

    # Step 2: Select VPLs from RSM
    # Heuristic: Select texels with direct illumination above a threshold
    vpl_list = []
    for texel in rsm_depth:
        if rsm_direct_light[texel] > threshold:
            vpl = {
                'position': texel_to_world_position(texel, rsm_depth, light.view_matrix),
                'normal': rsm_normal[texel],
                'albedo': rsm_albedo[texel],
                'intensity': rsm_direct_light[texel]
            }
            vpl_list.append(vpl)

    # Step 3: Splat VPLs to screen space
    indirect_buffer = zero_buffer(camera.resolution)
    for vpl in vpl_list:
        # Project VPL position to screen space
        screen_pos = world_to_screen(vpl.position, camera.view_matrix, camera.projection_matrix)
        
        # Calculate influence radius based on intensity and distance
        radius = calculate_influence_radius(vpl.intensity)
        
        # Splat contribution to nearby pixels
        for pixel in pixels_near(screen_pos, radius):
            # Simple diffuse approximation
            contribution = vpl.intensity * vpl.albedo / (distance_squared(vpl.position, pixel_world_position(pixel)))
            # Add cosine term if normal is available at pixel
            indirect_buffer[pixel] += contribution

    # Step 4: Final composite
    direct_image = render_direct_lighting(scene, light, camera)
    final_image = direct_image + indirect_buffer
    return final_image
```

## Subtopics

### Beer Shadow Maps
Beer Shadow Maps are a variant used primarily for volumetric cloud shadowing in Unreal Engine. They use cascaded shadow maps to support far shadow distances for clouds and ground casting [S2]. Compared to ray-marched shadows, they are faster and support far distances but lack volumetric self-shadow color and accuracy [S2]. They are recommended for console platforms like Xbox One and PlayStation 4 [S2]. Parameters include `Cloud Shadow Ray Sample Count Scale` (drives sample count), `Cloud Shadow Map Resolution Scale` (drives resolution and sunlight shaft cost), and `Cloud Shadow Extent` (reduces map radius for sharper results) [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of selected VPLs and the splatting resolution. Naive ray tracing per VPL is $O(N \cdot M)$ where $N$ is texels and $M$ is pixels. Screen-space splatting reduces this but still scales with VPL count and influence radius [S1, unofficial].
*   **Memory Complexity:** Requires storing additional textures (albedo, normal, direct light) in the RSM, increasing memory usage compared to standard shadow mapping [S1, unofficial].
*   **Strengths:** Efficient approximation of single-bounce indirect illumination; compatible with existing shadow map infrastructure; real-time performance with heuristics [S1, unofficial].
*   **Limitations:** Flickering due to temporal instability; limited to single-bounce; performance sensitive to VPL count; lacks volumetric self-shadowing color [S1, unofficial][S2].
*   **When to Use:** Real-time applications requiring diffuse interreflection where full global illumination is too expensive.
*   **When Not to Use:** Scenarios requiring multi-bounce GI, accurate volumetric shadows, or high temporal stability without significant post-processing.

## Background
Reflective Shadow Maps were inspired by the Virtual Point Lights technique used in the production of *Shrek 2* [S1, unofficial]. Tabellion and Lamorlette utilized a direct lighting pass and cached lighting to create one-bounce indirect lighting [S1, unofficial]. Dachsbacher and Stamminger formalized this into the Reflective Shadow Maps technique, rendering from the light's perspective and storing surface properties to treat texels as point lights [S1, unofficial]. The method was optimized by reversing the processing to splat lights in screen-space [S1, unofficial]. Kaplanyan's Light Propagation Volumes also aim for first-bounce GI but use 3D cells and spherical harmonics, contrasting with RSM's texel-based VPLs [S1, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Volumetric Cloud Component in Unreal Engine | Unreal Engine 5.8 Documentation | Epic Developer Community, https://dev.epicgames.com/documentation/en-us/unreal-engine/volumetric-cloud-component-in-unreal-engine, fetched 2026-10-30
