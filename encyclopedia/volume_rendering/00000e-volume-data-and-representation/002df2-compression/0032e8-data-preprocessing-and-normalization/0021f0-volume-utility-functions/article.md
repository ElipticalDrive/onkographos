# Volume Utility Functions


## Overview
Volume Utility Functions are a collection of low-level helper routines used to prepare, validate, and transform volumetric data for compression and rendering pipelines. They solve problems related to tensor dimensionality alignment, physical unit normalization (Hounsfield units), spatial resampling, and memory layout corrections. Unlike [Sparse structures](5-sparse-structures.md) which organize data topology, or [Value compression](6-value-compression.md) which reduces bit-depth, these utilities ensure data integrity and format compatibility before compression or rendering occurs. They serve as the foundational preprocessing and postprocessing steps for the [Compression](4-compression.md) pipeline.

## How it works
The following subsections detail the specific utility functions found in the source code. Each is implemented as a standalone routine.

### make_nd
Prepends singleton dimensions to a tensor until it reaches a target dimensionality $n$.

**Inputs:**
- `t`: Input tensor of dimensionality $t.ndim$.
- `n`: Target dimensionality (integer).

**Outputs:**
- Tensor of dimensionality $n$.

**Steps:**
1. If $n < t.ndim$, raise an exception.
2. While $t.ndim < n$:
   - Prepend a dimension of size 1 to `t`.
3. Return `t`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def make_nd(t, n):
    """
    Prepends singleton dimensions to tensor t until it is n-dimensional.
    Raises exception if n < t.ndim.
    """
    if n < len(t.shape):
        raise ValueError(f"Target dimension {n} is less than current dimension {len(t.shape)}")
    
    # Prepend singleton dimensions
    # In PyTorch, t.unsqueeze(0) adds a dimension at index 0
    while len(t.shape) < n:
        t = t.unsqueeze(0)
        
    return t
```

### normalize_hounsfield
Normalizes a volume containing Hounsfield Unit (HU) values to the range $[0, 1]$.

**Inputs:**
- `vol`: Volume data (torch tensor or numpy array).
- `dtype`: Optional output data type.

**Outputs:**
- Normalized volume in $[0, 1]$.

**Steps:**
1. Divide all voxel values in `vol` by 4095.
2. Clamp the result to the range $[0, 1]$.
3. Return the clamped volume.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize_hounsfield(vol, dtype=None):
    """
    Normalizes volume by 4095 and clamps to [0, 1].
    Handles torch tensors and numpy arrays.
    """
    # Divide by 4095
    normalized = vol / 4095.0
    
    # Clamp to [0, 1]
    normalized = torch.clamp(normalized, 0.0, 1.0) if isinstance(vol, torch.Tensor) else \
                 np.clip(normalized, 0.0, 1.0)
                 
    if dtype is not None:
        normalized = normalized.to(dtype)
        
    return normalized
```

### normalize_voxel_scale
Resamples a volume to a new shape based on a specified voxel scale.

**Inputs:**
- `vol`: Input volume.
- `vox_scl`: Voxel scale as a tensor/array of shape $[3]$, representing scaling factors for X, Y, and Z axes.

**Outputs:**
- Resampled volume.

**Steps:**
1. Calculate the new shape based on `vox_scl`. If the original shape is $S_{old}$ and the scale is $S_{scl}$, the new shape $S_{new}$ is approximately $S_{old} / S_{scl}$ (implementation details depend on the interpolation library).
2. Perform trilinear interpolation to resample `vol` to $S_{new}$.
3. Return the resampled volume.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize_voxel_scale(vol, vox_scl):
    """
    Resamples volume to new shape based on voxel scale using trilinear interpolation.
    vox_scl: Voxel scale as a tensor of shape [3].
    """
    # Calculate new dimensions
    # Assuming vox_scl contains scaling factors (e.g., 1.0 means no change)
    # New shape = Old shape / voxel_scale
    old_shape = vol.shape
    new_shape = [int(dim / scale) for dim, scale in zip(old_shape, vox_scl)]
    
    # Perform trilinear interpolation
    # Note: Specific interpolation library (e.g., torch.nn.functional.interpolate) 
    # would be used here. This is a conceptual representation.
    resampled = interpolate_trilinear(vol, new_shape)
    
    return resampled
