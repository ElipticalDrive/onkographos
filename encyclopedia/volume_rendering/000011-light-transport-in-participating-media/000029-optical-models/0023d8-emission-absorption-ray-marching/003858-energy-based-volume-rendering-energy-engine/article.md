# Energy-based Volume Rendering (Energy Engine)

Also called: Energy Mode Rendering, VTK-m Energy Tracing

## Overview
Energy-based Volume Rendering is a deterministic ray-marching technique that decouples absorption/scattering from emission into distinct ray buffers, processed via a connectivity-based tracer in `ENERGY_MODE` [S1, unofficial]. It differs from standard emission-absorption ray marching by maintaining separate accumulation paths for primary optical depth and secondary emission energy, allowing for precise control over how emitted light interacts with the medium's attenuation properties [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A structured dataset containing scalar fields for absorption/scattering coefficients and optionally for emission coefficients.
*   **Output:** A rendered image where pixel values represent the accumulated energy, derived from the primary absorption buffer and the secondary emission buffer.

**Data Structures:**
The method relies on the `RenderSettings` struct to configure the tracer. The layout is defined as follows [S1, unofficial]:
*   `m_render_mode`: Enum (`RenderMode`). Controls the rendering pass type.
*   `m_scattering_type`: Enum (`ScatteringType`). Defines the scattering model.
*   `m_ray_scope`: Enum (`RayScope`). Defines the ray traversal scope.
*   `m_color_table`: String identifier for the transfer function/color map (e.g., "cool2warm").
*   `m_primary_field`: String name of the dataset array containing absorption/scattering coefficients.
*   `m_secondary_field`: String name of the dataset array containing emission coefficients. An empty string disables emission.
*   `m_volume_settings`: Struct containing volume-specific settings.
*   `m_energy_settings`: Struct containing energy-specific settings.
*   `m_path_lengths`: Boolean flag (default `false`).

