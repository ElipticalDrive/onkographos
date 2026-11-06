# Wavelet-based Multiresolution
Also called: Wavelet Multiresolution, Wavelet LOD

## Overview
This family encompasses techniques that decompose volumetric data into a hierarchy of spatial-frequency components using wavelet transforms, enabling efficient Level-of-Detail (LOD) rendering and compression. It differs from sibling methods like [VoxelGrid Down Sample](1014-voxelgrid-down-sample.md) or [Mipmap Generation](2602-mipmap-generation.md) by operating in the frequency domain to preserve high-frequency details at coarse resolutions, rather than relying solely on spatial averaging or subsampling.

## Shared principles
The core mechanism involves a hierarchical decomposition where a volume is represented by a coarse approximation and a set of detail coefficients at multiple scales. Common mathematical and structural operations include:

1.  **Wavelet Decomposition:** Applying a 3D wavelet transform to split a data block into a low-pass subband (approximation) and high-pass subbands (detail coefficients). For a block of size $n$, this typically yields one subband of size $n/8$ and seven detail subbands of total size $7n/8$ [S1, unofficial].
2.  **Hierarchical Aggregation:** Organizing data into a tree structure where internal nodes store the aggregated low-resolution data from their children and the compressed detail coefficients required to reconstruct the children's raw data [S1, unofficial].
3.  **Thresholding and Compression:** Pruning detail coefficients with magnitudes below a user-defined threshold $T$ to zero, followed by lossless compression (e.g., run-length encoding and Huffman coding) to reduce storage overhead [S1, unofficial].
4.  **Error-Bounded Traversal:** Using a hierarchical error metric, often defined as the maximum Mean Square Error (MSE) between a parent's low-resolution data and its children's raw data, to guide LOD selection and ensure error bounds are met during rendering [S1, unofficial].

## Subtopics

### [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md)
Renders multiresolution data by projecting detail coefficients and approximations directly to screen-space quads of decreasing resolution. It differs from tree-based methods by avoiding explicit tree traversal for ray casting, instead treating each level as a separate geometric primitive.

### [Wavelet Tree Overview Map](5354-wavelet-tree-overview-map.md)
A visualization tool that maps the wavelet tree structure into a 2D triangular frame to display error distribution and LOD cuts. It complements the raw tree structure by providing an intuitive spatial overview of data quality.

### [Wavelet Tree](5390-wavelet-tree.md)
The foundational data structure that organizes volume blocks into a strict parent-child hierarchy. Internal nodes store low-resolution approximations and compressed detail coefficients, enabling efficient reconstruction and LOD selection.

## Background
The concept of using wavelets for multiresolution volume representation originated in the mid-1990s, with foundational work by researchers such as Kindlmann and Westin, and later expanded by Linsen et al. for interactive visualization. The specific tree construction and error metrics described in the child articles are derived from early implementations of wavelet-based volume compression and rendering, such as those by Linsen and Gröller. These methods are primarily used in scientific visualization for large-scale datasets where memory bandwidth and rendering performance are critical constraints.

## Sources
