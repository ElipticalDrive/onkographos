# Segmented Dataset Quantization

Also called: Block-based Segmented Quantization, Agus et al. Method

## Overview
Segmented Dataset Quantization is a lossy compression technique for volumetric segmentation data (uint32_t labels) that reduces storage by restricting small spatial blocks to contain only one or two distinct label values [S1]. It solves the problem of high memory bandwidth and storage requirements for large-scale segmented volumes by encoding the block type and the geometric splitting plane that separates the two components [S1]. Unlike general-purpose texture compression standards (e.g., VTC, ETC2, ASTC) which treat data as continuous color fields, this method explicitly models the discrete, categorical nature of segmentation labels, making it suitable for datasets with sharp, non-interpolating boundaries [S1].

## How it works
**Inputs and Outputs:**
- Input: A 3D segmentation volume $V$ consisting of uint32_t labels.
- Output: A compressed bitstream containing block types and splitting plane coordinates.

**Data Structures:**
The volume is partitioned into fixed-size 3D blocks (bricks). For each block, the following metadata is stored:
1. **Block Type**: An integer identifier indicating the quantization state (e.g., single-label, two-label split).
2. **Splitting Plane**: If the block contains two labels, the orientation and position of the plane separating them.

**Steps:**
1. **Block Partitioning**: Divide the input volume $V$ into non-overlapping 3D blocks of size $B \times B \times B$.
2. **Label Analysis**: For each block, identify the set of unique labels present.
3. **Quantization Decision**:
   - If the number of unique labels $|L| = 1$, assign the block type for "single label". Store the single label value.
   - If $|L| = 2$, assign the block type for "two-label split". Determine the splitting plane that best separates the two label regions.
   - If $|L| > 2$, the method as described in the source does not specify a fallback. [G] Typically, this would either increase block size, use a higher-order split, or fall back to a less compressed representation, but this detail is not given in the sources.
4. **Splitting Plane Encoding**: For blocks with two labels, encode the splitting plane. The plane is defined by its normal vector $\mathbf{n}$ and distance $d$ from the origin, or equivalently by its axis-aligned orientation and position. The source specifies "the splitting plane which subdivides the block" [S1].
5. **Compression**: The sequence of block types and splitting plane parameters is compressed using an entropy coder (e.g., rANS, as implied by the parent structure's use of rANS secondary compression [Parent]).

**Formulas:**
Let $B$ be the block size. For a block $b_i$, let $L_i$ be the set of unique labels in $b_i$.
- If $|L_i| = 1$, cost is minimal (label ID only).
- If $|L_i| = 2$, cost includes label IDs and plane parameters.
The splitting plane $P$ is defined such that for any voxel $v$ in the block:
$$ \text{label}(v) = \begin{cases} l_1 & \text{if } P(v) < 0 \\ l_2 & \text{if } P(v) \ge 0 \end{cases} $$
where $P(v) = \mathbf{n} \cdot v + d$. The exact encoding of $\mathbf{n}$ and $d$ is not specified in the source [S1].

**Parameters:**
- Block size $B$: Not given in the sources.
- Label precision: uint32_t.

**Edge Cases and Pitfalls:**
- **More than two labels**: The source does not specify behavior for blocks with $>2$ labels [S1].
- **Precision**: The splitting plane must be quantized to fit within the block. The source does not specify the precision of the plane coordinates [S1].

## Subtopics

### [Evolutionary Volumetric Data Structure (EVDS) / L-Block Compression (LBC)](11032-evolutionary-volumetric-data-structure-evds-l-block-compress.md)
The Evolutionary Volumetric Data Structure (EVDS), stored as an L-block Covering (LBC), is a hierarchical sparse data structure designed for the real-time incremental compression and segmentation of filamentary volumetric data, such as neuronal tissue. It solves the problem of managing massive, sparse datasets (e.g., 30 TB mouse brain scans) by packaging significant data into connected L-blocks (LBs) while ignoring "white space," allowing for on-the-fly generation without storing the entire volume in memory. Unlike standard AABB-trees, the LBC allows for easy incremental box building and maintains connectivity between nodes without requiring traversal through a parent node.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Proportional to the number of blocks times the average bits per block (1 label vs. 2 labels + plane).
- **Measured Performance**: Not given in the sources.
- **Strengths**: Efficient for segmented data with smooth or planar boundaries between regions. Reduces storage for homogeneous or bi-modal blocks.
- **Limitations**: Limited to blocks with at most two labels. Not suitable for complex multi-label regions without further subdivision or fallback.
- **When to use**: When the segmentation volume has large regions of uniform label or simple binary boundaries.
- **When not to use**: When labels are highly mixed within small blocks.

## Background
The method was proposed by Agus et al. [AGIM10] as a way to handle segmented datasets [S1]. It is distinct from texture compression methods like VTC [Cra04], ETC2 [SP07], and ASTC [NLP*12], which are extensions of 2D techniques to 3D and are designed for continuous color data [S1]. The method is relevant for real-time rendering of time-varying volumetric datasets where limited bandwidth makes compressed representations essential [S1].

## Sources
[S1] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
