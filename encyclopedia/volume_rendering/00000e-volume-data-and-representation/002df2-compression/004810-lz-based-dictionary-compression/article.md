# LZ-based Dictionary Compression

Also called: Lempel-Ziv family, LZ compression, Dictionary-based compression

## Overview
LZ-based dictionary compression is a lossless data compression technique that replaces repeated occurrences of data with references to a single copy of that data existing earlier in the input stream [S1]. It solves the problem of redundant data storage in volumetric datasets by exploiting spatial repetition [S1]. What sets it apart from general-purpose compressors like gzip or bzip2 is its optimization for fast decompression rather than maximum compression ratio [S1].

## Shared principles
All compression methods in this family share the following mechanism: [Compression](4-compression.md).

## Subtopics

### LZO1X-999 Compression
- **Definition:** A variant of the Lempel-Ziv-Oberhumer (LZO) real-time compression library that uses the same Lempel-Ziv dictionary coder as gzip but is optimized for fast decompression rather than maximum compression ratio [S1].
- **Parameters:** The algorithm variant is `LZO1X-999`. It is the slowest compressor in the LZO family (up to 8 times slower than the default variant) but does not influence decompression speed [S1].
- **Comparison:** Unlike gzip or bzip2, which are optimized for maximum compression ratio but are often too slow to decompress faster than reading uncompressed data, LZO is optimized for decompression speed [S1]. In time-varying volume data pipelines, the time for reading and uncompressing a data block must be less than the time needed to read the uncompressed block [S1].

### [LZV Algorithm](8069-lzv-algorithm.md)
- **Definition:** A dictionary-based lossless compression method that identifies repetitive sequences in volumetric data by mapping them to single indices within a dynamically constructed dictionary. It operates directly on the raw data stream without explicit vector quantization or orthogonal transforms, relying on greedy longest-match dictionary expansion.
- **Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lzv_compress(input_symbols):
    # Initialize dictionary with all single-character strings
    # Assuming symbols are integers 0-255 for byte data
    dictionary = {}
    next_index = 0
    
    # Standard initialization for byte-oriented LZW/LZV
    for i in range(256):
        dictionary[chr(i)] = next_index
        next_index += 1
        
    # Limit dictionary size to 4096 as per source
    MAX_DICT_SIZE = 4096
    
    output_indices = []
    current_string = ""
    
    for symbol in input_symbols:
        symbol_str = chr(symbol)
        candidate = current_string + symbol_str
        
        if candidate in dictionary:
            current_string = candidate
        else:
            # Emit index for the longest matched string
            output_indices.append(dictionary[current_string])
            
            # Add new string to dictionary if space allows
            if next_index < MAX_DICT_SIZE:
                dictionary[candidate] = next_index
                next_index += 1
            
            # Reset current string to the new symbol
            current_string = symbol_str
            
    # Emit the final string
    if current_string:
        output_indices.append(dictionary[current_string])
        
    return output_indices
```
- **Parameters:** Dictionary size is capped at 4096 entries.
- **Edge Cases:** If the dictionary fills, no new sequences are added, potentially reducing compression efficiency for highly repetitive but complex patterns. As a lossless method, it preserves exact data values, making it suitable for medical MRI data where lossy compression might cause diagnostic errors.

### [3D-VOI-LtoA](12632-3d-voi-ltoa.md)
- **Definition:** A lossless compression method for 3D medical images that separates the clinically significant foreground (Volume of Interest, VOI) from the zero-intensity background using a Selective Bounding Volume (SBV) technique, then encodes the VOI using a hybrid Lempel-Ziv-Welch (LZW) followed by Arithmetic Coding pipeline.
- **Comparison:** It distinguishes itself from sibling methods like [LZV Algorithm](8069-lzv-algorithm.md) by explicitly removing large, uninformative background regions prior to dictionary-based encoding, thereby reducing the data volume processed by the entropy coder and significantly increasing compression ratios for sparse medical volumes.

### Deflate Compression
- **Definition:** Compresses input bytes using Java's `Deflater` with a specified compression level [S2, unofficial].

## Background
LZ-based compression originates from the work of Abraham Lempel and Jacob Ziv in the early 1970s (LZ77 and LZ78) [G]. In volume rendering, these algorithms are adopted to handle large volumetric datasets by utilizing spatial coherence, though temporal coherence in time-varying data requires additional prediction models [S1]. The LZO library was introduced by Markus Oberhumer to address the bottleneck where decompression speed lagged behind data transfer speed [S1]. While standard LZ variants like Deflate are common in general-purpose compression, specialized variants like LZO1X-999 are chosen in volume rendering pipelines to ensure that the decompression overhead does not exceed the I/O latency of reading raw data [S1].

## Sources
[S1] A GPU-Supported Lossless Compression Scheme for Rendering Time-Varying Volume Data, https://doi.org/10.2312/vg/vg10/109-116, fetched 2026-10-26
[S2] https://github.com/rubensrudio/SeismicDataCompressor, https://github.com/rubensrudio/SeismicDataCompressor, fetched 2026-10-28 (unofficial)
