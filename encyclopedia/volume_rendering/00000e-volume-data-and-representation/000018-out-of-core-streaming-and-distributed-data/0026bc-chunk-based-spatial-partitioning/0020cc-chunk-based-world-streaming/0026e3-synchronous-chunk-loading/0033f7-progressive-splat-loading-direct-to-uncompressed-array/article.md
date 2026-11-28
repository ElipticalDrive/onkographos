# Progressive Splat Loading (Direct to Uncompressed Array)

Also called: Direct Splat Loading, Streaming Splat Ingestion

## Overview
Progressive Splat Loading ingests chunk-based splat data into an uncompressed array during synchronous chunk loading [S1, unofficial]. As chunks arrive, raw bytes are copied into a streaming buffer, and splats are parsed directly into the target array once enough bytes have accumulated [S1, unofficial]. This approach reduces initial visibility latency but requires careful management of partial data and memory layout [G].

## How it works
**Inputs and Outputs:**
- Input: Chunk data bytes arriving via the synchronous loader [S1, unofficial].
- Output: An `UncompressedSplatArray` containing fully parsed splat records ready for rendering [S1, unofficial].

**Data Structures:**
1. `directLoadBufferIn`: A byte buffer that accumulates incoming chunk data [S1, unofficial].
2. `standardLoadUncompressedSplatArray`: The target array storing parsed splat records [S1, unofficial].
3. Each record stores splat data; exact field sizes depend on the format [G].

**Numbered Steps:**
1. Initialize `UncompressedSplatArray` to store splats [S1, unofficial].
2. As chunks arrive from the synchronous loader, copy the raw bytes into `directLoadBufferIn` [S1, unofficial].
3. Check if `directLoadBufferIn` contains enough bytes to parse at least one complete splat record [G].
4. If enough bytes are available, parse the new splats from `directLoadBufferIn` into `standardLoadUncompressedSplatArray` [S1, unofficial].
5. Remove the parsed bytes from `directLoadBufferIn` to free space for new data [G].
6. Repeat steps 2–5 until all chunks for the current frame or region are processed [G].

**Formulas:**
- Parsing proceeds only when `len(directLoadBufferIn) >= B_splat`, where `B_splat` is the byte size of one complete splat record [G].

**Parameters:**
- `ProgressiveLoadSectionSize`: 262144 bytes [S1, unofficial].
- `ProgressiveLoadSectionDelayDuration`: 15 ms [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Partial Splats:** If `directLoadBufferIn` contains fewer bytes than a complete splat record, the data remains in the buffer until more bytes arrive [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import struct

class ProgressiveSplatLoader:
    def __init__(self):
        self.direct_load_buffer_in = bytearray()
        self.standard_load_uncompressed_splat_array = []
        self.bytes_per_splat = 40  # Example size; depends on format [G]

    def on_chunk_arrive(self, chunk_bytes: bytes):
        # Step 2: Copy incoming bytes to buffer
        self.direct_load_buffer_in.extend(chunk_bytes)
        
        # Step 3: Check buffer size [G]
        # Step 4 & 5: Parse and discard complete splats
        while len(self.direct_load_buffer_in) >= self.bytes_per_splat:
            splat_bytes = self.direct_load_buffer_in[:self.bytes_per_splat]
            del self.direct_load_buffer_in[:self.bytes_per_splat]
            
            splat = self.parse_splat(splat_bytes)
            self.standard_load_uncompressed_splat_array.append(splat)
            
        # Step 6: Loop repeats on next chunk arrival [G]

    def parse_splat(self, bytes_data: bytes):
        # Parse 10 floats from bytes [G]
        values = struct.unpack('<10f', bytes_data)
        return {
            'position': values[0:3],
            'covariance': values[3:6],
            'color': values[6:9],
            'opacity': values[9]
        }
```

## Subtopics

### Blocking Playback
A streaming mode for offline productions that guarantees all mips are streamed in when rendering the current frame by issuing IO requests and immediately waiting on them until complete [S2]. Pitfall: Blocking requests impact frame rate and cause 'skipping' of frames when driven by current game time, but this skipping issue is resolved when using Sequencer because it specifies which SVT frame to render regardless of real-time passage [S2].

## Operating characteristics
Not given in the sources.

## Background
This method is implemented in GaussianSplats3D [S1, unofficial] and discussed in the context of Sparse Volume Textures in Unreal Engine [S2]. It builds on chunk-based spatial partitioning and synchronous loading techniques [G].

## Sources
[S1] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
[S2] Sparse Volume Textures in Unreal Engine | Unreal Engine 5.8 ... - Epic Dev, https://dev.epicgames.com/documentation/en-us/unreal-engine/sparse-volume-textures-in-unreal-engine, fetched 2026-10-28
