# Compressed Random-Access Tree Framework

Also called: CRAT, Compressed adaptive tree

## Overview
The Compressed Random-Access Tree (CRAT) framework compresses adaptive multiresolution hierarchies into a compact, randomly-accessible structure [S1, unofficial]. Unlike prior schemes that use entropy coding on sequential traversals, CRAT employs fixed-rate encoding for both topology and data, enabling fine-grain random queries required by rendering algorithms [S1, unofficial]. It replaces pointers with local offsets, uses a forested mipmap structure, and applies vector quantization (VQ) to inter-level residuals, storing both offsets and codebook indices as byte records [S1, unofficial].

## How it works
**Inputs:** An adaptive multiresolution hierarchy (e.g., a quadtree or octree) representing spatially coherent data.
**Outputs:** A compressed bitstream or byte array containing fixed-rate encoded topology and data.

**Data Structures:**
The structure is a forest of trees (a mipmap-like hierarchy). Each node in the compressed representation is stored as a byte record.
1.  **Topology:** Encoded using fixed-rate fields. Pointers are replaced by local scaled offsets.
2.  **Data:** Inter-level residuals are vector-quantized. Codebook indices refer to data blocks, making the tree one level shallower than the finest resolution data [S1, unofficial].

**Algorithm Steps:**
1.  **Primal Subdivision:** Construct the adaptive hierarchy using primal subdivision. Note that slanting at boundaries may cause some subtrees to fall outside the input domain [S1, unofficial].
2.  **Topology Encoding:** Replace absolute pointers with local offsets. These offsets are encoded as fixed-rate byte records to allow random access without pointer indirection [S1, unofficial].
3.  **Residual Calculation:** Compute inter-level residuals between coarser and finer levels.
4.  **Vector Quantization (VQ):** Apply VQ over broods (groups of children) in the tree to compress the residuals [S1, unofficial].
5.  **Handling Undefined Data:** The compression scheme efficiently ignores undefined data in subtrees that fall outside the input domain due to boundary slanting [S1, unofficial].
6.  **Storage:** Store the resulting topology offsets and VQ codebook indices as 8-bit byte records [S1, unofficial].

**Pseudocode:**
The following pseudocode illustrates the evaluation of a 1D primal tree, demonstrating the interpolation logic used during random access. The compressed structure replaces the `Tree` pointers with offset-based lookups.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_1d(root_l, root_r, x, level):
    """
    Evaluate value at position x in a 1D tree.
    root_l, root_r: References to left and right child nodes (or roots of subtrees).
    x: Position in [0, 1].
    level: Current recursion depth.
    """
    # Interpolate current level values
    v_c = interp(root_l.val, root_r.val, x)
    
    if level <= 0.0:
        return v_c
    
    # Determine finer level value
    if x < 0.5:
        # Select left subtree
        # Recurse with scaled coordinate and decremented level
        v_f = evaluate_1d(root_l.L, root_l.R, (x - 0.0) * 2, level - 1.0)
    else:
        # Select right subtree
        v_f = evaluate_1d(root_r.L, root_r.R, (x - 0.5) * 2, level - 1.0)
        
    # Combine current and finer values (implementation detail for specific interpolation scheme)
    # Note: The source shows a recursive structure for traversal; 
    # the actual CRAT compression replaces the pointer access with offset decoding.
    return v_f

def interp(v1, v2, t):
    """Linear interpolation between v1 and v2."""
    return v1 * (1.0 - t) + v2 * t
```

**Parameters:**
-   **Codebook Size:** Not given in the sources.
-   **Bit Depth:** 8-bit fields for both topology and data [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Boundary Slanting:** Primal subdivision can cause subtrees to fall outside the input domain. The compression scheme must handle these undefined regions efficiently [S1, unofficial].
-   **Memory Bandwidth:** Tree structures may increase memory bandwidth due to multiple memory references per query. However, hierarchical access patterns are coherent, allowing memory caches to intercept most reads [S1, unofficial].

## Subtopics

### [Hierarchical Delta Encoding and Quantization](7225-hierarchical-delta-encoding-and-quantization.md)
Compresses hierarchical tree structures using top-down delta encoding and quantization for filesystem traversal.

### [Encoded Local Offsets for Tree Topology](7501-encoded-local-offsets-for-tree-topology.md)
Compresses tree topology by replacing absolute pointers with local scaled offsets for random access.

## Operating characteristics
**Time Complexity:** Random access requires traversing the tree depth. The fixed-rate encoding allows efficient parsing by CPU or GPU shaders [S1, unofficial].
**Memory Complexity:** Extremely compact; both topology and data are encoded in 8-bit fields [S1, unofficial].
**Performance:** Demonstrated real-time performance in pixel shaders for trilinear mipmap evaluation without specialized hardware [S1, unofficial].
**Strengths:** Enables fine-grain random queries; efficient handling of spatially coherent data; simple data structure suitable for GPU shaders.
**Limitations:** Not intended for detailed color images, which result in dense trees and are better handled by block-based schemes [S1, unofficial].
**Scaling:** Hierarchical access patterns are coherent, allowing memory caches to intercept most reads, mitigating bandwidth costs [S1, unofficial].

## Background
The framework was introduced by Sylvain Lefebvre and Hugues Hoppe [S1, unofficial]. It addresses the limitation of prior compressed tree schemes that relied on sequential traversal and entropy coding, which prevented the random access needed for rendering. The work demonstrates applications on light maps, alpha mattes, distance fields, and HDR images [S1, unofficial].

## Sources
[S1] Compressed Random-Access Trees for Spatially Coherent Data, https://doi.org/10.2312/egwr/egsr07/339-349, fetched 2026-10-26 (unofficial)
