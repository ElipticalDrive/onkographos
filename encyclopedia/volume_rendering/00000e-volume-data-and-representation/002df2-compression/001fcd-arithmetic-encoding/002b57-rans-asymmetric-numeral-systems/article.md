# rANS (Asymmetric Numeral Systems)

Also called: asymmetric numeral systems entropy coding, rANS Compression Framework, rANS Entropy Coding, range Asymmetric Numeral Systems, ryg_rANS adaptation

## Overview
rANS is a fast, lossless entropy coding scheme used to compress sequences of 4-bit operation symbols into a byte stream [S1][S2, unofficial]. It replaces a range of packed 4-bit elements with an rANS-encoded bytestream [S2, unofficial]. The method enables efficient and effective parallel decompression for GPU-based rendering [S1]. It builds on [Arithmetic Encoding](2127-arithmetic-encoding.md) by optimizing for integer arithmetic and parallel execution [S1][S2, unofficial].

## How it works
### Inputs and Outputs
- **Input**: A packed buffer of 4-bit symbols (half-bytes) starting at `start4bit` (inclusive) and ending at `end4bit` (exclusive), where the first element must be aligned to a 32-bit memory location [S2, unofficial].
- **Output**: An rANS-encoded bytestream for compression, or a decoded byte array for decompression [S2, unofficial].

### Data Structures
The implementation relies on specific frequency tables and state structures [S2, unofficial]:

1. **SymbolStats**:
    - `freqs[RANS_ALPHABET_SIZE]`: `uint32_t` array of symbol frequencies.
    - `cum_freqs[RANS_ALPHABET_SIZE + 1]`: `uint32_t` array of cumulative frequencies.
2. **RANS Class/Private Members**:
    - `esyms[RANS_ALPHABET_SIZE]`: Encoding symbols (`RansEncSymbol`).
    - `dsyms[RANS_ALPHABET_SIZE]`: Decoding symbols (`RansDecSymbol`).
    - `cum2sym[prob_scale]`: `uint8_t` array mapping cumulative frequency to symbol index.
    - `stats`: `SymbolStats` instance.
    - `has_frequency_tables`: `bool` flag.
3. **RANS_SYMBOL_TABLE (GLSL/Shader Context)**:
    - A constant array of 34 `uvec3` elements [S2, unofficial].
    - Each element represents decoding info: `(start, freq, cum_freq)` [S2, unofficial].
    - Elements 0–15 correspond to the 16 symbols [S2, unofficial].
    - Element 16 is a dummy entry `(0, 0, total_cum_freq)` [S2, unofficial].
    - The array is repeated twice (34 elements total) to support two frequency tables (base and detail) [S2, unofficial].

### Parameters
- `RANS_ALPHABET_SIZE`: 16 (constant size of the symbol alphabet for 4-bit values) [S2, unofficial].
- `prob_bits` / `RANS_PROB_BITS`: 14 (number of probability bits for scaling) [S2, unofficial].
- `prob_scale`: $2^{14} = 16384$ (scaling factor for probabilities) [S2, unofficial].
- `RANS_BYTE_L`: $2^{23} = 8,388,608$ (lower bound of the normalization interval) [S2, unofficial].

### Frequency Tables
Global frequency tables are used for the entire dataset because relative frequencies of operations are very similar across bricks [S1]. Static frequency tables are recomputed by selecting one of 8 predefined frequency arrays based on boolean flags `rle`, `palette_delta`, and `mtf` [S2, unofficial].

The index for the frequency table is calculated as:
$$ \text{index} = (\text{rle} ? 1 : 0) + (\text{palette\_delta} ? 2 : 0) + (\text{mtf} ? 4 : 0) $$

If `USE_DETAIL_FREQ` is defined, the finest Level of Detail (LoD) uses a detail frequency table with an offset of 17, otherwise it uses the base table with offset 0 [S2, unofficial].

### Encoding Process
The encoding process packs 4-bit elements from the input buffer into an rANS bytestream [S2, unofficial]. It operates symmetrically to decoding, updating the state $x$ and renormalizing by writing bytes when $x$ exceeds the upper bound, though the provided sources focus on the decoding implementation [S2, unofficial].

### Decoding Process
The decoding process is iterative and operates on the encoded byte stream [S2, unofficial].

1. **Initialization**:
    - Load the initial rANS state $x$ from the encoding array at the current byte index (aligned to 4 bytes) [S2, unofficial].
    - Advance the byte index by 4 [S2, unofficial].
