# Arithmetic Encoding

Also called: Arithmetic Coding

## Overview
Arithmetic encoding is a lossless entropy coding technique that maps a sequence of input symbols to a single fractional number within the interval $[0, 1)$ [S1, unofficial]. It solves the inefficiency of fixed-length or symbol-by-symbol variable-length codes by encoding the entire message as one number, achieving a bit rate approaching the Shannon entropy limit [S1, unofficial]. It differs from [Run-Length Encoding for Voxel Data](5051-run-length-encoding-for-voxel-data.md) by not relying on explicit run-length prefixes, and from [rANS (Asymmetric Numeral Systems)](5011-rans-asymmetric-numeral-systems.md) by using floating-point interval arithmetic rather than integer state machines, though rANS is often preferred for hardware parallelism [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A sequence of symbols $S = s_1, s_2, \dots, s_N$ drawn from an alphabet $\Sigma$, and a probability model $P(s_i | \text{context})$ for each symbol.
- Output: A binary bitstream representing the final sub-interval.

**Data Structures:**
- `current_interval`: A pair of floating-point numbers $[L, H)$ representing the current range, initialized to $[0.0, 1.0)$.
- `output_buffer`: A list of bits to be flushed.
- `pending_zeros`: An integer counter for deferred zero bits.

**Steps:**
1. **Initialization**: Set $L = 0.0$, $H = 1.0$.
2. **Encoding Loop**: For each symbol $s_i$ in the input sequence:
   a. Determine the cumulative probability range $[P_{cum}(s_i), P_{cum}(s_i) + P(s_i))$ based on the current context model. Let this range be $[p_{low}, p_{high})$ relative to $[0, 1)$.
   b. Calculate the interval width: $W = H - L$.
   c. Update the interval:
      $$L_{new} = L + W \times p_{low}$$
      $$H_{new} = L + W \times p_{high}$$
   d. Set $L = L_{new}$, $H = H_{new}$.
   e. **Renormalization**: While the interval is entirely within $[0, 0.5)$ or $[0.5, 1.0)$:
      - If $H \le 0.5$: Output bit `0`. Update $L = 2L$, $H = 2H$. Increment `pending_zeros` by 1.
      - If $L \ge 0.5$: Output bit `1`. Update $L = 2(L - 0.5)$, $H = 2(H - 0.5)$. Increment `pending_zeros` by 1.
3. **Finalization**: After processing all symbols, output the smallest number of bits that uniquely identifies the final interval $[L, H)$, followed by a bit opposite to the last output bit to disambiguate [G]. Flush any remaining `pending_zeros` by outputting the opposite bit and then zeros.

**Parameters:**
- Precision: Typically 32-bit or 64-bit floating point for interval calculations [G].
- Context Model: Probabilities are estimated dynamically based on previous symbols [S1, unofficial].

**Edge Cases:**
- **Precision Loss**: If the interval becomes too small relative to floating-point precision, renormalization must occur frequently. If precision is insufficient, the decoder may fail to distinguish the final interval.
- **Boundary Conditions**: The interval is half-open $[L, H)$. Care must be taken when $H$ approaches 1.0.

## Subtopics

### [MLZC 3-D Encoding/2-D Decoding](2501-mlzc-3-d-encoding-2-d-decoding.md)
Uses arithmetic encoding with context-adaptive models for wavelet subbands, enabling 2-D slice access.

### [rANS (Asymmetric Numeral Systems)](5011-rans-asymmetric-numeral-systems.md)
Integer-based alternative to arithmetic coding for high-throughput GPU decoding.

### [Bitstream Overloading via Flush Bytes and Markers](5237-bitstream-overloading-via-flush-bytes-and-markers.md)
Adds markers to arithmetic streams for random access.

### [Entropy Encoding](5360-entropy-encoding.md)
General category of which arithmetic encoding is a specific, optimal instance.

## Operating characteristics
- **Time Complexity**: Encoding and decoding are linear in the number of symbols $O(N)$, but each step involves floating-point multiplication and interval updates, which are more expensive than integer operations [S1, unofficial].
- **Memory Complexity**: Low, requiring only the current interval state and probability tables [S1, unofficial].
- **Performance**: Achieves near-optimal compression ratios, often outperforming Huffman coding [S1, unofficial]. However, it is sensitive to probability model accuracy; poor models lead to significant compression loss [S1, unofficial].
- **Strengths**: Optimal compression for known distributions; handles arbitrary alphabets.
- **Limitations**: Sequential dependency prevents parallel encoding/decoding of large blocks; floating-point precision issues can cause decoding errors if not handled carefully [S1, unofficial].

## Background
Arithmetic coding was originally developed by Shannon, Fano, and Elias in the 1940s and 1950s but became practical with the work of Rissanen in the 1970s [S1, unofficial]. It is widely used in modern standards such as JPEG 2000, H.264/AVC, and HEVC [S1, unofficial]. In volume rendering, it is used to compress anchor attributes and other metadata in neural representations [S1, unofficial].

## Sources
[S1] ECoNGS: Efficient Compressive Neural Gaussian Splats for Volume Visualization, https://doi.org/10.48550/arxiv.2607.18466, fetched 2026-10-26 (unofficial)
