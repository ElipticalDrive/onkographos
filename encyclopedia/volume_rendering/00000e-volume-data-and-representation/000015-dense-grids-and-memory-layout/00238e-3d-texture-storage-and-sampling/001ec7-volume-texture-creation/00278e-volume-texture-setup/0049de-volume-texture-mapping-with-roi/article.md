# Volume Texture Mapping with ROI

Also called: Texture Mapping Render Action, vzParameterVolumeTexture, vzParameterVolumeTexture Construction

## Overview
Volume Texture Mapping with ROI renders volumetric data by mapping a Data Region of Interest (ROI) within a source memory buffer to a Geometry ROI in world space, enabling sub-volume rendering without data copying [S1]. It differs from full-volume methods by restricting rendering to a specified data subset via coordinate mapping, and from sparse methods by operating on dense grid textures managed directly by the OpenGL pipeline [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A pointer to volumetric data (`dataPtr`), data dimensions (`dataDimensions`), a Data ROI defining the data subset (`dataROI`), data type (`dataType`), external format (`externalFormat`), and an optional internal OpenGL format (`internalFormat`) [S1].
*   **Output:** A configured 3D texture object and associated rendering state that maps voxel coordinates to world space coordinates, ready for use by a render action (e.g., `TMRenderAction`) [S1].

**Data Structures**
The core data structure is the `vzParameterVolumeTexture` object [S1]. Its constructor accepts the following parameters:
1.  `dataDimensions[3]`: Integer array specifying the dimensions of the texture data along the X, Y, and Z axes [S1].
2.  `dataROI[6]`: Integer array specifying the cuboidal region of interest within the volumetric data, formatted as `{minX, minY, minZ, maxX, maxY, maxZ}` [S1].
3.  `dataPtr`: Pointer to the raw texture data in memory [S1].
4.  `dataType`: Enum specifying the data type (e.g., unsigned byte, integer, float) [S1].
5.  `externalFormat`: Enum specifying the format of the data (e.g., luminance, RGBA) [S1].
6.  `internalFormat`: Enum specifying the OpenGL internal format, defaulting to `VZ_DEFAULT_INTERNAL_FORMAT` if unspecified [S1].

**Steps**
1.  **Initialize Texture:** Create a `vzParameterVolumeTexture` instance by passing the data dimensions, data ROI, data pointer, data type, external format, and internal format [S1].
2.  **Store Data Layout:** Ensure the data in `dataPtr` is stored in row-major order, interleaved for multi-channel formats (e.g., `{R1, G1, B1, A1}, {R2, G2, B2, A2}` for RGBA), matching `glTexImage3D()` requirements [S1].
3.  **Map ROI:** The system maps the Data ROI to the Geometry ROI, which is a bounding box in world space defaulting to `(0,0,0)` to `(1,1,1)` [S1]. Voxel samples at the border of the Data ROI map exactly to the boundaries of the Geometry ROI [S1].
4.  **Clip Geometry:** The render action clips geometry outside the Geometry ROI [S1]. If an OpenGL clipping plane is enabled before the `draw()` call, the action uses software clipping planes; otherwise, it uses OpenGL clipping planes [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Standard definitions for self-containment
VZ_DEFAULT_INTERNAL_FORMAT = "DEFAULT"
class vzTextureType: FLOAT = "FLOAT"; BYTE = "BYTE"
class vzExternalTextureFormat: RGBA = "RGBA"; LUMINANCE = "LUMINANCE"
def get_data_pointer(): return b"dummy_data"
class vzParameterVolumeTexture:
    def __init__(self, dataDimensions, dataROI, dataPtr, dataType, externalFormat, internalFormat):
        self.dataDimensions = dataDimensions
        self.dataROI = dataROI
        self.dataPtr = dataPtr
        self.dataType = dataType
        self.externalFormat = externalFormat
        self.internalFormat = internalFormat
    def setGeometryROI(self, min_world, max_world):
        self.min_world = min_world
        self.max_world = max_world
    def setDataPtr(self, new_ptr):
        self.dataPtr = new_ptr
class TMRenderAction:
    def draw(self, volume_texture):
        pass
render_action = TMRenderAction()

# Step 1: Initialize Texture
data_dimensions = [256, 256, 256]
data_roi = [64, 64, 64, 191, 191, 191]
data_ptr = get_data_pointer()
data_type = vzTextureType.FLOAT
external_format = vzExternalTextureFormat.RGBA
internal_format = VZ_DEFAULT_INTERNAL_FORMAT
volume_texture = vzParameterVolumeTexture(
    dataDimensions=data_dimensions,
    dataROI=data_roi,
    dataPtr=data_ptr,
    dataType=data_type,
    externalFormat=external_format,
    internalFormat=internal_format
)

# Step 2: Store Data Layout (handled by caller; data must be row-major interleaved)
# Example layout for RGBA: {R1, G1, B1, A1}, {R2, G2, B2, A2}

# Step 3: Map ROI
volume_texture.setGeometryROI(min_world=(0.0, 0.0, 0.0), max_world=(1.0, 1.0, 1.0))

# Step 4: Clip Geometry
# If OpenGL clipping planes are enabled before draw(), TMRenderAction uses software clipping.
# Otherwise, it uses OpenGL clipping planes.
render_action.draw(volume_texture)
```

**Parameters and Typical Values**
*   `dataDimensions`: Any positive integers; non-power-of-two dimensions are internally padded to the next higher power of two [S1].
*   `dataROI`: Must be within `dataDimensions`; example `{64, 64, 64, 191, 191, 191}` for a 128x128x128 sub-volume starting at offset 64 [S1].
*   `geometryROI`: Defaults to `(0,0,0)` to `(1,1,1)` [S1].
*   `internalFormat`: Defaults to `VZ_DEFAULT_INTERNAL_FORMAT`, which infers a suitable value from `dataType` and `externalFormat` [S1].

**Edge Cases and Pitfalls**
*   **Shallow Copy:** The API does not allocate memory for `dataPtr` [S1]. Modifying the data requires calling `setDataPtr()` to force a reload into texture memory [S1].
*   **Immutability:** `dataDimensions`, `dataROI`, `dataType`, `externalFormat`, and `internalFormat` cannot be modified after creation; a new texture object must be created to change these values [S1].
*   **Power of Two:** Non-power-of-two dimensions trigger internal padding to the next higher power of two [S1].
*   **Memory Limits:** Textures exceeding available texture memory are broken into smaller bricks internally to create OpenGL textures [S1].

## Subtopics

### 3D Texture Upload
Handles the transfer of texture data to GPU memory. Choose when explicit upload control is needed rather than automatic handling by the ROI mapping. [3D Texture Upload](4419-3d-texture-upload.md)

### VRDF Texture Construction for Label-Weighted Volumes
Constructs textures for label-weighted volumes. Choose when rendering semantic labels with weights instead of standard scalar/vector data. [VRDF Texture Construction for Label-Weighted Volumes](6959-vrdf-texture-construction-for-label-weighted-volumes.md)

### Sparse Volume Texture Setup
Sets up textures for sparse data structures. Choose when data is sparse and memory efficiency for empty space is critical. [Sparse Volume Texture Setup](7432-sparse-volume-texture-setup.md)

### Voxel Volume Initialization & Binding
Initializes and binds voxel volumes. Choose when low-level voxel buffer management is required before texture mapping. [Voxel Volume Initialization & Binding](12429-voxel-volume-initialization-binding.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** The method uses shallow copying and does not allocate additional memory for the data buffer itself, though it may pad dimensions to the next power of two, increasing texture memory usage [S1].
*   **Measured Performance:** Not given in the sources [S1].
*   **Scaling:** The system handles textures larger than available texture memory by breaking them into smaller bricks internally [S1].
*   **Strengths:** Allows rendering of sub-volumes without data copying; supports non-power-of-two dimensions via internal padding; provides flexible clipping via OpenGL or software planes [S1].
*   **Limitations:** Texture parameters are immutable after creation; requires manual reload if data changes; padding increases memory usage for non-power-of-two dimensions [S1].
*   **When to Use:** Rendering a specific region of a large volumetric dataset without copying data, or using non-power-of-two dimensions [S1].
*   **When Not to Use:** Frequent updates to texture dimensions or ROI are required (necessitating new texture creation), or data is sparse and a sparse representation is more efficient [S1].

## Background
The method originates from the SGI (Silicon Graphics) Volume Rendering API, specifically the `vzParameterVolumeTexture` class and `TMRenderAction` [S1]. It was designed to provide a simple abstraction of 3D textures and their position in 3D space, leveraging OpenGL's texture mapping capabilities for volume rendering [S1].

## Sources
[S1] Chapter 4. Texture Mapping Render Action, https://techpubs.jurassic.nl/library/manuals/4000/007-4389-010/sgi_html/ch04.html, fetched 2026-10-27