2. **Symbol Extraction Loop**:
    - **Get Cumulative Value**: Extract the cumulative value from the rANS state using the lower `RANS_PROB_BITS` (14 bits) [S2, unofficial].
    - **Find Symbol**: Determine the symbol index $s$ by finding the first entry in the frequency table where the next entry's cumulative frequency is greater than the current cumulative value, often accelerated using the `cum2sym` lookup table [S2, unofficial].
    - **State Advancement**: Update the rANS state $x$ using the formula:
        $$ x = F(s) \cdot (x \gg \text{RANS\_PROB\_BITS}) + (x \& \text{mask}) - L(s) $$
        Where:
        - $F(s)$ is the frequency of symbol $s$.
        - $L(s)$ is the start value (cumulative frequency of previous symbols) for symbol $s$.
        - $\text{mask} = (1 \ll \text{RANS\_PROB\_BITS}) - 1$ (i.e., $16383$).
        - $\gg$ is bitwise right shift.
        - $\&$ is bitwise AND.
    - **Renormalization**: If the updated state $x < RANS\_BYTE\_L$, read bytes from the encoding array until $x \ge RANS\_BYTE\_L$ [S2, unofficial]. Each byte read shifts $x$ left by 8 bits and adds the byte value [S2, unofficial]. When reading bytes from a `uint32` encoding array, the shift amount depends on the byte index within the `uint32` word: `idx % 4 * 8` [S2, unofficial].
3. **Repeat**: Continue the loop until the required number of output elements is decoded [S2, unofficial].

### Pseudocode (Decoding)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def decode_rans(encoding_array, num_symbols, start_byte_idx, freqs, cum_freqs, cum2sym):
    """
    Decodes num_symbols from encoding_array starting at start_byte_idx.
    encoding_array is a list of uint32 values.
    freqs: list of uint32 frequencies for each symbol.
    cum_freqs: list of uint32 cumulative frequencies (cum_freqs[s] acts as L(s)).
    cum2sym: list of uint8 mapping cumulative value to symbol index.
    """
    RANS_PROB_BITS = 14
    PROB_SCALE = 1 << RANS_PROB_BITS  # 16384
    RANS_BYTE_L = 1 << 23             # 8388608
    MASK = PROB_SCALE - 1             # 16383

    def get_byte(arr, idx):
        word = arr[idx // 4]
        byte_pos = idx % 4
        return (word >> (byte_pos * 8)) & 0xFF

    # 1. Initialization
    x = 0
    for i in range(4):
        b = get_byte(encoding_array, start_byte_idx + i)
        x = (x << 8) | b
    current_byte_idx = start_byte_idx + 4
    decoded_symbols = []

    # 2. Symbol Extraction Loop
    for _ in range(num_symbols):
        cum_val = x & MASK
        s = cum2sym[cum_val]

        # State advancement: x = F(s) * (x >> RANS_PROB_BITS) + (x & mask) - L(s)
        x = freqs[s] * (x >> RANS_PROB_BITS) + (x & MASK) - cum_freqs[s]

        # Renormalization
        while x < RANS_BYTE_L:
            b = get_byte(encoding_array, current_byte_idx)
            current_byte_idx += 1
            x = (x << 8) | b

        decoded_symbols.append(s)

    return decoded_symbols, current_byte_idx
```

### Edge Cases and Pitfalls
- **Alignment**: The first element of the packed 4-bit input must be aligned to a 32-bit memory location [S2, unofficial].
- **Byte Extraction**: When reading from a `uint32` array, the shift amount must account for the byte's position within the word (`idx % 4 * 8`) [S2, unofficial].
- **Frequency Table Selection**: Ensure the correct frequency table (base vs. detail) is selected based on the LoD and flags (`USE_DETAIL_FREQ`) [S2, unofficial].
- **Precision**: The use of 14 probability bits and a 23-bit lower bound requires careful integer arithmetic to avoid overflow or underflow during state updates and renormalization [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of symbols encoded or decoded [S1][S2, unofficial].
- **Memory Complexity**: Low overhead, requiring storage for frequency tables and a 32-bit rANS state per stream or brick [S2, unofficial].
- **Performance**: Achieves high throughput, up to gigabytes per second for both compression and decompression [S1].
- **Compression Ratio**: Strong compression ratios, typically 1% to 3% of the original dataset size for segmented volumes [S1].
- **Parallelism**: Designed for efficient parallel decompression, making it suitable for GPU-based rendering [S1].
- **Limitations**: Relies on global frequency tables, which may require recomputation or adaptation for highly non-stationary data streams [S1][S2, unofficial].

## Background
The method is used to compress sequences of operations for reconstructing segmented volume bricks [S1]. The specific implementation in compressed segmentation volumes is an adaptation of Fabian Giesen's public domain `ryg_rANS` code, optimized for encoding and decoding 4-bit symbols [S2, unofficial]. It enables interactive visualization with on-the-fly decompression and level-of-detail rendering [S1].

## Sources
[S1] Fast Compressed Segmentation Volumes for Scientific Visualization., https://doi.org/10.1109/tvcg.2023.3326573, fetched 2026-10-26
[S2] max-pio/compressed-segmentation-volumes, https://github.com/max-pio/compressed-segmentation-volumes, fetched 2026-10-26 (unofficial)
