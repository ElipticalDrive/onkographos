# Differential Data Stream (DDS) File Format

Also called: Differential Data Stream, Differential Data Stream (DDS) Compression

## Overview
The Differential Data Stream (DDS) is a lossless compression algorithm and file format designed for volume data, specifically optimizing for spatially correlated scalar fields. It solves the problem of efficiently storing large 3D datasets by encoding differential values (differences between adjacent voxels) using run-length encoding and adaptive bit allocation. Unlike general-purpose codecs, DDS is tailored for volume data structures, supporting multi-channel data through interleaving and configurable spatial strides. It distinguishes itself from siblings like [Predictive and Differential Coding](4393-predictive-and-differential-coding.md) by its specific implementation of variable-length bit packing based on the magnitude of differential values, and from [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) by operating directly in the spatial domain rather than a transform domain.

## How it works
**Inputs and Outputs:**
- Input: A volume dataset represented as a stream of bytes (voxel values), with parameters for channel count (`skip`) and spatial stride (`strip`).
- Output: A binary file containing a header and a compressed chunk.

**Data Structures:**
- **Header:**
  - Bytes 0-3: Magic number. `DDS_ID` for Version 1, `DDS_ID2` for Version 2.
  - Bytes 4-5: 2 bits for `skip - 1` (channel count 1-4), 16 bits for `strip - 1` (spatial stride).
  - Remaining bytes: Reserved or padding as needed by the specific implementation.
- **Chunk:**
  - 4 bytes: Size of the compressed data in bytes.
  - Variable length: The compressed bitstream.

**Constants:**
- `DDS_BLOCKSIZE`: Block processing limit (implementation dependent, often related to cache lines or block size).
- `DDS_INTERLEAVE`: Threshold for byte count. If `bytes > DDS_INTERLEAVE`, Version 2 is used; otherwise Version 1.
- `DDS_RL`: Maximum run length encoded in 7 bits (value 127).
- `DDS_shiftl(x, n)`: Left shift operation `x << n`.

**Algorithm Steps:**

1. **Initialization:**
   - Determine version: If input byte size `> DDS_INTERLEAVE`, set `version = 2`, else `version = 1`.
   - Initialize lookup table `lookup[256]` mapping differential values in range `[-128, 127]` to required bit counts.
     - For `i` from -128 to 127:
       - If `i <= 0`: Find smallest `bits` such that `(1 << bits) / 2 < -i`.
       - If `i > 0`: Find smallest `bits` such that `(1 << bits) / 2 <= i`.
       - `lookup[i + 128] = bits`.

2. **Encoding (DDS_encode):**
   - Write Header:
     - Write `DDS_ID` if `version == 1`, else `DDS_ID2`.
     - Write 2 bits: `skip - 1`.
     - Write 16 bits: `strip - 1`.
   - Process Data Blocks:
     - Iterate through the input data in blocks of size `DDS_BLOCKSIZE`.
     - For each block, compute differential values `act`.
       - Let `pre1` be the previous voxel value.
       - Let `tmp1` be the current voxel value.
       - If `strip == 1`: `act = tmp1 - pre1`.
       - If `strip > 1`: `act = tmp1 - pre1 - *(ptr1 - strip) + *(ptr1 - strip - 1)`. (Note: This formula implies a 2D or higher spatial relationship; for 1D, it simplifies. The source specifies this general form).
       - Clamp `act` to `[-128, 127]`.
       - Map `act` to bit count `bits` using `lookup[act + 128]`.
     - Run-Length Encoding of Bit Counts:
       - Track current run of identical bit counts: `cnt1` (count), `bits1` (bit count).
       - Track previous run: `cnt2`, `bits2`.
       - Merge runs if: `cnt1 + cnt2 < (1 << DDS_RL)` AND `(cnt1 + cnt2) * max(bits1, bits2) < cnt1 * bits1 + cnt2 * bits2 + DDS_RL + 3`.
       - When a run ends (or merge condition fails):
         - Write run length `cnt2` using `DDS_writebits(cnt2, DDS_RL)`.
         - Write bit count `bits2` using `DDS_writebits(DDS_code(bits2), 3)`. (Note: `DDS_code` maps bit count to a 3-bit code, likely a Huffman or fixed code for 1-7 bits).
         - For each value in the run, write the differential value `act2` using `DDS_writebits(act2 + (1 << bits2) / 2, bits2)`. This centers the unsigned bit representation around zero.
   - Flush remaining bits.
   - If `version == 2`, deinterleave the output bitstream based on `skip` and `strip` using `DDS_deinterleave`.
   - Write Chunk Size and Chunk Data to file.

