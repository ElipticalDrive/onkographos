# LZV Algorithm
Also called: Lembel‐Ziv‐Welch Coding, Lempel–Ziv–Welch (LZW) Coding

## Overview
The LZV algorithm is a dictionary-based lossless compression method that identifies repetitive sequences in volumetric data by mapping them to single indices within a dynamically constructed dictionary [S1]. It distinguishes itself from sibling methods like 3D-VOI-LtoA by operating directly on the raw data stream (or slice streams) without explicit vector quantization or orthogonal transforms, relying instead on greedy longest-match dictionary expansion [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A sequence of symbols (e.g., voxel values, pixel values in a slice) from the volume data.
- Output: A stream of dictionary indices representing the compressed data.

**Data Structures:**
- **Dictionary:** A lookup table mapping indices to character sequences (strings). The dictionary size is limited to a maximum of 4096 entries [S1].
- **Input Buffer:** The remaining unprocessed input symbols.
- **Output Buffer:** The stream of emitted indices.

**Algorithmic Steps:**
1.  **Initialization:** Initialize the dictionary such that all possible input strings of length one are present. Each single symbol is assigned a unique index [S1].
2.  **Longest Match:** For the current input symbol, identify the longest string in the dictionary that matches the beginning of the remaining input [S1].
3.  **Emit and Remove:** Emit the dictionary index corresponding to this longest matching string to the output. Remove the matched string from the input [S1].
4.  **Dictionary Update:** Add the new sequence formed by the matched string plus the next input symbol to the dictionary, if the dictionary is not full [S1].
5.  **Iteration:** Move to the next symbol in the input and repeat steps 2–4 until all input symbols are processed [S1].

**Pseudocode:**
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
        
    # Limit dictionary size to 4096 as per source [S1]
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

**Parameters:**
- **Dictionary Size:** Maximum 4096 character sequences [S1].
- **Compression Type:** Lossless [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Dictionary Overflow:** The dictionary is capped at 4096 entries. If the dictionary fills, no new sequences are added, potentially reducing compression efficiency for highly repetitive but complex patterns [S1].
- **Precision:** As a lossless method, it preserves exact data values, making it suitable for medical MRI data where lossy compression might cause diagnostic errors [S2, unofficial].

## Subtopics

### 3D-VOI-LtoA
A sibling method that uses vector quantization and transform coding. Choose LZV when lossless preservation is critical and data exhibits repetitive string patterns; choose 3D-VOI-LtoA for higher compression ratios on smooth data where some loss is acceptable. [3D-VOI-LtoA](12632-3d-voi-ltoa.md)

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of input symbols, assuming efficient dictionary lookup (e.g., hash table or trie) [S1].
- **Memory Complexity:** Proportional to the dictionary size (4096 entries) plus the input/output buffers [S1].
- **Performance:** Performs better in volume space and on differential data compared to other methods [S2, unofficial]. It offers faster compression and decompression speeds compared to Huffman coding [S2, unofficial].
- **Strengths:** Lossless preservation, fast speed, effective on repetitive data [S1][S2, unofficial].
- **Limitations:** Limited dictionary size (4096) may restrict compression gains on very large, diverse datasets; not suitable for lossy scenarios where higher compression is needed [S2, unofficial].

## Background
The LZV algorithm, also known as Lempel–Ziv–Welch (LZW) coding, is a dictionary-based compression technique originally developed for image and text data [S1]. It has been adapted for volumetric data compression, particularly in medical imaging (e.g., MRI), where lossless compression is often required to prevent diagnostic errors [S2, unofficial]. The method is greedy, modifying the dictionary dynamically with each new string encountered [S1].

## Sources
[S1] On a hybrid lossless compression technique for three‐dimensional medical images, https://doi.org/10.1002/acm2.12960, fetched 2026-11-08
[S2] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
