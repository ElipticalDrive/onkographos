# 4DGS v1 Progressive Loader

Also called: legacy per-timestep streaming

## Overview
The 4DGS v1 Progressive Loader is a streaming parser for the legacy `.4dsplat` v1 file format, designed to load animated 4D Gaussian Splatting scenes in the browser. It solves the problem of high memory consumption and latency by streaming independent Gaussian blocks per timestep, allowing the viewer to re-upload textures on every timestep change without downloading the entire file at once [S1, unofficial]. It sets itself apart from the v2 format by treating each timestep as a separate, self-contained block rather than a unified spacetime structure with shared static data and dynamic trajectories [S1, unofficial].

## How it works
### Inputs and Outputs
- **Input**: A URL pointing to a `.4dsplat` v1 file hosted on an HTTP server.
- **Output**: A sequence of Gaussian sets, one per timestep, emitted via callbacks (e.g., `onTimestep`).

### Data Structures
The loader maintains internal state for the file header, directory, and current parsing position.

**File Header Layout (44 bytes, Little-Endian):**
- Bytes 0-3: Magic string `'4DGS'` (ASCII).
- Bytes 4-7: Version (`uint32`).
- Bytes 8-11: Number of timesteps (`numTimesteps`, `uint32`).
- Bytes 12-15: Padding.
- Bytes 16-19: Frames per second (`fps`, `float32`).
- Bytes 20-43: Bounding box (`bbox`, 6 `float32` values: `min_x, min_y, min_z, max_x, max_y, max_z`).

**Directory Layout (follows header):**
- Bytes 44 to $44 + 4 \times \text{numTimesteps} - 1$: Counts (array of `uint32`, one per timestep).
- Bytes $44 + 4 \times \text{numTimesteps}$ to end: Offsets (array of `uint32`, one per timestep).

**Gaussian Record Layout (32 bytes per Gaussian):**
- Bytes 0-11: Position (`float32[3]`).
- Bytes 12-23: Scale (`float32[3]`).
- Bytes 24-27: Color (`uint8[4]`, RGBA).
- Bytes 28-31: Rotation (`int8[4]`, quaternion components).

**Parameters:**
- `TEX_WIDTH`: 4096. The width of the data texture, must match the shader [S1, unofficial].
- `HEAD_PROBE`: 8236 bytes. Calculated as `HEADER_SIZE` (44) + 8192 [S1, unofficial].

### Steps
1. **Probe Header and Directory**:
   - Issue an HTTP `Range` request for bytes `0` to `HEAD_PROBE - 1`.
   - If the server returns `206 Partial Content`:
     - Parse the header and directory from the received buffer.
     - Validate `numTimesteps` and offsets.
   - If the server returns `200 OK` (ignoring Range):
     - The entire file was downloaded. Fall back to full file parsing [S1, unofficial].

2. **Parse Timestep 0**:
   - Determine if the first timestep block is fully contained within the probe buffer.
   - Condition: `blockEnd(0) <= probeBuf.byteLength`.
   - If true: Use the probe buffer directly.
   - If false: Fetch the specific range for timestep 0 using `fetchRange`.
   - Call `parseTimestep` on the data and emit `onTimestep(0, ...)`.

3. **Stream Remaining Timesteps**:
   - Calculate `total` size from the file or directory.
   - Issue a single HTTP `Range` request for bytes `m.offsets[1]` to `total - 1`.
   - Allocate a `Uint8Array` of size `total - m.offsets[1]`.
   - Read chunks from the response body into this array.
   - After each chunk, call `flush()`:
     - Iterate through `nextT` starting at 1.
     - Check if `blockEnd(nextT) - restStart <= received`.
     - If true, call `parseTimestep` and `onTimestep(nextT, ...)`, then increment `nextT`.

### Formulas
**Center Calculation from Bounding Box:**
$$
center_x = \frac{bbox_{min,x} + bbox_{max,x}}{2}
$$
$$
center_y = \frac{bbox_{min,y} + bbox_{max,y}}{2}
$$
$$
center_z = \frac{bbox_{min,z} + bbox_{max,z}}{2}
$$
$$
radius = 0.5 \times \sqrt{(bbox_{max,x} - bbox_{min,x})^2 + (bbox_{max,y} - bbox_{min,y})^2 + (bbox_{max,z} - bbox_{min,z})^2}
$$
If `radius` is 0, use 1 [S1, unofficial].

**Rotation Normalization:**
$$
q_i = \frac{v_i}{127}
$$
where $v_i$ is the signed `int8` value from the file [S1, unofficial].

