# Auto-trigger RLE with Variable Length Codes

Also called: RLE with auto-trigger, auto-trigger(2) RLE

## Overview
Auto-trigger RLE with Variable Length Codes is a lossless compression method for voxel data that eliminates redundancy by encoding runs of identical voxels using variable-length codes (VLCs) triggered only after a specific repetition threshold [S1, unofficial]. It differs from standard "all-counts" RLE by avoiding the overhead of encoding short, non-repeating sequences as zero-length runs, and from fixed-length RLE by adapting to the skewed statistical distribution of voxel types and run lengths [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A compressed bitstream buffer containing the encoded world block [S1, unofficial].
*   **Output:** A 3D voxel array `out_voxels[128]` representing a $16 \times 16 \times 128$ block, accessed linearly as `linear_voxels` [S1, unofficial].
*   **Data Structures:**
    *   `bit_offset`: Current read position in the input bitstream [S1, unofficial].
    *   `byte_offset`: Current write position in the output linear buffer [S1, unofficial].
    *   `auto_run`: Integer counter tracking consecutive identical voxels [S1, unofficial].
    *   `last`: The voxel type of the previously decoded voxel [S1, unofficial].

**VLC Definitions:**
The method uses two prefix codes. A prefix code is uniquely decodable, meaning no code is a prefix of another, allowing decoding by traversing a binary tree until a leaf is reached [S1, unofficial].

1.  **Voxel Type Code (16 types):**
    *   `00`: Air
    *   `01`: Dirt
    *   `1xxxx`: 16 other stuff types (5 bits total: 1 prefix bit + 4 data bits) [S1, unofficial].

2.  **Run Length Code:**
    *   `0xxxx`: Run length $0-15$ (5 bits total: 1 prefix bit + 4 data bits) [S1, unofficial].
    *   `10xxxxxx`: Run length $16-79$ (8 bits total: 2 prefix bits + 6 data bits) [S1, unofficial].
    *   `11xxxxxxxxxxxxxxx`: Run length $80-32847$ (17 bits total: 2 prefix bits + 15 data bits) [S1, unofficial].

**Parameters:**
*   **Trigger Value:** 2. The decoder enters run mode only after observing 2 consecutive identical voxels [S1, unofficial].

**Algorithm Steps:**
1.  Initialize `bit_offset = 0`, `byte_offset = 0`, `auto_run = 0`, `last = -1` (invalid type) [S1, unofficial].
2.  Loop while `byte_offset < 16 \times 16 \times 128` (32,768 voxels) [S1, unofficial]:
    a.  Decode the current voxel type `this_tile` from the bitstream using the Voxel Type Code, advancing `bit_offset` [S1, unofficial].
    b.  Write `this_tile` to `linear_voxels[byte_offset]` [S1, unofficial].
    c.  Increment `byte_offset` [S1, unofficial].
    d.  If `this_tile == last`:
        i.  Increment `auto_run` [S1, unofficial].
        ii. If `auto_run == 2`:
            1.  Decode the run length `run_length` from the bitstream using the Run Length Code, advancing `bit_offset` [S1, unofficial].
            2.  Fill `run_length` copies of `this_tile` into `linear_voxels` starting at `byte_offset` [S1, unofficial].
            3.  Increment `byte_offset` by `run_length` [S1, unofficial].
            4.  Reset `auto_run = 0` [S1, unofficial].
        iii. Else (`auto_run < 2`):
            1.  Do nothing (continue to next voxel) [S1, unofficial].
    e.  Else (`this_tile != last`):
        i.  Reset `auto_run = 0` [S1, unofficial].
    f.  Update `last = this_tile` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decode_auto_trigger_rle(buffer: bytes) -> list:
    """
    Decodes a voxel world block using auto-trigger RLE with VLCs.
    
    Args:
        buffer: The compressed bitstream.
        
    Returns:
        A list of 32,768 voxel types (uint8).
    """
    BLOCK_SIZE = 16 * 16 * 128  # 32,768
    TRIGGER = 2
    
    bit_offset = 0
    byte_offset = 0
    auto_run = 0
    last = -1
    out_voxels = [0] * BLOCK_SIZE
    
    def read_bits(n: int) -> int:
        """Read n bits from the bitstream, advancing bit_offset."""
        nonlocal bit_offset
        val = 0
        for _ in range(n):
            byte_idx = bit_offset // 8
            bit_idx = 7 - (bit_offset % 8)
            val = (val << 1) | ((buffer[byte_idx] >> bit_idx) & 1)
            bit_offset += 1
        return val
    
    while byte_offset < BLOCK_SIZE:
        # Decode voxel type
        b0 = read_bits(1)
        b1 = read_bits(1)
        if b0 == 0 and b1 == 0:
            this_tile = 0  # Air
        elif b0 == 0 and b1 == 1:
            this_tile = 1  # Dirt
        else:
            this_tile = 2 + read_bits(4)
            
        out_voxels[byte_offset] = this_tile
        byte_offset += 1
        
        if this_tile == last:
            auto_run += 1
            if auto_run == TRIGGER:
                # Decode run length
                b0 = read_bits(1)
                if b0 == 0:
                    run_length = read_bits(4)
                else:
                    b1 = read_bits(1)
                    if b1 == 0:
                        run_length = 16 + read_bits(6)
                    else:
                        run_length = 80 + read_bits(15)
                        
                # Fill run
                out_voxels[byte_offset : byte_offset + run_length] = [this_tile] * run_length
                byte_offset += run_length
                auto_run = 0
        else:
            auto_run = 0
            
        last = this_tile
        
    return out_voxels
```

**Edge Cases and Pitfalls:**
*   **Misfiring on Alternating Sequences:** If the data contains frequent short alternating patterns (e.g., `aabbcc`), the trigger of 2 causes the encoder to emit `aa(0)bb(0)cc(0)`, which is wasteful because it encodes zero-length runs. If this occurs frequently, the trigger value should be increased (e.g., to 3) [S1, unofficial].
*   **Run Length Exceeding Block Size:** The maximum encoded run length is 32,847, which exceeds the block size of 32,768, but this is safe because a single run cannot physically exceed the block boundaries [S1, unofficial].

## Operating characteristics
*   **Compression Ratio:** Achieves an average compression ratio of 123:1 on the tested dataset (250 world blocks) [S1, unofficial]. Performance varies from 48:1 to 964:1 depending on the data distribution [S1, unofficial].
*   **Comparison to Alternatives:**
    *   **Fixed-length RLE:** Significantly outperforms fixed-length codes by adapting to the skewed distribution of voxel types and run lengths [S1, unofficial].
    *   **bzip2:** Generally ~30% less efficient than bzip2 (which achieves 157:1 on the same data), but offers simpler, asymmetric decoding [S1, unofficial].
*   **Complexity:**
    *   **Decoding:** Simple and fast, suitable for real-time game engines where content is read many times [S1, unofficial].
    *   **Encoding:** More complex than decoding, but acceptable as content is created once [S1, unofficial].
*   **Memory:** Low memory overhead for the decoder, requiring only state variables and the output buffer [S1, unofficial].
*   **When to Use:** Ideal for voxel-based worlds with high redundancy (e.g., large homogeneous regions like air or dirt) and where asymmetric compression (easy decoding) is prioritized over maximum compression ratio [S1, unofficial].
*   **When Not to Use:** When data lacks long runs or has frequent short alternating patterns that trigger misfiring, or when maximum compression ratio is more critical than decoding speed [S1, unofficial].

## Background
*   **Origin:** Proposed by Arthur Ouellet and implemented by the author of the source article [S1, unofficial].
*   **Context:** Developed for compressing voxel world blocks in a game engine, specifically targeting $16 \times 16 \times 128$ blocks composed of 16 different stuff types (notably air and dirt) [S1, unofficial].
*   **History:** The method evolved from an initial idea to use RLE on $16 \times 16$ tiles with fixed codes, to a linear series of voxels with auto-trigger RLE and VLCs to better exploit the skewed distribution of data [S1, unofficial].
*   **Usage:** Designed for game engines where content is compressed once and decompressed many times, favoring simple, fast decoding [S1, unofficial].

## Sources
[S1] Interesting article about RLE compression with voxels, https://hbfs.wordpress.com/2011/03/22/compressing-voxel-worlds/, fetched 2026-10-27 (unofficial)
