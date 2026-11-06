# VRDF File Format / VRDF Loader

Also called: Volume Rendering Data Format

## Overview
The VRDF (Volume Rendering Data Format) is a binary file format designed for storing 3D voxel grids alongside associated metadata and transfer functions. It solves the problem of interoperability between medical imaging pipelines and volume rendering engines by packaging geometry, intensity data, and rendering parameters into a single self-describing file. Unlike [Binvox File Parsing](2862-binvox-file-parsing.md), which is limited to binary occupancy grids, VRDF supports continuous floating-point data, multi-channel interleaved data (e.g., label and weight), and embedded JSON metadata. It builds on [Voxel File I/O](2786-voxel-file-i-o.md) by providing a structured schema for complex volume data rather than simple raw bytes.

## How it works

### Inputs and Outputs
**Input:** A binary file conforming to the VRDF specification.
**Output:** A structured volume object containing:
- `dim`: Array of 3 integers `[dimX, dimY, dimZ]`.
- `spacing_mm`: Array of 3 floats `[sx, sy, sz]`.
- `affine`: 4x4 transformation matrix.
- `mode`: String indicating data type.
- `volumeData`: Array of voxel values (layout depends on `mode` and `channels`).
- `transferFunction`: Parsed transfer function definition.

### Data Structures

#### File Layout
The file consists of sequential blocks. All multi-byte integers are Little-Endian (LE).

1. **Magic**: 8 bytes ASCII `"VRDF0001"`.
2. **Total Size**: 8 bytes `uint64 LE`. The total size of the file in bytes.
3. **Metadata Block**:
   - 8 bytes `uint64 LE`: `metaLen` (length of JSON payload).
   - `metaLen` bytes: UTF-8 encoded JSON string.
4. **Transfer Function Block**:
   - 8 bytes `uint64 LE`: `tfLen` (length of JSON payload).
   - `tfLen` bytes: UTF-8 encoded JSON string.
5. **Raw Data Block**:
   - 8 bytes `uint64 LE`: `rawLen` (length of voxel data).
   - `rawLen` bytes: Raw voxel data.

#### VRDFMeta Struct
Parsed from the Metadata JSON. Fields:
- `dim`: `int[3]`
- `spacing_mm`: `float[3]`
- `dtype`: `string`
- `intensity_range`: `float[2]` (min, max)
- `affine`: `float[4,4]` (16 floats, row-major order in JSON)
- `mode`: `string` (`"labelmap"`, `"continuous"`, `"anatomy_label"`, `"activity_weight"`, `"anatomy_label_weighted"`)
- `endianness`: `string`
- `order`: `string` (e.g., `"XYZ"`)
- `channels`: `int`
- `channel_meaning`: `List<string>`

#### VRDFTransferFunction Struct
Parsed from the Transfer Function JSON. Fields:
- `type`: `string`
- `entries`: `List<VRDFTFEntry>`
- `curve`: `List<TFPoint>`
- `origin`: `string`
- `channel_names_hint`: `List<string>`

#### VRDFTFEntry Struct
- `label`: `float`
- `color`: `float[3]` (RGB)
- `alpha`: `float`
- `name`: `string`

#### TFPoint Struct
- `x`: `float`
- `color`: `float[3]`
- `alpha`: `float`

### Steps

1. **Open File**: Open the input file in binary read mode.
2. **Validate Magic**: Read 8 bytes. If not equal to `"VRDF0001"`, abort with error.
3. **Read Total Size**: Read 8 bytes as `uint64 LE` into `totalSize`.
4. **Read Metadata**:
   - Read 8 bytes as `uint64 LE` into `metaLen`.
   - Read `metaLen` bytes.
   - Decode as UTF-8 JSON.
   - Parse into `VRDFMeta` struct.
5. **Read Transfer Function**:
   - Read 8 bytes as `uint64 LE` into `tfLen`.
   - Read `tfLen` bytes.
   - Decode as UTF-8 JSON.
   - Parse into `VRDFTransferFunction` struct.
