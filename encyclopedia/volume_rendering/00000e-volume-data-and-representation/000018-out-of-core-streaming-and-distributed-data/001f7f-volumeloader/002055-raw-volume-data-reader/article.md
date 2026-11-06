# Raw Volume Data Reader
Also called: BrickDensityRegion Volume Slice Loading, RawReader, Volume Data Loading

## Overview
The Raw Volume Data Reader loads raw binary volume data files directly into memory [S1, unofficial]. It parses a fixed header, reads voxel values, applies density clamping and normalization, and maps the data to 3D coordinates [S2, S3, unofficial]. It differs from generic loaders by handling coordinate mapping and density scaling directly during the load phase rather than treating the data as opaque bytes [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A raw binary file containing a header and voxel data [S3, unofficial].
- Output: A contiguous memory block of processed voxel values and coordinates [S1, S2, unofficial].

**Data Structures:**
The core data structure `RawReader` contains:
- `m_data`: A pointer to `unsigned short` (`unsigned short*`) pointing to a contiguous memory block of size $Z \times Y \times X \times \text{sizeof(unsigned short)}$ [S1, unofficial].

**Parameters:**
- `iso_min`, `iso_max`: Thresholds for iso-clamping; values outside this range are set to 0 [S2, unofficial].
- `m_density_mult`: A multiplier for density calculation [S2, unofficial].
- `max`: The maximum value used for normalization scaling [S2, unofficial].
- `vol_res`: The target volume resolution dimensions (X, Y, Z) [S2, unofficial].
- `file_res`: The source file resolution dimensions (X, Z) [S2, unofficial].
- `multiply`: Scaling factor calculated as `vol_res.x() / file_res.x()` [S2, unofficial].
- `VOLUME_MIN_DENSITY`, `VOLUME_MAX_DENSITY`: Constants defining the clamping range for volume density values [S3, unofficial].

**Steps:**
1. Open the file in binary read mode with the `ate` flag to determine the file size [S1, unofficial].
2. Read the file header: 3 integers for grid dimensions and 3 floats for bounding box size [S3, unofficial].
3. Calculate the expected file size as $Z \times Y \times X \times \text{sizeof(unsigned short)}$ and assert that the actual file size matches this value [S1, unofficial].
4. Allocate memory for `m_data` using `malloc` [S1, unofficial].
5. Seek to the beginning of the file and read the entire content into the allocated memory [S1, unofficial].
6. Close the file [S1, unofficial].
7. Iterate through the buffer to find the minimum and maximum values for the normalization range [S2, unofficial].
8. For each element, apply iso-clamping: if $tmp < iso\_min$ or $tmp > iso\_max$, set $tmp = 0$ [S2, unofficial].
9. Calculate the density value: $val = tmp \times m\_density\_mult \times multiply / max$ [S2, unofficial].
10. Calculate the 3D voxel coordinates: $x = i \times multiply$, $y = vol\_res.y() - y\_val - 1$, $z = j \times multiply$, where $i$ iterates from 0 to `file_res.x()`, $j$ iterates from 0 to `file_res.z()`, and $y\_val$ corresponds to the current slice index [S2, unofficial].
11. Store the processed values in `m_data` [S1, S2, unofficial].
12. Free the allocated memory in the destructor [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import os
import struct

def load_raw_volume(filename, vol_res, file_res, iso_min, iso_max, m_density_mult, max_val):
    # Step 1: Open file with ate flag to get size
    file_size = os.path.getsize(filename)
    
    # Step 2: Read header (3 ints, 3 floats)
    with open(filename, 'rb') as f:
        header = f.read(24) # 3*4 + 3*4 bytes
        grid_dims = struct.unpack('3i', header[:12])
        bbox_size = struct.unpack('3f', header[12:24])
        
        # Step 3: Assert file size matches expected volume size
        expected_size = grid_dims[0] * grid_dims[1] * grid_dims[2] * 2 # 2 bytes for unsigned short
        if file_size != expected_size:
            raise ValueError("File size mismatch")
            
        # Step 4-5: Allocate memory and read entire file content
        f.seek(0)
        raw_data = f.read(expected_size)
        
    # Step 6: Close file (handled by context manager)
    
    # Step 7: Unpack unsigned shorts and find min/max
    buffer = list(struct.unpack('<' + str(expected_size // 2) + 'H', raw_data))
    min_val = min(buffer)
    max_val_actual = max(buffer)
    
    # Step 8-10: Process data
    multiply = vol_res[0] / file_res[0]
    processed_data = []
    
    for j in range(file_res[2]):
        for i in range(file_res[0]):
            tmp = buffer[j * file_res[0] + i]
            
            # Iso-clamping
            if tmp < iso_min or tmp > iso_max:
                tmp = 0
            
            # Density calculation
            val = tmp * m_density_mult * multiply / max_val
            
            # Coordinate calculation
            x = i * multiply
            y = vol_res[1] - j - 1  # y_val corresponds to slice index j
            z = j * multiply
            
            processed_data.append(val)
            
    return processed_data

def free_raw_volume(m_data):
    del m_data # Explicit deallocation handled by Python runtime
```

**Edge Cases and Pitfalls:**
- File size mismatch: The method asserts that the file size matches the expected volume size; if the file is corrupted or incomplete, this will fail [S1, unofficial].
- Precision: The method uses `unsigned short` (16-bit) for storage; if the source data has higher precision, it is truncated to 16-bit during the initial read [S1, unofficial].
- Boundary conditions: The iso-clamping sets values outside the range to 0, which may result in loss of data if the range is not chosen carefully [S2, unofficial].

## Subtopics

### [Voxel data loading and quantization](2478-voxel-data-loading-and-quantization.md)
Defines the procedure for ingesting raw binary volume data, applying domain-specific quantization to reduce precision, and preparing the data for GPU transfer. It differs by embedding specific quantization logic (scaling and thresholding) directly into the loading pipeline.

## Operating characteristics
- **Time Complexity:** $O(N)$ for reading and processing, where $N$ is the number of voxels [S1, S2, unofficial].
- **Memory Complexity:** $O(N)$, storing all voxels in memory as `unsigned short` [S1, unofficial].
- **Strengths:** Direct loading of raw binary data without complex format parsing [S1, unofficial].
- **Limitations:** Lacks support for standard medical imaging formats and does not handle coordinate system alignment or resampling [S1, S2, unofficial].
- **When to use:** When working with raw binary volume data files and needing a quick loading mechanism [S1, unofficial].
- **When not to use:** When dealing with standard medical imaging formats or requiring advanced features like resampling [S1, S2, unofficial].

## Background
The Raw Volume Data Reader is part of the VolumeLoader framework, which supports various medical imaging formats [Parent]. It is used for ingesting raw binary data into memory for further processing [S1, unofficial].

## Sources
[S1] upupming/marching-cubes, https://github.com/upupming/marching-cubes, fetched 2026-10-27 (unofficial)
[S2] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S3] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
