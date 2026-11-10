# Energy-based Volume Rendering (Energy Engine)

Also called: Energy Mode Rendering, VTK-m Energy Tracing

## Overview
Energy-based Volume Rendering is a deterministic ray-marching technique that separates absorption/scattering (primary field) and emission (secondary field) into distinct ray buffers, processed in `ENERGY_MODE` by a connectivity-based ray tracer [S1, unofficial]. It differs from standard emission-absorption ray marching by maintaining separate accumulation paths for primary optical depth and secondary emission energy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A structured dataset containing scalar fields for absorption/scattering coefficients and optionally for emission coefficients [S1, unofficial].
*   **Output:** A rendered image where pixel values represent the accumulated energy, derived from the primary absorption buffer and the secondary emission buffer [S1, unofficial].

**Data Structures:**
The method relies on the `RenderSettings` struct to configure the tracer [S1, unofficial]. The layout is defined as follows [S1, unofficial]:
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
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Self-contained implementation matching the Rover/ConnectivityProxy API
class ConnectivityProxy:
    def __init__(self, dataset):
        self.dataset = dataset
        self.render_mode = None
        self.primary_field = None
        self.secondary_field = None
        self.primary_buffer = None
        self.emission_buffer = None
        self.render_settings = None
    def set_render_mode(self, mode): self.render_mode = mode
    def assign_field(self, name, field_type):
        if field_type == "PRIMARY": self.primary_field = name
        else: self.secondary_field = name
    def set_primary_buffer(self, buf): self.primary_buffer = buf
    def set_emission_buffer(self, buf): self.emission_buffer = buf
    def set_render_settings(self, settings): self.render_settings = settings
    def partial_trace(self): pass
    def get_result(self): return self.dataset

def energy_based_volume_rendering(dataset, primary_field_name, secondary_field_name=None, num_bins=1):
    # 1. Create ConnectivityProxy from input dataset and set render mode to ENERGY_MODE
    tracer = ConnectivityProxy(dataset)
    tracer.set_render_mode("energy")

    # 2. Assign primary dataset array (absorption/scattering) to the tracer
    tracer.assign_field(primary_field_name, "PRIMARY")

    # 3. Assign secondary dataset array as emission field if provided
    if secondary_field_name is not None and secondary_field_name != "":
        tracer.assign_field(secondary_field_name, "SECONDARY")
    else:
        tracer.secondary_field = ""

    # 4. Initialize primary ray buffer (index 0) with num_bins channels, set all to 1.0
    primary_buffer = [1.0] * num_bins
    tracer.set_primary_buffer(primary_buffer)

    # 5. Initialize emission buffer with num_bins channels, set all to 0.0 if enabled
    if tracer.secondary_field != "":
        emission_buffer = [0.0] * num_bins
        tracer.set_emission_buffer(emission_buffer)
    else:
        tracer.set_emission_buffer(None)

    # 6. Configure RenderSettings
    settings = {
        "m_render_mode": "energy",
        "m_scattering_type": "non_scattering",
        "m_ray_scope": "global_rays",
        "m_color_table": "cool2warm",
        "m_primary_field": primary_field_name,
        "m_secondary_field": tracer.secondary_field,
        "m_unit_scalar": 1.0,
        "m_path_lengths": False
    }
    tracer.set_render_settings(settings)

    # 7. Execute ray traversal and accumulation
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
*   **Time Complexity:** Linear with respect to the number of rays and the number of steps per ray (determined by volume resolution and ray length) [G].
*   **Memory Complexity:** Proportional to the number of rays times the number of bins (`num_bins`) for the primary and emission buffers, plus the storage for the input dataset arrays [S1, unofficial].
*   **Strengths:** Separates absorption and emission logic, allowing for independent control and debugging of energy loss and gain [S1, unofficial]. Suitable for scenarios where emission needs to be attenuated by the medium's absorption properties in a physically plausible way [S1, unofficial].
*   **Limitations:** Requires managing two separate buffers. Performance may be impacted by the overhead of the connectivity proxy and buffer management compared to simpler single-buffer ray marchers [S1, unofficial].
*   **When to Use:** Use when you need to render volumes where emission is significant and must be correctly attenuated by absorption, or when you need to analyze the separate contributions of absorption and emission [S1, unofficial].
*   **When Not to Use:** Avoid for simple opacity-based rendering where emission is negligible or handled via standard compositing [G]. Avoid if memory bandwidth is a critical bottleneck and single-buffer accumulation suffices [G].

## Background
Energy-based Volume Rendering is implemented within the Rover framework, developed by Lawrence Livermore National Security, LLC [S1, unofficial]. The method leverages the `ConnectivityProxy` and `EnergyEngine` components to handle the specific requirements of energy mode rendering, separating the optical depth calculation from the emission accumulation [S1, unofficial].

## Sources
[S1] llnl/rover, https://github.com/llnl/rover, fetched 2026-10-26 (unofficial)
