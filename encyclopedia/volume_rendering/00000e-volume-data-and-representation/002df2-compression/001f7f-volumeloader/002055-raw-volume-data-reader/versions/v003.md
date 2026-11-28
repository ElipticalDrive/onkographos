# Raw Volume Data Reader

Also called: 3D Raw Volume Texture Loading, BrickDensityRegion Volume Slice Loading, Raw CT Data Loading, Raw Volume Data Loading, RawReader, Volume Data Loading

## Overview
The Raw Volume Data Reader ingests raw binary volume data (typically from CT scanners or specific model formats) into memory, applying domain-specific quantization to reduce precision, and preparing the data for GPU transfer or further processing [S1, S2, S4, S5, unofficial]. It solves the problem of converting high-precision raw sensor data (e.g., 12-bit Hounsfield units) into a format suitable for real-time raycasting or sparse voxel octree construction, often involving thresholding and bit-depth reduction [S1, S2, S4, S5, unofficial]. It differs from generic loaders by embedding specific quantization logic (scaling and thresholding) directly into the loading pipeline rather than treating data as opaque bytes [S1, S2, S4, S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A raw binary file containing voxel data. The file format may vary:
    *   Format A: A header of 3 integers (grid dimensions $X, Y, Z$) and 3 floats (bounding box size), followed by $X \times Y \times Z$ floats in row-major order (X fastest, Z slowest) [S5, unofficial].
    *   Format B: A sequence of 2-byte unsigned shorts representing the volume [S2, S3, unofficial].
    *   Format C: A sequence of 8-bit unsigned integers representing slices (e.g., 512x512 pixels per slice) [S1, S4, unofficial].
*   **Output:** A contiguous memory buffer (CPU or GPU) containing the processed voxel data, ready for texture upload or raycasting.

**Data Structures:**
*   **RawReader (C++):** Contains a private field `m_data` of type `unsigned short*` pointing to a contiguous memory block of size $Z \times Y \times X \times \text{sizeof(unsigned short)}$ [S2, unofficial].
*   **VolumeSampleData (CUDA):** A managed CUDA memory array of size `grid_dim.x * grid_dim.y * grid_dim.z` storing processed voxel samples [S5, unofficial].
*   **GLubyte Buffer:** A dynamically allocated array of size `dimension.x * dimension.y * dimension.z` used for OpenGL texture upload [S4, unofficial].

**Steps:**

1.  **Open and Validate File:**
    *   Open the file in binary read mode [S1, S2, S4, unofficial].
    *   Determine file size. If the format is known (e.g., Format B), assert that `file_size == Z * Y * X * sizeof(unsigned short)` [S2, unofficial].
    *   If a header is present (Format A), read the first 12 bytes to extract grid dimensions ($X, Y, Z$) and bounding box dimensions [S5, unofficial].

2.  **Allocate Memory:**
    *   Allocate a buffer `m_data` of size $Z \times Y \times X \times \text{sizeof(unsigned short)}$ [S2, unofficial].
    *   Alternatively, allocate a `GLubyte` buffer of size $X \times Y \times Z$ for 8-bit data [S4, unofficial].
    *   For CUDA implementations, allocate managed memory for `VolumeSampleData` [S5, unofficial].

3.  **Read Raw Data:**
    *   Read the entire file content into the allocated buffer [S1, S2, unofficial].
    *   If the data is 16-bit unsigned short, read it directly [S2, S3, unofficial].
    *   If the data is 8-bit, read it into a byte array `pix08` [S1, unofficial].

4.  **Process and Quantize (Per-Voxel):**
    *   Iterate through the raw data. The indexing depends on the source format:
        *   For 2D slice-based data (Format C): Iterate through slices. For each slice, create a bitmap source using `BitmapSource.Create(width, height, 96, 96, PixelFormats.Gray8, null, singleImage, stride)` where `stride = (width * 8) / 8` [S1, unofficial].
        *   For 3D volume data (Format A/B): Iterate through indices $i$ (X), $j$ (Z), $k$ (Y) [S3, S5, unofficial].
    *   **Clamping and Scaling:**
        *   Find min and max values in the buffer to determine normalization range [S3, unofficial].
        *   Apply iso-clamping: If `tmp < iso_min` or `tmp > iso_max`, set `tmp = 0` [S3, unofficial].
        *   Compute density value: $val = tmp \times m\_density\_mult \times multiply / max$ [S3, unofficial].
        *   For CUDA loading: Clamp float values to $[VOLUME\_MIN\_DENSITY, VOLUME\_MAX\_DENSITY]$ [S5, unofficial].
    *   **Coordinate Transformation (if applicable):**
        *   Calculate 3D voxel coordinates:
            $$x = i \times multiply$$
            $$y = vol\_res.y() - y\_val - 1$$
            $$z = j \times multiply$$
            where $multiply = vol\_res.x() / file\_res.x()$ [S3, unofficial].

5.  **Transfer to GPU (Optional):**
    *   Allocate a `GLubyte` buffer of size $X \times Y \times Z$ [S4, unofficial].
    *   Read size bytes into the buffer [S4, unofficial].
    *   Create an OpenGL texture object and bind it to `GL_TEXTURE_3D` [S4, unofficial].
    *   Set filters: `GL_TEXTURE_MAG_FILTER` and `GL_TEXTURE_MIN_FILTER` to `GL_LINEAR` [S4, unofficial].
    *   Set wrap modes: `GL_TEXTURE_WRAP_S`, `GL_TEXTURE_WRAP_T`, and `GL_TEXTURE_WRAP_R` to `GL_REPEAT` [S4, unofficial].
    *   Upload data:
        ```python
        glTexImage3D(GL_TEXTURE_3D, 0, GL_RED, dimension.x, dimension.y, dimension.z, 0, GL_RED, GL_UNSIGNED_BYTE, m_LocalBuffer)
        ```
    *   Delete the CPU buffer [S4, unofficial].

6.  **Cleanup:**
    *   Close the file [S2, unofficial].
    *   Free allocated memory (e.g., `free(m_data)`) in the destructor [S2, unofficial].

**Parameters:**
*   `dimension` (glm::vec3): Dimensions of the volume in voxels (x, y, z) [S4, unofficial].
*   `texture_path` (std::string): File path to the raw data [S4, unofficial].
*   `VOLUME_MIN_DENSITY`, `VOLUME_MAX_DENSITY`: Constants defining the clamping range for volume density values [S5, unofficial].
*   `iso_min`, `iso_max`: Thresholds for iso-clamping [S3, unofficial].
*   `m_density_mult`: Density multiplication factor [S3, unofficial].
*   Slice dimensions: Typically 512x512 pixels, assumed constant in some implementations [S1, unofficial].
*   Pixel format: 8-bit grayscale (`PixelFormats.Gray8`) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision Mismatch:** Ensure the data type in the file (e.g., 16-bit unsigned short) matches the expected size in the assertion [S2, unofficial].
*   **Coordinate System:** The Y-axis inversion ($y = vol\_res.y() - y\_val - 1$) is critical for correct orientation in some medical imaging contexts [S3, unofficial].
*   **Memory Overflow:** Assert file size matches expected volume size to prevent buffer overflows or incomplete reads [S2, unofficial].

## Subtopics

### [Voxel data loading and quantization](2478-voxel-data-loading-and-quantization.md)
Defines the procedure for ingesting raw binary volume data, applying domain-specific quantization to reduce precision, and preparing the data for GPU transfer. It differs from generic loaders by embedding specific quantization logic (scaling and thresholding) directly into the loading pipeline.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels, for reading, processing, and uploading.
*   **Memory Complexity:** $O(N)$ for storing the raw data buffer and the processed data buffer.
*   **Performance:** Loading speed is limited by disk I/O and memory bandwidth. GPU upload speed depends on PCIe bandwidth and texture upload efficiency.
*   **Strengths:** Simple implementation, direct mapping to raw sensor data, efficient for fixed-format data.
*   **Limitations:** Lacks support for complex metadata (DICOM tags), compression, or variable slice thickness without additional logic. Requires prior knowledge of data format (header structure, byte order, bit depth).
*   **When to Use:** When data is already in a raw binary format and no complex metadata parsing is required.
*   **When Not to Use:** When data is compressed, contains complex metadata, or requires resampling/interpolation during loading.

## Background
The Raw Volume Data Reader is a fundamental component in volume rendering pipelines, particularly in medical imaging and scientific visualization. It originated from the need to efficiently load raw CT scan data, which is often stored in proprietary or simple binary formats without complex headers. The method has been implemented in various research projects and educational examples, such as marching cubes implementations and CUDA-based volume renderers [S1, S2, S3, S4, S5, unofficial].

## Sources
[S1] How to generate 3D mesh from CT Slices (.raw) using marching cubes algorithm in C#?, https://stackoverflow.com/questions/65498236/how-to-generate-3d-mesh-from-ct-slices-raw-using-marching-cubes-algorithm-in, fetched 2026-10-27 (unofficial)
[S2] upupming/marching-cubes, https://github.com/upupming/marching-cubes, fetched 2026-10-27 (unofficial)
[S3] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S4] Conradyen/Slice-Base-Volume-Rendering, https://github.com/Conradyen/Slice-Base-Volume-Rendering, fetched 2026-10-27 (unofficial)
[S5] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
