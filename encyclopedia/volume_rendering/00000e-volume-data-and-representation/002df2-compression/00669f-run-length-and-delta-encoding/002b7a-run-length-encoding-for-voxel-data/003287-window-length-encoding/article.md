# Window-Length Encoding
Also called: RL encoding, Run-Length Encoding

## Overview
Window-Length Encoding is a data compression technique that scans a source and encodes repeated sequences of a single symbol (runs) by replacing them with a symbol-count pair [S1, unofficial]. In volume rendering, it is applied to intermediate images to reduce network transfer time [S2]. It sets itself apart from standard RLE by using a fixed-length counter based on an expected run length $m$, which simplifies implementation at the cost of efficiency if $m$ is not a power of 2 [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A binary source sequence (assumed to be 0s and 1s, with probability of 0 > probability of 1) [S1, unofficial].
- Output: A bitstream of encoded run lengths [S1, unofficial].

**Data Structures:**
- Counter $C$: An integer variable of width $h$ bits [S1, unofficial].
- Parameter $m$: The expected run length [S1, unofficial].
- Parameter $h$: The counter width in bits, calculated as $h = \lceil \log_2 m \rceil$ [S1, unofficial].

**Steps:**
1. Select a value $m$ indicating the expected run length [S1, unofficial].
2. Calculate the counter width $h = \lceil \log_2 m \rceil$ [S1, unofficial].
3. Initialize the counter $C$ to zero [S1, unofficial].
4. Scan the source for runs of zeros (since $p > q$, only runs of zeros are counted; a 1 marks the end of a run) [S1, unofficial].
5. For each zero in a run, increment $C$ [S1, unofficial].
6. If $C == m$:
   - Output a `1` bit [S1, unofficial].
   - Reset $C$ to zero [S1, unofficial].
   - Continue counting the current run [S1, unofficial].
7. If the end of the run is detected (a `1` in the source is encountered):
   - Output a `0` bit [S1, unofficial].
   - Output the current value of $C$ using $h$ bits [S1, unofficial].
8. Repeat until the source is exhausted [S1, unofficial].

**Formulas:**
- Counter width: $h = \lceil \log_2 m \rceil$ [S1, unofficial].
- Bits per counter output: $h + 1$ bits (one bit for the end-of-run marker, $h$ bits for the counter value) [S1, unofficial].

**Parameters:**
- $m$: Expected run length. Typical values are not specified, but $m=14$ is used in examples [S1, unofficial].
- Encoding time: ~1 ms [S2].
- Decoding time: ~1 ms [S2].

**Edge Cases and Pitfalls:**
- If $m$ is not a power of 2, the counter can represent $2^h$ values, but only $2^h - m$ are unused, wasting efficiency [S1, unofficial].
- Encoding efficiency depends on image content. Views within the volume produce large images which cannot be window encoded. Images with contiguous colour runs (e.g. distant views) are efficiently run-length encoded [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def window_length_encode(source_bits, m):
    """
    Encodes a binary source using Window-Length Encoding.
    
    Args:
        source_bits: List of integers (0 or 1) representing the binary source.
        m: Expected run length (integer).
        
    Returns:
        List of integers (0 or 1) representing the encoded bitstream.
    """
    h = math.ceil(math.log2(m))
    counter = 0
    encoded_bits = []
    
    for bit in source_bits:
        if bit == 0:
            # Increment counter for runs of zeros
            counter += 1
            if counter == m:
                # Output 1 bit to indicate full counter, reset counter
                encoded_bits.append(1)
                counter = 0
        else:
            # End of a run of zeros detected
            # Output 0 bit followed by the counter value
            encoded_bits.append(0)
            # Append counter value in h bits
            for i in range(h - 1, -1, -1):
                encoded_bits.append((counter >> i) & 1)
            counter = 0
            
    return encoded_bits
```

## Subtopics

### Window-Length Encoded Image Compositing
This variant involves compositing input intermediate images, including implicit handling of the window-length encoded images [S2]. Compositing time is approximately 10 ms per input intermediate image [S2].

## Operating characteristics
- **Time Complexity:** Encoding and decoding each take ~1 ms [S2]. Compositing takes ~10 ms per input intermediate image [S2].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** On a 1000 Mb/s Ethernet network, window-length encoding reduces image propagation time to <1 ms, compared to ~10 ms without compression for 500x500 pixel images [S2].
- **Scaling:** Efficiency depends on image content. Distant views with contiguous colour runs are efficiently encoded, while views within the volume are not [S2].
- **Strengths:** Significant reduction in network transfer time for suitable images [S2].
- **Limitations:** Inefficient if $m$ is not a power of 2 due to unused counter values [S1, unofficial]. Not effective for images without contiguous runs [S2].
- **When to use:** When transmitting intermediate images over a network where bandwidth is a constraint and images have long runs of identical values [S2].
- **When not to use:** When images lack contiguous runs (e.g., views within the volume) [S2].

## Background
Window-Length Encoding is a variant of Run-Length Encoding (RLE) [S1, unofficial]. It was applied in the context of a distributed-data implementation of the perspective shear-warp volume rendering algorithm for visualization of large astronomical cubes [S2]. The standard RLE method uses a fixed-length counter, whereas Golomb coding uses a variable-length counter to improve efficiency [S1, unofficial].

## Sources
[S1] Length-Limited Data Transformation and Compression, https://digital.library.unt.edu/ark:/67531/metadc874691/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
[S2] "A Distributed-Data Implementation of the Perspective Shear-Warp Volume Rendering Algorithm for Visualisation of Large Astronomical Cubes", http://web.eng.ucsd.edu/~jschulze/projects/Beeson2003.pdf, fetched 2026-10-29
