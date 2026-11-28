# 4DGS v1 Progressive Loader

Also called: legacy per-timestep streaming

## Overview
The 4DGS v1 Progressive Loader is a streaming parser for volumetric splat files (`.4dsplat` v1) that loads independent Gaussian blocks per timestep [S1, unofficial]. It solves the problem of rendering animated 4D Gaussian Splatting scenes in web browsers by allowing the viewer to re-upload textures on every timestep change without downloading the entire file [S1, unofficial]. Unlike the [4DGS v2 Spacetime Loader](23028-4dgs-v2-spacetime-loader.md), which separates static and dynamic data for continuous interpolation, this method treats each timestep as a self-contained block, requiring full texture updates when scrubbing through time [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A URL to a `.4dsplat` v1 file, an HTTP client supporting Range requests, and a callback `onTimestep(index, data)`.
*   **Output:** Parsed Gaussian data structures for each timestep, emitted sequentially.

**Data Structures:**
*   **Header (44 bytes):**
    *   Bytes 0-3: Magic string `'4DGS'`.
    *   Bytes 4-7: Version (`uint32` LE).
    *   Bytes 8-11: Number of timesteps (`uint32` LE).
    *   Bytes 12-15: Padding.
    *   Bytes 16-19: FPS (`float32` LE).
    *   Bytes 20-43: Bounding box (6 `float32` LE: `min_x, min_y, min_z, max_x, max_y, max_z`).
*   **Directory (after header):**
    *   Counts: `numTimesteps * 4` bytes (`uint32` LE).
    *   Offsets: `numTimesteps * 4` bytes (`uint32` LE).
*   **Gaussian Record (32 bytes):**
    *   Bytes 0-11: Position (`float32[3]` LE).
    *   Bytes 12-23: Scale (`float32[3]` LE).
    *   Bytes 24-27: Color (`uint8[4]`, normalized by dividing by 255).
    *   Bytes 28-31: Rotation (`int8[4]`, normalized by dividing by 127).

**Parameters:**
*   `TEX_WIDTH`: 4096. Width of the data texture, must match the shader [S1, unofficial].
*   `HEADER_SIZE`: 44 bytes [S1, unofficial].
*   `HEAD_PROBE`: `HEADER_SIZE + 8192` (8236 bytes). Size of the initial HTTP Range request to probe the header and directory [S1, unofficial].

**Steps:**
1.  **Probe Header and Directory:**
    *   Fetch bytes `0` to `HEAD_PROBE - 1` using an HTTP Range request.
    *   If the server returns `206 Partial Content`, parse the directory from the probe buffer to extract `numTimesteps`, counts, and offsets.
    *   If the server returns `200 OK` (ignoring the Range request), the entire file has been downloaded. Parse the full buffer as a complete file and return [S1, unofficial].

2.  **Parse Timestep 0:**
    *   Determine if the first timestep block is fully contained within the probe buffer by checking if `blockEnd(0) <= probeBuf.byteLength`.
    *   If contained, use the probe buffer directly.
    *   If not, fetch the specific byte range for timestep 0 using `fetchRange`.
    *   Call `parseTimestep` on the data and emit `onTimestep(0, ...)` [S1, unofficial].

3.  **Stream Remaining Timesteps:**
    *   Fetch bytes from `m.offsets[1]` to `total - 1` in a single HTTP Range request.
    *   Allocate a `Uint8Array` of size `total - m.offsets[1]`.
    *   Read chunks from the response body into this array.
    *   After each chunk, call `flush()`:
        *   Iterate through `nextT` starting at 1.
        *   Check if `blockEnd(nextT) - restStart <= received`.
        *   If true, call `parseTimestep` and `onTimestep`, then increment `nextT` [S1, unofficial].

**Formulas:**
*   **Center Calculation from Bounding Box:**
    $$center_x = \frac{bbox_{min,x} + bbox_{max,x}}{2}$$
    $$center_y = \frac{bbox_{min,y} + bbox_{max,y}}{2}$$
    $$center_z = \frac{bbox_{min,z} + bbox_{max,z}}{2}$$
    $$radius = 0.5 \times \sqrt{(bbox_{max,x} - bbox_{min,x})^2 + (bbox_{max,y} - bbox_{min,y})^2 + (bbox_{max,z} - bbox_{min,z})^2}$$
    If radius is 0, use 1 [S1, unofficial].

*   **Rotation Normalization:**
    $$q_i = \frac{v_i}{127}$$
    where $v_i$ is the signed int8 value from the file [S1, unofficial].

*   **Color Normalization:**
    $$c_i = \frac{u_i}{255}$$
    where $u_i$ is the unsigned uint8 value from the file [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_4dgs_v1_progressive(url, on_timestep):
    # Constants
    HEADER_SIZE = 44
    HEAD_PROBE_SIZE = HEADER_SIZE + 8192
    
    # Step 1: Probe
    probe_response = http_get_range(url, start=0, end=HEAD_PROBE_SIZE - 1)
    
    if probe_response.status != 206:
        # Server ignored Range request, download full file
        full_buffer = probe_response.body # Assuming body contains full file if 200
        return load_full_file(full_buffer, on_timestep)
    
    # Parse header and directory from probe
    header = parse_header(probe_response.body[:HEADER_SIZE])
    directory = parse_directory(probe_response.body[HEADER_SIZE:])
    
    num_timesteps = header.numTimesteps
    offsets = directory.offsets
    counts = directory.counts
    
    # Step 2: Parse Timestep 0
    t0_end = offsets[0] + counts[0]
    if t0_end <= len(probe_response.body):
        t0_data = probe_response.body[offsets[0]:t0_end]
    else:
        # Fetch specific range for t0
        t0_data = http_get_range(url, start=offsets[0], end=offsets[0] + counts[0] - 1).body
    
    gaussians_t0 = parse_timestep(t0_data, header.bbox)
    on_timestep(0, gaussians_t0)
    
    # Step 3: Stream remaining timesteps
    if num_timesteps > 1:
        remaining_start = offsets[1]
        remaining_size = offsets[-1] + counts[-1] - remaining_start # Assuming last block ends at file end or use total size
        
        # Fetch remaining data
        remaining_response = http_get_range(url, start=remaining_start, end=remaining_size - 1)
        remaining_buffer = remaining_response.body
        
        next_t = 1
        received = 0
        
        # Simulate chunk processing
        for chunk in remaining_response.chunks:
            remaining_buffer.extend(chunk)
            received += len(chunk)
            
            while next_t < num_timesteps:
                block_end = offsets[next_t] + counts[next_t]
                # Adjust for relative start in remaining buffer
                if block_end - remaining_start <= received:
                    t_data = remaining_buffer[offsets[next_t] - remaining_start : block_end - remaining_start]
                    gaussians = parse_timestep(t_data, header.bbox)
                    on_timestep(next_t, gaussians)
                    next_t += 1
                else:
                    break

def parse_header(data):
    magic = data[0:4]
    if magic != b'4DGS':
        raise ValueError("Invalid magic")
    version = struct.unpack_from('<I', data, 4)[0]
    num_timesteps = struct.unpack_from('<I', data, 8)[0]
    fps = struct.unpack_from('<f', data, 16)[0]
    bbox = struct.unpack_from('<ffffff', data, 20)
    return {'version': version, 'numTimesteps': num_timesteps, 'fps': fps, 'bbox': bbox}

def parse_directory(data):
    # Directory follows header. Size depends on numTimesteps.
    # Counts: numTimesteps * 4 bytes
    # Offsets: numTimesteps * 4 bytes
    # Note: Exact start offset depends on header size, passed as 'data' starting after header
    num_t = len(data) // 8 # Assuming equal counts and offsets
    counts = struct.unpack_from('<' + 'I' * num_t, data, 0)
    offsets = struct.unpack_from('<' + 'I' * num_t, data, num_t * 4)
    return {'counts': counts, 'offsets': offsets}

def parse_timestep(data, bbox):
    gaussians = []
    stride = 32
    num_gaussians = len(data) // stride
    
    for i in range(num_gaussians):
        offset = i * stride
        pos = struct.unpack_from('<fff', data, offset)
        scale = struct.unpack_from('<fff', data, offset + 12)
        color_u8 = struct.unpack_from('<BBBB', data, offset + 24)
        rot_i8 = struct.unpack_from('<iiii', data, offset + 28)
        
        # Normalize
        color = [c / 255.0 for c in color_u8]
        rot = [r / 127.0 for r in rot_i8]
        
        gaussians.append({
            'pos': pos,
            'scale': scale,
            'color': color,
            'rot': rot
        })
    return gaussians
```

## Subtopics

### [4DGS v2 Spacetime Loader](23028-4dgs-v2-spacetime-loader.md)
The 4DGS v2 Spacetime Loader is a parsing and loading strategy for `.4dsplat` v2 files that separates a Gaussian Splatting scene into a static "canvas" of persistent primitives and a set of dynamic "tracks" defined by temporal control points and position trajectories.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of timesteps and gaussians per timestep. Parsing is fast, but texture uploads occur per timestep change.
*   **Memory Complexity:** Proportional to the size of the currently loaded timestep block. The streaming approach avoids loading all timesteps into memory simultaneously.
*   **Performance:** Performance depends on network latency and server support for Range requests. If the server ignores Range requests, the entire file is downloaded, increasing initial load time significantly [S1, unofficial].
*   **Strengths:** Simple implementation, works with standard HTTP servers, allows immediate rendering of the first timestep.
*   **Limitations:** Inefficient for scrubbing through time as it requires re-uploading textures for each timestep. File size scales with clip length [S1, unofficial].
*   **When to use:** Legacy `.4dsplat` v1 files, or when the viewer does not support the v2 spacetime format.
*   **When not to use:** For long clips where file size and continuous interpolation are critical; use [4DGS v2 Spacetime Loader](23028-4dgs-v2-spacetime-loader.md) instead.

## Background
The 4DGS v1 Progressive Loader is part of the "4D" web app ecosystem, which transforms videos into renderable Gaussian Splats [S1, unofficial]. It was developed to support the initial `.4dsplat` v1 format, which stores independent Gaussian blocks per timestep [S1, unofficial]. The method is implemented in the `web/components/Splat/loader.ts` file of the devarshh/volumetric-video-4dgs repository [S1, unofficial].

## Sources
[S1] devarshh/volumetric-video-4dgs, https://github.com/devarshh/volumetric-video-4dgs, fetched 2026-10-26 (unofficial)
