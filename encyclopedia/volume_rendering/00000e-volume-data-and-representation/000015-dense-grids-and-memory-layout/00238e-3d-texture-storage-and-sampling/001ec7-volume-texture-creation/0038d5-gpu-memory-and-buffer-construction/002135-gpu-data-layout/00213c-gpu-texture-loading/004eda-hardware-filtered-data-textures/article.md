# Hardware-Filtered Data Textures

Also called: GPU Texture Format Selection for Sampling

## Overview
Hardware-Filtered Data Textures enable efficient single-fetch filtered lookups by mapping volume data to specific GPU texture formats that match the sampling kernel, avoiding the high cost of manual interpolation on integer textures [S1, unofficial]. This method differs from general texture loading by prioritizing the internal format's compatibility with hardware filtering units over raw transfer bandwidth, whereas sibling methods like CUDA-accelerated Vector Field Reading focus on vector magnitude computation rather than scalar field sampling optimization [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar volume data (uint8, uint16) or raw pixel buffers.
*   **Output:** A GPU 3D texture configured for hardware-filtered sampling.

**Data Structures:**
The method relies on OpenGL texture specifications. The critical fields are the internal format (determining bits per pixel and hardware interpretation) and the format (determining the layout of components in memory).

**Steps:**
1.  **Select Format for uint8 Data:**
    *   Use the `R8` internal format. This maps 8-bit unsigned integers directly to the texture unit, enabling hardware trilinear filtering with a single fetch [S1, unofficial].
2.  **Select Format for uint16 Data:**
    *   Check for the `EXT_texture_norm16` extension support [S1, unofficial].
    *   If supported, use the `R16` internal format. This allows 16-bit unsigned integers to be sampled with hardware filtering [S1, unofficial].
    *   If the extension is missing, fall back to the integer path (manual interpolation) [S1, unofficial].