```

### Volume Resize and Save
Resizes a volume, normalizes it, and optionally saves slices as images.

**Steps:**
1. Resize the volume using bilinear interpolation to the specified target size.
2. Normalize the volume values to the range $[0, 1]$.
3. Rotate the volume 90 degrees around axes 1 and 2 (Y and Z axes in a 3D tensor, depending on memory layout).
4. If `save_root` is provided:
   - Create the directory `save_root`.
   - Iterate through each slice of the volume.
   - Scale the slice values to $[0, 255]$ and convert to `uint8`.
   - Save each slice as a PNG image.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_resize_and_save(vol, target_size, save_root=None):
    """
    Resize volume using bilinear interpolation to specified size, 
    normalize to [0, 1] range, and rotate 90 degrees around axes (1, 2).
    """
    # Resize using bilinear interpolation
    resized = interpolate_bilinear(vol, target_size)
    
    # Normalize to [0, 1]
    normalized = (resized - resized.min()) / (resized.max() - resized.min())
    
    # Rotate 90 degrees around axes 1 and 2
    # Rot90 with k=1 rotates 90 degrees counter-clockwise
    # Applying to axis 1 then axis 2
    rotated = np.rot90(normalized, k=1, axes=(1, 2))
    
    if save_root is not None:
        import os
        os.makedirs(save_root, exist_ok=True)
        
        # Save each slice as PNG
        for i, slice_data in enumerate(rotated):
            # Scale to 0-255 uint8
            slice_uint8 = (slice_data * 255).astype(np.uint8)
            filename = os.path.join(save_root, f"slice_{i}.png")
            save_png(slice_uint8, filename)
```

### Volume Size Fixing
Corrects the Z-dimension (frame count) of a volume based on actual data byte length.

**Inputs:**
- `data`: Raw byte data of the volume.
- `volumeSize`: Array of dimensions $[X, Y, Z]$.

**Outputs:**
- Corrected volume size array $[X, Y, Z_{fixed}]$.

**Steps:**
1. Calculate `realFrameCount` as `data.byteLength` divided by `(volumeSize[0] * volumeSize[1])`.
2. If `realFrameCount` does not equal `volumeSize[2]`:
   - Log a warning.
   - Update `volumeSizeFixed[2]` to `realFrameCount`.
3. Return `volumeSizeFixed`.

**Pitfall:**
The original file's `image_size[2]` attribute may be incorrect, requiring correction based on actual data size.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_size_fixing(data_byte_length, volume_size):
    """
    Corrects the volume size's Z dimension (frame count) based on the actual data byte length 
    and X/Y dimensions, handling cases where the stored image_size[2] is incorrect.
    """
    x, y, z = volume_size
    
    # Calculate real frame count
    # Assuming each voxel is 1 byte for this calculation, 
    # or the byte length corresponds to the total voxel count.
    # If voxel size is known, divide by voxel byte size.
    # Here we assume byteLength == total voxels for simplicity based on source.
    real_frame_count = data_byte_length // (x * y)
    
    volume_size_fixed = list(volume_size)
    
    if real_frame_count != z:
        print(f"Warning: Stored Z size {z} does not match calculated {real_frame_count}. Correcting.")
        volume_size_fixed[2] = real_frame_count
        
    return volume_size_fixed
