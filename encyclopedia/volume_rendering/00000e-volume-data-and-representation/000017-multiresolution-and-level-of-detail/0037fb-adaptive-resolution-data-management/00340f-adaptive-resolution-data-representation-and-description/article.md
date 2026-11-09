# Adaptive-Resolution Data Representation and Description

Also called: Adaptive Resolution Data Representation, Block-based Adaptive Resolution

## Overview
Adaptive-Resolution Data Representation is a scheme where a volume dataset is partitioned into spatial blocks, each assigned one of three discrete resolution levels (High, Medium, or Low) to balance fidelity and storage [S1, unofficial]. It differs from flat blocking by allowing variable block sizes at lower resolutions to maintain constant data size per level, achieving different compression ratios [S2, unofficial].

## How it works
**Inputs:** A volume dataset and a data description file defining block geometry and resolution.
**Outputs:** A structured representation of blocks with associated resolution codes.

**Data Structures:**
The core structure is defined by a data description file. Each block entry contains:
1.  `Start-X`, `Start-Y`, `Start-Z`: Starting coordinates of the block.
2.  `End-X`, `End-Y`, `End-Z`: Ending coordinates of the block.
3.  `Resolution`: A string code indicating the resolution level.

**Resolution Codes:**
*   High: `/`
*   Medium: `/s`
*   Low: `/s/s`

**Virtual vs. Actual Dimensions:**
*   **Virtual Dimensions:** The dimensions of the block at the highest resolution (High). For compositing, subimages from lower-resolution blocks are expanded to these virtual dimensions.
*   **Actual Dimensions:** The dimensions of the data points actually stored for that block.
    *   High: Same as virtual.
    *   Medium: Half the virtual dimensions in each axis.
    *   Low: Quarter the virtual dimensions in each axis (or half of Medium).

**Example Calculation (from [S1, unofficial]):**
If a block has virtual dimensions $196 \times 56 \times 56$:
*   High Actual: $196 \times 56 \times 56$
*   Medium Actual: $98 \times 28 \times 28$
*   Low Actual: $49 \times 14 \times 14$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_data_description_file(filepath):
    """
    Parses the data description file to extract block definitions.
    Returns a list of block dictionaries.
    """
    blocks = []
    with open(filepath, 'r') as f:
        lines = f.readlines()
    
    # First line: Total number of blocks
    try:
        num_blocks = int(lines[0].strip())
    except ValueError:
        raise ValueError("First line must be an integer representing total blocks")
    
    # Remaining lines: Block definitions
    # Format: Start-X, Start-Y, Start-Z, End-X, End-Y, End-Z, Resolution
    # Note: The source implies a specific format. We assume comma or whitespace separation.
    # The source shows:
    # Block-1-Start-X
    # Block-1-Start-Y
    # ...
    # Resolution
    # This suggests each field might be on a new line or space-separated.
    # Based on "Figure 18 shows the format", and typical CSV-like structures in such theses,
    # we assume space-separated or similar. However, the text lists them vertically.
    # Let's assume a standard space-separated line for each block for robustness,
    # or handle the vertical format if strictly required.
    # Given the ambiguity of "Figure 18" without the image, we assume a line-based record.
    
    for i in range(1, num_blocks + 1):
        if i - 1 < len(lines):
            line = lines[i].strip()
            # Split by whitespace
            parts = line.split()
            if len(parts) < 7:
                raise ValueError(f"Line {i+1} does not contain enough fields")
            
            start_x = int(parts[0])
            start_y = int(parts[1])
            start_z = int(parts[2])
            end_x = int(parts[3])
            end_y = int(parts[4])
            end_z = int(parts[5])
            resolution_code = parts[6]
            
            # Validate resolution code
            if resolution_code not in ['/', '/s', '/s/s']:
                raise ValueError(f"Invalid resolution code: {resolution_code}")
            
            # Calculate dimensions
            width = end_x - start_x + 1
            height = end_y - start_y + 1
            depth = end_z - start_z + 1
            
            # Virtual dimensions are the dimensions at High resolution
            virtual_width = width
            virtual_height = height
            virtual_depth = depth
            
            # Actual dimensions depend on resolution
            if resolution_code == '/':
                actual_width = virtual_width
                actual_height = virtual_height
                actual_depth = virtual_depth
            elif resolution_code == '/s':
                actual_width = virtual_width // 2
                actual_height = virtual_height // 2
                actual_depth = virtual_depth // 2
            elif resolution_code == '/s/s':
                actual_width = virtual_width // 4
                actual_height = virtual_height // 4
                actual_depth = virtual_depth // 4
            
            blocks.append({
                'id': i,
                'start': (start_x, start_y, start_z),
                'end': (end_x, end_y, end_z),
                'virtual_dims': (virtual_width, virtual_height, virtual_depth),
                'actual_dims': (actual_width, actual_height, actual_depth),
                'resolution_code': resolution_code
            })
    
    return blocks
```

**Edge Cases:**
*   **Boundary Alignment:** Blocks must align with the resolution grid. If a block's virtual dimensions are not divisible by 2 or 4, actual dimensions may require rounding or padding. The source example uses dimensions divisible by 4 ($196, 56$), so this is not explicitly addressed [S1, unofficial].
*   **Empty Blocks:** The source does not specify how empty blocks are handled in this specific representation. [G] Typically, empty blocks might be flagged or stored as null references.

## Subtopics

### Hierarchical Blocking
A multiresolution volume representation using hierarchical blocking where lower resolution blocks cover increasing spatial size, maintaining constant data size per level [S2, unofficial]. It achieves a data reduction of 2.2:1, whereas flat blocking achieves 3.8:1 for the same levels [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Parsing the description file is $O(N)$ where $N$ is the number of blocks. Rendering time depends on the number of high-resolution blocks accessed.
*   **Memory Complexity:** Depends on the distribution of resolution levels. High-resolution blocks consume more memory per unit volume than low-resolution blocks.
*   **Performance:** The source does not provide specific performance metrics for the flat/block-based adaptive resolution method [S1, unofficial].
*   **Strengths:** Allows regions of interest to be rendered at higher fidelity while less important regions use lower resolution [S1, unofficial].
*   **Limitations:** Requires a data description file and complex logic to handle different resolutions during compositing [S1, unofficial].

## Background
This method was developed for parallel visualization of spatial adaptive-resolution data, specifically using MHD pressure data [S1, unofficial]. The goal was to create a dummy AR Data Representation for temporary use in a project focused on parallel visualization techniques [S1, unofficial].

## Sources
[S1] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
[S2] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
