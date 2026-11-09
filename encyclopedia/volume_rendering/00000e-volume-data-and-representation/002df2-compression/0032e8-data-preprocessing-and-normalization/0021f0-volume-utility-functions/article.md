# Volume Utility Functions


## Overview
Volume Utility Functions provide a collection of low-level, domain-specific operations for preparing, validating, and transforming volumetric data prior to rendering or analysis. Unlike [Volume Normalization](1823-volume-normalization.md), which focuses on standardizing intensity distributions for visualization, these functions handle structural adjustments (dimensionality, voxel scale), format conversions (half-precision, normal encoding), and data integrity checks (NIfTI analysis, region validation). They serve as the foundational preprocessing layer described in [Data Preprocessing and Normalization](6880-data-preprocessing-and-normalization.md).

## How it works
The module comprises several distinct utility functions, each addressing a specific data transformation or validation need.

### make_nd
Ensures a tensor has a specific number of dimensions by prepending singleton dimensions.

**Inputs:**
- `t`: Input tensor (PyTorch or NumPy).
- `n`: Target number of dimensions (integer).

**Outputs:**
- Tensor with shape `(1, 1, ..., t.shape)` such that `len(output.shape) == n`.

**Steps:**
1. Check if `n < t.ndim`. If true, raise an exception [S1, unofficial].
2. While `t.ndim < n`, prepend a dimension of size 1 to `t` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def make_nd(t, n):
    """
    Prepends singleton dimensions to tensor t until it is n-dimensional.
    """
    if n < t.ndim:
        raise ValueError(f"Target dimensions {n} less than current dimensions {t.ndim}")
    
    # Prepend (n - t.ndim) singleton dimensions
    for _ in range(n - t.ndim):
        t = t.unsqueeze(0)
        
    return t
```

### normalize_hounsfield
Normalizes a medical volume (Hounsfield Units) to the range $[0, 1]$.

**Inputs:**
- `vol`: Input volume (NumPy array or PyTorch tensor).
- `dtype`: Optional output data type.

**Outputs:**
- Volume with values in $[0, 1]$.

**Steps:**
1. Divide all voxel values by 4095 [S1, unofficial].
2. Clamp the result to the range $[0, 1]$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize_hounsfield(vol, dtype=None):
    """
    Normalizes Hounsfield Unit volume by 4095 and clamps to [0, 1].
    """
    # Divide by max HU range
    normalized = vol / 4095.0
    
    # Clamp to [0, 1]
    normalized = torch.clamp(normalized, 0.0, 1.0) if isinstance(normalized, torch.Tensor) else np.clip(normalized, 0.0, 1.0)
    
    if dtype is not None:
        normalized = normalized.to(dtype)
        
    return normalized
```

### normalize_voxel_scale
Resamples a volume to a new shape based on a target voxel scale.

**Inputs:**
- `vol`: Input volume.
- `vox_scl`: Target voxel scale as a tensor/array of shape `[3]` (sx, sy, sz) [S1, unofficial].

**Outputs:**
- Resampled volume with shape adjusted to match the new voxel scale.