6. **Read Raw Data**:
   - Read 8 bytes as `uint64 LE` into `rawLen`.
   - Read `rawLen` bytes into a byte buffer.
7. **Parse Voxel Data**: Dispatch based on `meta.mode` and `meta.channels`.

### Formulas and Validation

**Voxel Count**:
$$ N = dimX \times dimY \times dimZ $$

**Validation for `anatomy_label_weighted` (2 channels)**:
If `meta.mode == "anatomy_label_weighted"` and `meta.channels == 2`:
$$ rawLen == N \times 2 \times 4 \text{ bytes} $$
Data is interleaved `[label, weight]` pairs. Each pair is 8 bytes.
For voxel $i$ from $0$ to $N-1$:
- Label offset: $i \times 8$
- Weight offset: $i \times 8 + 4$
- Both are `float32` (4 bytes).

**Validation for Legacy/Single-Channel**:
If `meta.mode` is `"labelmap"`, `"continuous"`, `"anatomy_label"`, or `"activity_weight"`:
$$ rawLen == N \times 4 \text{ bytes} $$
Data is a contiguous array of `float32`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct
import json
import numpy as np

def load_vrdf(file_path):
    with open(file_path, 'rb') as f:
        # 1. Magic
        magic = f.read(8)
        if magic != b'VRDF0001':
            raise ValueError(f"Bad magic '{magic}', expected 'VRDF0001'.")

        # 2. Total Size
        total_size = struct.unpack('<Q', f.read(8))[0]

        # 3. Metadata
        meta_len = struct.unpack('<Q', f.read(8))[0]
        meta_json_bytes = f.read(meta_len)
        meta_dict = json.loads(meta_json_bytes.decode('utf-8'))
        
        # Parse Meta Struct
        meta = {
            'dim': tuple(meta_dict['dim']),
            'spacing_mm': tuple(meta_dict['spacing_mm']),
            'mode': meta_dict['mode'],
            'channels': meta_dict['channels'],
            'affine': np.array(meta_dict['affine']).reshape(4, 4), # Row-major in JSON
            'intensity_range': tuple(meta_dict['intensity_range'])
        }

        # 4. Transfer Function
        tf_len = struct.unpack('<Q', f.read(8))[0]
        tf_json_bytes = f.read(tf_len)
        tf_dict = json.loads(tf_json_bytes.decode('utf-8'))
        
        # Parse TF Struct (simplified)
        tf = {
            'type': tf_dict.get('type', 'none'),
            'entries': tf_dict.get('entries', [])
        }

        # 5. Raw Data
        raw_len = struct.unpack('<Q', f.read(8))[0]
        raw_bytes = f.read(raw_len)

        # 6. Parse Voxel Data
        dim = meta['dim']
        voxel_count = dim[0] * dim[1] * dim[2]
        
        # Handle Affine Fallback [G]
        if meta['affine'] is None or not np.isfinite(meta['affine']).all():
            spacing = meta['spacing_mm']
            meta['affine'] = np.diag([spacing[0], spacing[1], spacing[2], 1.0])

        volume_data = None
        label_data = None
        weight_data = None

        mode = meta['mode']
        channels = meta['channels']

        if mode == "anatomy_label_weighted" and channels == 2:
            # Interleaved Label/Weight
            if raw_len != voxel_count * 8:
                raise ValueError(f"Raw length mismatch for interleaved data: {raw_len} != {voxel_count * 8}")
            
            label_data = np.empty(voxel_count, dtype=np.float32)
            weight_data = np.empty(voxel_count, dtype=np.float32)
            
            stride = 8
            for i in range(voxel_count):
                off = i * stride
                # Unpack interleaved floats
                lbl = struct.unpack_from('<f', raw_bytes, off + 0)[0]
                wgt = struct.unpack_from('<f', raw_bytes, off + 4)[0]
                label_data[i] = lbl
                weight_data[i] = wgt
                
        else:
            # Single Channel / Legacy
            expected_len = voxel_count * 4
            if raw_len != expected_len:
                raise ValueError(f"Raw length mismatch for single channel: {raw_len} != {expected_len}")
            
            # Buffer.BlockCopy equivalent in Python: direct view
            volume_data = np.frombuffer(raw_bytes, dtype=np.float32, count=voxel_count)

    return {
        'meta': meta,
        'tf': tf,
        'volume_data': volume_data,
        'label_data': label_data,
        'weight_data': weight_data
    }