3. **Decoding (DDS_decode):**
   - Read Header:
     - Read 4 bytes. Compare with `DDS_ID`. If match, `version = 1`. Else compare with `DDS_ID2`. If match, `version = 2`. Else error.
     - Read 2 bits: `skip = value + 1`.
     - Read 16 bits: `strip = value + 1`.
   - Read Chunk:
     - Read 4 bytes for chunk size.
     - Read chunk data.
   - If `version == 2`, re-interleave the bitstream using `DDS_interleave` based on `skip` and `strip`.
   - Initialize lookup table (same as encoding).
   - Iterate through chunk:
     - Read run length `cnt` using `DDS_readbits(DDS_RL)`.
     - Read bit count code `code` using `DDS_readbits(3)`.
     - Map `code` back to `bits` (inverse of `DDS_code`).
     - For `i` from 0 to `cnt - 1`:
       - Read `bits` bits to get unsigned value `val`.
       - Compute differential `act = val - (1 << bits) / 2`.
       - Reconstruct voxel value: `voxel = pre1 + act` (adjusting for `strip` logic if necessary, though decoding typically reverses the encoding difference).
       - Update `pre1`.
   - Return reconstructed data.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dds_encode(data, bytes_count, skip, strip, version):
    # 1. Initialize Lookup Table
    lookup = [0] * 256
    for i in range(-128, 128):
        if i <= 0:
            bits = 0
            while (1 << bits) // 2 < -i:
                bits += 1
        else:
            bits = 0
            while (1 << bits) // 2 <= i:
                bits += 1
        lookup[i + 128] = bits

    # 2. Write Header
    if version == 1:
        write_bytes(b'DDS_ID')
    else:
        write_bytes(b'DDS_ID2')
    
    write_bits(skip - 1, 2)
    write_bits(strip - 1, 16)

    # 3. Encode Data
    # Note: The source describes a complex run-length encoding of bit counts.
    # We simulate the core logic.
    
    prev_voxel = 0 # Assuming initial condition or first voxel stored separately
    run_cnt = 0
    run_bits = 0
    prev_run_cnt = 0
    prev_run_bits = 0
    
    bitstream = []
    
    for idx in range(bytes_count):
        current_voxel = data[idx]
        
        # Compute differential
        # Simplified for 1D/strip=1 for clarity; source formula is more complex for strip > 1
        if strip == 1:
            act = current_voxel - prev_voxel
        else:
            # Source formula: tmp1 - pre1 - *(ptr1 - strip) + *(ptr1 - strip - 1)
            # This requires access to previous voxels at specific offsets.
            # Assuming data is linearized and ptr1 is index.
            ptr1 = idx
            pre1 = prev_voxel
            # Note: Accessing ptr1 - strip requires bounds checking and handling of previous blocks
            # This is a simplified placeholder for the complex spatial dependency
            act = current_voxel - pre1 
        
        # Clamp
        act = max(-128, min(127, act))
        
        # Map to bits
        bits = lookup[act + 128]
        
        # Run-length encoding logic
        if bits == run_bits:
            run_cnt += 1
        else:
            # End of previous run
            if prev_run_cnt > 0:
                # Write previous run
                write_bits(prev_run_cnt, 7) # DDS_RL = 7 bits
                write_bits(dds_code(prev_run_bits), 3)
                for _ in range(prev_run_cnt):
                    # Find act for previous run? 
                    # In a real implementation, we'd store acts in a buffer for the run.
                    # Here we assume we can retrieve them or store them.
                    pass 
            
            # Merge logic check (simplified)
            # If merge condition met, we would combine prev_run and current run
            # For this pseudocode, we just start a new run
            prev_run_cnt = run_cnt
            prev_run_bits = run_bits
            run_cnt = 1
            run_bits = bits
        
        prev_voxel = current_voxel

    # Flush last run
    if prev_run_cnt > 0:
        write_bits(prev_run_cnt, 7)
        write_bits(dds_code(prev_run_bits), 3)
        # Write values for the last run...

    # 4. Interleave if Version 2
    if version == 2:
        bitstream = dds_deinterleave(bitstream, skip, strip)
        
    return bitstream

