# 2-D-PROG

Also called: Bi-dimensional MLZC

## Overview
2-D-PROG is the bi-dimensional variant of the MLZC algorithm, used as a benchmark for 2-D wavelet-based coding systems and for independent encoding of subbands within the 3-D/2-D MLZC framework [S1]. It distinguishes itself from general 2-D standards like JPEG2000 by its specific integration into the MLZC context, utilizing spatial support and interband conditioning to optimize lossless compression rates for medical data [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2-D image block (subband) derived from the volume decomposition [S1].
*   **Output:** An encoded bitstream representing the quantized wavelet coefficients [S1].

**Data Structures:**
The method relies on context modeling defined by a spatial support window and optional interband conditioning [S1].
*   **Spatial Support:** A neighborhood of coefficients used to predict the current coefficient [G]. The sources specify that supports like (060) and (070) are optimal [S1].
*   **Interband Conditioning:** Optional [S1].

**Parameters:**
*   **Spatial Support Size:** The (060) and (070) configurations yield the highest compression ratios [S1].
*   **Interband Conditioning:** Reduces the minimum lossless rate by approximately 1.2% [S1].

**Steps:**
1.  **Decomposition:** The input 2-D image is decomposed using a Discrete Wavelet Transform (DWT) implemented via lifting steps, as used in JPEG2000 [S1].
2.  **Context Modeling:** For each coefficient, a context is formed based on the chosen spatial support (e.g., neighbors at specific offsets) and optionally interband neighbors [S1].
3.  **Symbol Generation:** The current coefficient's quantized value is mapped to a symbol based on the context statistics [G]. The sparseness of the conditioning space limits the effective support size; larger supports are beneficial for larger datasets but degrade if the space becomes too sparse for adequate statistics [S1].
4.  **Encoding:** The generated symbols are encoded into a bitstream [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_2d_prog(image, spatial_support, use_interband):
    # 1. Wavelet Decomposition via lifting steps [S1]
    subbands = _dwt_2d_lifting(image)
    bitstream = []
    
    for subband in subbands:
        for y in range(subband.height):
            for x in range(subband.width):
                # 2. Context Modeling
                context = _get_spatial_neighbors(subband, x, y, spatial_support)
                if use_interband:
                    context += _get_interband_neighbors(subbands, x, y)
                
                # 3. Symbol Generation
                symbol = _map_to_symbol(subband[y, x], context)
                
                # 4. Encoding
                bitstream.append(_arithmetic_encode(symbol, context))
                
    return bitstream

def _dwt_2d_lifting(data):
    # Standard 2D lifting scheme DWT: apply 1D lifting along rows, then columns
    row_transformed = [_dwt_1d_lifting(row) for row in data]
    col_transformed = [_dwt_1d_lifting(col) for col in zip(*row_transformed)]
    return col_transformed

def _dwt_1d_lifting(signal):
    # Placeholder for standard lifting step implementation [G]
    return signal

def _get_spatial_neighbors(subband, x, y, support):
    # Returns coefficients at offsets defined by spatial_support
    neighbors = []
    for dy, dx in support:
        ny, nx = y + dy, x + dx
        if 0 <= ny < subband.height and 0 <= nx < subband.width:
            neighbors.append(subband[ny, nx])
    return neighbors

def _get_interband_neighbors(subbands, x, y):
    # Returns coefficients from adjacent subbands at the same spatial coordinates
    neighbors = []
    for subband in subbands:
        if 0 <= y < subband.height and 0 <= x < subband.width:
            neighbors.append(subband[y, x])
    return neighbors

def _map_to_symbol(value, context):
    # Maps quantized coefficient to a symbol using context statistics [G]
    return value

def _arithmetic_encode(symbol, context):
    # Updates arithmetic coder state with symbol and context [G]
    return symbol
```

**Edge Cases and Pitfalls:**
*   **Sparseness Limit:** If the spatial support is too wide for the dataset size, the conditioning space becomes too sparse, preventing adequate representation of symbol statistics and degrading performance [S1].
*   **Dataset Size Dependency:** Larger volumes benefit from wider spatial supports [S1].

## Subtopics

### [LPL-PROG Mode](5897-lpl-prog-mode.md)
LPL-PROG Mode is a bitstream organization scheme for the MLZC 3-D encoding/2-D decoding system that enables quality scalability for both the entire volume and individual 2-D slices. It is derived from G-PROG mode by inserting markers after every quantization layer of every subband image. It is chosen when 2-D local-scale conditioning is used and slice-level progressiveness is required, at the cost of increased bitstream overhead due to markers.

## Operating characteristics
*   **Compression Efficiency:** The (060) and (070) spatial supports lead to the highest compression ratios [S1].
*   **Impact of Interband Conditioning:** Using interband conditioning reduces the minimum lossless rate by about 1.2% [S1].
*   **Scalability:** Not given in the sources.
*   **Performance:** It serves as a benchmark for 2-D systems [S1].

## Background
2-D-PROG is the bi-dimensional version of the MLZC algorithm, designed to handle 2-D slices independently or as part of the 3-D/2-D MLZC approach [S1]. It is compared against JPEG2000, a standard designed to overcome JPEG limitations with improved compression, lossy-to-lossless performance, and scalability [S1]. The algorithm uses DWT implemented by lifting steps, similar to MLZC and EZW-3-D [S1].

## Sources
[S1] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
