# Volume Utility Functions


## Overview
Volume Utility Functions provide atomic preprocessing and post-processing operations for volumetric data, including tensor dimensionality adjustment, intensity normalization, geometric resampling, memory alignment, and format-specific metadata handling [S1, S2, S3, unofficial][S4, S5][S6, S7, S8, unofficial]. They solve problems related to preparing volume data for rendering, compression, or analysis by offering specific, isolated operations rather than sequential workflows [S1, S2, S3, unofficial][S4, S5][S6, S7, S8, unofficial].

## How it works

### make_nd
Prepends singleton dimensions to a tensor until it reaches a target dimensionality $n$ [S1, unofficial].

**Inputs:**
- `t`: Input tensor with shape $(d_1, d_2, \dots, d_k)$.
- `n`: Target number of dimensions (integer).

**Outputs:**
- A tensor of shape $(1, 1, \dots, 1, d_1, d_2, \dots, d_k)$ with total length $n$ [S1, unofficial].

**Steps:**
1. Check if $n < \text{len}(t.\text{shape})$. If true, raise an exception [S1, unofficial].
2. Calculate the number of dimensions to prepend: $k = n - \text{len}(t.\text{shape})$.
3. Prepend $k$ singleton dimensions to `t` by reshaping [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def make_nd(t, n):
    """
    Prepends singleton dimensions to tensor t until it is n-dimensional.
    Raises exception if n < t.ndim.
    """
    if n < len(t.shape):
        raise ValueError(f"Target dimensions {n} is less than current dimensions {len(t.shape)}")
    
    new_shape = (1,) * (n - len(t.shape)) + t.shape
    return t.reshape(new_shape)
```

### normalize_hounsfield
Normalizes a volume assuming Hounsfield Unit (HU) scale, clamping values to $[0, 1]$ [S1, unofficial].

**Inputs:**
- `vol`: Input volume (array of voxel values).
- `dtype`: Optional output data type.

**Outputs:**
- Normalized volume with values in $[0, 1]$ [S1, unofficial].

**Steps:**
1. Divide all voxel values in `vol` by 4095 [S1, unofficial].
2. Clamp the result to the range $[0, 1]$ [S1, unofficial].

**Formula:**
$$ v_{\text{out}} = \max(0, \min(1, v_{\text{in}} / 4095)) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clamp(x, lo, hi):
    return max(lo, min(hi, x))

def normalize_hounsfield(vol, dtype=None):
    """
    Normalizes volume by 4095 and clamps to [0, 1].
    Handles torch tensors and numpy arrays.
    """
    normalized = [clamp(v / 4095.0, 0.0, 1.0) for v in vol]
    
    if dtype is not None:
        normalized = [x.astype(dtype) if hasattr(x, 'astype') else x for x in normalized]
        
    return normalized
```

### normalize_voxel_scale
Resamples a volume to a new shape based on a target voxel scale using trilinear interpolation [S1, unofficial].

**Inputs:**
- `vol`: Input volume.
- `vox_scl`: Target voxel scale as a tensor/array of shape $[3]$ (e.g., $[s_x, s_y, s_z]$) [S1, unofficial].

**Outputs:**
- Resampled volume [S1, unofficial].

**Steps:**
1. Calculate the new shape based on the original physical size and `vox_scl` [S1, unofficial].
2. Apply trilinear interpolation to resample `vol` to the new shape [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalize_voxel_scale(vol, vox_scl):
    """
    Resamples volume to new shape based on voxel scale using trilinear interpolation.
    vox_scl: Voxel scale as a tensor of shape [3].
    """
    orig_shape = vol.shape
    # Assuming original spacing is 1.0 or derived from metadata [G]
    orig_spacing = [1.0, 1.0, 1.0]
    
    new_shape = [int(orig_shape[i] * orig_spacing[i] / vox_scl[i]) for i in range(3)]
    
    # Apply trilinear interpolation to resample to new_shape [G]
    return trilinear_resample(vol, new_shape)

def trilinear_resample(vol, new_shape):
    """Standard trilinear interpolation implementation [G]"""
    # Implementation omitted for brevity; applies standard 3D trilinear kernel
    pass
```

### Volume Size Fixing
Corrects the Z dimension (frame count) of a volume based on actual data byte length [S2, unofficial].

**Inputs:**
- `volume_size`: Array $[X, Y, Z]$ from file header [S2, unofficial].
- `data_byte_length`: Raw byte buffer length [S2, unofficial].

**Outputs:**
- Corrected `volume_size_fixed` [S2, unofficial].

**Steps:**
1. Calculate `realFrameCount` as `data_byte_length` divided by `(volume_size[0] * volume_size[1])` [S2, unofficial].
2. If `realFrameCount` does not equal `volume_size[2]`, log a warning [S2, unofficial].
3. Update `volume_size_fixed[2]` to `realFrameCount` [S2, unofficial].
4. Return `volume_size_fixed` [S2, unofficial].

**Pitfall:** The original file's `image_size[2]` attribute may be incorrect, requiring correction based on actual data size [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_size_fixing(volume_size, data_byte_length):
    """
    Corrects the volume size's Z dimension based on actual data byte length.
    """
    x, y, z = volume_size
    real_frame_count = data_byte_length // (x * y)
    
    if real_frame_count != z:
        print(f"Warning: Correcting Z dimension from {z} to {real_frame_count}")
        
    volume_size_fixed = [x, y, real_frame_count]
    return volume_size_fixed
```

### Region Normalization and Validation
Clamps region coordinates to valid chunk bounds and checks for empty regions [S3, unofficial].

**Inputs:**
- `region`: Array $[min_x, min_y, min_z, max_x, max_y, max_z]$ [S3, unofficial].
- `height`: Height of the volume [S3, unofficial].
- `chunk_size`: Size of the processing chunk [S3, unofficial].
- `first_index`: Minimum valid index [S3, unofficial].

**Outputs:**
- Clamped region coordinates and boolean indicating if empty [S3, unofficial].

**Steps:**
1. Extract min and max coordinates from the region [S3, unofficial].
2. Clamp min coordinates: $X, Z \in [\text{FIRST\_INDEX}, \text{CHUNK\_SIZE}]$, $Y \in [\text{FIRST\_INDEX}, \text{height}]$ [S3, unofficial].
3. Clamp max coordinates: $X, Z \in [\text{min\_x}, \text{CHUNK\_SIZE}]$, $Y \in [\text{min\_y}, \text{height}]$ [S3, unofficial].
4. Check if region is empty: if $\text{min} \ge \text{max}$ in any axis, the region is empty [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def region_normalization(region, height, chunk_size, first_index):
    """
    Clamps region coordinates to valid chunk bounds and checks if a region is empty.
    """
    min_x, min_y, min_z, max_x, max_y, max_z = region
    
    # Clamp min coordinates
    min_x = max(first_index, min_x)
    min_z = max(first_index, min_z)
    min_y = max(first_index, min_y)
    
    # Clamp max coordinates
    max_x = min(chunk_size, max_x)
    max_z = min(chunk_size, max_z)
    max_y = min(height, max_y)
    
    # Check if empty
    is_empty = (min_x >= max_x) or (min_y >= max_y) or (min_z >= max_z)
    
    return [min_x, min_y, min_z, max_x, max_y, max_z], is_empty
```

### VolPack Normal Vector Encoding/Decoding
Encodes/decodes normalized 3D vectors into 16-bit integer format for VolPack shaders [S4].

**Inputs:**
- Encoding: Normalized vector $(n_x, n_y, n_z)$ where $n_x^2 + n_y^2 + n_z^2 = 1$ [S4].
- Decoding: 16-bit integer `n` [S4].

**Outputs:**
- Encoding: 16-bit integer [S4].
- Decoding: Components $(n_x, n_y, n_z)$ [S4].

**Steps:**
1. **Encode:** Call `vpNormalIndex(nx, ny, nz)`. The components must be normalized. Returns a 16-bit encoded normal [S4].
2. **Decode:** Call `vpNormal(n, &nx, &ny, &nz)`. Decodes `n` into the provided pointers [S4].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vp_normal_index(nx, ny, nz):
    """
    Encodes a normalized 3D vector into a 16-bit integer.
    nx, ny, nz must satisfy nx^2 + ny^2 + nz^2 == 1.
    """
    # VolPack 16-bit encoding scheme is implementation-specific [G]
    raise NotImplementedError("VolPack 16-bit encoding scheme is implementation-specific [G]")

def vp_normal(n):
    """
    Decodes a 16-bit encoded normal back into its vector components.
    """
    # VolPack 16-bit decoding scheme is implementation-specific [G]
    raise NotImplementedError("VolPack 16-bit decoding scheme is implementation-specific [G]")
```

### Memory Alignment / Word Alignment
Rounds up a byte count to the nearest word size [S5].

**Inputs:**
- `byteCount`: Input byte count [S5].
- `wordSize`: Alignment size in bytes [S5].

**Outputs:**
- Aligned byte count [S5].

**Steps:**
1. Calculate aligned size: $\lceil \frac{\text{byteCount}}{\text{wordSize}} \rceil \times \text{wordSize}$ [S5].

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
Decodes packed scale data from a buffer into floating-point log-scale values [S6, unofficial].

**Inputs:**
- `src`: Byte buffer [S6, unofficial].
- `coordCount`: Number of scale values (pointCount * 3) [S6, unofficial].

**Outputs:**
- `out`: Array of floating-point log-scale values [S6, unofficial].

**Steps:**
1. Iterate `coordCount` times [S6, unofficial].
2. For each byte `src[i]`, compute `out[i] = src[i] / 16.0 - 10.0` [S6, unofficial].

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

### Modality Detection via Intensity Threshold
Heuristically classifies volume data as CT or MRI based on minimum scalar value [S8, unofficial].

**Inputs:**
- `volume`: Volume array [S8, unofficial].

**Outputs:**
- String: 'CT' or 'MRI' [S8, unofficial].

**Steps:**
1. Calculate the minimum intensity value in the volume array [S8, unofficial].
2. If $\min < -500$, classify as 'CT' [S8, unofficial].
3. Otherwise, classify as 'MRI' [S8, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def modality_detection(volume):
    """
    Heuristically classifies loaded volume data as CT or MRI based on minimum scalar value.
    """
    min_val = min(volume)
    if min_val < -500:
        return 'CT'
    else:
        return 'MRI'
```

## Subtopics

### [Volume Resize and Save](2904-volume-resize-and-save.md)
Performs post-processing on a reconstructed 3D volume by resizing it to a target dimension, normalizing intensity values, and optionally saving the data as image slices or raw binary files. It solves the problem of converting raw reconstruction outputs into standardized formats for visualization or downstream analysis. It differs from [Image Resizing Utilities](11843-image-resizing-utilities.md) by including specific normalization and rotation steps, and from [Vector Normalization](3145-vector-normalization.md) by handling 3D tensor structures and file I/O.

### [Vector Normalization](3145-vector-normalization.md)
Converts a 3D vector to unit length or identifies its dominant axis, serving as a fundamental preprocessing step for vector field visualization, compression, and geometric calculations in volume rendering. The method encompasses standard Euclidean normalization (unitizing), maximum-index identification, and L-infinity normalization for specific eigenvector storage contexts.

### [Half-Precision Float Conversion](3267-half-precision-float-conversion.md)
Converts between 32-bit IEEE 754 single-precision floats and 16-bit binary16 half-precision floats, and vice versa. It solves the problem of reducing memory bandwidth and storage for volume data while maintaining sufficient precision for rendering, avoiding undefined behavior associated with direct type punning. Unlike sibling utilities that normalize vectors or resize images, this method performs strict numerical format conversion with specific handling for subnormal numbers, overflow, and NaN preservation.

### [Image Resizing Utilities](11843-image-resizing-utilities.md)
Provides a suite of deterministic functions for manipulating image arrays, including resizing (downsampling, upsampling, rescaling), type conversion, and geometric adjustments. They solve the problem of preparing image data for neural network inputs or downstream processing by ensuring correct data types, dimensions, and divisibility constraints. Unlike generic resizing libraries, these utilities enforce strict integer-scale operations where possible to avoid interpolation artifacts and provide specific helpers like `make_divisible` to align dimensions to hardware or model requirements.

### [VTK Image Data Construction from NumPy](15604-vtk-image-data-construction-from-numpy.md)
Imports volume data from NumPy arrays or NIfTI files into VTK, ensuring correct memory layout, data spacing, and origin configuration for normalized volume rendering, particularly for Diffusion Weighted Imaging (DWI) images. It differs from generic array conversion by explicitly handling the geometric metadata (spacing/origin) required to prevent visual artifacts such as volumes appearing too thin.

### NIfTI Analysis and Multi-Channel Debugging
A Python script `analyze.py` inspects NIfTI files, displaying metadata, voxel statistics, multi-channel analysis, and winner-takes-all (argmax) labelmap simulation [S7, unofficial]. It loads NIfTI files using `nibabel`, extracts `fdata` as `np.float32`, and prints file path, dtype, shape, dimensions, voxel size (mm), affine 4x4 matrix, axis orientation, and finite voxel statistics (min, max, mean, std, non-zero count) [S7, unofficial]. For 4D data, it prints central voxel values across channels, argmax channel, per-channel statistics, and generates a hard labelmap histogram using `np.argmax` and thresholding [S7, unofficial].

### Volume Histogram Computation
A global operation that computes the frequency distribution of voxel intensities within a volume-of-interest across a distributed network [S9, unofficial]. Each processor computes a histogram for its local voxel data within the specified `transform.sizes` bounds and applies a `threshold.value` [S9, unofficial]. Partial histograms are sent to a host node, which merges them into a complete 256-bin integer histogram, plots the result, and generates a histogram equalization table for automatic thresholding or edge detection [S9, unofficial].

## Operating characteristics
The utility functions are generally $O(N)$ with respect to the number of voxels or bytes processed, where $N$ is the volume size or data length [S1, S2, S3, unofficial][S4, S5][S6, S7, S8, unofficial]. Memory complexity is $O(1)$ auxiliary space for functions like `make_nd` (which may allocate a new tensor) and $O(N)$ for resampling functions like `normalize_voxel_scale` which produce a new volume [S1, S2, S3, unofficial][S4, S5][S6, S7, S8, unofficial]. Performance is dominated by memory bandwidth for I/O-heavy utilities (e.g., `volume_size_fixing`, `scale_unpacking`) and interpolation cost for `normalize_voxel_scale` [S1, S2, S3, unofficial][S4, S5][S6, S7, S8, unofficial]. These functions are lightweight and suitable for preprocessing pipelines [S1, S2, S3, unofficial][S4, S5][S6, S7, S8, unofficial]. They are not suitable for real-time rendering without caching, as they are designed for data preparation [S1, S2, S3, unofficial][S4, S5][S6, S7, S8, unofficial].

## Background
These utilities originate from various open-source volume rendering and medical imaging projects. `make_nd` and `normalize_hounsfield` are from `flashdrr` [S1, unofficial]. `volume_size_fixing` is from `web-vol-viewer` [S2, unofficial]. `region_normalization` is from `mc-meshing` [S3, unofficial]. `VolPack` functions are from the VolPack User's Guide [S4]. `align_up` is from OpenVDB [S5]. `scale_unpacking` is from `spz` [S6, unofficial]. `nifti_analysis` is from `vrdf-sdk` [S7, unofficial]. `modality_detection` is from `medviewer` [S8, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
[S2] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
[S3] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S4] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S5] OpenVDB: nanovdb::math Namespace Reference, https://openvdb.org/documentation/doxygen/namespacenanovdb_1_1math.html, fetched 2026-10-24
[S6] nianticlabs/spz, https://github.com/nianticlabs/spz, fetched 2026-10-27 (unofficial)
[S7] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
[S8] caitlin-leonard/medviewer, https://github.com/caitlin-leonard/medviewer, fetched 2026-10-27 (unofficial)
[S9] DTIC ADA245410: Voxel Data Processing on a Transputer Network, https://archive.org/details/DTIC_ADA245410, fetched 2026-10-26 (unofficial)