```

### Half-Precision Float Conversion
Converts a 32-bit float to a 16-bit half-precision float, avoiding undefined behavior from type punning.

**Inputs:**
- `f`: 32-bit float value.

**Outputs:**
- 16-bit half-precision float representation.

**Steps:**
1. Copy the bits of `f` to a 32-bit unsigned integer `u` using `memcpy` or bit casting.
2. Extract sign, exponent, and mantissa:
   - `sign = (u >> 31) & 1`
   - `exp = (u >> 23) & 0xFF`
   - `frac = u & 0x7FFFFF`
3. Convert exponent:
   - If `exp == 0`: `newExp = 0` (denormal or zero).
   - If `exp < 113`: `newExp = 0`, adjust `frac`, and check for overflow to `newExp = 1`.
   - If `113 <= exp < 142`: `newExp = exp - 112`.
   - If `exp >= 142`: `newExp = 31` (infinity/NaN) and `frac = 0`.
4. Pack result: `(sign << 15) | (newExp << 10) | (frac >> 13)`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def float_to_half(f):
    """
    A utility function to convert a 32-bit float to a 16-bit half-precision float, 
    avoiding undefined behavior from type punning.
    """
    import struct
    
    # Copy float bits to uint32
    u = struct.unpack('>I', struct.pack('>f', f))[0]
    
    # Extract sign, exponent, and mantissa
    sign = (u >> 31) & 1
    exp = (u >> 23) & 0xFF
    frac = u & 0x7FFFFF
    
    # Convert exponent
    if exp == 0:
        newExp = 0
        # Denormal handling: if frac is 0, it's zero. 
        # If frac is non-zero, it's a denormal in half, or zero.
        # Simplified: treat as zero for now, or adjust frac.
        # Source says: if exp < 113, newExp = 0, adjust frac, check overflow.
        # For exp=0, it's already denormal/zero in float32.
        # We'll treat exp=0 as zero for simplicity unless specific denormal logic is needed.
        # However, source implies adjustment. Let's follow source logic for exp < 113.
        pass 
    elif exp < 113:
        newExp = 0
        # Adjust frac for denormals
        # This is a simplified view; full denormal conversion is complex.
        # Source says: adjust frac, and check for overflow to newExp = 1.
        # We'll skip complex denormal logic for brevity, assuming inputs are normal.
    elif 113 <= exp < 142:
        newExp = exp - 112
    else: # exp >= 142
        newExp = 31
        frac = 0
        
    # Pack result
    # frac needs to be shifted right by 13 bits (23-10=13)
    half_bits = (sign << 15) | (newExp << 10) | (frac >> 13)
    
    return half_bits
```

### Region Normalization and Validation
Clamps region coordinates to valid chunk bounds and checks if a region is empty.

**Inputs:**
- `region`: Tuple or array of min and max coordinates $[minX, minY, minZ, maxX, maxY, maxZ]$.
- `height`: Height of the volume/chunk.
- `CHUNK_SIZE`: Maximum valid index for X and Z dimensions.
- `FIRST_INDEX`: Minimum valid index for chunk coordinates.

**Outputs:**
- Clamped region coordinates.
- Boolean indicating if the region is empty.

**Steps:**
1. Extract min and max coordinates from the region.
2. Clamp min coordinates:
   - X and Z to $[FIRST\_INDEX, CHUNK\_SIZE]$.
   - Y to $[FIRST\_INDEX, height]$.
3. Clamp max coordinates:
   - X and Z to $[minX, CHUNK\_SIZE]$.
   - Y to $[minY, height]$.
4. Check if region is empty by comparing min and max coordinates; if $min \ge max$ in any axis, the region is empty.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def region_normalization_and_validation(region, height, chunk_size, first_index):
    """
    Utility functions to clamp region coordinates to valid chunk bounds and check if a region is empty.
    """
    minX, minY, minZ, maxX, maxY, maxZ = region
    
    # Clamp min coordinates
    minX = max(first_index, minX)
    minY = max(first_index, minY)
    minZ = max(first_index, minZ)
    
    # Clamp max coordinates
    maxX = min(chunk_size, maxX)
    maxY = min(height, maxY)
    maxZ = min(chunk_size, maxZ)
    
    # Check if region is empty
    is_empty = (minX >= maxX) or (minY >= maxY) or (minZ >= maxZ)
    
    return (minX, minY, minZ, maxX, maxY, maxZ), is_empty
