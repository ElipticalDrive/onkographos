# Differential Data Stream (DDS) File Format

Also called: Differential Data Stream, Differential Data Stream (DDS) Compression

## Overview
The Differential Data Stream (DDS) file format is a lossless compression method for volume data that encodes differential values using run-length encoding and adaptive bit allocation per run [S1, unofficial]. It is designed to handle multi-channel volume data by interleaving channels and applying spatial/temporal differencing, making it distinct from transform-based methods like [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) or [Laplacian Pyramid](2250-laplacian-pyramid.md) by operating directly on raw pixel/voxel values without frequency decomposition. It builds on [Value compression](6-value-compression.md) principles by focusing on the statistical redundancy of differences between adjacent voxels.

## How it works
**Inputs and Outputs:**
- Input: A 3D volume array $V$ with dimensions $W \times H \times D$, containing $C$ channels (1-4). Data type is assumed to be 8-bit signed or unsigned integers (clamped to 8-bit range during processing).
- Output: A binary file containing a header and a compressed chunk.

**Data Structures:**
- **Header:**
  - Bytes 0-3: Magic number `DDS_ID` (Version 1) or `DDS_ID2` (Version 2).
  - Bytes 4-5: 2 bits for `skip - 1` (channel count - 1), 14 unused bits (or part of the 16-bit strip field depending on alignment, see below). *Note: The source specifies "2 bits for skip - 1, 16 bits for strip - 1". This implies a 18-bit field, likely padded to 24 or 32 bits in the byte stream.*
  - Bytes 6-7 (or 6-9): 16 bits for `strip - 1` (spatial/temporal stride - 1).
- **Chunk:** A variable-length byte stream containing the encoded differential data.

**Parameters:**
- `DDS_BLOCKSIZE`: Constant determining block processing limits (value not explicitly given, used in loop condition `while (blkcnt == DDS_BLOCKSIZE)`).
- `DDS_INTERLEAVE`: Threshold for version selection. If total bytes > `DDS_INTERLEAVE`, Version 2 is used; otherwise Version 1.
- `DDS_RL`: Maximum run length for run-length encoding, encoded in 7 bits (max value 127).
- `skip`: Number of channels (1-4). Stored as 2 bits (`skip - 1`).
- `strip`: Spatial/temporal stride for differential calculation. Stored as 16 bits (`strip - 1`).

**Algorithm Steps:**

1. **Version Selection:**
   - Calculate total size of input data in bytes.
   - If size > `DDS_INTERLEAVE`, set `version = 2`. Else, `version = 1`.

2. **Header Writing:**
   - Write `DDS_ID` if `version == 1`, else write `DDS_ID2`.
   - Write `skip - 1` in 2 bits.
   - Write `strip - 1` in 16 bits.

