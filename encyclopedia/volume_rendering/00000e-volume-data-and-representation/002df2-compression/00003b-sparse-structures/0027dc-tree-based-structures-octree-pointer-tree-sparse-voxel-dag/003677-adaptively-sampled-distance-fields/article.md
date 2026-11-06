# Adaptively Sampled Distance Fields

Also called: ASDF

## Overview
Adaptively Sampled Distance Fields (ASDF) are a tree-based compression method for representing 3D shapes and vector outlines by encoding a signed distance field (SDF) rather than binary occupancy [S1, unofficial]. Unlike traditional binary-valued trees that require fine refinement to capture smooth boundaries, ASDF exploits the smoothness of the distance function to achieve high compression ratios while preserving the topological accuracy of the shape's zero set [S1, unofficial]. It is particularly suited for applications requiring resolution-independent antialiasing and magnification of complex vector outlines [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A target shape defined by a signed distance field (SDF) sampled on a regular grid, or a parametric vector representation converted to an SDF.
*   **Output:** A compressed random-access tree structure representing the SDF, optimized for spatial coherence.

**Data Structures:**
The method uses a compressed tree structure (specifically a random-access compressed tree, as detailed in the parent article [4139-tree-based-structures-octree-pointer-tree-sparse-voxel-dag.md]). The tree nodes store quantized distance values. The compression utilizes a codebook (either specialized or universal) to encode the residuals or coefficients of the tree nodes [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Construct or load the signed distance field for the target shape at a specified resolution (e.g., $1024^2$ or higher, as cited in examples) [S1, unofficial].
2.  **Tree Construction:** Build the hierarchical tree structure over the SDF domain.
3.  **Adaptive Simplification with Sign Preservation:**
    *   Apply tree simplification (pruning/coarsening) to reduce the number of nodes.
    *   **Constraint:** Unlike standard compression which minimizes global error, this step restricts simplification to **preserve the sign** of the approximated data $a_l$ everywhere [S1, unofficial].
    *   Let $\tau$ be the tolerance for the approximation. The simplification allows a larger $\tau$ but ensures that for every point in the domain, the sign of the approximated distance value matches the sign of the original distance value [S1, unofficial]. This guarantees that the zero isosurface (the shape boundary) is preserved exactly in topology.
4.  **Compression:** Encode the simplified tree using a codebook (universal or specialized) to generate the final compressed bitstream [S1, unofficial].

**Formulas:**
*   **Sign Preservation Condition:** For all points $x$ in the domain, $\text{sign}(\hat{d}(x)) = \text{sign}(d(x))$, where $d(x)$ is the original distance and $\hat{d}(x)$ is the approximated distance [S1, unofficial].
*   **Accuracy Metric:** Geometric PSNR is calculated by extracting the zero isocurve from the compressed SDF and measuring the RMS distance between points on this curve and the original curve [S1, unofficial].

**Parameters:**
*   **Tolerance ($\tau$):** The approximation tolerance is set larger than in standard binary tree compression to allow more aggressive pruning, relying on the sign preservation constraint to maintain shape fidelity [S1, unofficial].
*   **Resolution:** Examples cite input resolutions such as $1024^2$ (noted as "10252" in source text, likely a typo for $1024^2$ or similar high resolution) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Set Accuracy:** The primary risk is violating the sign preservation constraint, which would shift the shape boundary. The algorithm must strictly enforce $\text{sign}(\hat{d}(x)) = \text{sign}(d(x))$ [S1, unofficial].
*   **Compression vs. Size:** While ASDF is more compact than binary trees for smooth shapes, it may be less efficient for highly noisy or non-smooth data where the distance field lacks smoothness [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptively_sampled_distance_field_compression(original_sdf, codebook):
    """
    Compress a signed distance field using ASDF with sign preservation.
    
    Args:
        original_sdf: 3D array of signed distance values.
        codebook: Pre-trained codebook for compression.
        
    Returns:
        compressed_tree: Compressed representation of the SDF.
    """
    # Step 1: Build the initial tree structure from the SDF
    tree = build_tree_from_sdf(original_sdf)
    
    # Step 2: Perform adaptive simplification with sign preservation
    # The key difference from standard tree compression is the constraint
    # that the sign of the approximated data must match the original.
    simplified_tree = adaptive_simplify_with_sign_preservation(tree, tolerance=large_tau)
    
    # Step 3: Compress the simplified tree using the codebook
    compressed_data = compress_tree(simplified_tree, codebook)
    
    return compressed_data

def adaptive_simplify_with_sign_preservation(tree, tolerance):
    """
    Simplify the tree by pruning nodes, ensuring the sign of the distance
    field is preserved everywhere.
    """
    # Traverse the tree
    for node in tree.nodes:
        # Check if the node can be simplified (pruned/merged)
        # without violating the sign preservation constraint.
        # The approximation error can be up to 'tolerance',
        # but the sign must remain unchanged.
        if can_simplify(node, tolerance):
            if sign_preserved(node.original_value, node.approximated_value):
                prune_node(node)
            else:
                keep_node(node)
        else:
            keep_node(node)
    return tree

def sign_preserved(original_val, approximated_val):
    """
    Check if the sign of the approximated value matches the original.
    """
    return (original_val >= 0 and approximated_val >= 0) or \
           (original_val < 0 and approximated_val < 0)
```

## Operating characteristics
*   **Compression Ratio:** ASDF achieves significantly higher compression ratios than traditional binary-valued trees for smooth shapes because it exploits the smoothness of the distance field [S1, unofficial]. For example, a vector shape represented as an SDF in a compressed tree was 7.8 KB, compared to 3.2 KB for the original parametric representation and 131 KB for a binary image at the same resolution [S1, unofficial].
*   **Accuracy:** The method provides high geometric accuracy, with reported PSNR of 82 dB for the zero isocurve extraction, implying errors are not visible at resolutions below 10K pixels [S1, unofficial].
*   **Rendering Benefits:** The distance field representation enables high-quality antialiasing and resolution-independent magnification, which are not possible with traditional binary images [S1, unofficial].
*   **Limitations:** The method is less effective for shapes that do not have smooth vector outlines, as the compression gain relies on the smoothness of the distance field [S1, unofficial].

## Background
Adaptively Sampled Distance Fields were introduced by Fedkiw, Perrin, Rausch, and James in 2000 [S1, unofficial]. The method was developed to address the limitations of binary-valued tree representations for complex shapes, which require excessive refinement to capture smooth boundaries. By using a signed distance field and enforcing sign preservation during tree simplification, ASDF provides a compact and accurate representation for vector outlines and 3D shapes [S1, unofficial].

## Sources
[S1] Compressed Random-Access Trees for Spatially Coherent Data, https://doi.org/10.2312/egwr/egsr07/339-349, fetched 2026-10-26 (unofficial)