3.  **Optimize Transfer Bandwidth (NVIDIA Specific):**
    *   When loading data into graphics memory, use the `GL_BGRA` external format with `GL_RGBA8` internal format [S2].
    *   This avoids driver-side swizzling overhead because NVIDIA hardware internally stores 8-bit pixels in BGRA layout, matching Microsoft GDI pixel layout [S2].
    *   Target a texture size of 256 KB for optimal loading bandwidth on NVIDIA GeForce 6800 Ultra hardware [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def create_hardware_filtered_texture(data, data_type, gl_context):
    """
    Creates a GPU texture optimized for hardware-filtered sampling.
    data: numpy array or buffer of volume data.
    data_type: 'uint8' or 'uint16'.
    gl_context: Object providing glTexImage3D, glGetExtension, etc.
    """
    # Step 1 & 2: Determine internal format based on data type
    if data_type == 'uint8':
        internal_format = 'GL_R8'
        external_format = 'GL_RED' # Standard for R8
        # Note: For transfer optimization on NVIDIA, see Subtopics
    elif data_type == 'uint16':
        # Check for extension support
        if gl_context.check_extension('EXT_texture_norm16'):
            internal_format = 'GL_R16'
            external_format = 'GL_RED'
        else:
            # Fallback to integer path (manual trilinear)
            raise RuntimeError("EXT_texture_norm16 missing; use integer path")
    else:
        raise ValueError("Unsupported data type for hardware filtering")

    # Step 3: If loading via buffer transfer on NVIDIA, optimize format
    # This applies specifically to the upload phase, not necessarily the sampling format
    if gl_context.is_nvidia_hardware():
        # For 8-bit data, use BGRA external format to avoid swizzling
        if data_type == 'uint8':
            upload_external_format = 'GL_BGRA'
            upload_internal_format = 'GL_RGBA8'
            # Note: This requires packing data into BGRA layout or using specific drivers
            # that handle the swizzle automatically.
        else:
            upload_external_format = external_format
            upload_internal_format = internal_format
    else:
        upload_external_format = external_format
        upload_internal_format = internal_format

    # Create the texture object
    tex_id = gl_context.create_texture_3d()
    gl_context.bind_texture_3d(tex_id)
    
    # Upload data
    # The dimensions must be specified. 
    # For optimal bandwidth on NVIDIA GeForce 6800 Ultra, aim for 256 KB total size [S2].
    gl_context.tex_image_3d(
        target='GL_TEXTURE_3D',
        internal_format=upload_internal_format,
        width=data.shape[0],
        height=data.shape[1],
        depth=data.shape[2],
        format=upload_external_format,
        type='GL_UNSIGNED_BYTE' if data_type == 'uint8' else 'GL_UNSIGNED_SHORT',
        data=data
    )
    
    # Set filtering to enable hardware trilinear sampling
    gl_context.tex_parameteri('GL_TEXTURE_3D', 'GL_TEXTURE_MIN_FILTER', 'GL_LINEAR_MIPMAP_LINEAR')
    gl_context.tex_parameteri('GL_TEXTURE_3D', 'GL_TEXTURE_MAG_FILTER', 'GL_LINEAR')
    
    return tex_id
```

**Parameters:**
*   `EXT_texture_norm16`: Required for uint16 hardware filtering [S1, unofficial].
*   Texture Size: 256 KB yields peak bandwidth on NVIDIA GeForce 6800 Ultra [S2].

**Edge Cases:**
*   **Missing Extension:** If `EXT_texture_norm16` is not available, uint16 data cannot use hardware filtering and must fall back to manual interpolation, which is 3–4× slower [S1, unofficial].
*   **Precision:** Filtered textures match the integer path to within one 8-bit step [S1, unofficial].
*   **Hardware Dependency:** Transfer bandwidth optimizations are highly hardware-dependent (specifically NVIDIA GeForce 6800 Ultra results cited) [S2].

## Subtopics

### Texture Format Optimization for Loading
This variant focuses on the host-to-device transfer phase rather than the sampling phase. It identifies that NVIDIA graphics cards match Microsoft GDI pixel layout (BGRA) internally [S2]. Transferring non-BGRA data requires driver swizzling, which reduces bandwidth. The optimal texture format for loading is `GL_BGRA` external format with `GL_RGBA8` internal format [S2]. This optimization is specific to 8-bit textures and NVIDIA hardware [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Sampling:** O(1) per sample with hardware filtering, as it performs one filtered fetch instead of eight manual fetches [S1, unofficial].
*   **Loading:** Bandwidth is maximized at specific texture sizes (256 KB on NVIDIA GeForce 6800 Ultra) [S2].

**Measured Performance:**
*   Manual trilinear filtering on integer textures is approximately 3–4× the GPU time per frame compared to hardware-filtered textures [S1, unofficial].
*   Hardware-filtered textures allow the kernel's cost to "mostly disappear" [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Significant performance gain for uint8 and uint16 data; images match integer path precision to within one 8-bit step [S1, unofficial].
*   **Limitations:** Requires specific extensions (`EXT_texture_norm16`) for uint16; transfer optimizations are hardware-specific (NVIDIA) [S1, unofficial][S2]. 32- and 64-bit integers remain on the exact integer path [S1, unofficial].

**When to Use:**
*   Use for uint8 and uint16 volume data when `EXT_texture_norm16` is supported and performance is critical [S1, unofficial].
*   Use BGRA upload format for 8-bit data on NVIDIA hardware to maximize load bandwidth [S2].

## Background
The method originated from work on histotomography data rendering, specifically involving 2.8 TB volumes at 16-bit depth [S1, unofficial]. The transfer bandwidth optimization was detailed in research on distributed shared memory for roaming large volumes, tested on NVIDIA GeForce 6800 Ultra hardware [S2].

## Sources
[S1] Volume rendering: progressive multiscale loading, sorted compositing, lighting and more from a histotomography fork (2.8 TB volume @ 16bits) - what, if anything, would be worth upstreaming?, https://github.com/google/neuroglancer/issues/1134, fetched 2026-10-26 (unofficial)
[S2] Distributed Shared Memory for Roaming Large Volumes, https://doi.org/10.1109/tvcg.2006.135, fetched 2026-10-26
