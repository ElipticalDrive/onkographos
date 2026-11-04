# Raw Volume Data Reader
Also called: BrickDensityRegion, Volume Slice Loading, RawReader

## Overview
The Raw Volume Data Reader is a specialized loader for ingesting raw binary volume data files directly into memory. It solves the problem of converting high-precision raw sensor data (e.g., from CT scanners) into a usable format for volume rendering or sparse voxel construction. It differs from generic loaders by embedding specific quantization logic (scaling and thresholding) directly into the loading pipeline rather than treating data as opaque bytes [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A raw binary file containing unsigned short (16-bit) values.
- Output: A contiguous memory block of `unsigned short` values representing the 3D volume, optionally normalized and clamped.

**Data Structures:**
The core data structure `RawReader` (or `BrickDensityRegion`) contains:
- `m_data`: A pointer to `unsigned short` (`unsigned short*`) pointing to a contiguous memory block of size $Z \times Y \times X \times \text{sizeof(unsigned short)}$ [S1, unofficial].

**Parameters:**
- `iso_min`, `iso_max`: Thresholds for iso-clamping. Values outside this range are set to 0 [S2, unofficial].
- `m_density_mult`: A multiplier for density calculation [S2, unofficial].
- `max`: The maximum value used for normalization scaling [S2, unofficial].
- `vol_res`: The target volume resolution (X, Y, Z) [S2, unofficial].
- `file_res`: The source file resolution (X, Z) [S2, unofficial].
- `multiply`: Scaling factor calculated as `vol_res.x() / file_res.x()` [S2, unofficial].

**Steps:**
1. Open the file in binary read mode with the `ate` flag to determine file size [S1, unofficial].
2. Calculate expected file size: $Size = Z \times Y \times X \times \text{sizeof(unsigned short)}$ [S1, unofficial].
3. Assert that the actual file size matches the expected size [S1, unofficial].
4. Allocate memory for the data buffer using `malloc` [S1, unofficial].
5. Seek to the beginning of the file and read the entire content into the allocated memory [S1, unofficial].
6. Close the file [S1, unofficial].
7. Iterate through the data to apply normalization and clamping:
   - Find min and max values in the buffer to determine normalization range [S2, unofficial].
   - For each element, apply iso-clamping: if value $< iso\_min$ or value $> iso\_max$, set to 0 [S2, unofficial].
   - Calculate density value: $val = tmp \times m\_density\_mult \times multiply / max$ [S2, unofficial].
   - Calculate 3D voxel coordinates:
     - $x = i \times multiply$
     - $y = vol\_res.y() - y\_val - 1$
     - $z = j \times multiply$
     where $i$ iterates from 0 to `file_res.x()` and $j$ iterates from 0 to `file_res.z()` [S2, unofficial].
8. Store the processed values in `m_data` [S1, S2, unofficial].
9. Free allocated memory in the destructor [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_raw_volume(filename, vol_res, file_res, iso_min, iso_max, m_density_mult, max_val):
    # Step 1-3: Open file and verify size
    file_size = get_file_size(filename)
    expected_size = vol_res.z * vol_res.y * vol_res.x * 2  # 2 bytes for unsigned short
    if file_size != expected_size:
        raise ValueError("File size mismatch")

    # Step 4-6: Read file into memory
    m_data = malloc(expected_size)
    with open(filename, 'rb') as f:
        f.seek(0)
        data = f.read(expected_size)
        # Convert bytes to unsigned short array
        buffer = unpack_unsigned_shorts(data)
    
    # Step 7: Process data
    multiply = vol_res.x / file_res.x
    processed_data = []
    
    # Find min/max if not provided
    min_val = min(buffer)
    max_val_actual = max(buffer)
    
    for j in range(file_res.z):
        for i in range(file_res.x):
            tmp = buffer[j * file_res.x + i]
            
            # Iso-clamping
            if tmp < iso_min or tmp > iso_max:
                tmp = 0
            
            # Density calculation
            val = tmp * m_density_mult * multiply / max_val
            
            # Coordinate calculation (for mapping if needed)
            x = i * multiply
            y = vol_res.y - (j * multiply) - 1  # Note: y_val logic from source
            z = j * multiply
            
            processed_data.append(val)
            
    # Store in m_data structure
    m_data = processed_data
    return m_data

def free_raw_volume(m_data):
    free(m_data)
```

**Edge Cases and Pitfalls:**
- File size mismatch: The method asserts that the file size matches the expected volume size. If the file is corrupted or incomplete, this will fail [S1, unofficial].
- Precision: The method uses `unsigned short` (16-bit) for storage. If the source data has higher precision, it may be truncated during the initial read or normalization [S1, S2, unofficial].
- Boundary conditions: The iso-clamping sets values outside the range to 0, which may result in loss of data if the range is not chosen carefully [S2, unofficial].

## Subtopics

### [Voxel data loading and quantization](2478-voxel-data-loading-and-quantization.md)
Defines the procedure for ingesting raw binary volume data, applying domain-specific quantization to reduce precision, and preparing the data for GPU transfer. It differs by embedding specific quantization logic (scaling and thresholding) directly into the loading pipeline.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of voxels, due to single-pass reading and processing [S1, S2, unofficial].
- **Memory Complexity:** $O(N)$, storing all voxels in memory as `unsigned short` [S1, unofficial].
- **Strengths:** Simple, fast, and direct loading of raw binary data without complex format parsing [S1, S2, unofficial].
- **Limitations:** Lacks support for various medical imaging formats (e.g., DICOM, NIfTI) and does not handle coordinate system alignment or resampling [S1, S2, unofficial].
- **When to use:** When working with raw binary volume data files and needing a quick, simple loading mechanism [S1, S2, unofficial].
- **When not to use:** When dealing with standard medical imaging formats or requiring advanced features like resampling or coordinate alignment [S1, S2, unofficial].

## Background
The Raw Volume Data Reader is part of the VolumeLoader framework, which supports various medical imaging formats. It is used in contexts where raw binary data from CT scanners or specific model formats needs to be quickly ingested into memory for further processing or rendering [S1, S2, unofficial].

## Sources
[S1] upupming/marching-cubes, https://github.com/upupming/marching-cubes, fetched 2026-10-27 (unofficial)
[S2] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
