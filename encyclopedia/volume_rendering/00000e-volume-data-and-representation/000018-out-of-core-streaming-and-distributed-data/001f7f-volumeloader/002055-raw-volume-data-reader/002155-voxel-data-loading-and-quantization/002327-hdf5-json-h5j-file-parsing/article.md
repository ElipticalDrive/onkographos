# HDF5 JSON (H5J) File Parsing

Also called: H5J Attribute Parsing

## Overview
Parses HDF5 JSON (H5J) file attributes to extract volume dimensions, voxel spacing, and channel specifications, applying word alignment to spatial dimensions. It differs from raw binary readers by interpreting the JSON metadata structure to construct the `volSize`, `voxSize`, and `chanSpecs` objects required for subsequent voxel loading and quantization.

## How it works
**Inputs:** A parsed JSON object representing the HDF5 file's attribute hierarchy (typically accessible via a library like `web-h5j-loader` or standard JSON parsing of the `.h5j` metadata).
**Outputs:** An object containing:
- `volSize`: `[number, number, number]` representing width, height, and frames.
- `voxSize`: `[number, number, number]` representing voxel dimensions in physical units.
- `chanSpecs`: `Array<{name: string, content: string}>` mapping channel names to their data types.

**Data Structures:**
- `volSize`: Array of 3 integers.
- `voxSize`: Array of 3 numbers (floats).
- `chanSpecs`: Array of objects with `name` (string) and `content` (string) fields.

**Steps:**
1. Initialize `volSize` and `voxSize` arrays to `[0, 0, 0]`.
2. Check if the `attrs` object contains the key `'image_size'`. If present, assign `attrs.image_size` to `volSize`.
3. Check if the `attrs` object contains the key `'voxel_size'`. If present, assign `attrs.voxel_size` to `voxSize`.
4. Extract `attrs.channels`.
   - If `attrs.channels` contains `'width'`, set `volSize[0]` to `attrs.channels.width[0]`.
   - If `attrs.channels` contains `'height'`, set `volSize[1]` to `attrs.channels.height[0]`.
   - If `attrs.channels` contains `'frames'`, set `volSize[2]` to `attrs.channels.frames[0]`.
   - *Note:* If `image_size` was present in step 2, these channel-specific overrides may replace or supplement the initial `volSize`. The sources imply a fallback or override mechanism where channel dimensions take precedence if explicitly defined in the channels object.
5. Align `volSize[0]` (width) and `volSize[1]` (height) to 8-byte word boundaries.
   - Formula: `alignedSize = Math.ceil(size / 8) * 8`.
   - Apply this to `volSize[0]` and `volSize[1]`.
6. Construct `chanSpecs` from `attrs.channels.names` and `attrs.channels.content_types`.
   - Iterate through the indices of the names array.
   - For each index `i`, create an object `{ name: attrs.channels.names[i], content: attrs.channels.content_types[i] }`.
   - Append to `chanSpecs`.
7. Return the object `{ volSize, voxSize, chanSpecs }`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_h5j_attributes(attrs):
    # Step 1: Initialize
    vol_size = [0, 0, 0]
    vox_size = [0, 0, 0]
    
    # Step 2: Check image_size
    if 'image_size' in attrs:
        vol_size = list(attrs['image_size'])
        
    # Step 3: Check voxel_size
    if 'voxel_size' in attrs:
        vox_size = list(attrs['voxel_size'])
        
    # Step 4: Extract channel dimensions
    if 'channels' in attrs:
        channels = attrs['channels']
        
        if 'width' in channels:
            vol_size[0] = channels['width'][0]
            
        if 'height' in channels:
            vol_size[1] = channels['height'][0]
            
        if 'frames' in channels:
            vol_size[2] = channels['frames'][0]
            
    # Step 5: Word alignment for width and height
    # Align to 8-byte boundaries
    vol_size[0] = math.ceil(vol_size[0] / 8) * 8
    vol_size[1] = math.ceil(vol_size[1] / 8) * 8
    
    # Step 6: Construct channel specs
    chan_specs = []
    if 'channels' in attrs:
        channels = attrs['channels']
        names = channels.get('names', [])
        content_types = channels.get('content_types', [])
        
        # Assuming names and content_types are aligned by index
        for i in range(len(names)):
            name = names[i]
            content = content_types[i] if i < len(content_types) else "unknown"
            chan_specs.append({
                "name": name,
                "content": content
            })
            
    # Step 7: Return result
    return {
        "volSize": vol_size,
        "voxSize": vox_size,
        "chanSpecs": chan_specs
    }
```

**Parameters:**
- Word alignment size: 8 bytes.

**Edge Cases:**
- Missing `image_size` or `voxel_size`: The arrays remain `[0, 0, 0]` unless populated by channel-specific attributes.
- Missing `channels` object: `chanSpecs` will be an empty array.
- Mismatched lengths in `names` and `content_types`: The pseudocode guards against index out-of-bounds by checking `i < len(content_types)`.

## Subtopics

### H5J Volume Data Loading with Cross-Origin Isolation
Loading H5J format volumes, which may contain HDF5 containers with 12-bit HEVC compressed 3D data, often requires the `web-h5j-loader` library. This library relies on multi-threaded `ffmpeg.wasm` via `SharedArrayBuffer` for efficient decoding. To enable `SharedArrayBuffer`, the server must serve the H5J files with the headers `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp`. If these headers are missing, the browser will throw an exception: `'SharedArrayBuffer is not defined'`. This is a security restriction; `SharedArrayBuffer` usage is prevented unless cross-origin isolation is explicitly enabled to mitigate side-channel attacks like Spectre.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of attributes and channels. Parsing JSON is generally fast, dominated by the structure traversal.
- **Memory Complexity:** Minimal, proportional to the size of the metadata object.
- **Strengths:** Provides structured access to volume metadata without loading the actual voxel data. Handles word alignment automatically, which is critical for memory alignment in subsequent GPU/CPU processing.
- **Limitations:** Does not load the voxel data itself. Requires the JSON metadata to be fully available. The alignment step may introduce padding, meaning the logical volume size may differ from the physical data size if not accounted for in the loader.
- **When to use:** When initializing a volume viewer or loader, to determine memory allocation and channel configuration before loading raw voxel data.
- **When not to use:** When only raw binary data is available without JSON metadata, or when the metadata structure differs from the Janelia standard.

## Background
Originating from the Janelia Research Campus's web-based volume viewer tools, this parsing method is part of the `web-vol-viewer` ecosystem. It is designed to handle the specific metadata conventions used in Janelia's HDF5 JSON (H5J) format, which is commonly used for storing large-scale biological imaging data. The method ensures that volume dimensions are correctly aligned for efficient memory access, a common requirement in high-performance volume rendering.

## Sources
[S1] JaneliaSciComp/web-vol-viewer, https://github.com/JaneliaSciComp/web-vol-viewer, fetched 2026-10-26 (unofficial)
