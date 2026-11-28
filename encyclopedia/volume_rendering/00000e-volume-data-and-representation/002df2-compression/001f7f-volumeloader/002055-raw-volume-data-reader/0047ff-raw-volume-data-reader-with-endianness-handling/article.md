# Raw Volume Data Reader with Endianness Handling

Also called: Raw Binary Volume Data Loading with Endianness Handling

## Overview
This method extends the [Raw Volume Data Reader](2259-raw-volume-data-reader.md) by explicitly handling byte-order mismatches between the source file and the host system. It solves the problem of corrupted voxel values when reading 16-bit signed integer (`short`) data from machines with different endianness (e.g., big-endian files on little-endian x86 hardware). It sets itself apart from generic readers by enforcing a specific byte-combination logic for 16-bit values and guarding against sign-extension errors during byte masking.

## How it works
**Inputs:**
- Raw binary file stream containing 16-bit signed integers.
- Dimensions: `CT_z_axis`, `CT_y_axis`, `CT_x_axis`.
- `isCorrectEndian`: Boolean flag indicating if the file's byte order matches the system's native order.

**Outputs:**
- 3D array `volume` of type `short` with layout `[CT_z_axis][CT_y_axis][CT_x_axis]`.
- `min` and `max` values of the loaded data.

**Data Structures:**
- `volume`: A 3D array of `short` (16-bit signed integers).
- `min`: `short`, initialized to `Short.MAX_VALUE`.
- `max`: `short`, initialized to `Short.MIN_VALUE`.

**Steps:**
1. Initialize `min = Short.MAX_VALUE` and `max = Short.MIN_VALUE` [S1, unofficial].
2. Iterate through the volume dimensions in order: `k` from `0` to `CT_z_axis-1`, `j` from `0` to `CT_y_axis-1`, `i` from `0` to `CT_x_axis-1` [S1, unofficial].
3. Read two bytes from the input stream: `b1` and `b2`. Each byte must be masked with `0xff` to prevent sign-extension of negative byte values into the 16-bit integer [S1, unofficial].
   - `b1 = in.readByte() & 0xff`
   - `b2 = in.readByte() & 0xff`
4. Combine the bytes into a 16-bit value based on `isCorrectEndian` [S1, unofficial]:
   - If `isCorrectEndian` is `false` (byte swap required): `value = (b2 << 8) | b1`
   - If `isCorrectEndian` is `true` (native order): `value = (b1 << 8) | b2`
5. Cast the result to `short` and assign to `volume[k][j][i]` [S1, unofficial].
6. Update `min` and `max` if the new `value` is smaller or larger, respectively [S1, unofficial].

**Pitfalls:**
- **Sign Extension:** Reading raw bytes without masking with `0xff` will sign-extend negative bytes (e.g., `0xFF` becomes `0xFFFF` or `-1` in 16-bit) to 16-bit integers, corrupting the value. The code explicitly masks: `& 0xff` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_raw_volume_with_endianness(file_stream, CT_z_axis, CT_y_axis, CT_x_axis, is_correct_endian):
    # Initialize bounds
    min_val = 32767  # Short.MAX_VALUE
    max_val = -32768 # Short.MIN_VALUE
    
    # Initialize 3D array
    volume = [[[0 for _ in range(CT_x_axis)] for _ in range(CT_y_axis)] for _ in range(CT_z_axis)]
    
    for k in range(CT_z_axis):
        for j in range(CT_y_axis):
            for i in range(CT_x_axis):
                # Read two bytes and mask to prevent sign extension
                b1 = file_stream.read_byte() & 0xff
                b2 = file_stream.read_byte() & 0xff
                
                # Combine bytes based on endianness
                if not is_correct_endian:
                    # Swap bytes: big-endian file on little-endian machine (or vice versa)
                    raw_value = (b2 << 8) | b1
                else:
                    # Native order
                    raw_value = (b1 << 8) | b2
                
                # Cast to signed short
                # Note: In Python, integers are arbitrary precision. 
                # To simulate 16-bit signed short behavior for min/max logic:
                if raw_value > 32767:
                    raw_value -= 65536
                
                value = raw_value
                
                volume[k][j][i] = value
                
                # Update min/max
                if value < min_val:
                    min_val = value
                if value > max_val:
                    max_val = value
                    
    return volume, min_val, max_val
```

## Subtopics

### Byte Order Detection and Swapping
Handles endianness differences when reading binary data files across different processor architectures. It is a procedure to handle binary data files created on machines with different byte orders (big-endian vs. little-endian) by detecting the file's byte order and the machine's byte order, then performing byte-swapping if necessary [S2]. A key pitfall is that DSN6 file readers assume big-endian byte order; if read on little-endian machines (Intel/AMD x86), byte-swapping is required to get correct values [S2]. DSN6 file readers assume the file was written in big-endian byte order. If read on a little-endian machine without correction, values will be incorrect, though the reader handles reading on both types [S2]. In comparison, DSN6 readers assume big-endian, whereas other file readers automatically detect and swap byte order as needed [S2].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the total number of voxels ($CT\_z\_axis \times CT\_y\_axis \times CT\_x\_axis$), as each voxel requires reading two bytes and combining them.
- **Memory Complexity:** $O(N)$ to store the `short` array.
- **Strengths:** Correctly interprets 16-bit data across different hardware architectures; prevents subtle data corruption from sign-extension errors.
- **Limitations:** Requires prior knowledge of the file's endianness relative to the host system (via `isCorrectEndian` flag) unless automatic detection is implemented in a wrapper.
- **When to use:** When loading raw binary volume data from external sources or cross-platform transfers where endianness is not guaranteed to match the host.
- **When not to use:** When the data is already in native endianness and performance is critical, as byte swapping adds overhead.

## Background
The method originates from implementations for loading raw volume data in Java-based volume rendering applications [S1, unofficial]. The concept of handling endianness in binary volume data is well-established in scientific visualization, particularly in tools like UCSF ChimeraX which handle DSN6 files and other formats that may originate from different architectures [S2]. The specific pitfall regarding sign-extension of bytes is a common error in low-level binary parsing in languages like Java where `byte` is signed [S1, unofficial].

## Sources
[S1] joshcodd/volume-rendering, https://github.com/joshcodd/volume-rendering, fetched 2026-10-26 (unofficial)
[S2] Command: volume, https://www.rbvi.ucsf.edu/chimerax/docs/user/commands/volume.html, fetched 2026-10-27
