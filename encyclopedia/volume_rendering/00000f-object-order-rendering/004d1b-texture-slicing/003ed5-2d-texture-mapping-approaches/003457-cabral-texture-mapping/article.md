# Cabral Texture Mapping

Also called: Filtered Back Projection via Texture Mapping, Texture Mapping Backprojection, Trilinear interpolation of texture mapped slices

## Overview
Cabral Texture Mapping implements the filtered back projection (FBP) algorithm for Computed Tomography (CT) reconstruction using 2D texture mapping hardware and an accumulation buffer [S2, unofficial]. It differs from standard ray-casting volume rendering by treating the reconstruction as a back-projection of slice data into a frame buffer, leveraging graphics hardware for interpolation and blending [S2, unofficial]. Unlike [2D Texture-Based Volume Rendering](916-2d-texture-based-volume-rendering.md) which focuses on direct visualization, this method is specifically designed for reconstructing volumetric data from projection data, though it shares the underlying texture-mapping primitives [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 2D projection slices (or reconstructed intermediate slices) derived from CT projection data.
*   **Output:** A reconstructed 3D volume scalar field, accumulated into a frame buffer or accumulation buffer.

**Data Structures:**
*   **Texture Maps:** 2D textures representing the projection slices. The exact layout is not specified in the sources, but they are loaded into texture memory [S2, unofficial].
*   **Accumulation Buffer:** A buffer used to sum the back-projected contributions. The sources mention the use of an accumulation buffer on Raster Manager Reality Engine Onyx machines [S2, unofficial].

**Steps:**
1.  **Load Slices:** Load the 2D projection slices into texture memory [S2, unofficial].
2.  **Back-Projection via Texture Mapping:** For each slice, map the texture coordinates to the 3D volume space. The sources describe this as "filtered back projection" implemented via texture mapping [S2, unofficial].
3.  **Trilinear Interpolation:** During the mapping process, the scalar values and texture-mapped slices are tri-linearly interpolated [S1, unofficial]. This interpolation occurs as the slice data is projected into the volume space.
4.  **Blending:** The interpolated slices are blended into the frame buffer (or accumulation buffer) in a back-to-front manner [S1, unofficial]. This accumulation step effectively sums the back-projected contributions to form the final volume data.

**Formulas:**
*   The specific mathematical formulation of the filtered back projection algorithm is not provided in the sources. The method relies on the hardware implementation of the back-projection geometry [S2, unofficial].
*   Trilinear interpolation is used for value estimation, but the specific interpolation weights are not detailed [S1, unofficial].

**Parameters:**
*   **Interpolation Mode:** Trilinear interpolation is specified [S1, unofficial].
*   **Blending Order:** Back-to-front [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Transfer Function Interpolation:** A known drawback of early texture-mapping approaches is that non-linear transfer functions are not interpolated correctly by the texture hardware, potentially requiring re-shading and re-loading of the volume when viewing parameters change [S2, unofficial].
*   **Hardware Dependency:** The original implementation relied on specific hardware (Raster Manager Reality Engine Onyx) [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Cabral Texture Mapping
# Based on [S1, unofficial] and [S2, unofficial]

def cabral_texture_mapping(projection_slices, accumulation_buffer):
    """
    Implements filtered back projection using texture mapping.
    
    Args:
        projection_slices: List of 2D projection slices to be back-projected.
        accumulation_buffer: Buffer to accumulate the back-projected data.
    """
    
    # Load slices into texture memory
    # Note: Exact loading mechanism depends on hardware (e.g., Raster Manager) [S2, unofficial]
    textures = load_slices_into_texture_memory(projection_slices)
    
    # Initialize accumulation buffer
    accumulation_buffer.clear()
    
    # Back-project each slice
    # The sources specify blending in a back-to-front manner [S1, unofficial]
    for slice_data in textures:
        # Map slice to volume space using texture mapping
        # This step performs the back-projection geometry
        mapped_slice = apply_texture_mapping(slice_data)
        
        # Tri-linearly interpolate the scalar value and texture mapped slices [S1, unofficial]
        interpolated_slice = trilinear_interpolate(mapped_slice)
        
        # Blend into the frame-buffer/accumulation buffer in back-to-front manner [S1, unofficial]
        # The specific blending function (e.g., additive) is not detailed, 
        # but accumulation implies summation for reconstruction.
        accumulate_into_buffer(accumulation_buffer, interpolated_slice)
        
    return accumulation_buffer

def trilinear_interpolate(data):
    """
    Performs trilinear interpolation on the mapped slice data.
    Implementation details are not provided in the sources [S1, unofficial].
    """
    # Placeholder for trilinear interpolation logic
    # This would involve sampling the texture with interpolation weights
    pass

def accumulate_into_buffer(buffer, slice_data):
    """
    Accumulates the slice data into the buffer.
    The sources mention an accumulation buffer [S2, unofficial].
    """
    # Add slice data to the buffer
    # Exact accumulation function (e.g., addition, max) is not specified
    pass
```

## Subtopics

### Multi-resolution Representation for Intermediate Template
A technique to overcome resolution limitations and glancing angle projection artifacts by generating the intermediate template at different resolutions for different areas, rather than using a single finest resolution for the whole template [S3, unofficial]. It defines resolution intervals $\Delta u = 1/(x_d - 1)$ and $\Delta v = 1/(y_d - 1)$ and uses continuous variables $u$ and $v$ to adjust the origin and direction of the tracing ray during DVR/DSR generation of the intermediate template, allowing different resolution intervals to be used [S3, unofficial]. This is chosen when different areas have different resolution requirements (e.g., wrinkles around eyes), as rendering the whole template at the finest resolution is impractical and unnecessary [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The original implementation used four Raster Manager Reality Engine Onyx machines [S2, unofficial].
*   **Strengths:** Leverages existing texture mapping hardware for back-projection, potentially offering real-time performance on specialized hardware [S2, unofficial].
*   **Limitations:** Non-linear transfer functions are not interpolated correctly by the texture hardware, requiring re-shading and re-loading when viewing parameters change [S2, unofficial]. High network bandwidth requirements if distributed rendering is used for composition [S2, unofficial].
*   **When to use:** When implementing filtered back projection on hardware with strong texture mapping capabilities and accumulation buffers.
*   **When not to use:** When non-linear transfer functions are critical and hardware interpolation is insufficient, or when modern GPU-based ray casting is available.

## Background
The method was proposed by Cabral et al. [11] and implemented on Raster Manager Reality Engine Onyx machines [S2, unofficial]. It is part of the history of early volume rendering and CT reconstruction techniques that utilized graphics hardware acceleration [S2, unofficial]. The approach is related to other texture-based methods like those by Dachille and Kaufman [19] and shear-warp methods [63] [S2, unofficial].

## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S2] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S3] Texture Mapping Volume Objects, https://doi.org/10.2312/vvg.20051006, fetched 2026-10-30 (unofficial)
