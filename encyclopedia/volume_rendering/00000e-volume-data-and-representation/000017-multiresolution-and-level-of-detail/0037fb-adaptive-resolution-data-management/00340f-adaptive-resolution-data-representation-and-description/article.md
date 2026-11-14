# Adaptive-Resolution Data Representation and Description

Also called: Adaptive Resolution Data Representation, Block-based Adaptive Resolution

## Overview
This method defines a block-based spatial partitioning scheme where a volume dataset is divided into discrete blocks, each assigned one of three specific resolution levels (High, Medium, or Low) [S1, unofficial]. It solves the problem of efficiently storing and transmitting heterogeneous volume data by allowing regions of interest to retain high fidelity while less important regions use lower resolution [S1, unofficial]. It differs from flat blocking by using a hierarchical structure where lower-resolution blocks cover larger spatial extents, maintaining a constant data size per level rather than reducing sample counts [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A high-resolution volume dataset and a data description file defining the spatial partitioning and resolution assignment for each block [S1, unofficial].
*   **Output:** A set of subimages (blocks) at varying resolutions, ready for compositing [S1, unofficial].

**Data Structures:**
The core data structure is the **Data Description File**, a plain text format [S1, unofficial].
*   **Line 1:** Integer specifying the total number of blocks $N_{blocks}$ [S1, unofficial].
*   **Lines 2 to $N_{blocks}+1$:** Each line defines one block with the following fields:
    1.  `Start-X`: Integer coordinate of the block's starting x-index [S1, unofficial].
    2.  `Start-Y`: Integer coordinate of the block's starting y-index [S1, unofficial].
    3.  `Start-Z`: Integer coordinate of the block's starting z-index [S1, unofficial].
    4.  `End-X`: Integer coordinate of the block's ending x-index [S1, unofficial].
    5.  `End-Y`: Integer coordinate of the block's ending y-index [S1, unofficial].
    6.  `End-Z`: Integer coordinate of the block's ending z-index [S1, unofficial].
    7.  `Resolution`: A string token indicating the resolution level [S1, unofficial].

**Resolution Codes:**
The `Resolution` field uses specific string tokens [S1, unofficial]:
*   High: `/`
*   Medium: `/s`
*   Low: `/s/s`

**Virtual vs. Actual Dimensions:**
The method differentiates between Virtual Dimensions and Actual Dimensions [S1, unofficial]:
1.  **Virtual Dimensions:** The dimensions of the block at the highest resolution (High) [S1, unofficial]. All subimages are expanded to these dimensions for compositing [S1, unofficial].
2.  **Actual Dimensions:** The dimensions of the data points actually stored for that block at its assigned resolution [S1, unofficial].

**Algorithm Steps:**
1.  Parse the Data Description File to determine $N_{blocks}$ and the properties of each block $i$ [S1, unofficial].
2.  For each block $i$, calculate its Virtual Dimensions $(V_x, V_y, V_z)$ based on the bounding box defined by Start/End coordinates [S1, unofficial].
3.  Determine the Actual Dimensions $(A_x, A_y, A_z)$ based on the assigned Resolution code:
    *   If High: $(A_x, A_y, A_z) = (V_x, V_y, V_z)$.
    *   If Medium: $(A_x, A_y, A_z) = (V_x/2, V_y/2, V_z/2)$.
    *   If Low: $(A_x, A_y, A_z) = (V_x/4, V_y/4, V_z/4)$ [S1, unofficial].
4.  Extract or generate the voxel data for block $i$ using the Actual Dimensions [S1, unofficial].
5.  For compositing, upsample or pad the block data to match the Virtual Dimensions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parse_data_description_file(filepath):
    blocks = []
    with open(filepath, 'r') as f:
        lines = f.readlines()
    
    # Line 1: Total number of blocks
    num_blocks = int(lines[0].strip())
    
    for i in range(1, num_blocks + 1):
        line = lines[i].strip().split()
        # Format: Start-X Start-Y Start-Z End-X End-Y End-Z Resolution
        start_x = int(line[0])
        start_y = int(line[1])
        start_z = int(line[2])
        end_x = int(line[3])
        end_y = int(line[4])
        end_z = int(line[5])
        resolution_str = line[6]
        
        # Calculate Virtual Dimensions (High Resolution Size)
        virtual_x = end_x - start_x + 1
        virtual_y = end_y - start_y + 1
        virtual_z = end_z - start_z + 1
        
        # Determine Actual Dimensions based on Resolution Code
        if resolution_str == '/':
            # High
            actual_x = virtual_x
            actual_y = virtual_y
            actual_z = virtual_z
        elif resolution_str == '/s':
            # Medium
            actual_x = virtual_x // 2
            actual_y = virtual_y // 2
            actual_z = virtual_z // 2
        elif resolution_str == '/s/s':
            # Low
            actual_x = virtual_x // 4
            actual_y = virtual_y // 4
            actual_z = virtual_z // 4
        else:
            raise ValueError(f"Unknown resolution code: {resolution_str}")
            
        blocks.append({
            'id': i,
            'start': (start_x, start_y, start_z),
            'end': (end_x, end_y, end_z),
            'virtual_dims': (virtual_x, virtual_y, virtual_z),
            'actual_dims': (actual_x, actual_y, actual_z),
            'resolution': resolution_str
        })
        
    return blocks

def get_block_data(block_info, source_volume):
    """
    Extracts data for a block.
    In a real implementation, this would read from disk or memory.
    Here we assume source_volume contains the high-res data.
    """
    sx, sy, sz = block_info['start']
    ex, ey, ez = block_info['end']
    ax, ay, az = block_info['actual_dims']
    
    # Extract high-res sub-volume
    high_res_block = source_volume[sx:ex+1, sy:ey+1, sz:ez+1]
    
    # Downsample if necessary to get actual dimensions
    if block_info['resolution'] == '/s':
        # [G] Standard 2x2x2 average pooling not specified in sources
        pass
    elif block_info['resolution'] == '/s/s':
        # [G] Standard 4x4x4 average pooling not specified in sources
        pass
        
    return high_res_block
```

**Edge Cases and Pitfalls:**
*   **Boundary Alignment:** The start/end coordinates must align with the resolution scaling factors to avoid fractional voxel indices [S1, unofficial].
*   **Compositing Expansion:** Subimages must be correctly expanded to virtual dimensions, though the specific interpolation kernel is not specified [S1, unofficial].
*   **Resolution Code Parsing:** The string tokens are strict (`/`, `/s`, `/s/s`) [S1, unofficial].

## Subtopics

### Hierarchical Blocking
A multiresolution volume representation where lower-resolution blocks cover increasing spatial sizes, maintaining a constant data size per level [S2, unofficial]. It achieves a data reduction of 2.2:1, whereas flat blocking achieves 3.8:1 for the same levels [S2, unofficial]. Hierarchical blocking is chosen when memory efficiency per level is prioritized over the simplicity of flat structures [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Parsing the description file is $O(N_{blocks})$ [S1, unofficial]. Data extraction and downsampling depend on the volume size and the number of blocks [S1, unofficial].
*   **Memory Complexity:** The total memory usage is the sum of the actual dimensions of all blocks [S1, unofficial].
*   **Performance:** The method allows for parallel visualization by processing blocks independently [S1, unofficial].
*   **Strengths:** Flexible spatial adaptation and a simple text-based description format [S1, unofficial].
*   **Limitations:** Limited to three discrete resolution levels in the primary description [S1, unofficial]. Hierarchical blocking achieves a data reduction of 2.2:1, whereas flat blocking achieves 3.8:1 for the same levels [S2, unofficial].

## Background
The method was developed for the parallel visualization of spatial adaptive-resolution data, specifically using MHD pressure data [S1, unofficial]. It was part of a project aimed at creating a dummy adaptive data representation for temporary use in visualization research [S1, unofficial]. The concept of hierarchical blocking was further explored in the context of GPU-based volume ray-casting to optimize LOD based on visual error [S2, unofficial].

## Sources
[S1] Volume visualization of time-varying data using parallel, multiresolution and adaptive-resolution techniques, https://scholars.unh.edu/cgi/viewcontent.cgi?article=1245&context=thesis, fetched 2026-10-27 (unofficial)
[S2] GPU-Based Volume Ray-Casting with Advanced Illumination, https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2008/advanced_illumination.pdf, fetched 2026-10-29 (unofficial)