```

### Parameters
- **Magic**: `"VRDF0001"` (8 bytes).
- **Endianness**: Little-Endian for all integers.
- **Float Precision**: `float32` (4 bytes).
- **JSON Encoding**: UTF-8, compact separators `(",",":")` recommended for size efficiency.

### Edge Cases and Pitfalls
- **Affine Matrix Order**: JSON stores the affine matrix in row-major order. When loading into column-major systems (e.g., Unity's `Matrix4x4`), use row-setting methods or transpose carefully. If the affine is missing or malformed, fallback to a diagonal matrix using `spacing_mm` and zero translation [S1, unofficial].
- **Interleaved Data**: `Buffer.BlockCopy` or direct memory mapping cannot efficiently unpack interleaved `[label, weight]` pairs due to stride. Manual iteration with `struct.unpack` or equivalent is required [S1, unofficial].
- **4D Volumes**: Some export modes handle 4D data (X,Y,Z,T) by splitting into multiple VRDF files (e.g., `_t00.vrdf`). The loader should expect single 3D volumes per file unless explicitly handling a sequence [S1, unofficial].

## Subtopics

### Binvox File Parsing
- **What it is**: A loader for the `.binvox` format, a compact binary format for 3D voxel grids.
- **Difference**: Supports only binary occupancy (0 or 1) and lacks metadata/transfer functions. VRDF supports continuous data and rich metadata.
- **When to choose**: Use for simple occupancy grids where file size is critical and no intensity/label data is needed.
- **Link**: [Binvox File Parsing](2862-binvox-file-parsing.md)

### IMS File Format & Image Stack Class
- **What it is**: A loader for IMS (Image Stack) files, often used in microscopy.
- **Difference**: IMS is typically a stack of 2D images with specific header structures for microscopy metadata. VRDF is a unified 3D volume format with embedded rendering parameters.
- **When to choose**: Use when working with microscopy data in its native IMS container.
- **Link**: [IMS File Format & Image Stack Class](3322-ims-file-format-image-stack-class.md)

## Operating characteristics

### Time and Memory Complexity
- **Time**: $O(N)$ where $N$ is the number of voxels. Parsing JSON is $O(M)$ where $M$ is metadata size (typically small). Interleaved unpacking is $O(N)$ with higher constant factor than contiguous copy.
- **Memory**: $O(N)$ for the volume data. Metadata and TF are $O(1)$ relative to $N$.

### Measured Performance
- **Hardware**: Standard CPU.
- **Resolution**: Performance scales linearly with voxel count.
- **Scaling**: Efficient for large volumes due to binary layout. JSON parsing overhead is negligible compared to I/O and data unpacking.

### Strengths and Limitations
- **Strengths**: Self-describing (metadata embedded), supports complex data types (label+weight), compact binary storage, easy to parse with standard libraries.
- **Limitations**: No compression (raw data is uncompressed), JSON metadata can be verbose for very large scenes if not optimized, single-threaded parsing.

### When to Use
- Use when you need to exchange 3D volume data with intensity, labels, and rendering parameters between tools.
- Use when interoperability with Python/NumPy ecosystems is required (as the SDK is Python-based).

### When Not to Use
- Do not use for massive datasets requiring compression (consider [Binary Voxel RLE Encoding](2861-binary-voxel-rle-encoding.md)).
- Do not use if strict binary-only metadata is required (JSON is text-based).

## Background
The VRDF format was developed by Guillaume Schneider as part of the `vrdf-sdk` repository. It is an unofficial format designed to bridge the gap between medical imaging data (often in NIfTI or DICOM) and real-time volume rendering engines. The format emphasizes simplicity and self-containment, embedding all necessary rendering parameters (transfer functions) within the file.

## Sources
[S1] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