**Steps:**
1. Calculate the new shape based on the original physical size and `vox_scl` [S1, unofficial].
2. Resample the volume using trilinear interpolation to the new shape [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize_voxel_scale(vol, vox_scl):
    """
    Resamples volume to new shape based on voxel scale using trilinear interpolation.
    """
    # Assuming vol is a tensor/array with shape (D, H, W) or (C, D, H, W)
    # Calculate new dimensions based on physical size / new voxel size
    # Note: Exact implementation depends on library (e.g., torch.nn.functional.interpolate)
    # This is a conceptual representation of the trilinear resampling step.
    
    # 1. Determine new shape
    # original_spacing = get_original_spacing(vol) # Helper not defined in sources
    # new_shape = calculate_new_shape(original_spacing, vox_scl, vol.shape)
    
    # 2. Resample
    # resampled = interpolate_trilinear(vol, new_shape)
    
    # Placeholder for library-specific resampling logic
    raise NotImplementedError("Trilinear resampling implementation depends on specific tensor library (e.g., PyTorch, NumPy)")
```

### Vector Normalization
Converts a 3D vector into a unit vector.

**Inputs:**
- `v`: Input 3D vector.

**Outputs:**
- Unit vector $\hat{v} = \frac{v}{\|v\|}$.
- Zero vector if $\|v\| < \epsilon$ (precision threshold) [S3, unofficial].

**Steps:**
1. Compute magnitude $\|v\|$ [S3, unofficial].
2. If $\|v\| < \epsilon$, return zero vector [S3, unofficial].
3. Otherwise, return $v / \|v\|$ [S3, unofficial].
4. Some implementations modify the input vector in place [S2, unofficial].

### Half-Precision Float Conversion
Converts between IEEE 754 single-precision (32-bit) and half-precision (16-bit) formats.

**Inputs:**
- 32-bit float or 16-bit half-precision value.

**Outputs:**
- Converted value.
- Packed 32-bit integer containing two half-precision values [S2, unofficial].

**Steps:**
1. Use explicit bit manipulation and `memcpy` to avoid undefined behavior from type punning (e.g., pointer casting) [S2, unofficial].
2. For packing: combine two 16-bit halves into a 32-bit integer [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

def float_to_half(f):
    """Converts a 32-bit float to a 16-bit half-precision float using struct."""
    # Pack as 32-bit float, unpack as 2 16-bit integers (little-endian)
    packed = struct.pack('<f', f)
    # Note: Direct bit manipulation is preferred in C++, but struct is safe in Python
    # This is a simplified representation; actual IEEE 754 conversion requires bit shifting
    # to handle exponent bias and mantissa truncation.
    # For strict implementation, use a library or explicit bit logic.
    raise NotImplementedError("Explicit IEEE 754 bit manipulation required for accurate float16 conversion")

def half_to_float(h):
    """Converts a 16-bit half-precision float to a 32-bit float."""
    raise NotImplementedError("Explicit IEEE 754 bit manipulation required")

def pack_two_halves(h1, h2):
    """Packs two half-precision values into a 32-bit integer."""
    # h1 and h2 are 16-bit integers
    return (h1 & 0xFFFF) | ((h2 & 0xFFFF) << 16)
```

### Volume Resize and Save
Resizes a volume, normalizes it, and optionally saves slices as images.

**Inputs:**
- Volume data.
- Target size.
- `save_root`: Optional directory path.

**Steps:**
1. Resize volume using bilinear interpolation to specified size [S2, unofficial].
2. Normalize values to $[0, 1]$ range [S2, unofficial].
3. Rotate 90 degrees around axes (1, 2) [S2, unofficial].
4. If `save_root` is provided:
   - Create directory.
   - Save each slice as a PNG image scaled to $0-255$ uint8 [S2, unofficial].

### Volume Size Fixing
Corrects the Z dimension (frame count) of a volume based on actual data byte length.

**Inputs:**
- Volume data buffer.
- `volumeSize`: Array `[X, Y, Z]`.

**Outputs:**
- Corrected `volumeSizeFixed`.

**Steps:**
1. Calculate `realFrameCount` as `data.byteLength / (volumeSize[0] * volumeSize[1])` [S3, unofficial].
2. If `realFrameCount != volumeSize[2]`:
   - Log a warning [S3, unofficial].
   - Update `volumeSizeFixed[2]` to `realFrameCount` [S3, unofficial].
3. Return `volumeSizeFixed` [S3, unofficial].

**Pitfall:** The original file's `image_size[2]` attribute may be incorrect, requiring correction based on actual data size [S3, unofficial].

### Region Normalization and Validation
Clamps region coordinates to valid chunk bounds and checks for empty regions.

**Inputs:**
- Region coordinates (min, max).
- `CHUNK_SIZE`: Maximum chunk dimension.
- `height`: Y-dimension height.
- `FIRST_INDEX`: Minimum valid index [S4, unofficial].

**Steps:**
1. Extract min and max coordinates from the region [S4, unofficial].
2. Clamp min coordinates:
   - X and Z to $[FIRST\_INDEX, CHUNK\_SIZE]$ [S4, unofficial].
   - Y to $[FIRST\_INDEX, height]$ [S4, unofficial].
3. Clamp max coordinates:
   - X and Z to $[minX, CHUNK\_SIZE]$ [S4, unofficial].
   - Y to $[minY, height]$ [S4, unofficial].
4. Check if region is empty: if `min >= max` in any axis, the region is empty [S4, unofficial].

### VolPack Normal Vector Encoding/Decoding
Encodes/decodes normalized 3D vectors into 16-bit integer format for VolPack shaders.

**Inputs:**
- Normal vector $(nx, ny, nz)$ where $nx^2 + ny^2 + nz^2 = 1$ [S5].
- Encoded 16-bit integer `n` [S5].

**Outputs:**
- Encoded 16-bit integer [S5].
- Decoded components $(nx, ny, nz)$ [S5].

**Steps:**
1. **Encode:** Call `vpNormalIndex(nx, ny, nz)` to return a 16-bit encoded normal [S5].
2. **Decode:** Call `vpNormal(n, &nx, &ny, &nz)` to decode `n` into components stored at pointers [S5].

### Memory Alignment / Word Alignment
Rounds up a byte count to the nearest word size.

**Inputs:**
- `byteCount`: Input byte count.
- `wordSize`: Alignment size in bytes [S6].

**Outputs:**
- Aligned byte count.

**Steps:**
1. Compute `aligned = ((byteCount + wordSize - 1) / wordSize) * wordSize` [S6].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def align_up(byteCount, wordSize):
    """
    Rounds up byteCount to the nearest wordSize.
    """
    return ((byteCount + wordSize - 1) // wordSize) * wordSize
```

### Scale Unpacking
Decodes packed scale data from a buffer into floating-point log-scale values.

**Inputs:**
- Buffer `src` of bytes.
- `coordCount`: Number of scale values (pointCount * 3) [S7, unofficial].

**Outputs:**
- Array `out` of floating-point values.

**Steps:**
1. Iterate `i` from 0 to `coordCount - 1` [S7, unofficial].
2. Compute `out[i] = src[i] / 16.0f - 10.0f` [S7, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def scale_unpack(src, coordCount):
    """
    Decodes packed scale data from a buffer into floating-point log-scale values.
    """
    out = [0.0] * coordCount
    for i in range(coordCount):
        out[i] = src[i] / 16.0 - 10.0
    return out
```

### NIfTI Analysis and Multi-Channel Debugging
Inspects NIfTI files, displaying metadata, voxel statistics, and multi-channel analysis.

**Inputs:**
- Path to NIfTI file.

**Outputs:**
- Printed metadata and statistics.

**Steps:**
1. **Load:** Load NIfTI file using `nibabel`, get data as `np.float32` [S8, unofficial].
2. **Describe:** Print file path, dtype, shape, num dimensions, voxel size (mm), affine 4x4 matrix, axis orientation, and finite voxel statistics (min, max, mean, std, non-zero count) [S8, unofficial].
3. **Header Fields:** Print header keys: `descrip`, `datatype`, `bitpix`, `scl_slope`, `scl_inter`, `qform_code`, `sform_code`, `xyzt_units`, `toffset`, `intent_name`, `intent_code`, `cal_min`, `cal_max` [S8, unofficial].
4. **Multi-Channel:** If 4D:
   - Print central voxel values across all channels [S8, unofficial].
   - Compute `winner_channel = np.argmax(data, axis=3)` and `winner_value = np.max(data, axis=3)` [S8, unofficial].
   - Create `hard_labels`: if `winner_value > 0`, label is `winner_channel + 1`, else 0 [S8, unofficial].
   - Print histogram of winning labels with counts and percentages [S8, unofficial].

## Subtopics

### [Vector Normalization](3145-vector-normalization.md)
Converts a 3D vector into a unit vector, or returns a zero vector if the input magnitude is below a precision threshold. Used to process direction data such as sun vectors.

### [Half-Precision Float Conversion](3267-half-precision-float-conversion.md)
Converts between IEEE 754 single-precision (32-bit) and half-precision (16-bit) floating-point formats, enabling efficient storage and transmission of volume data using reduced precision formats.

## Operating characteristics
- **Time Complexity:** Depends on the operation. `make_nd` is $O(1)$ (dimension adjustment). `normalize_hounsfield` and `normalize_voxel_scale` are $O(N)$ where $N$ is the number of voxels. `scale_unpack` is $O(N)$ where $N$ is the number of scale values.
- **Memory Complexity:** Generally $O(N)$ for resampling and normalization operations. `make_nd` adds minimal overhead.
- **Strengths:** Provides essential, low-level utilities for data integrity and format compatibility. Handles edge cases like incorrect metadata (Volume Size Fixing) and empty regions.
- **Limitations:** Some functions (e.g., `normalize_voxel_scale`) rely on external interpolation libraries, making implementation details variable. Half-precision conversion requires careful bit manipulation to avoid undefined behavior.
- **When to Use:** Use for preprocessing volumes before rendering or analysis, especially when dealing with medical data (Hounsfield units), multi-channel data, or specific hardware requirements (half-precision, VolPack shaders).

## Background
Volume Utility Functions are a collection of utilities found in various open-source volume rendering and medical imaging projects. They address common preprocessing needs such as dimensionality adjustment, intensity normalization, voxel resampling, and data validation. Key implementations are found in [flashdrr](https://github.com/pcarnah/flashdrr) [S1, unofficial], [DGR](https://github.com/wskingdom/DGR) [S2, unofficial], [web-vol-viewer](https://github.com/JaneliaSciComp/web-vol-viewer) [S3, unofficial], [mc-meshing](https://github.com/nerima-games/mc-meshing) [S4, unofficial], [VolPack](http://www-graphics.stanford.edu/software/volpack/vp_userguide.html) [S5], [OpenVDB](https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html) [S6], [spz](https://github.com/nianticlabs/spz) [S7, unofficial], and [vrdf-sdk](https://github.com/guillaume-schneider/vrdf-sdk) [S8, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S2] wskingdom/DGR, https://github.com/wskingdom/DGR, fetched 2026-10-26 (unofficial)
[S3] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
[S4] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S5] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S6] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
[S7] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
[S8] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
