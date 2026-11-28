# Simulated Fluorescence Process (SFP) Renderer
Also called: SFP Renderer, SFP volume rendering algorithm

## Overview
The Simulated Flucescence Process (SFP) Renderer is a volume rendering algorithm that interprets image intensity as a distribution of fluorescent dye, modeling physical light-matter interaction to render scenes as they would appear under real fluorescence microscopy conditions [S1]. Unlike standard isosurface rendering or methods requiring surface detection and segmentation, SFP operates directly on 3D data by simulating two distinct light transport stages: excitation by a distant light source and subsequent emission propagation to the viewer [S1]. This approach allows for the visualization of internal structures with realistic shadowing and absorption effects without preprocessing [S1].

## How it works
The SFP algorithm simulates the physical process of fluorescence in two sequential stages. It treats the input volume data as a distribution of fluorescent dye where intensity values determine absorption and emission properties.

**Inputs and Outputs**
*   **Input:** A 3D volume dataset (typically multi-channel microscopy data) where intensity represents dye concentration.
*   **Input:** A light source located at infinity (directional).
*   **Input:** A virtual table plane below the object.
*   **Input:** Penetration Depth parameter (scalar).
*   **Output:** A 2D image representing the scene as viewed by an observer.

**Data Structures**
The algorithm processes the volume as a grid of voxels. Each voxel $v$ at position $\mathbf{p}$ contains intensity values for one or more channels. Each channel is interpreted as a distinct dye with specific properties: transparency, color, and brightness [S1].

**Parameters**
*   **Penetration Depth ($d_p$):** A scalar parameter that sets the absorption and emission transparencies for all channels simultaneously [S1].
    *   Low values: Objects appear solid with hard shadows; excitation light is absorbed immediately.
    *   High values: Light penetrates deeper, revealing internal structures with softer shadows [S1].

**Algorithm Steps**

1.  **Excitation Stage (Lighting):**
    The fluorescent distribution is illuminated by a light source located at infinity [S1].
    *   For each voxel in the volume, calculate the excitation intensity based on its orientation relative to the light source.
    *   Areas facing the light source are more excited because the dye absorbs the excitation light [S1].
    *   Each voxel in the light beam is affected by shadowing from its predecessors along the light ray. Let $L_{exc}(\mathbf{p})$ be the excitation intensity at position $\mathbf{p}$. This is determined by integrating the extinction of the excitation light through the volume from the light source to $\mathbf{p}$ [S1].

2.  **Emission Stage (Rendering):**
    Light emitted by the object and the table below it propagates towards the viewer [S1].
    *   Each voxel emits light proportional to its excitation intensity $L_{exc}(\mathbf{p})$ and its channel-specific brightness [S1].
    *   The emitted light travels along the view ray towards the camera.
    *   As the light passes through the dye, it gets absorbed. Objects can be obscured from view by other objects (shadowing from predecessors along the view ray) [S1].
    *   The final pixel color is the accumulated emission from all voxels along the view ray, attenuated by the absorption of the dye [S1].