def dds_decode(chunk, bytes_count, skip, strip, version):
    # 1. Read Header
    magic = read_bytes(4)
    if magic == b'DDS_ID':
        version = 1
    elif magic == b'DDS_ID2':
        version = 2
    else:
        raise Error("Invalid DDS file")
        
    skip = read_bits(2) + 1
    strip = read_bits(16) + 1
    
    # 2. Read Chunk
    chunk_size = read_bytes(4)
    chunk_data = read_bytes(chunk_size)
    
    # 3. Deinterleave if Version 2
    if version == 2:
        chunk_data = dds_interleave(chunk_data, skip, strip)
        
    # 4. Decode Data
    lookup = [0] * 256
    for i in range(-128, 128):
        if i <= 0:
            bits = 0
            while (1 << bits) // 2 < -i:
                bits += 1
        else:
            bits = 0
            while (1 << bits) // 2 <= i:
                bits += 1
        lookup[i + 128] = bits
        
    data = []
    prev_voxel = 0
    
    while chunk_data not empty:
        run_cnt = read_bits(7)
        code = read_bits(3)
        bits = dds_decode_code(code)
        
        for _ in range(run_cnt):
            val = read_bits(bits)
            act = val - (1 << bits) // 2
            current_voxel = prev_voxel + act
            data.append(current_voxel)
            prev_voxel = current_voxel
            
    return data
```

## Subtopics

### DDS Bitstream Packing/Unpacking
A variable-length bitstream encoder/decoder that packs/unpacks bits into 32-bit words with automatic cache management and endianness handling. It masks input values to the required bit width, packs them into a buffer, and flushes 32-bit words to cache when full, handling byte swapping for little-endian systems. Reading splits across buffer and cache if the requested bits span a word boundary. It uses `DDS_BLOCKSIZE` (1 MiB) for cache allocation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of voxels, $O(N)$, due to single-pass encoding and decoding. The run-length encoding of bit counts adds a small constant overhead for merging runs.
- **Memory Complexity:** $O(1)$ auxiliary memory for the encoding/decoding state, plus $O(N)$ for the input/output buffers. The cache size is fixed at 1 MiB (`DDS_BLOCKSIZE`) [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales well with data size due to linear complexity. The interleaving in Version 2 may add overhead for large datasets but supports multi-channel data.
- **Strengths:** Lossless compression tailored for volume data; efficient for data with small differential values (common in smooth volumes); supports multi-channel data.
- **Limitations:** Lossless only; performance depends on the spatial correlation of the data; complex spatial dependencies for `strip > 1` may be difficult to implement efficiently.
- **When to use:** When lossless compression of volume data is required, and the data exhibits strong spatial correlation with small differences between adjacent voxels.
- **When not to use:** When lossy compression is acceptable and higher compression ratios are needed; when data is random or has large discontinuities.

## Background
The Differential Data Stream (DDS) format is described in the unofficial repository `SRaimondi/VolumeRender` [S1, unofficial]. It appears to be a specialized codec for volume rendering applications, likely derived from or inspired by techniques used in medical imaging or scientific visualization where lossless compression of 3D scalar fields is beneficial. The specific algorithm details, including the run-length encoding of bit counts and the differential calculation, are unique to this implementation.

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
