# Differential Data Stream (DDS) File Format

Also called: Differential Data Stream, Differential Data Stream (DDS) Compression

## Overview
The Differential Data Stream (DDS) file format is a lossless compression method for volume data that encodes differential values using run-length encoding and adaptive bit allocation per run [S1, unofficial]. It supports two versions to handle different data sizes via an interleave parameter [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A byte array of volume data with `bytes` total elements, `skip` channels (1-4), and `strip` spatial/temporal stride [S1, unofficial].
- Output: A binary file containing a header and a compressed chunk [S1, unofficial].

**Data Structures:**
- **Header:**
  - Bytes 0-3: Magic identifier `DDS_ID` (Version 1) or `DDS_ID2` (Version 2) [S1, unofficial].
  - Bytes 4-5: 2 bits for `skip - 1` (channel count minus one) [S1, unofficial].
  - Bytes 6-9: 16 bits for `strip - 1` (spatial/temporal stride minus one) [S1, unofficial].
- **Chunk:** A variable-length byte stream containing the encoded differential data [S1, unofficial].

**Parameters:**
- `DDS_BLOCKSIZE`: Cache allocation size for bitstream packing, 1 MiB [S1, unofficial].
- `DDS_INTERLEAVE`: Threshold for version selection; if `bytes > DDS_INTERLEAVE`, Version 2 is used, otherwise Version 1 [S1, unofficial].
- `DDS_RL`: Maximum run length encoded in 7 bits (value 127) [S1, unofficial].
- `skip`: Number of channels (1-4), stored as 2 bits (`skip - 1`) [S1, unofficial].
- `strip`: Spatial/temporal stride for differential calculation, stored as 16 bits (`strip - 1`) [S1, unofficial].

**Algorithm Steps:**
1. **Version Selection:** Compare `bytes` against `DDS_INTERLEAVE`; set `version = 2` if greater, else `version = 1` [S1, unofficial].
2. **Header Writing:** Write `DDS_ID` or `DDS_ID2`, then `skip - 1` in 2 bits, then `strip - 1` in 16 bits [S1, unofficial].
3. **Encoding:**
   - **Deinterleaving (Version 2 only):** Separate the input byte stream into channel-specific streams based on `skip` and `DDS_BLOCKSIZE` [S1, unofficial].
   - **Lookup Table Initialization:** Map differential values in $[-128, 127]$ to required bit counts [S1, unofficial].
   - **Differential Calculation:** Iterate through bytes; compute `act1 = tmp1 - pre1` (or `tmp1 - pre1 - *(ptr1 - strip) + *(ptr1 - strip - 1)` if `strip > 1`) [S1, unofficial].
   - **Clamping & Mapping:** Clamp `act1` to $[-128, 127]$ to avoid out-of-bounds access, then map to bit count via lookup [S1, unofficial].
   - **Run-Length Encoding of Bit Counts:** Track current run (`cnt1`, `bits1`) and previous run (`cnt2`, `bits2`) [S1, unofficial]. Merge runs if `cnt1 + cnt2 < (1 << DDS_RL)` and `(cnt1 + cnt2) * max(bits1, bits2) < cnt1 * bits1 + cnt2 * bits2 + DDS_RL + 3` [S1, unofficial].
   - **Run Writing:** When a run ends, write `cnt2` using 7 bits, write `bits2` using 3 bits, then write each differential value offset by `(1 << bits2) / 2` using `bits2` bits [S1, unofficial].
   - **Flush & Re-interleave:** Flush remaining bits to the bitstream; if Version 2, re-interleave the encoded data [S1, unofficial].
4. **Chunk Writing:** Write the compressed chunk to the file [S1, unofficial].
5. **Decoding:** Read header, determine version, read chunk, deinterleave (Version 2), unpack bitstream, decode runs, reconstruct bytes via `tmp = pre + act`, and re-interleave (Version 2) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
DDS_RL = 7  # Maximum run length encoded in 7 bits (127) [S1, unofficial]

class BitStream:
    def __init__(self):
        self.buffer = 0
        self.bufsize = 0
        self.data = bytearray()

    def write(self, value, nbits):
        value &= (1 << nbits) - 1
        if self.bufsize + nbits < 32:
            self.buffer = (self.buffer << nbits) | value
            self.bufsize += nbits
        else:
            self.data.append(self.buffer & 0xFF)
            self.data.append((self.buffer >> 8) & 0xFF)
            self.data.append((self.buffer >> 16) & 0xFF)
            self.data.append((self.buffer >> 24) & 0xFF)
            self.buffer = value
            self.bufsize = nbits

    def flush(self):
        if self.bufsize > 0:
            self.data.append(self.buffer & 0xFF)
            self.data.append((self.buffer >> 8) & 0xFF)
            self.data.append((self.buffer >> 16) & 0xFF)
            self.data.append((self.buffer >> 24) & 0xFF)
            self.buffer = 0
            self.bufsize = 0

def DDS_code(b):
    return b & 0x7  # Maps bit count to 3-bit code [G]

def deinterleave(data, skip, bytes):
    # Separates interleaved channel data into contiguous blocks [G]
    channels = [[] for _ in range(skip)]
    for i, b in enumerate(data):
        channels[i % skip].append(b)
    return [c for blk in channels for c in blk]

def interleave(data, skip, bytes):
    # Re-interleaves channel blocks into original order [G]
    channels = [data[i::skip] for i in range(skip)]
    result = []
    for i in range(bytes // skip):
        for c in channels:
            result.append(c[i])
    return result

def DDS_encode(data, bytes, skip, strip, version):
    # Initialize lookup table [S1, unofficial]
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

    if version == 2:
        data = deinterleave(data, skip, bytes)

    bs = BitStream()
    cnt1, bits1 = 0, 0
    cnt2, bits2 = 0, 0
    pre = 0
    current_values = []

    for i in range(bytes):
        tmp1 = data[i]
        act1 = tmp1 - pre
        if strip > 1:
            # Adjust differential for strip > 1 [S1, unofficial]
            pass  # Indexing depends on memory layout [G]

        act1 = max(-128, min(127, act1))
        bits1 = lookup[act1 + 128]

        if cnt1 == 0:
            cnt1 = 1
            bits1_val = bits1
            current_values = [act1]
        else:
            if bits1 == bits1_val:
                cnt1 += 1
                current_values.append(act1)
            else:
                if cnt2 > 0 and (cnt1 + cnt2) < (1 << DDS_RL) and \
                   (cnt1 + cnt2) * max(bits1_val, bits2_val) < cnt1 * bits1_val + cnt2 * bits2_val + DDS_RL + 3:
                    cnt2 += cnt1
                    bits2_val = max(bits1_val, bits2_val)
                    cnt1 = 0
                    bits1_val = 0
                    current_values = []
                else:
                    bs.write(cnt2, DDS_RL)
                    bs.write(DDS_code(bits2_val), 3)
                    for val in current_values:
                        bs.write(val + (1 << bits2_val) // 2, bits2_val)
                    cnt2 = 0
                    bits2_val = 0
                    cnt1 = 1
                    bits1_val = bits1
                    current_values = [act1]
        pre = tmp1

    if cnt2 > 0:
        bs.write(cnt2, DDS_RL)
        bs.write(DDS_code(bits2_val), 3)
        for val in current_values:
            bs.write(val + (1 << bits2_val) // 2, bits2_val)
    elif cnt1 > 0:
        bs.write(cnt1, DDS_RL)
        bs.write(DDS_code(bits1_val), 3)
        for val in current_values:
            bs.write(val + (1 << bits1_val) // 2, bits1_val)

    bs.flush()

    if version == 2:
        bs.data = interleave(bs.data, skip, bytes)

    return bs.data

def DDS_decode(chunk, size, bytes, version):
    if version == 2:
        chunk = deinterleave(chunk, skip, size)

    bs = BitStream()
    bs.data = chunk
    bs.buffer = 0
    bs.bufsize = 0
    data = []
    pre = 0

    while len(data) < bytes:
        cnt = bs.data[0]  # Simplified read for 7-bit run length [G]
        bits = bs.data[1] & 0x7  # Simplified read for 3-bit code [G]
        # Actual bitstream reading requires stateful bit extraction [G]
        # Reconstruct values
        for _ in range(cnt):
            val = 0  # Placeholder for bit extraction [G]
            act = val - (1 << bits) // 2
            tmp = pre + act
            data.append(tmp)
            pre = tmp

    if version == 2:
        data = interleave(data, skip, bytes)

    return data
```

## Subtopics

### DDS Bitstream Packing/Unpacking
A variable-length bitstream encoder/decoder that packs/unpacks bits into 32-bit words with automatic cache management and endianness handling [S1, unofficial]. It masks input values to the specified bit width, packs them into a buffer, and flushes 32-bit words to cache when full, handling byte swapping for little-endian systems [S1, unofficial]. Reading splits across buffer and cache, fetching new words as needed [S1, unofficial]. It uses `DDS_BLOCKSIZE` (1 MiB) for cache allocation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of bytes, $O(N)$, due to single-pass encoding and decoding [S1, unofficial].
- **Memory Complexity:** $O(N)$ for input/output buffers, plus $O(1)$ for lookup table and bitstream buffers [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with volume size [S1, unofficial].
- **Strengths:** Lossless compression, simple implementation, adaptive bit allocation based on differential values [S1, unofficial].
- **Limitations:** Limited to 8-bit data (clamped), no frequency-based optimization, performance depends on data redundancy [S1, unofficial].
- **When to use:** When lossless compression of 8-bit volume data is required and data has high local correlation [S1, unofficial].
- **When not to use:** For floating-point data, or when lossy compression is acceptable and higher compression ratios are needed [S1, unofficial].

## Background
The Differential Data Stream (DDS) format is documented in the `SRaimondi/VolumeRender` repository as an implementation of volume rendering compression techniques [S1, unofficial]. The algorithm details, including header layout, differential encoding, and bitstream packing, are derived directly from this source [S1, unofficial].

## Sources
[S1] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