3. **Encoding (`DDS_encode`):**
   - **Interleaving (Version 2 only):** If `version == 2`, deinterleave the input byte stream based on channel skip and block size using `DDS_deinterleave`. This separates channels for independent processing.
   - **Initialization:** Initialize a lookup table `lookup[256]` mapping differential values in $[-128, 127]$ to required bit counts.
     ```python
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
     ```
   - **Differential Calculation:** Iterate through the data bytes. For each byte `tmp1` at position `ptr1`:
     - Calculate previous byte `pre1` (from the same channel/stream).
     - Compute differential `act1 = tmp1 - pre1`.
     - If `strip > 1`, adjust differential: `act1 = tmp1 - pre1 - *(ptr1 - strip) + *(ptr1 - strip - 1)`. *Note: The exact indexing depends on the memory layout and whether `strip` refers to spatial neighbors in a specific dimension.*
     - Clamp `act1` to $[-128, 127]$ to avoid out-of-bounds access in the lookup table.
     - Map `act1` to bit count `bits1 = lookup[act1 + 128]`.
   - **Run-Length Encoding of Bit Counts:**
     - Track runs of identical bit counts. Maintain two runs: current (`cnt1`, `bits1`) and previous (`cnt2`, `bits2`).
     - Merge runs if: `cnt1 + cnt2 < (1 << DDS_RL)` AND `(cnt1 + cnt2) * max(bits1, bits2) < cnt1 * bits1 + cnt2 * bits2 + DDS_RL + 3`.
     - When a run ends (or merge condition fails):
       - Write run length `cnt2` using `DDS_writebits(cnt2, DDS_RL)`.
       - Write bit count `bits2` using `DDS_writebits(DDS_code(bits2), 3)`. *Note: `DDS_code` likely maps bit counts to a 3-bit code, possibly using Huffman or fixed mapping for small integers.*
       - For each value in the run, write the actual differential value `act2` using `DDS_writebits(act2 + (1 << bits2) // 2, bits2)`. This offsets the signed value to unsigned for writing.
   - **Flush:** Flush any remaining bits in the bitstream buffer.
   - **Re-interleaving (Version 2 only):** If `version == 2`, re-interleave the original data (or the encoded stream? The source says "re-interleave the original data", which is ambiguous. Given it's a compression format, it likely means the encoded data is re-interleaved if the encoding was done on separated channels. However, standard practice is to encode then interleave the *encoded* blocks or encode interleaved data. The source says "Deinterleave... to compress... re-interleave the original data". This suggests the *input* was deinterleaved for processing, and the *output* might need re-interleaving if the format expects channel-interleaved compressed blocks. Given the ambiguity, we assume the encoded chunk is written as is, or re-interleaved if the format requires it. The source explicitly says "re-interleave the original data", which might be a misstatement for "re-interleave the encoded data" or "the data was deinterleaved for processing, but the output is a single chunk". We will assume the chunk contains the encoded data for all channels, potentially interleaved if version 2).

4. **Chunk Writing:**
   - Write the compressed chunk to the file.

5. **Decoding (`DDS_decode`):**
   - Read header and determine version.
   - Read chunk size and data.
   - **Deinterleaving (Version 2 only):** If `version == 2`, deinterleave the input byte stream based on channel skip and block size using `DDS_deinterleave`.
   - **Bitstream Unpacking:** Use `DDS_bitstream_unpack` to read bits from the 32-bit word buffer.
   - **Run-Length Decoding:**
     - Read run length `cnt` using `DDS_readbits(DDS_RL)`.
     - Read bit count code `code` using `DDS_readbits(3)`.
     - Decode `bits` from `code` (inverse of `DDS_code`).
     - For each value in the run:
       - Read `val` using `DDS_readbits(bits)`.
       - Convert to signed differential: `act = val - (1 << bits) // 2`.
       - Reconstruct original byte: `tmp = pre + act`.
       - Update `pre = tmp`.
   - **Re-interleaving (Version 2 only):** If `version == 2`, re-interleave the decoded data to restore the original channel order.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def DDS_encode(data, bytes, skip, strip, version):
    # Initialize lookup table
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

    # Deinterleave if version 2
    if version == 2:
        data = DDS_deinterleave(data, skip, bytes)

    # Initialize bitstream buffer
    buffer = 0
    bufsize = 0
    bitstream = []

    # Process data
    pre = 0
    cnt1, bits1 = 0, 0
    cnt2, bits2 = 0, 0

    for i in range(bytes):
        tmp1 = data[i]
        # Calculate differential
        act1 = tmp1 - pre
        if strip > 1:
            # Adjust for strip
            # Note: Exact indexing for strip depends on data layout
            # Assuming 1D stream for simplicity, strip refers to previous element in a specific stride
            # This part is ambiguous in sources, assuming simple previous element for now
            # If strip > 1, it might refer to a 3D neighbor. 
            # For implementation, assume 1D stream and strip is ignored or handled by data ordering
            pass 
        
        # Clamp
        act1 = max(-128, min(127, act1))
        
        # Map to bits
        bits1 = lookup[act1 + 128]
        
        # Run-length encoding logic
        if cnt1 == 0:
            cnt1 = 1
            bits1_val = bits1
        else:
            if bits1 == bits1_val:
                cnt1 += 1
            else:
                # End of run
                # Merge logic
                if cnt2 > 0 and cnt1 + cnt2 < (1 << DDS_RL) and \
                   (cnt1 + cnt2) * max(bits1_val, bits2_val) < cnt1 * bits1_val + cnt2 * bits2_val + DDS_RL + 3:
                    cnt2 += cnt1
                    bits2_val = max(bits1_val, bits2_val) # Or keep bits2_val? Source says max(bits1, bits2)
                    cnt1 = 0
                    bits1_val = 0
                else:
                    # Write run 2
                    DDS_writebits(bitstream, cnt2, DDS_RL)
                    DDS_writebits(bitstream, DDS_code(bits2_val), 3)
                    for _ in range(cnt2):
                        # Need to store actual values. This requires storing values in a buffer.
                        # Simplified: assume values are stored in a separate buffer or re-calculated
                        # This is a major gap in the source description.
                        pass 
                    cnt2 = 0
                    bits2_val = 0
                    cnt1 = 1
                    bits1_val = bits1
        
        pre = tmp1

    # Flush remaining
    if cnt2 > 0:
        DDS_writebits(bitstream, cnt2, DDS_RL)
        DDS_writebits(bitstream, DDS_code(bits2_val), 3)
        # Write values...

    if version == 2:
        bitstream = DDS_interleave(bitstream, skip, bytes)

    return bitstream

def DDS_decode(chunk, size, bytes, version):
    # Deinterleave if version 2
    if version == 2:
        chunk = DDS_deinterleave(chunk, skip, size)

    # Initialize bitstream buffer
    buffer = 0
    bufsize = 0
    data = []

    pre = 0

    while len(data) < bytes:
        # Read run length
        cnt = DDS_readbits(chunk, DDS_RL)
        
        # Read bit count code
        code = DDS_readbits(chunk, 3)
        bits = DDS_decode_code(code)
        
        # Read values
        for _ in range(cnt):
            val = DDS_readbits(chunk, bits)
            act = val - (1 << bits) // 2
            tmp = pre + act
            data.append(tmp)
            pre = tmp

    if version == 2:
        data = DDS_interleave(data, skip, bytes)

    return data
```

## Subtopics

### DDS Bitstream Packing/Unpacking
A variable-length bitstream encoder/decoder that packs/unpacks bits into 32-bit words with automatic cache management and endianness handling [S1, unofficial]. It masks input values to the specified bit width, packs them into a buffer, and flushes 32-bit words to cache when full, handling byte swapping for little-endian systems. Reading splits across buffer and cache, fetching new words as needed. It uses `DDS_BLOCKSIZE` (1 MiB) for cache allocation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of bytes, $O(N)$, due to single-pass encoding and decoding.
- **Memory Complexity:** $O(N)$ for input/output buffers, plus $O(1)$ for lookup table and bitstream buffers.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with volume size.
- **Strengths:** Lossless compression, simple implementation, adaptive bit allocation based on differential values.
- **Limitations:** Limited to 8-bit data (clamped), no frequency-based optimization, performance depends on data redundancy.
- **When to use:** When lossless compression of 8-bit volume data is required and data has high local correlation.
- **When not to use:** For floating-point data, or when lossy compression is acceptable and higher compression ratios are needed.

## Background
The Differential Data Stream (DDS) format is described in the context of the `SRaimondi/VolumeRender` repository, which appears to be an unofficial implementation or study of volume rendering techniques [S1, unofficial]. The specific algorithm details are derived from this source, which provides C-like pseudocode and implementation notes. The method is not associated with a specific peer-reviewed paper in the provided sources.

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
