# Splat Data Encoding & Disabled Splat Detection

Also called: Disabled Splat Detection, Splat Data Encoding

## Overview
This method defines the binary encoding scheme for individual Gaussian splats within a contiguous buffer and provides the logic to identify "disabled" splats that should be excluded from rendering and spatial queries. It solves the problem of efficiently managing inactive or removed splats without reallocating memory, by reserving a specific bit pattern that acts as a sentinel value. It differs from standard active-splat processing by requiring explicit bit-level inspection of the data layout to filter out invalid entries before sorting or rasterization.

## How it works
**Inputs and Outputs**
- Input: A contiguous block of splat data stored in two `Uint32Array` buffers (or equivalent flat integer arrays).
- Output: A boolean flag indicating if a splat is disabled, and modified coordinate values (NaN) for sorting purposes if disabled.

**Data Structures**
Each splat occupies 8 words (32 bytes) of storage, split across two arrays:
- `splatA`: Contains words 0–3 (16 bytes).
- `splatB`: Contains words 4–7 (16 bytes).
For the $i$-th splat, the indices in `splatB` are $i \times 4 + 1$ and $i \times 4 + 2$ (assuming 0-based indexing for the 32-bit words within the second array's segment for that splat, or more precisely, the second and third 32-bit words of the second half of the splat's 64-bit total footprint).

**Disabled Splat Detection Logic**
A splat is considered disabled if specific bit patterns are present in the second and third words of the second array (`splatB`) associated with that splat. Let $i$ be the splat index. The detection checks:
1. Read the word at `splatB[i4 + 1]`.
2. Read the word at `splatB[i4 + 2]`.
3. The splat is disabled if:
   - The upper 16 bits of `splatB[i4 + 1]` equal `0xfc00`.
   - `splatB[i4 + 2]` equals `0xfc00fc00`.

**Sorting and Spatial Queries**
When extracting centers for spatial sorting or querying:
- If a splat is detected as disabled, its $x, y, z$ coordinates must be set to `Number.NaN`.
- This ensures that the splat is excluded from spatial data structures (e.g., BVH, grids) and does not contribute to rendering passes.

**Pitfalls**
- Disabled splats must be explicitly detected via their specific bit pattern. If not checked, they will be processed as valid splats, leading to rendering artifacts or incorrect sorting behavior [S1, unofficial].
- The bit pattern is rigid; any deviation means the splat is active.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_splat_disabled(splat_b, splat_index):
    """
    Check if a splat is disabled based on the specific bit pattern in splatB.
    
    Args:
        splat_b (list or array): The second array containing words 4-7 of each splat.
        splat_index (int): The index of the splat to check.
        
    Returns:
        bool: True if the splat is disabled, False otherwise.
    """
    # Calculate the offset in splat_b for the current splat's second and third words
    # Each splat has 4 words in splat_b.
    offset = splat_index * 4
    
    word_1 = splat_b[offset + 1]
    word_2 = splat_b[offset + 2]
    
    # Check upper 16 bits of word_1
    upper_16_bits = (word_1 >> 16) & 0xFFFF
    
    # Check if upper 16 bits are 0xfc00 and word_2 is 0xfc00fc00
    if upper_16_bits == 0xfc00 and word_2 == 0xfc00fc00:
        return True
    
    return False

def get_splat_center_for_sorting(splat_a, splat_b, splat_index):
    """
    Extract center coordinates for sorting. If disabled, return NaN.
    
    Args:
        splat_a (list or array): The first array containing words 0-3 of each splat.
        splat_b (list or array): The second array containing words 4-7 of each splat.
        splat_index (int): The index of the splat.
        
    Returns:
        tuple: (x, y, z) coordinates, or (NaN, NaN, NaN) if disabled.
    """
    if is_splat_disabled(splat_b, splat_index):
        return (float('nan'), float('nan'), float('nan'))
    
    # Note: The exact mapping of words to x, y, z is not specified in the source.
    # This is a placeholder for the actual extraction logic which depends on the 
    # specific encoding of the remaining words (e.g., rotation, scale, opacity).
    # [G] Assuming standard 3D Gaussian encoding where x, y, z are stored in 
    # specific words of splat_a or splat_b. Without source details, we cannot 
    # provide the exact extraction formula.
    
    # Example placeholder (must be replaced with actual encoding logic):
    # x = splat_a[splat_index * 4 + 0]
    # y = splat_a[splat_index * 4 + 1]
    # z = splat_a[splat_index * 4 + 2]
    
    return (0.0, 0.0, 0.0) # Placeholder
```

## Subtopics

## Operating characteristics
- **Time Complexity**: Detection is $O(1)$ per splat. Sorting becomes $O(N \log N)$ where $N$ is the number of active splats, as disabled splats are filtered out or marked NaN.
- **Memory Complexity**: $O(1)$ extra space per splat for the detection logic; no additional memory allocation for disabled splats.
- **Strengths**: Efficiently handles dynamic scene changes (adding/removing splats) without memory reallocation.
- **Limitations**: Requires strict adherence to the bit pattern; any corruption of the data may lead to incorrect detection.

## Background
This method is part of the implementation details of **Gaussian-Splat-Lite**, an unofficial optimization of the 3D Gaussian Splatting technique. It is designed for efficient GPU-based rendering and data management.

## Sources
[S1] WilliamLiu-1997/Gaussian-Splat-Lite, https://github.com/WilliamLiu-1997/Gaussian-Splat-Lite, fetched 2026-10-27 (unofficial)
