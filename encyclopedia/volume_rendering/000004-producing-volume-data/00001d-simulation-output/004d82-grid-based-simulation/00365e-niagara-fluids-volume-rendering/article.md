# Niagara Fluids Volume Rendering

Also called: Grid-based simulation rendering, Niagara Fluids Plugin, Niagara Fluids Plugin adaptation

## Overview
Niagara Fluids Volume Rendering is a method for rendering large volumetric datasets in Unreal Engine 5 by repurposing the engine's built-in grid-based fluid simulation system [S1, S2, S3, unofficial]. It ignores the simulation physics ("do nothing" simulation) and utilizes the system's rendering pipeline and material integration to display volume textures [S2, S3, unofficial]. It differs from dedicated ray marchers like [TBRayMarcher](52-tbray-marcher.md) by offering native engine integration and material flexibility, but suffers from excessive VRAM overhead and lower effective resolution due to the underlying simulation grid allocation [S1, S2, unofficial].

## How it works
The method treats the volume data as a static field within a Niagara fluid simulation system. The core mechanism involves creating a custom sampling interface and configuring the Niagara system to render the volume without updating physical states.

### Inputs and Outputs
*   **Input**: Volume texture data (e.g., imported via MHD loader from [TBRayMarcher](52-tbray-marcher.md) or other sources) [S2, S3, unofficial].
*   **Output**: Rendered 2D image of the volume, composited into the scene [S2, S3, unofficial].

### Data Structures
The system relies on the internal data structures of the Niagara Fluids plugin:
1.  **Simulation Grid**: A structured 3D grid allocated for simulation variables (velocity, pressure, density) [S2, S3, unofficial]. This grid is allocated even if unused for simulation, causing VRAM overhead [S2, unofficial].
2.  **Volume Texture**: The actual data container holding the voxel values [S2, S3, unofficial].
3.  **Niagara System**: A particle system structure where the volume is rendered as a volumetric effect [S2, S3, unofficial].

### Steps
1.  **Import Volume Data**: Load the volume dataset into the engine. The sources note that [TBRayMarcher](52-tbray-marcher.md)'s MHD loader was used to import data into a format compatible with Niagara [S2, S3, unofficial].
2.  **Create Custom Niagara Module Script**: Since no built-in module script existed for sampling Volume textures at the time, a custom script must be written to sample the volume texture [S2, S3, unofficial].
    *   The script takes a 3D coordinate and the volume texture reference.
    *   It performs trilinear (or nearest) sampling of the scalar value at the coordinate.
    *   It returns the sampled value to the Niagara system.
3.  **Configure Niagara System**:
    *   Create a new Niagara Fluid System.
    *   Set the simulation step to "Do Nothing" or disable time updates to treat the data as static [S2, S3, unofficial].
    *   Assign the Volume Texture to the system's data container.
    *   Configure the material to use the sampled values from the custom module script for shading (density, color, alpha) [S2, S3, unofficial].
4.  **Transfer Function Authoring**:
    *   Simple transfer functions can be authored interactively in the Editor using the material system [S2, S3, unofficial].
    *   For efficiency, a four-channel RGBA-1D-lookup texture is preferred over per-voxel evaluation for color and alpha curves [S2, unofficial].
