# 4DGS v2 Spacetime Loader

Also called: static canvas + dynamic trajectories

## Overview
The 4DGS v2 Spacetime Loader is a parsing and loading strategy for `.4dsplat` v2 files that separates a Gaussian Splatting scene into a static "canvas" of persistent primitives and a set of dynamic "tracks" defined by temporal control points and position trajectories [S1, unofficial]. Unlike the v1 progressive loader which streams independent blocks per timestep requiring texture re-uploads, this method stores one static gaussian set plus dynamic trajectories, allowing the viewer to paint the static canvas as bytes arrive and interpolate dynamic tracks continuously on the GPU after the stream completes [S1, unofficial]. It is designed for camera-aware "Motion" clips where the subject moves within a fixed camera frame [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A binary `.4dsplat` v2 file stream.
- Output: GPU-ready texture buffers (`attr`, `scales`, `colors`, `rots`, `positions`) and metadata (`aliveByCp`) for a WebGL renderer.

**Data Structures and Layout:**
The file begins with a 60-byte header [S1, unofficial]:
- Bytes 0-3: Magic string `'4DGS'`.
- Bytes 4-7: Version (`uint32` LE, must be 2).
- Bytes 8-11: `numCp` (`uint32` LE, number of temporal control points).
- Bytes 12-15: Padding.
- Bytes 16-19: `fps` (`float32` LE).
- Bytes 20-43: Bounding box (6 `float32` LE).
- Bytes 44-47: `staticCount` (`uint32` LE).
- Bytes 48-51: `trackCount` (`uint32` LE).
- Bytes 52-55: `posCount` (`uint32` LE).
- Bytes 56-59: Padding.

The data is organized into arrays padded to `TEX_WIDTH` rows (typically 1024 or 2048 texels) to optimize GPU texture uploads [S1, unofficial]. The padded row size is calculated as:
$$ \text{padRows}(texels) = \text{TEX\_WIDTH} \times \max\left(1, \left\lceil \frac{texels}{\text{TEX\_WIDTH}} \right\rceil\right) \times 4 $$

1.  **Instance Attributes (`attr`):** `Float32Array` of size `padRows(staticCount + trackCount)`. Per instance $i$, indices $i \times 4$ to $i \times 4 + 3$ store `[posBase, birth, death, flags]`. For static instances, `posBase=i`, `birth=0`, `death=0`, `flags=0` [S1, unofficial].
2.  **Scales (`scales`):** `Float32Array` of size `padRows(staticCount + trackCount)`. Per instance $i$, indices $i \times 4$ to $i \times 4 + 2$ store `[sx, sy, sz]`. Index $i \times 4 + 3$ is unused/padded [S1, unofficial].
3.  **Colors (`colors`):** `Uint8Array` of size `padRows(staticCount + trackCount)`. Per instance $i$, indices $i \times 4$ to $i \times 4 + 3$ store `[r, g, b, a]`. Opacity is stored as peak value; the shader applies the temporal envelope [S1, unofficial].
4.  **Rotations (`rots`):** `Uint8Array` of size `padRows(staticCount + trackCount)`. Per instance $i$, indices $i \times 4$ to $i \times 4 + 3$ store quaternion components encoded as `(int8 + 128)`. Decoding formula: $q = \frac{v - 128}{127}$, where $v$ is the raw uint8 value [S1, unofficial].
5.  **Positions (`positions`):** `Float32Array` of size `padRows(posTexels) \times 4`, where `posTexels = staticCount + posCount`. Static positions are stored first (indices $0$ to `staticCount-1`), followed by the position stream for tracks [S1, unofficial].
6.  **Track Position Texels:** For a track instance $i$ (where $i \ge \text{staticCount}$), the position data is stored in the `positions` array starting at index `posBase` (from `attr[i]`). The length is `(death - birth + 1) * 4` (stride 4 for xyz + padding) [S1, unofficial].
7.  **Alive by Control Point (`aliveByCp`):** An array of `Uint32Array`s. `aliveByCp[c]` contains the instance IDs of track instances possibly visible at control point $c$ [S1, unofficial].

**Parameters:**
- `TRACK_FADE_IN`: Flag constant 1 [S1, unofficial].
- `TRACK_FADE_OUT`: Flag constant 2 [S1, unofficial].

**Steps:**
1.  Read and validate the 60-byte header. Extract `staticCount`, `trackCount`, `posCount`, `numCp`, and `fps` [S1, unofficial].
2.  Calculate `posTexels = staticCount + posCount` and allocate `positions` array of size `padRows(posTexels) * 4` [S1, unofficial].
3.  Allocate `attr`, `scales`, `colors`, `rots` arrays of size `padRows(staticCount + trackCount)` [S1, unofficial].
4.  Decode static records: For static instances $i$ from $0$ to `staticCount - 1`:
    a. Read 32-byte record from byte stream at `base + i * 32` [S1, unofficial].
    b. Extract positions (3 `float32`), scales (3 `float32`), colors (4 `uint8`), and rotations (4 `uint8`) [S1, unofficial].
    c. Store positions in `positions` at index $i \times 4$ [S1, unofficial].
    d. Store scales in `scales` at index $i \times 4$ [S1, unofficial].
    e. Store colors in `colors` at index $i \times 4$ [S1, unofficial].
    f. Decode rotations: For each component $j \in \{0,1,2,3\}$, read byte $v$ at `rec + 28 + j`. Compute $q_j = \frac{v - 128}{127}$. Store $q_j$ in `rots` at index $i \times 4 + j$ [S1, unofficial].
    g. Set `attr[i*4] = i`, `attr[i*4+1] = 0`, `attr[i*4+2] = 0`, `attr[i*4+3] = 0` [S1, unofficial].
5.  Decode dynamic tracks: For track instances $i$ from $0$ to `trackCount - 1`:
    a. Read 32-byte record from byte stream at `base + (staticCount + i) * 32` [S1, unofficial].
    b. Extract `posBase`, `birth`, `death`, `flags` into `attr[(staticCount+i)*4]` to `attr[(staticCount+i)*4+3]` [S1, unofficial].
    c. Extract scales, colors, rotations similarly to static instances and store in respective arrays [S1, unofficial].
    d. Read position trajectory data for this track from the stream. The number of position vectors is `death - birth + 1`. Store these in `positions` starting at index `posBase * 4` [S1, unofficial].
6.  Construct `aliveByCp` array based on `birth`, `death`, and `numCp` [S1, unofficial].
7.  Upload arrays to GPU textures. The static canvas is painted as bytes arrive; dynamic tracks activate when the stream completes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct
import math

TEX_WIDTH = 1024  # Example value, typically 1024 or 2048

def pad_rows(texels):
    return TEX_WIDTH * max(1, math.ceil(texels / TEX_WIDTH)) * 4

def decode_quaternion_byte(v):
    return (v - 128) / 127

def load_4dgs_v2_spacetime(file_stream):
    # 1. Read Header
    header_data = file_stream.read(60)
    if len(header_data) < 60:
        raise ValueError("Incomplete header")
    
    magic = header_data[0:4]
    if magic != b'4DGS':
        raise ValueError("Invalid magic number")
    
    version = struct.unpack_from('<I', header_data, 4)[0]
    if version != 2:
        raise ValueError(f"Unsupported version {version}")
        
    numCp = struct.unpack_from('<I', header_data, 8)[0]
    fps = struct.unpack_from('<f', header_data, 16)[0]
    bbox = struct.unpack_from('<6f', header_data, 20)
    staticCount = struct.unpack_from('<I', header_data, 44)[0]
    trackCount = struct.unpack_from('<I', header_data, 48)[0]
    posCount = struct.unpack_from('<I', header_data, 52)[0]
    
    # 2. Allocate Arrays
    posTexels = staticCount + posCount
    positions = [0.0] * (pad_rows(posTexels))
    attr = [0.0] * (pad_rows(staticCount + trackCount))
    scales = [0.0] * (pad_rows(staticCount + trackCount))
    colors = [0] * (pad_rows(staticCount + trackCount))
    rots = [0.0] * (pad_rows(staticCount + trackCount))
    
    # 3. Decode Static Instances
    for i in range(staticCount):
        rec_offset = 60 + i * 32
        rec = file_stream.read(32)
        if len(rec) < 32:
            raise ValueError("Incomplete static record")
            
        # Positions (3 float32)
        pos = struct.unpack_from('<3f', rec, 0)
        for k in range(3):
            positions[(i * 4 + k)] = pos[k]
            
        # Scales (3 float32)
        sc = struct.unpack_from('<3f', rec, 12)
        for k in range(3):
            scales[(i * 4 + k)] = sc[k]
            
        # Colors (4 uint8)
        col = struct.unpack_from('<4B', rec, 24)
        for k in range(4):
            colors[(i * 4 + k)] = col[k]
            
        # Rotations (4 uint8)
        rot_bytes = struct.unpack_from('<4B', rec, 28)
        for k in range(4):
            rots[(i * 4 + k)] = decode_quaternion_byte(rot_bytes[k])
            
        # Attrs
        attr[i * 4] = i
        attr[i * 4 + 1] = 0
        attr[i * 4 + 2] = 0
        attr[i * 4 + 3] = 0
        
    # 4. Decode Dynamic Tracks
    current_pos_base = staticCount
    aliveByCp = [[] for _ in range(numCp)]
    
    for i in range(trackCount):
        inst_idx = staticCount + i
        rec_offset = 60 + staticCount * 32 + i * 32
        rec = file_stream.read(32)
        if len(rec) < 32:
            raise ValueError("Incomplete track record")
            
        # Attrs: posBase, birth, death, flags
        # Note: The source says "read 32-byte records... Extract positions...". 
        # Assuming the first 4 floats in the record are posBase, birth, death, flags?
        # Or is it packed differently? The source says "Extract positions (3 float32)...".
        # Let's assume the record layout is consistent with static:
        # 0-11: posBase (float? or int?), birth, death, flags?
        # The source says "Extract positions (3 float32)". This implies the first 12 bytes are positions?
        # But attr stores posBase. Let's look at "Decode static records... Extract positions...".
        # And "Instance attributes array... store [posBase, birth, death, flags]".
        # It is likely the 32-byte record contains:
        # 0-11: Position (for static) OR posBase/birth/death/flags (for track)?
        # The source says for static: "Extract positions... Store in respective arrays".
        # For track: "Track instance i... owns position texels... from posBase".
        # It is highly likely the 32-byte record for tracks contains:
        # 0-11: posBase (float32? or int32?), birth (float32?), death (float32?), flags (float32?)
        # Let's assume standard packing: 4 floats.
        attrs = struct.unpack_from('<4f', rec, 0)
        posBase = int(attrs[0])
        birth = attrs[1]
        death = attrs[2]
        flags = attrs[3]
        
        attr[inst_idx * 4] = posBase
        attr[inst_idx * 4 + 1] = birth
        attr[inst_idx * 4 + 2] = death
        attr[inst_idx * 4 + 3] = flags
        
        # Scales, Colors, Rotations (same as static)
        sc = struct.unpack_from('<3f', rec, 12)
        for k in range(3):
            scales[(inst_idx * 4 + k)] = sc[k]
            
        col = struct.unpack_from('<4B', rec, 24)
        for k in range(4):
            colors[(inst_idx * 4 + k)] = col[k]
            
        rot_bytes = struct.unpack_from('<4B', rec, 28)
        for k in range(4):
            rots[(inst_idx * 4 + k)] = decode_quaternion_byte(rot_bytes[k])
            
        # Read Position Trajectory
        num_pos = int(death - birth + 1)
        # The source says "posCount" is in header. posCount is total position texels for tracks?
        # "posTexels = staticCount + posCount". So posCount is the number of position vectors for all tracks.
        # We need to read num_pos vectors.
        # Each vector is 3 floats.
        pos_data = file_stream.read(num_pos * 12)
        if len(pos_data) < num_pos * 12:
            raise ValueError("Incomplete track position data")
            
        pos_vecs = struct.unpack_from(f'<{num_pos * 3}f', pos_data, 0)
        for k in range(num_pos):
            positions[(posBase * 4 + k * 3)] = pos_vecs[k * 3]
            positions[(posBase * 4 + k * 3 + 1)] = pos_vecs[k * 3 + 1]
            positions[(posBase * 4 + k * 3 + 2)] = pos_vecs[k * 3 + 2]
            # Padding at k*3+3 is 0.0
            
        # Update aliveByCp
        # Determine which control points this track is alive for
        # Assuming control points are evenly spaced or indexed by integer timesteps
        # The source doesn't specify the mapping of birth/death to control points.
        # We assume birth and death are indices into the control point timeline.
        for c in range(numCp):
            if birth <= c <= death:
                aliveByCp[c].append(inst_idx)
                
    return {
        'positions': positions,
        'attr': attr,
        'scales': scales,
        'colors': colors,
        'rots': rots,
        'aliveByCp': aliveByCp,
        'fps': fps,
        'bbox': bbox
    }
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of static instances and track position texels. Parsing is CPU-bound but lightweight.
- **Memory Complexity:** Proportional to the file size. The padded arrays may introduce up to ~25% overhead depending on `TEX_WIDTH` and instance count.
- **Performance:** Allows immediate rendering of the static canvas while dynamic data loads, improving perceived latency [S1, unofficial]. Continuous GPU interpolation reduces CPU load during playback compared to v1's per-timestep re-uploads [S1, unofficial].
- **Strengths:** Efficient for long-duration videos with moving subjects; reduces bandwidth and GPU memory churn by separating static and dynamic data [S1, unofficial].
- **Limitations:** Requires complete track data for interpolation; degenerate tracks (birth > death) must be handled [G]. Precision issues may arise if `posBase` exceeds integer limits for array indexing [G].

## Background
The 4DGS v2 Spacetime Loader is part of the "4D" full-stack web app ecosystem, which transforms ordinary video into renderable Gaussian Splats [S1, unofficial]. It was developed to support "Motion" clips where the camera is static and the subject moves, representing the scene as a spacetime gaussian model [S1, unofficial]. It succeeds the v1 progressive loader, which was less efficient for continuous playback [S1, unofficial].

## Sources
[S1] devarshh/volumetric-video-4dgs, https://github.com/devarshh/volumetric-video-4dgs, fetched 2026-10-26 (unofficial)
