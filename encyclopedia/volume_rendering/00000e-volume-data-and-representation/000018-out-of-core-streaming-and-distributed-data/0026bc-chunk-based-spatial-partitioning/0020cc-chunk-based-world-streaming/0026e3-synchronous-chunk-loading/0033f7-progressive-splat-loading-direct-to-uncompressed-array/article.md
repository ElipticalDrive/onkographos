# Progressive Splat Loading (Direct to Uncompressed Array)

Also called: Direct Splat Loading, Streaming Splat Ingestion

## Overview
Progressive Splat Loading is a method for ingesting chunk-based splat data into an uncompressed array during synchronous chunk loading, enabling immediate rendering of available data without waiting for full chunk assembly. It differs from standard chunk loading by parsing splats directly from a streaming buffer into a global array as bytes arrive, rather than holding them in a temporary structure until the entire chunk is received. This reduces latency for initial visibility but requires careful management of partial data and memory layout.

## How it works
**Inputs and Outputs:**
- Input: A stream of chunk data bytes arriving via the synchronous loader.
- Output: An `UncompressedSplatArray` containing fully parsed splat records ready for rendering.

**Data Structures:**
1. `directLoadBufferIn`: A byte buffer that accumulates incoming chunk data. Its size is not fixed; it grows as data arrives.
2. `standardLoadUncompressedSplatArray`: The target array storing parsed splat records. Each record typically contains position, covariance, color, and opacity fields, though exact field sizes depend on the splat format (e.g., 3D Gaussian Splatting uses 3 floats for position, 3 for covariance, 3 for color, 1 for opacity, totaling 10 floats or 40 bytes per splat).
3. `UncompressedSplatArray`: A dynamic array or pre-allocated buffer that holds the final splat records.

**Numbered Steps:**
1. Initialize `UncompressedSplatArray` to store splats. [S1, unofficial]
2. As chunks arrive from the synchronous loader, copy the raw bytes into `directLoadBufferIn`. [S1, unofficial]
3. Check if `directLoadBufferIn` contains enough bytes to parse at least one complete splat record.
4. If enough bytes are available, parse the new splats from `directLoadBufferIn` into `standardLoadUncompressedSplatArray`. [S1, unofficial]
5. Remove the parsed bytes from `directLoadBufferIn` to free space for new data.
6. Repeat steps 2–5 until all chunks for the current frame or region are processed.

**Formulas:**
- Bytes per splat ($B_{splat}$): For standard 3D Gaussian Splatting, $B_{splat} = 10 \times 4 = 40$ bytes (assuming 32-bit floats). [G]
- Number of splats parseable ($N_{parse}$): $N_{parse} = \lfloor \frac{Size(directLoadBufferIn)}{B_{splat}} \rfloor$.

**Parameters:**
- `ProgressiveLoadSectionSize`: 262144 bytes (256 KB). This defines the chunk size for loading sections. [S1, unofficial]
- `ProgressiveLoadSectionDelayDuration`: 15 ms. A delay parameter possibly used to batch loads or manage thread scheduling. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Partial Splats:** If `directLoadBufferIn` contains fewer than $B_{splat}$ bytes, no splats are parsed. The buffer retains the partial data for the next arrival.
- **Memory Fragmentation:** Frequent parsing and shifting of `directLoadBufferIn` may cause fragmentation if not managed with a circular buffer or compacting strategy.
- **Precision:** Ensure that the parsing logic correctly handles endianness and float precision (32-bit vs 64-bit) as per the source format. [G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class ProgressiveSplatLoader:
    def __init__(self):
        self.direct_load_buffer_in = bytearray()
        self.standard_load_uncompressed_splat_array = []
        self.bytes_per_splat = 40  # 10 floats * 4 bytes

    def on_chunk_arrive(self, chunk_bytes: bytes):
        # Step 2: Copy incoming bytes to buffer
        self.direct_load_buffer_in.extend(chunk_bytes)
        
        # Step 3 & 4: Parse splats if enough data
        while len(self.direct_load_buffer_in) >= self.bytes_per_splat:
            # Extract one splat's worth of bytes
            splat_bytes = self.direct_load_buffer_in[:self.bytes_per_splat]
            # Remove from buffer
            self.direct_load_buffer_in = self.direct_load_buffer_in[self.bytes_per_splat:]
            
            # Parse splat record (simplified)
            splat = self.parse_splat(splat_bytes)
            self.standard_load_uncompressed_splat_array.append(splat)

    def parse_splat(self, bytes_data: bytes):
        # Parse 10 floats from bytes
        # Assuming little-endian, 32-bit floats
        import struct
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
A streaming mode for offline productions that guarantees all mips are streamed in when rendering the current frame by issuing IO requests and immediately waiting on them until complete. [S2] Pitfall: Blocking requests impact frame rate and cause 'skipping' of frames when driven by current game time, but this skipping issue is resolved when using Sequencer because it specifies which SVT frame to render regardless of real-time passage. [S2]

## Operating characteristics
- **Time Complexity:** Parsing is linear with respect to the number of splats ($O(N)$), but the overhead of buffer management and synchronization may add constant factors.
- **Memory Complexity:** Requires additional buffer space for `directLoadBufferIn` and the `UncompressedSplatArray`. The buffer size depends on chunk size and arrival rate.
- **Performance:** Improves initial visibility by allowing rendering of partial data. However, blocking variants can cause frame rate drops if IO is slow. [S2]
- **Strengths:** Low latency for initial display; efficient use of bandwidth by processing data as it arrives.
- **Limitations:** Requires careful buffer management; may not be suitable for real-time applications with strict frame time budgets if IO is blocking.
- **When to Use:** Offline rendering, pre-visualization, or scenarios where initial frame latency is more critical than consistent frame pacing.
- **When Not to Use:** Real-time interactive applications with strict frame time constraints and non-blocking IO preferred.

## Background
Originating from implementations like GaussianSplats3D [S1, unofficial] and Sparse Volume Textures in Unreal Engine [S2], this method addresses the need for efficient streaming of large-scale splat data. It builds on chunk-based spatial partitioning and synchronous loading techniques.

## Sources
[S1] mkkellogg/GaussianSplats3D, https://github.com/mkkellogg/GaussianSplats3D, fetched 2026-10-27 (unofficial)
[S2] Sparse Volume Textures in Unreal Engine | Unreal Engine 5.8 ... - Epic Dev, https://dev.epicgames.com/documentation/en-us/unreal-engine/sparse-volume-textures-in-unreal-engine, fetched 2026-10-28
