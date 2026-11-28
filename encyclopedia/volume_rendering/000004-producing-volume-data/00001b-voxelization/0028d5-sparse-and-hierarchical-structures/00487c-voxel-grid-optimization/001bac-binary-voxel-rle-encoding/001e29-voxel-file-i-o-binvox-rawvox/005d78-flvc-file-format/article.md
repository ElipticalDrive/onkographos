# FLVC File Format

Also called: Free Lossless Voxel Compression, FLVC

## Overview
FLVC is a container format for voxel data that supports zlib-compressed content for non-empty volumes [S1]. It is designed to store sparse voxel octrees (SVOs) and their associated attributes, providing a structured way to serialize voxel grids with metadata [S1]. Unlike raw binary dumps, FLVC includes a header defining volume dimensions, offsets, and attribute schemas, allowing for flexible storage of color, normals, and custom data [S1].

## How it works
**Inputs:** A voxel volume (typically an SVO) with associated attributes (e.g., color, normals).
**Outputs:** A binary file with the `.flvc` extension.

**Data Structures:**
The file consists of a fixed-size header followed by optional compressed content.

1.  **Type Definitions:**
    *   `u8`: 8-bit unsigned integer.
    *   `u16`, `u32`, `u64`: Little-endian unsigned integers of 16, 32, and 64 bits respectively.
    *   `i8`, `i16`, `i32`, `i64`: Two's complement signed integers of 8, 16, 32, and 64 bits respectively (little-endian for >8 bits) [S1].

2.  **Helper Struct `string8`:**
    *   `size`: A non-zero `u8` indicating the length of the string.
    *   `content`: `size` bytes of ASCII-encoded characters [S1].

3.  **Attribute Type Enum (`attribute_type`, `u8`):**
    *   `BOOL` = 0x01 (nonzero is true)
    *   `INT_8` = 0x11, `INT_16` = 0x12, `INT_32` = 0x14, `INT_64` = 0x18
    *   `UINT_8` = 0x21, `UINT_16` = 0x22, `UINT_32` = 0x24, `UINT_64` = 0x28
    *   `FLOAT_32` = 0x41, `FLOAT_64` = 0x42 [S1]

4.  **Attribute Modifier Enum (`attribute_modifier`, `u16`):**
    *   `NON_LOCAL` = 1 (no spatial locality, e.g., unique IDs)
    *   `ALPHA_LAST` = 2 (only for 'color'; alpha is appended last, overriding default ARGB order) [S1]

5.  **Main Header Layout (40 bytes fixed + variable definitions):**
    *   `magic` (`u64`): `0xff11_33cc_666c_7663` (hex). Corresponds to bytes `\xff\x11\x33\xccflvc` [S1].
    *   `version_major` (`u8`): Currently `0` [S1].
    *   `version_minor` (`u8`): Currently `1` [S1].
    *   `volume_offset` (`i32[3]`): Offset of the volume from the origin [S1].
    *   `volume_size` (`u32[3]`): Exact dimensions of the voxel volume [S1].
    *   `empty` (`u8`): `1` if the volume contains no voxels, `0` otherwise [S1].
    *   `padding` (`u8[5]`): Fixed padding bytes with value `"(def)"` to ensure the header is 40 bytes before variable parts [S1].
    *   `definitions_size` (`u16`): Number of attribute definitions following [S1].
    *   `definitions` (`attribute_definition[definitions_size]`): Variable-length array of attribute schemas [S1].
    *   `reserved` (`u8`): Value `'|'` (0x7C). Serves as a visual separator in hex editors and helps detect reading past the header [S1].
    *   `content` (`zlib_stream`): Present only if `empty` is `0` [S1].

6.  **Attribute Definition Struct:**
    *   `identifier` (`string8`): Unique identifier (e.g., `"color"`, `"normal"`) [S1].
    *   `atype` (`attribute_type`): Data type of the attribute [S1].
    *   `cardinality` (`u8`): `0` for variable-sized arrays; non-zero for fixed-size arrays/vectors. Position attributes must have cardinality `3` [S1].
    *   `modifiers` (`u16`): Bitmap of `attribute_modifier` flags [S1].

**Pseudocode for Header Parsing:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_flvc_header(file_stream):
    # Read magic
    magic = read_u64_le(file_stream)
    if magic != 0xff11_33cc_666c_7663:
        raise ValueError("Invalid FLVC magic bytes")

    # Read versions
    version_major = read_u8(file_stream)
    version_minor = read_u8(file_stream)

    # Read volume metadata
    volume_offset = (read_i32_le(file_stream), read_i32_le(file_stream), read_i32_le(file_stream))
    volume_size = (read_u32_le(file_stream), read_u32_le(file_stream), read_u32_le(file_stream))

    # Read empty flag
    empty = read_u8(file_stream)

    # Read padding (5 bytes)
    padding = read_bytes(file_stream, 5)
    if padding != b"(def)":
        # Note: Sources state padding value is "(def)", but implementations should handle potential variations or strict checks
        pass 

    # Read definitions
    definitions_size = read_u16_le(file_stream)
    definitions = []
    for _ in range(definitions_size):
        attr = parse_attribute_definition(file_stream)
        definitions.append(attr)

    # Read reserved byte
    reserved = read_u8(file_stream)
    if reserved != ord('|'):
        # Warning: Reserved byte mismatch, possible corruption or version mismatch
        pass

    return {
        "version": (version_major, version_minor),
        "volume_offset": volume_offset,
        "volume_size": volume_size,
        "empty": empty,
        "definitions": definitions
    }

def parse_attribute_definition(file_stream):
    # Parse string8 identifier
    size = read_u8(file_stream)
    identifier = read_bytes(file_stream, size).decode('ascii')

    # Parse type and cardinality
    atype = read_u8(file_stream)
    cardinality = read_u8(file_stream)

    # Parse modifiers
    modifiers = read_u16_le(file_stream)

    return {
        "identifier": identifier,
        "atype": atype,
        "cardinality": cardinality,
        "modifiers": modifiers
    }
```

**Edge Cases and Pitfalls:**
*   **SVO Dimensions:** Sparse Voxel Octrees (SVOs) internally require power-of-two dimensions, even if `volume_size` specifies exact non-power-of-two dimensions [S1].
*   **Empty Volumes:** If `empty` is `1`, no `zlib_stream` content follows the reserved byte. Attempting to read compressed data will fail or read garbage [S1].
*   **Reserved Byte:** The `'|'` byte is critical for debugging. If an implementation reads past the header without checking this byte, it may misinterpret the start of the zlib stream or garbage data [S1].

## Operating characteristics
*   **Compression:** Uses zlib compression for the voxel data payload. Compression ratio depends on the sparsity and redundancy of the voxel data.
*   **Memory:** The header is small (40 bytes + attribute definitions). The payload size depends on the compressed voxel data.
*   **Performance:** Decompression speed depends on zlib performance. Parsing the header is fast and deterministic.
*   **Strengths:** Flexible attribute system, standard compression, clear structure for debugging.
*   **Limitations:** SVO internal power-of-two constraint may lead to wasted space for non-power-of-two volumes. Not given in the sources: specific compression ratios or throughput metrics.

## Background
FLVC was produced based on research results into voxel compression [S1]. It is part of the "Voxel Compression" project, aiming to provide a robust file format for voxel data [S1].

## Sources
[S1] Free Lossless Voxel Compression (FLVC), https://eisenwave.github.io/voxel-compression-docs/flvc/flvc.html, fetched 2026-10-27