**Pseudocode**
The following pseudocode outlines the core logic. Note that specific integration formulas for optical depth are not provided in the source; standard Beer-Lambert integration is implied by the description of shadowing and absorption [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_sfp(volume, light_direction, view_direction, penetration_depth, table_plane):
    """
    Renders a volume using the Simulated Fluorescence Process.
    
    Args:
        volume: 3D array of intensity values (multi-channel supported).
        light_direction: Unit vector for the light source at infinity.
        view_direction: Unit vector for the camera view.
        penetration_depth: Scalar controlling absorption/emission transparency.
        table_plane: Plane definition for the shadow-casting table.
        
    Returns:
        2D image array.
    """
    # Initialize output image
    image = zeros_like(view_direction.xy) 
    
    # Pre-calculate channel properties based on penetration depth
    # Note: Exact mapping of penetration_depth to absorption coefficient is not specified [S1]
    # [G] Assume absorption coefficient sigma_a is inversely proportional to penetration_depth
    absorption_coeff = 1.0 / penetration_depth
    
    # Step 1: Excitation Stage
    # Calculate excitation intensity for each voxel, accounting for shadowing from light
    excitation_map = zeros_like(volume)
    
    # Ray march along light_direction for each voxel to determine shadowing
    # This is a simplified representation; actual implementation requires efficient ray casting
    for voxel in volume:
        # Calculate transmittance from light source to this voxel
        # T_exc = exp(-integral(sigma_a * density) along light ray)
        # [G] Standard exponential attenuation is implied by "absorbs excitation light"
        transmittance = calculate_transmittance(voxel.position, light_direction, volume, absorption_coeff)
        
        # Excitation is proportional to incident light and local dye concentration
        # [G] Assuming linear relationship between intensity and dye concentration
        excitation_map[voxel] = voxel.intensity * transmittance

    # Step 2: Emission Stage
    # Ray march along view_direction for each pixel
    for pixel in image:
        accumulated_radiance = 0.0
        current_transmittance = 1.0
        
        # March from near to far along view ray
        for voxel in volume_along_ray(pixel, view_direction):
            # Emitted light from this voxel
            # [G] Emission is proportional to excitation and channel brightness
            emitted_light = excitation_map[voxel] * voxel.channel_brightness
            
            # Add contribution to pixel, attenuated by remaining path
            # dL = L_e * exp(-tau) * dt
            accumulated_radiance += emitted_light * current_transmittance
            
            # Update transmittance for subsequent voxels (absorption of emission light)
            # [G] Standard exponential attenuation
            current_transmittance *= exp(-absorption_coeff * voxel.density * voxel.step_size)
            
            # Check for table intersection to add table shadow/emission if applicable
            # [S1] "Light emitted by the object and a table below it propagates towards the viewer"
            if voxel.is_on_table:
                table_emission = get_table_emission(voxel)
                accumulated_radiance += table_emission * current_transmittance
                
        image[pixel] = accumulated_radiance
        
    return image

def calculate_transmittance(start_pos, direction, volume, absorption_coeff):
    """
    Calculates the transmittance of light from the source to a point.
    """
    # [G] Standard ray marching for optical depth
    optical_depth = 0.0
    # Ray march from start_pos backwards along -direction to infinity or volume boundary
    # Integrate absorption coefficient along the path
    # [S1] "Each voxel in the light beam is affected by shadowing from its predecessors"
    for voxel in volume_along_ray(start_pos, -direction):
        optical_depth += absorption_coeff * voxel.density * voxel.step_size
    return exp(-optical_depth)
```

**Edge Cases and Pitfalls**
*   **No Surface Detection Required:** The algorithm does not require surface detection, sharp gradients, or segmentation, making it suitable for direct visualization of 3D microscopy data [S1].
*   **Multi-Channel Interpretation:** Each channel of a multi-channel image is interpreted as a dye with different properties (transparencies, color, brightness) [S1]. Incorrect channel mapping may lead to unrealistic colors.
*   **Penetration Depth Sensitivity:** Low penetration depth results in solid objects with hard shadows, potentially hiding internal details. High penetration depth reveals internal structures but may reduce contrast [S1].

## Subtopics

### SFP Isosurface Integration
An isosurface added to the SFP scene that acts as a completely impenetrable barrier for both excitation and emission light [S1]. The isosurface casts hard shadows on the SFP material beneath it and on the table [S1]. It is partially obscured by SFP material located in front of it [S1]. The isosurface can also cast shadows on itself, allowing for complex lighting interactions even without fluorescent material [S1]. This feature creates a greater sense of space and depth compared to standard surface rendering [S1].

## Operating characteristics
*   **Performance:** The performance of the SFP Renderer greatly benefits from GPU acceleration [S1].
*   **Strengths:**
    *   Does not require surface detection, sharp gradients, or segmentation [S1].
    *   Suitable for direct visualization of 3D microscopy data [S1].
    *   Provides realistic lighting and shadowing effects (excitation and emission) [S1].
    *   Supports multi-channel data interpretation [S1].
    *   Supports cutting planes (up to 6) for internal visualization [S1].
*   **Limitations:**
    *   Relies on the assumption that image intensity maps to fluorescent dye distribution [S1].
    *   Specific performance metrics (FPS, memory usage) are not provided in the sources [S1].
*   **When to use:** Direct visualization of 3D microscopy data where realistic fluorescence appearance is desired without preprocessing [S1].
*   **When not to use:** When surface geometry is the primary focus and fluorescence effects are irrelevant [S1].

## Background
The SFP Renderer is part of the Huygens software suite by Scientific Volume Imaging (SVI) [S1]. It was introduced in Huygens 19.10, which added features like cutting planes [S1]. The algorithm is designed for use with data acquired from microscopes, such as the Leica Digital Light Sheet microscope, and is often used in conjunction with deconvolution algorithms like CMLE [S1]. It is used in biological research, as evidenced by testimonials from institutions like Maastruth University and EMBL Heidelberg [S1].

## Sources
[S1] SFP Volume Renderer, https://svi.nl/Huygens-SFP-Renderer, fetched 2026-10-29
