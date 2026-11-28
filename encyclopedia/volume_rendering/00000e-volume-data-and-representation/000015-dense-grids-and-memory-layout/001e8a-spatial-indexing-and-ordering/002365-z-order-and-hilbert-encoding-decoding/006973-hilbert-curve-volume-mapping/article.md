# Hilbert Curve Volume Mapping

Also called: 3D space-filling curve mapping, Hilbert curve-based mapping, Hilbert-based volume embedding

## Overview
Hilbert Curve Volume Mapping is a technique to map 1D sequential data (such as similarity scores from genomic alignments) into a 3D volume grid using a space-filling Hilbert curve to maximize spatial coherence [S1, S2, unofficial]. Unlike naive scan-line traversal, which places distant 1D positions adjacent in 3D space (e.g., position 0 and 4096 in a $64^3$ grid), Hilbert mapping ensures that neighboring positions in the 1D sequence remain adjacent in 3D space, facilitating easier interpretation of data and visuals [S1, S2, unofficial]. It builds on [Z-Order and Hilbert Encoding/Decoding](3013-z-order-and-hilbert-encoding-decoding.md) by applying the encoding specifically to populate a volumetric dataset for rendering.

## How it works
**Inputs:**
- A 1D sequence of data values (e.g., similarity scores) of length $N$.
- Target volume dimensions $(D_x, D_y, D_z)$, typically forming a cube where $N = D_x \times D_y \times D_z$ [S2, unofficial].

**Outputs:**
- A 3D volume grid (voxel array) where each voxel $(x, y, z)$ contains the data value from the 1D sequence mapped to that position.

**Data Structures:**
- **Volume Grid:** A 3D array of size $D_x \times D_y \times D_z$. The layout is determined by the Hilbert curve traversal order.
- **Hilbert Curve State:** To map index $i$ to $(x, y, z)$, the algorithm requires the current position, orientation, and size of the sub-cube being traversed. This is typically managed recursively or iteratively using bit manipulation or octant navigation logic as described in [Max 1998] [S1, S2, unofficial].

**Steps:**
1. **Generate/Traverse the 3D Hilbert Curve:** Construct the path of the Hilbert curve for the target volume dimensions (e.g., $64^3$) [S2, unofficial]. The curve visits every voxel exactly once.
2. **Map 1D Index to 3D Coordinates:** For each index $i$ in the 1D sequence ($0 \le i < N$), determine the corresponding 3D coordinates $(x, y, z)$ along the Hilbert curve. This is done using the encoding algorithm described by Max [15] [S1, S2, unofficial].
3. **Assign Data:** Assign the data value at sequence index $i$ to the voxel at coordinates $(x, y, z)$ [S2, unofficial].

**Pseudocode:**
The following pseudocode assumes a recursive implementation of the 3D Hilbert curve traversal to fill the volume. It uses the standard recursive definition where a curve of order $n$ is composed of 8 sub-curves of order $n-1$.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hilbert_volume_map(data_1d, dim):
    """
    Maps 1D data into a 3D volume using a Hilbert curve.
    
    Args:
        data_1d: List of values of length dim^3.
        dim: Integer dimension of the cubic volume (e.g., 64).
        
    Returns:
        volume: 3D numpy-like array of shape (dim, dim, dim).
    """
    # Initialize empty volume
    volume = [[[None for _ in range(dim)] for _ in range(dim)] for _ in range(dim)]
    
    # Helper function to traverse the Hilbert curve and fill the volume
    # order: current recursion depth (log2(dim))
    # x, y, z: current starting coordinate of the sub-cube
    # orientation: tuple indicating the entry/exit orientation of the sub-cube
    # data_index: current index in the 1D data sequence
    # Returns: next data_index after filling this sub-cube
    
    def traverse(order, x, y, z, orientation, data_index):
        if order == 0:
            # Base case: single voxel
            volume[x][y][z] = data_1d[data_index]
            return data_index + 1
        
        sub_dim = 1 << (order - 1) # 2^(order-1)
        
        # Define the 8 sub-cubes and their orientations based on the current orientation
        # This logic depends on the specific Hilbert curve variant (e.g., Max's algorithm).
        # For a standard 3D Hilbert curve, the sequence of sub-cubes and their 
        # required orientations to maintain continuity is fixed.
        
        # Example structure for one orientation (entry from -x, exit to +x):
        # The actual mapping of sub-cube indices to (dx, dy, dz) offsets and 
        # new orientations must be derived from the Hilbert curve definition.
        
        # Placeholder for the specific recursive step logic from Max [15]
        # In practice, this involves:
        # 1. Determining the 8 sub-cubes' relative positions.
        # 2. Determining the orientation (rotation/reflection) of each sub-cube
        #    so that the exit of one connects to the entry of the next.
        # 3. Recursively calling traverse for each sub-cube.
        
        # Since the exact orientation table for 3D Hilbert is complex and 
        # source [S1, S2, unofficial] refers to [Max 1998] without detailing the table,
        # we describe the iterative bit-manipulation approach often used 
        # for encoding/decoding, which is more implementable from text.
        
        return data_index

    # Alternative: Iterative Encoding Approach (if decode function is available)
    # 1. Precompute or implement a function hilbert_decode(index, dim) -> (x, y, z)
    # 2. Loop through data_1d and assign.
    
    # Given the constraint to be self-contained and the reference to Max [15],
    # the recursive structure is the conceptual model. 
    # For implementation, one would typically use the bit-twiddling 
    # Hilbert encode/decode routines from [3013-z-order-and-hilbert-encoding-decoding.md].
    
    # Simplified iterative assignment assuming hilbert_decode exists:
    for i, value in enumerate(data_1d):
        x, y, z = hilbert_decode(i, dim)
        volume[x][y][z] = value
        
    return volume

