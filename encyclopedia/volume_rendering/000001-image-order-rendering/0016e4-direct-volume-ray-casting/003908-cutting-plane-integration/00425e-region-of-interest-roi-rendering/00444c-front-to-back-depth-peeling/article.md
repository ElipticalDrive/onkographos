# Front-to-Back Depth Peeling

Also called: FtBDP

## Overview
Front-to-Back Depth Peeling (FtBDP) is a technique for Region of Interest (ROI) rendering that defines an ROI as a 3D envelope by combining polygon-based depth peeling with volume ray casting [S1, unofficial]. It solves the problem of visualizing internal volume features bounded by an arbitrary surface by breaking the ray casting process into stages synchronized with intermediate peeling results [S1, unofficial]. Unlike standard depth peeling which only handles polygonal models, FtBDP interleaves polygon rendering and volume ray casting to correctly handle volume samples both inside and outside the envelope [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Polygonal envelope geometry (tagged with a vertex attribute key), volume dataset, MIDA visualization settings for inside/outside regions, initial ray starting positions (optional, for optimization) [S1, unofficial].
*   **Outputs:** Final composited image with ROI visualization [S1, unofficial].

**Data Structures**
*   **Depth Buffer:** Stores depth values of fragments for tracking peeled surfaces [S1, unofficial].
*   **Color Buffer:** Stores accumulated color from previous peeling and ray casting passes [S1, unofficial].
*   **Ray Position Buffer:** Stores 3D intersection positions of rays with peeled envelope surfaces from previous steps [S1, unofficial].
*   **Envelope Geometry Tag:** A vertex attribute key distinguishing envelope polygons from regular scene geometry [S1, unofficial].

**Numbered Steps**
The method proceeds in a sequence of peeling and ray casting stages [S1, unofficial]:
1.  **Initial Peeling:** Render the front-most polygonal geometry (the front surface of the envelope) normally. Store the depths of all corresponding fragments for every pixel in the depth buffer [S1, unofficial].
2.  **First Ray Casting Stage:** Initiate MIDA ray casting. Pass the color and depth buffers from Step 1 as textures. Perform a depth test; render only volume parts that fall *outside* the envelope. Blend the colors from the previous peeling pass with the ray casting results. Store the positions of rays intersecting the peeled surface (from Step 1) in a separate buffer [S1, unofficial].
3.  **Next Peeling Stage:** Initiate the next envelope peeling stage. Pass the depth results of the previous peeling pass as a texture. Choose new fragments that satisfy the depth test condition (depth > stored texture values). Store the depth values of all newly chosen fragments in the depth buffer [S1, unofficial].
4.  **Subsequent Ray Casting Stage:** Initiate MIDA ray casting. Pass the color and depth buffers from Step 3, as well as the ray positions and accumulated color results from Step 2, as textures. Initialize ray starting positions from the stored values. Perform the depth test. Render sampled values inside or outside the envelope using different MIDA settings. Blend the colors from the previous peeling pass and the previous ray casting pass with the current ray casting results. Store the positions of rays intersecting the newly peeled surface (from Step 3) in the ray position buffer [S1, unofficial].
5.  **Iteration:** Repeat Steps 3 and 4 until the envelope peeling stage reports 0 newly chosen fragments [S1, unofficial].

**Formulas**
*   Fragment selection condition: $d_{frag} > d_{texture}$, where $d_{frag}$ is the depth of a candidate fragment and $d_{texture}$ is the stored depth value from the previous peeling pass [S1, unofficial].
*   Inside/Outside determination: A ray sample is classified as *inside* if it lies between a front-facing and a back-facing envelope fragment; otherwise it is *outside* [S1, unofficial].

**Parameters**
*   **MIDA Settings:** Visualization parameters for rendering volume samples inside vs. outside the envelope [S1, unofficial].
*   **Depth Test Condition:** Comparison operator (e.g., `>`) to select fragments behind previously peeled surfaces [S1, unofficial].
*   Typical values are not given in the sources.

**Edge Cases and Pitfalls**
*   **Open Surfaces:** Extreme cases where the envelope geometry crosses metaball grid boundaries and no longer forms a closed surface are handled correctly [S1, unofficial].
*   **Overdraw:** The full geometry of the scene must be drawn and tested in each polygon peeling pass, leading to overdraw [S1, unofficial].
*   **Shader Overhead:** There is overhead associated with shader code initialization and execution per stage [S1, unofficial].
*   Precision and boundary handling are not specified beyond the open-surface case [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def front_to_back_depth_peeling(envelope_geometry, volume_data, inside_settings, outside_settings):
    # Data structures
    depth_buffer = []
    color_buffer = []
    ray_position_buffer = []

    # Step 1: Initial Peeling
    # Render front-most polygonal geometry (envelope front surface)
    # Store depths of all corresponding fragments for every pixel
    depth_buffer = render_front_surface(envelope_geometry)

    # Step 2: First Ray Casting Stage
    # Pass color and depth buffers from Step 1 as textures
    # Perform depth test; render only volume parts outside the envelope
    # Blend previous pass colors with ray casting results
    # Store ray intersection positions in a separate buffer
    ray_cast_result_2 = cast_volume_rays(
        volume_data, ray_origin=None, ray_direction=None,
        depth_texture=depth_buffer, color_texture=color_buffer,
        settings=outside_settings, mode="outside"
    )
    color_buffer = blend_buffers(color_buffer, ray_cast_result_2.color)
    ray_position_buffer = ray_cast_result_2.intersections

    # Steps 3 & 4: Iterative Peeling and Ray Casting
    while True:
        # Step 3: Next Peeling Stage
        # Pass depth results of previous peeling pass as a texture
        # Choose new fragments satisfying depth test condition (depth > stored texture values)
        # Store new fragment depths in the depth buffer
        new_fragments = select_fragments(envelope_geometry, depth_buffer, condition="depth > texture")
        if not new_fragments:
            break
        depth_buffer = update_depth_buffer(depth_buffer, new_fragments)

        # Step 4: Subsequent Ray Casting Stage
        # Pass color and depth buffers from Step 3, and ray positions/accumulated colors from Step 2 as textures
        # Initialize ray starting positions from stored values
        # Perform depth test. Render sampled values inside/outside envelope using different MIDA settings
        # Blend previous peeling and ray casting results with current results
        # Store new ray intersection positions
        ray_cast_result_4 = cast_volume_rays(
            volume_data, ray_origin=ray_position_buffer, ray_direction=None,
            depth_texture=depth_buffer, color_texture=color_buffer,
            settings_inside=inside_settings, settings_outside=outside_settings
        )
        color_buffer = blend_buffers(color_buffer, ray_cast_result_4.color)
        ray_position_buffer = ray_cast_result_4.intersections

    return color_buffer

# Helper functions (standard operations)
def render_front_surface(geometry):
    # Renders front-facing envelope polygons, returns depth buffer
    return [fragment.depth for fragment in geometry.front_faces]

def cast_volume_rays(volume, ray_origin, ray_direction, depth_texture, color_texture, settings, mode="outside", settings_inside=None, settings_outside=None):
    # Casts rays through volume, applies depth test against depth_texture,
    # samples volume using settings based on mode/face orientation,
    # returns color buffer and intersection positions
    color = []
    intersections = []
    # Standard ray marching logic would go here
    return type('Result', (), {'color': color, 'intersections': intersections})()

def blend_buffers(prev_color, new_color):
    # Alpha blends new_color over prev_color
    return prev_color

def select_fragments(geometry, depth_texture, condition):
    # Selects fragments where depth satisfies condition against depth_texture
    return []

def update_depth_buffer(depth_buffer, fragments):
    # Updates depth buffer with new fragment depths
    return depth_buffer
```

## Operating characteristics
*   **Time Complexity:** The method involves multiple passes. Each peeling pass requires drawing the full geometry and performing depth tests. Each ray casting pass involves volume sampling and blending. The number of passes depends on the complexity of the envelope depth layers [S1, unofficial].
*   **Memory Complexity:** Requires storage for depth buffers, color buffers, and ray position buffers [S1, unofficial].
*   **Performance:** No additional GPU computational workload for ray casting is introduced by keeping and initializing ray positions between stages [S1, unofficial]. However, overhead exists from shader initialization/execution per stage and overdraw from the depth peeling algorithm [S1, unofficial].
*   **Strengths:** Allows for complex ROI definitions using implicit surfaces (metaballs) and handles open envelope surfaces. Correctly blends polygonal and volumetric data [S1, unofficial].
*   **Limitations:** Overdraw and shader overhead can impact performance. Requires careful management of buffer passing between passes [S1, unofficial].

## Background
Front-to-Back Depth Peeling was introduced in the context of a 3D painting metaphor for volume image exploration [S1, unofficial]. It extends the concept of depth peeling, which traditionally deals only with polygonal models, to integrate with volume ray casting [S1, unofficial]. The technique was developed to support an envelope definition using GPU-accelerated metaballs, allowing for growing and editing of the ROI in a manner similar to placing brush strokes [S1, unofficial]. The term "Front-to-Back" refers to the order of peeling, which matches the front-to-back order of ray casting [S1, unofficial].

## Sources
[S1] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
