# Voxel Chunk Compression

Also called: Compress

## Overview
Voxel Chunk Compression reduces the storage size of volumetric data blocks by applying general-purpose compression algorithms to raw voxel buffers [S1, unofficial]. It solves the problem of high memory bandwidth and storage requirements for large-scale voxel worlds by exploiting spatial redundancy within fixed-size data blocks [G]. It is distinguished from sparse structures by operating on dense data layouts and from specialized volume codecs by relying on standard compression primitives rather than volumetric-specific transforms [S1, G, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A contiguous buffer of voxel data (`Data`) containing `NumVoxels` elements of type `FVoxelData`, and a preferred compression codec (`PreferredCodec`) [S1, unofficial].
- Output: A binary buffer (`OutBuffer`) containing a header followed by the compressed payload [S1, unofficial].

**Data Structures:**
- `FVoxelData`: The raw voxel element type [S1, unofficial].
- `Header`: A fixed-size structure containing `Magic`, `FormatVersion`, `CodecId`, `VoxelFormatVersion`, `ChunkSize`, and `UncompressedBytes` [S1, unofficial].
- `Payload`: The compressed byte stream [S1, unofficial].

**Parameters:**
- `PreferredCodec`: An enumeration `EVoxelChunkCodec` specifying the compression algorithm [S1, unofficial].

**Steps:**
1. If `NumVoxels` is 0, return false [S1, unofficial].
2. Calculate the uncompressed size in bytes:
   $$ \text{RawBytes} = \text{NumVoxels} \times \text{sizeof}(FVoxelData) $$
3. Determine `bPlanar` as true if `PreferredCodec` is `LZ4Planar` or `OodlePlanar` [S1, unofficial].
4. If `bPlanar` is true, de-interleave `Data` into a scratch buffer `PlaneScratch` and set source `Src` to `PlaneScratch`; otherwise set `Src` to `Data` [S1, unofficial].
5. Map `PreferredCodec` to compression format `Fmt` [S1, unofficial]. Calculate the maximum compressed size bound:
   $$ \text{Bound} = \text{CompressMemoryBound}(Fmt, \text{RawBytes}) $$
   Compress `Src` into `Payload` [S1, unofficial]. If compression succeeds and `CompressedSize < RawBytes`, set `UsedCodec = PreferredCodec`; otherwise reset `Payload` [S1, unofficial].
6. If `UsedCodec` is `Raw`, copy `Data` to `Payload` via `memcpy` [S1, unofficial].
7. Construct `Header` with `Magic`, `FormatVersion`, `CodecId` (set to `UsedCodec`), `VoxelFormatVersion`, `ChunkSize`, and `UncompressedBytes` [S1, unofficial].
8. Write `Header` then `Payload` to `OutBuffer` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def CompressVoxelChunk(Data: list, NumVoxels: int, PreferredCodec: str) -> bytes:
    if NumVoxels == 0:
        return False
    RawBytes = NumVoxels * sizeof_FVoxelData
    bPlanar = PreferredCodec in ("LZ4Planar", "OodlePlanar")
    if bPlanar:
        Src = DeInterleave(Data)
    else:
        Src = Data
    Fmt = CodecToFormat(PreferredCodec)
    Bound = CompressMemoryBound(Fmt, RawBytes)
    Payload = bytearray(Bound)
    CompressedSize = CompressMemory(Fmt, Src, Payload, RawBytes)
    if CompressedSize is not None and CompressedSize < RawBytes:
        UsedCodec = PreferredCodec
        Payload = Payload[:CompressedSize]
    else:
        UsedCodec = "Raw"
        Payload = bytearray(Src)
    Header = {
        "Magic": 0x564F5843,
        "FormatVersion": 1,
        "CodecId": UsedCodec,
        "VoxelFormatVersion": 1,
        "ChunkSize": (0, 0, 0),
        "UncompressedBytes": RawBytes
    }
    OutBuffer = SerializeHeader(Header) + Payload
    return OutBuffer

# Helper definitions (standard operations)
sizeof_FVoxelData = 4  # Not given in the sources.
def DeInterleave(data): ... # Not given in the sources.
def CodecToFormat(codec): return codec  # Not given in the sources.
def CompressMemoryBound(fmt, size): return size * 2  # Not given in the sources.
def CompressMemory(fmt, src, dst, size): ... # Not given in the sources.
def SerializeHeader(header): ... # Not given in the sources.
```

**Edge Cases and Pitfalls:**
- **Empty Chunks:** Chunks with zero voxels must be explicitly handled to return false immediately [S1, unofficial].
- **Compression Failure:** If the compressed size is not strictly less than the raw size, the method falls back to storing raw data via `memcpy` [S1, unofficial].
- **Planar Layout:** Planar compression (`LZ4Planar`, `OodlePlanar`) requires memory reordering into a scratch buffer before compression [S1, unofficial].

## Subtopics

### Encoded Mesh Decoding
Decodes a compressed voxel mesh representation by scaling down encoded vertex positions and reconstructing the triangle list [S2, unofficial]. It is distinguished by its specific handling of fixed-point position data and index-based triangle reconstruction [S2, unofficial]. Choose this when rendering voxel meshes that have been stored in a compressed, quantized format to save space [S2, unofficial]. The method scales vertex positions by dividing by 256.0f to convert from compressed integer representation back to floating-point world coordinates, copies vertex attributes (material ID, normal), validates that the index count is a multiple of 3, iterates through indices in steps of 3 to form triangles, and copies the mesh offset to the decoded mesh [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Compression time depends on the chosen codec (LZ4/Oodle) [G]. Planar variants add $O(N)$ overhead for de-interleaving [G].
- **Memory Complexity:** Requires additional memory for the scratch buffer during planar compression ($O(N)$) and the payload buffer ($O(N)$) [G].
- **Performance:** Provides high compression ratios for homogeneous or repetitive voxel data [G]. Performance is hardware-dependent on the underlying compression library [G].
- **Strengths:** Simple to implement, leverages highly optimized standard compression libraries, supports multiple codecs [G].
- **Limitations:** Does not exploit volumetric-specific redundancy as effectively as sparse structures [G]. Planar layout increases memory usage during compression [G].
- **When to Use:** When general-purpose compression is sufficient and volumetric-specific codecs are not available or too complex [G].
- **When Not to Use:** When the data is highly sparse or when specific perceptual quality metrics are required [G].

## Background
Voxel Chunk Compression is a technique used in voxel-based game engines and simulation software to manage the large memory footprint of voxel worlds [G]. It is commonly implemented in engines like Unreal Engine's voxel plugins, where it serves as a foundational layer for storing and transmitting voxel data [G]. The method relies on established compression algorithms rather than novel volumetric transforms, making it a robust solution for general voxel data storage [S1, G, unofficial].

## Sources
[S1] draquel/VoxelWorlds, https://github.com/draquel/VoxelWorlds, fetched 2026-10-27 (unofficial)
[S2] Jay2645/VoxelGenerator, https://github.com/Jay2645/VoxelGenerator, fetched 2026-10-27 (unofficial)