5.  **Rendering**:
    *   The Niagara system renders the volume. If the dataset is too large, it must be chunked into multiple Niagara Systems [S2, S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Custom Niagara Module Script for Volume Sampling
# Note: This is a conceptual implementation of the custom script required
# because no native module existed [S2, S3, unofficial].

function SampleVolumeTexture(volume_texture: Texture3D, coordinate: Vector3) -> float:
    """
    Samples a scalar value from a 3D volume texture.
    coordinate: Normalized 3D coordinates (0.0 to 1.0) within the volume bounds.
    Returns: The interpolated scalar value at the coordinate.
    """
    # Clamp coordinates to valid range [0, 1]
    clamped_coord = clamp(coordinate, 0.0, 1.0)
    
    # Perform trilinear interpolation
    # The underlying engine API handles the texture sampling
    value = volume_texture.SampleLevel(clamped_coord, mip_level=0)
    
    return value

# Niagara System Setup
function SetupNiagaraFluidSystem(volume_texture: Texture3D):
    """
    Configures the Niagara Fluid System to render the volume without simulation.
    """
    system = CreateNiagaraFluidSystem()
    
    # Disable simulation updates
    system.simulation_enabled = False
    
    # Assign volume data
    system.volume_data = volume_texture
    
    # Attach custom module for sampling
    system.add_module(SampleVolumeTexture)
    
    # Configure material to use sampled density
    system.material.set_density_input(SampleVolumeTexture.output)
    
    return system
```

### Parameters
*   **Max Size**: Approximately 0.45 gigavoxels at low resolution [S1, unofficial].
*   **Internal Resolution**: Must be significantly smaller than the actual data resolution to avoid VRAM crashes. For example, a per-chunk resolution of 384 for the largest dimension was tested [S2, unofficial].

### Edge Cases and Pitfalls
*   **VRAM Overhead**: The simulation grid is allocated on top of the texture data, causing heavy VRAM consumption [S2, unofficial]. This limits the effective rendering resolution [S1, unofficial].
*   **Chunking Artifacts**: Chunking large datasets into multiple Niagara Systems results in lighting artifacts at chunk borders, as each chunk's lighting is computed independently [S1, S2, S3, unofficial].
*   **Transfer Function Limitations**: Transfer function authoring is restricted to Editor mode and does not officially support Game mode or NDisplay devices for large textures [S2, unofficial].
*   **Performance**: Performance is poor for large datasets due to low effective resolution and VRAM constraints [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Not explicitly defined, but rendering performance is described as "Poor" for full datasets due to low resolution and VRAM bottlenecks [S1, unofficial].
*   **Memory Complexity**: High. The method allocates simulation grid data in addition to the volume texture data [S2, unofficial]. This leads to excessive VRAM consumption [S1, unofficial].
*   **Measured Performance**: On an RTX 3500 Ada GPU (12 GB VRAM), showing more than five chunks of 384 resolution caused out-of-VRAM crashes [S2, unofficial]. On an RTX 5000 (16 GB VRAM), the full dataset fit but performance was severely impacted by increased resolution [S2, unofficial].
*   **Strengths**: Native engine integration, flexibility via the material system, and interactive transfer function authoring in the Editor [S2, S3, unofficial].
*   **Limitations**: Excessive VRAM usage, low effective resolution, lighting artifacts at chunk borders, and lack of runtime transfer function support [S1, S2, S3, unofficial].
*   **When to Use**: Small to medium datasets where native integration is prioritized over performance and resolution.
*   **When Not to Use**: Large datasets requiring high resolution or seamless full rendering, where [Sparse Volume Textures](52-sparse-volume-textures.md) or [Heterogeneous Volumes](52-heterogeneous-volumes.md) are more suitable [S1, unofficial].

## Background
The method was explored as part of an investigation into rendering large volume datasets (e.g., the Kolumbo seismic dataset) in Unreal Engine 5 [S1, S2, S3, unofficial]. It was identified as a potential solution due to its sustainability and native integration, but was ultimately deemed impractical for large-scale use due to VRAM constraints [S1, S2, unofficial]. The approach was documented in a survey of volume rendering techniques in UE5 [S1, S2, S3, unofficial].

## Sources
[S1] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://www.emergentmind.com/papers/2504.07485, fetched 2026-10-26 (unofficial)
[S2] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://arxiv.org/abs/2504.07485, fetched 2026-10-27 (unofficial)
[S3] Rendering Large Volume Datasets in Unreal Engine 5: A Survey, https://www.researchgate.net/publication/390671564_Rendering_Large_Volume_Datasets_in_Unreal_Engine_5_A_Survey, fetched 2026-10-28 (unofficial)