**Parameters:**
*   `m_unit_scalar`: Float scaling factor for energy values during tracing. Default: `1.0` [S1, unofficial].
*   `m_render_mode`: Default: `volume` (supports `energy`) [S1, unofficial].
*   `m_scattering_type`: Default: `non_scattering` [S1, unofficial].
*   `m_ray_scope`: Default: `global_rays` [S1, unofficial].
*   `m_color_table`: Default: `"cool2warm"` [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Create a `ConnectivityProxy` from the input dataset. Set its render mode to `ENERGY_MODE` [S1, unofficial].
2.  **Primary Field Assignment:** Assign the dataset array specified by `m_primary_field` to the tracer. This field represents the absorption/scattering coefficients [S1, unofficial].
3.  **Secondary Field Assignment:** If `m_secondary_field` is not empty, assign the corresponding dataset array as the emission field. If empty, emission is disabled [S1, unofficial].
4.  **Primary Ray Buffer Setup:** Initialize the primary ray buffer (index 0) with `num_bins` channels. Set all initial values to `1.0`. This buffer tracks the transmittance (energy remaining after absorption) [S1, unofficial].
5.  **Emission Buffer Setup:** If emission is enabled (step 3), allocate an emission buffer with `num_bins` channels. Initialize all values to `0.0`. This buffer accumulates emitted energy along the ray [S1, unofficial].
6.  **Ray Traversal:** Execute the ray traversal and accumulation by calling `PartialTrace` on the tracer. The tracer uses the connectivity information to traverse the volume, updating the primary buffer based on absorption and the emission buffer based on emission contributions [S1, unofficial].

**Pseudocode:**
The following pseudocode illustrates the setup and execution flow based on the `Rover` API and `EnergyEngine` logic [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def energy_based_volume_rendering(dataset, primary_field_name, secondary_field_name=None, num_bins=1):
    """
    Performs energy-based volume rendering using the Rover/ConnectivityProxy API.
    
    Args:
        dataset: The input vtkm dataset.
        primary_field_name: Name of the array for absorption/scattering.
        secondary_field_name: Name of the array for emission. None or empty string disables emission.
        num_bins: Number of bins/channels for the ray buffers.
        
    Returns:
        The rendered result image.
    """
    # 1. Create ConnectivityProxy and set render mode
    tracer = ConnectivityProxy(dataset)
    tracer.set_render_mode(ENERGY_MODE)
    
    # 2. Assign primary field (absorption/scattering)
    tracer.assign_field(primary_field_name, field_type=PRIMARY_FIELD)
    
    # 3. Assign secondary field (emission) if provided
    if secondary_field_name is not None and secondary_field_name != "":
        tracer.assign_field(secondary_field_name, field_type=SECONDARY_FIELD)
    
    # 4. Initialize primary ray buffer (index 0)
    # Set all values to 1.0 (full transmittance at start)
    primary_buffer = create_buffer(num_bins)
    primary_buffer.fill(1.0)
    tracer.set_primary_buffer(primary_buffer)
    
    # 5. Initialize emission buffer if emission is enabled
    if secondary_field_name is not None and secondary_field_name != "":
        emission_buffer = create_buffer(num_bins)
        emission_buffer.fill(0.0)
        tracer.set_emission_buffer(emission_buffer)
    else:
        tracer.set_emission_buffer(None)
        
    # 6. Configure RenderSettings
    settings = RenderSettings()
    settings.m_render_mode = ENERGY_MODE
    settings.m_scattering_type = NON_SCATTERING
    settings.m_ray_scope = GLOBAL_RAYS
    settings.m_color_table = "cool2warm"
    settings.m_primary_field = primary_field_name
    settings.m_secondary_field = secondary_field_name if secondary_field_name else ""
    settings.m_unit_scalar = 1.0
    settings.m_path_lengths = False
    
    tracer.set_render_settings(settings)
    
    # 7. Execute ray traversal
    # PartialTrace performs the accumulation along rays
    tracer.partial_trace()
    
    # 8. Retrieve results
    result_image = tracer.get_result()
    return result_image
```

**Edge Cases and Pitfalls:**
*   **Precision:** The implementation supports both single (`vtkm::Float32`) and double (`vtkm::Float64`) precision. Switching precision requires deleting and recreating the scheduler [S1, unofficial].
*   **MPI Parallelism:** If MPI is enabled, the communication handle must be set correctly using `set_mpi_comm_handle` [S1, unofficial].
*   **Empty Secondary Field:** If `m_secondary_field` is empty, the emission buffer is not allocated, and no emission accumulation occurs [S1, unofficial].

## Subtopics

### Volumetric Liquid Rendering
A sibling method focused on simulating liquid-like optical properties. Choose this when simulating fluids with specific refraction and reflection characteristics rather than pure emission-absorption energy tracking. See [Volumetric Liquid Rendering](4017-volumetric-liquid-rendering.md).

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and the number of steps per ray (determined by volume resolution and ray length). The connectivity-based traversal may optimize empty space skipping depending on the dataset structure.
*   **Memory Complexity:** Proportional to the number of rays times the number of bins (`num_bins`) for the primary and emission buffers, plus the storage for the input dataset arrays.
*   **Strengths:** Separates absorption and emission logic, allowing for independent control and debugging of energy loss and gain. Suitable for scenarios where emission needs to be attenuated by the medium's absorption properties in a physically plausible way.
*   **Limitations:** Requires managing two separate buffers. Performance may be impacted by the overhead of the connectivity proxy and buffer management compared to simpler single-buffer ray marchers.
*   **When to Use:** Use when you need to render volumes where emission is significant and must be correctly attenuated by absorption, or when you need to analyze the separate contributions of absorption and emission.
*   **When Not to Use:** Avoid for simple opacity-based rendering where emission is negligible or handled via standard compositing. Avoid if memory bandwidth is a critical bottleneck and single-buffer accumulation suffices.

## Background
Energy-based Volume Rendering is implemented within the Rover framework, developed by Lawrence Livermore National Security, LLC [S1, unofficial]. The method leverages the `ConnectivityProxy` and `EnergyEngine` components to handle the specific requirements of energy mode rendering, separating the optical depth calculation from the emission accumulation. It is part of the broader effort to provide high-performance volume rendering capabilities within the VTK-m ecosystem [S1, unofficial].

## Sources
[S1] llnl/rover, https://github.com/llnl/rover, fetched 2026-10-26 (unofficial)