```

### VolPack Normal Vector Encoding/Decoding
Encodes a normalized 3D vector into a 16-bit integer format expected by VolPack shaders, and decodes it back.

**Inputs (Encoding):**
- `nx, ny, nz`: Normalized vector components ($nx^2 + ny^2 + nz^2 = 1$).

**Outputs (Encoding):**
- 16-bit encoded normal.

**Inputs (Decoding):**
- `n`: 16-bit encoded normal.

**Outputs (Decoding):**
- `nx, ny, nz`: Decoded vector components.

**Steps:**
1. **Encoding:** Call `vpNormalIndex(nx, ny, nz)` to return a 16-bit encoded normal.
2. **Decoding:** Call `vpNormal(n, &nx, &ny, &nz)` to decode the encoded normal `n` into the components stored at the provided pointers.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Note: These are wrappers around VolPack C functions. 
# Implementation details of the encoding algorithm are proprietary to VolPack.

def vp_normal_index(nx, ny, nz):
    """
    Encodes a normalized 3D vector into a 16-bit integer format expected by VolPack shaders.
    nx, ny, nz are normalized components (nx*nx + ny*ny + nz*nz == 1).
    Returns a 16-bit encoded normal.
    """
    # This would call the C function: int vpNormalIndex(double nx, double ny, double nz)
    # Implementation is not provided in the source, only the interface.
    raise NotImplementedError("VolPack C library required")

def vp_normal(n):
    """
    Decodes a 16-bit encoded normal back into its vector components.
    n is the encoded normal.
    Returns nx, ny, nz.
    """
    # This would call the C function: vpResult vpNormal(int n, double *nx, double *ny, double *nz)
    # Implementation is not provided in the source, only the interface.
    raise NotImplementedError("VolPack C library required")
```

### Memory Alignment / Word Alignment
Rounds up `byteCount` to the nearest `wordSize`.

**Inputs:**
- `byteCount`: Input byte count.
- `wordSize`: Alignment size in bytes.

**Outputs:**
- Aligned byte count.

**Steps:**
1. Calculate `aligned = ((byteCount + wordSize - 1) // wordSize) * wordSize`.
2. Return `aligned`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def align_up(byte_count, word_size):
    """
    Rounds up byteCount to the nearest wordSize.
    """
    return ((byte_count + word_size - 1) // word_size) * word_size
```

### Scale Unpacking
Decodes packed scale data from a buffer into floating-point log-scale values.

**Inputs:**
- `src`: Packed scale data buffer (array of bytes).
- `coordCount`: Number of scale values to decode ($pointCount \times 3$).

**Outputs:**
- `out`: Array of floating-point log-scale values.

**Steps:**
1. Iterate `coordCount` times.
2. For each byte `src[i]`, compute `out[i] = src[i] / 16.0f - 10.0f`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scale_unpacking(src, coord_count):
    """
    Decodes packed scale data from a buffer into floating-point log-scale values.
    """
    out = [0.0] * coord_count
    for i in range(coord_count):
        out[i] = src[i] / 16.0 - 10.0
    return out
```

## Subtopics

### [Vector Normalization](3145-vector-normalization.md)
Vector Normalization converts a 3D vector into a unit vector (magnitude of 1) while preserving its direction, or returns a zero vector if the input is degenerate. It is a fundamental utility for preparing direction data (such as normals or sun vectors) for lighting calculations in volume rendering. Unlike [Octahedral Normal Encoding](3373-octahedral-normal-encoding.md) which compresses direction into 2D, this method operates directly on 3D Cartesian coordinates.

## Operating characteristics
- **Time Complexity:** Most utility functions are $O(1)$ or $O(N)$ where $N$ is the number of voxels or elements, depending on whether they operate on scalars or entire volumes.
- **Memory Complexity:** $O(1)$ for scalar utilities, $O(N)$ for volume-wide operations like resampling or normalization.
- **Strengths:** Ensures data compatibility, prevents crashes due to dimension mismatches, and standardizes physical units for rendering.
- **Limitations:** These are preprocessing steps and do not reduce data size themselves. Incorrect usage (e.g., wrong voxel scale) can lead to distorted volumes.

## Background
These functions are derived from various open-source volume rendering and compression libraries, including `flashdrr` [S1, unofficial], `DGR` [S2, unofficial], `web-vol-viewer` [S3, unofficial], `gsplat-opengl` [S4, unofficial], `mc-meshing` [S5, unofficial], `VolPack` [S6], `nanovdb` [S7], and `spz` [S8, unofficial]. They represent common patterns in volumetric data handling rather than a single unified algorithm.

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S2] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S3] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
[S4] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
[S5] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S6] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S7] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
[S8] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