**Color Normalization:**
$$
c_i = \frac{u_i}{255}
$$
where $u_i$ is the unsigned `uint8` value from the file [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def parse_4dgs_v1_progressive(url):
    HEAD_PROBE = 8236  # 44 + 8192
    HEADER_SIZE = 44
    
    # Step 1: Probe
    probe_buf = fetch_range(url, 0, HEAD_PROBE - 1)
    status = probe_buf.status
    
    if status == 206:
        # Parse header and directory from probe_buf
        header = parse_header(probe_buf[0:HEADER_SIZE])
        directory = parse_directory(probe_buf[HEADER_SIZE:])
        num_timesteps = header.numTimesteps
        offsets = directory.offsets
        counts = directory.counts
        total_size = offsets[-1] + counts[-1] * 32  # Approximate total size
    elif status == 200:
        # Fallback: full file loaded
        header = parse_header(probe_buf[0:HEADER_SIZE])
        directory = parse_directory(probe_buf[HEADER_SIZE:])
        num_timesteps = header.numTimesteps
        offsets = directory.offsets
        counts = directory.counts
        total_size = len(probe_buf)
    else:
        raise Error("Failed to fetch header")

    # Step 2: Parse Timestep 0
    block0_end = offsets[0] + counts[0] * 32
    if block0_end <= len(probe_buf):
        t0_data = probe_buf[offsets[0]:block0_end]
    else:
        t0_data = fetch_range(url, offsets[0], block0_end - 1)
    
    gaussians_0 = parse_timestep(t0_data)
    on_timestep(0, gaussians_0)

    # Step 3: Stream remaining timesteps
    if num_timesteps > 1:
        remaining_start = offsets[1]
        remaining_size = total_size - remaining_start
        remaining_buf = allocate_uint8array(remaining_size)
        
        # Fetch remaining data in chunks
        chunks = fetch_range_chunks(url, remaining_start, total_size - 1)
        received = 0
        
        for chunk in chunks:
            remaining_buf[received:received+len(chunk)] = chunk
            received += len(chunk)
            
            # Flush logic
            next_t = 1
            while next_t < num_timesteps:
                block_end = offsets[next_t] + counts[next_t] * 32
                # Adjusted end relative to remaining_buf start
                adjusted_end = block_end - remaining_start
                if adjusted_end <= received:
                    # Data for timestep next_t is available
                    t_data = remaining_buf[0:adjusted_end] # Note: offsets are absolute, need to adjust
                    # Correct extraction:
                    abs_start = offsets[next_t]
                    abs_end = offsets[next_t] + counts[next_t] * 32
                    rel_start = abs_start - remaining_start
                    rel_end = abs_end - remaining_start
                    t_data = remaining_buf[rel_start:rel_end]
                    
                    gaussians = parse_timestep(t_data)
                    on_timestep(next_t, gaussians)
                    next_t += 1
                else:
                    break

def parse_header(buf):
    magic = buf[0:4]
    if magic != b'4DGS':
        raise Error("Invalid magic")
    version = read_uint32_le(buf[4:8])
    num_timesteps = read_uint32_le(buf[8:12])
    fps = read_float32_le(buf[16:20])
    bbox = [read_float32_le(buf[20+i*4:24+i*4]) for i in range(6)]
    return {
        'version': version,
        'numTimesteps': num_timesteps,
        'fps': fps,
        'bbox': bbox
    }

def parse_directory(buf):
    # Directory follows header immediately
    # Counts: numTimesteps * 4 bytes
    # Offsets: numTimesteps * 4 bytes
    # Note: In v1, directory is at the end of the header block or immediately after?
    # Based on notes: "Bytes 44+: directory of counts ... followed by offsets"
    # Assuming buf starts at byte 44
    num_t = len(buf) // 8 # Each timestep has 4 bytes count + 4 bytes offset
    counts = []
    offsets = []
    for i in range(num_t):
        counts.append(read_uint32_le(buf[i*8:i*8+4]))
        offsets.append(read_uint32_le(buf[i*8+4:i*8+8]))
    return {'counts': counts, 'offsets': offsets}

def parse_timestep(data):
    gaussians = []
    stride = 32
    num_gaussians = len(data) // stride
    
    for i in range(num_gaussians):
        offset = i * stride
        pos = [read_float32_le(data[offset+j*4:offset+j*4+4]) for j in range(3)]
        scale = [read_float32_le(data[offset+12+j*4:offset+12+j*4+4]) for j in range(3)]
        color_u8 = data[offset+24:offset+28]
        rot_i8 = data[offset+28:offset+32]
        
        color = [c / 255.0 for c in color_u8]
        rot = [r / 127.0 for r in rot_i8]
        
        gaussians.append({
            'position': pos,
            'scale': scale,
            'color': color,
            'rotation': rot
        })
    return gaussians

def fetch_range(url, start, end):
    # Simulate HTTP Range request
    # Returns buffer and status code
    pass

def read_uint32_le(buf):
    return int.from_bytes(buf, byteorder='little', signed=False)

def read_float32_le(buf):
    import struct
    return struct.unpack('<f', buf)[0]
```

### Edge Cases and Pitfalls
- **Server Ignoring Range Requests**: If the server returns `200 OK` instead of `206 Partial Content`, the entire file is downloaded. The loader detects this by checking `probe.status !== 206` and falls back to `loadFull` [S1, unofficial].
- **Absurd Timestep Count**: If the directory end offset exceeds the probe buffer size, the loader bails to a full fetch to avoid parsing errors [S1, unofficial].
- **Precision**: Rotations are normalized by dividing by 127, not 128, which may introduce slight quantization errors compared to standard float representations [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of timesteps and gaussians per timestep. Network latency is mitigated by chunked streaming.
- **Memory Complexity**: Proportional to the size of the current timestep being parsed, not the entire file. This allows rendering large animated scenes on devices with limited memory.
- **Strengths**: Low initial load time, memory efficiency for large files, supports incremental updates.
- **Limitations**: Requires HTTP server support for Range requests. Higher overhead per timestep due to repeated parsing and texture uploads compared to v2's unified structure.
- **When to Use**: When dealing with legacy `.4dsplat` v1 files or when memory constraints prevent loading the entire scene at once.
- **When Not to Use**: For new projects, prefer the v2 format for better compression and unified spacetime handling.

## Background
The 4DGS v1 Progressive Loader is part of the "4D" web app ecosystem, which transforms video into renderable Gaussian Splats. It was developed to support the legacy v1 file format, which stores independent Gaussian blocks per timestep. This approach contrasts with the newer v2 format, which uses a unified spacetime structure. The loader is implemented in TypeScript for the web viewer, leveraging HTTP Range requests for efficient streaming [S1, unofficial].

## Sources
[S1] devarshh/volumetric-video-4dgs, https://github.com/devarshh/volumetric-video-4dgs, fetched 2026-10-26 (unofficial)