def hilbert_decode(index, dim):
    """
    Decodes a 1D Hilbert index to 3D coordinates.
    This is a placeholder for the specific algorithm in [Max 1998].
    Standard 3D Hilbert decoding involves bit manipulation.
    """
    # Not given in the sources. Implementation requires the specific 
    # bit-manipulation logic for 3D Hilbert curves (e.g., from [3013-z-order-and-hilbert-encoding-decoding.md]).
    pass
```

**Parameters:**
- **Volume Dimensions:** Typically a cube, e.g., $64^3$ [S2, unofficial]. The dimension must be a power of 2 for standard recursive Hilbert curve definitions.
- **Data Length:** Must equal the number of voxels ($D^3$) [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Naive Traversal:** Using scan-line traversal places distant 1D positions adjacent in 3D (e.g., position 0 and 4096 in a $64^3$ grid), making interpretation difficult [S1, S2, unofficial].
- **Orientation Loss:** Navigating in 3D space can cause users to lose track of their position in the 1D sequence. This is mitigated by providing a 1D representation alongside the 3D view [S2, unofficial].
- **Precision:** The mapping relies on exact integer arithmetic for indices. Floating-point errors in coordinate calculation should be avoided [G].

## Subtopics

### Octant-Based Navigation
A navigation method that leverages the fractal nature of the Hilbert curve to organize data in an octree-like structure, allowing users to zoom in by selecting octants [S2, unofficial].
- **How it works:**
    1. Divide the currently displayed cubic sequence portion into eight octants [S2, unofficial].
    2. Allow the user to select an octant to zoom in [S2, unofficial].
    3. Zooming out moves to the next lower level of detail [S2, unofficial].
- **UI/Interaction:**
    - Display a 1D bar representing the complete sequence and a lower bar for the displayed portion [S2, unofficial].
    - Highlight the displayed portion in the upper bar [S2, unofficial].
    - Allow the user to click on an octant in the volume or the corresponding part of the 1D bar to zoom [S2, unofficial].
    - Display a vertical line marker in the 1D sequence display [S2, unofficial].
    - Allow the user to drag the marker to move through the 3D volume along the Hilbert curve [S2, unofficial].
- **When to choose:** When interactive exploration of hierarchical data structures within the volume is required, particularly for genomic alignments where local regions of interest need to be examined in detail [S2, unofficial].

## Operating characteristics
- **Time Complexity:** The mapping process is $O(N)$ where $N$ is the number of data elements, as each element is mapped exactly once [S1, S2, unofficial]. The decoding/traversal step is also linear if implemented efficiently.
- **Memory Complexity:** $O(N)$ to store the 3D volume grid [S1, S2, unofficial].
- **Measured Performance:** Applied to multiple alignment datasets, such as the Stem Cell Leukemia (SCL) dataset with 150,000 basepairs from five species (human, mouse, chicken, pufferfish, zebrafish) [S2, unofficial].
- **Strengths:** Maximizes spatial coherence, ensuring that neighboring data points in the 1D sequence are adjacent in 3D space, which aids in visual interpretation of features like conserved regions or specific mutations [S1, S2, unofficial].
- **Limitations:** Interpretation of data is difficult if naive scan-line traversal is used instead [S1, S2, unofficial]. Navigation in 3D can be disorienting without supplementary 1D context [S2, unofficial].
- **When to use:** When visualizing 1D sequential data (e.g., genomic alignments, time-series) in a 3D volume to leverage spatial coherence for feature detection [S1, S2, unofficial].
- **When not to use:** When the data does not have a strong 1D sequential structure or when spatial coherence is not a priority for the visualization task [G].

## Background
- **Origin:** The method was developed for the volume visualization of multiple alignment of large genomic DNA sequences [S1, S2, unofficial].
- **History:** Work by Keim et al. [1995] and Voorhies [1991] showed that Hilbert curve-based mapping is among the most coherent space-filling curve approaches [S1, S2, unofficial]. Coherence is defined as the amount by which neighboring pixels (voxels) are at sequential positions on the curve [Wong et al. 2003] [S2, unofficial]. The specific mapping algorithm used is described by Max [1998] [S1, S2, unofficial].
- **Applications:** Used in bioinformatics for visualizing multiple sequence alignments, such as the SCL dataset, to discover regulatory elements in DNA sequences [S2, unofficial].

## Sources
[S1] Volume visualization of multiple alignment of large genomicDNA, https://digital.library.unt.edu/ark:/67531/metadc887169/?q=multiresolution volume rendering, fetched 2026-10-28 (unofficial)
[S2] Volume visualization of multiple alignment of genomic DNA, https://digital.library.unt.edu/ark:/67531/metadc780327/?q=multiresolution volume rendering, fetched 2026-10-28 (unofficial)
