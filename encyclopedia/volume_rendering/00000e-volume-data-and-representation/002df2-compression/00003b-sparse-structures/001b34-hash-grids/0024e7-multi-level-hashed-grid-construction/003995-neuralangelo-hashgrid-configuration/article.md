# Neuralangelo Hashgrid Configuration

Also called: Neuralangelo Hashgrid Parameters

## Overview
Neuralangelo Hashgrid Configuration defines the specific hyperparameters for the multi-level hash grid encoding used in the Neuralangelo framework for neural implicit surface reconstruction [S1]. It solves the problem of balancing representational capacity against memory footprint in coordinate-based neural networks by specifying a fixed hierarchy of spatial resolutions and feature dimensions [S1]. It differs from sibling methods like Multi-Grid Merging by using a static, pre-defined resolution schedule rather than dynamic merging, and from Hierarchical Voxel Block Hashing by relying on uniform hash table lookups rather than octree-based spatial partitioning [S1][S2, unofficial].

## How it works
The configuration specifies the architecture of the hash grid encoder, which maps 3D coordinates to feature vectors via multi-resolution hashing [S1].

**Inputs and Outputs**
- Input: 3D coordinate vector $\mathbf{x} \in \mathbb{R}^3$.
- Output: Feature vector $\mathbf{f} \in \mathbb{R}^{N_{levels} \times F}$, where $N_{levels}$ is the number of hash levels and $F$ is the feature size per level.

**Data Structures**
The hash grid consists of $N_{levels}$ independent hash tables. Each table $l$ (where $l \in \{0, \dots, N_{levels}-1\}$) has:
- A hash map of size $H = 2^{19}$ entries [S1].
- A resolution $R_l$ defining the spatial granularity [S1].
- A feature vector of size $F = 8$ stored at each entry [S1].

**Parameters**
- Number of levels ($N_{levels}$): 16 [S1].
- Hashmap size ($H$): $2^{19}$ (524,288) [S1].
- Feature size ($F$): 8 [S1].
- Resolution range: Minimum resolution $R_{min} = 32$, Maximum resolution $R_{max} = 4096$ [S1].
- Training epochs: 100,000 with scaled schedulers [S1].

**Resolution Schedule**
The resolutions are distributed logarithmically between $R_{min}$ and $R_{max}$ [S1]. For level $l$, the resolution $R_l$ is calculated as:
$$ R_l = R_{min} \times \left( \frac{R_{max}}{R_{min}} \right)^{\frac{l}{N_{levels}-1}} $$
This ensures that lower levels capture coarse global structure while higher levels capture fine local details [S1].

**Lookup Process**
1. For each level $l$, quantize the input coordinate $\mathbf{x}$ to the grid defined by $R_l$.
2. Compute the hash index $h$ for the grid cell containing $\mathbf{x}$ using a hash function $Hash(\cdot)$ with modulo $H$:
   $$ h = Hash(\text{grid\_cell}(\mathbf{x}, R_l)) \pmod H $$
3. Retrieve the feature vector $\mathbf{f}_l$ from the hash table at index $h$ [S1].
4. Concatenate all $\mathbf{f}_l$ to form the final feature vector [S1].

**Edge Cases and Pitfalls**
- Hash collisions: With $H=2^{19}$ and high-resolution levels, collisions are possible. The method relies on the MLP to average out noise from collisions, which is effective for dense data but less so for sparse data [S2, unofficial].
- Precision: Floating-point precision in coordinate quantization can affect lookup consistency [G].

## Subtopics

### [Adaptive Feature Grids](10458-adaptive-feature-grids.md)
Adaptive Feature Grids replaces fixed uniform grid placement with multiple spatially adaptive grids, each equipped with learnable transformation matrices to concentrate representational capacity in high-error regions. Unlike standard hash encodings which rely on collision averaging for sparse data, this approach optimizes parameter distribution via a feature density loss, making it superior for dense, complex scientific volumes but underperforming on highly sparse datasets.

### [Structure-Aware Containment Aggregation](17258-structure-aware-containment-aggregation.md)
Structure-Aware Containment Aggregation (SACA) is a topology-aware module that modulates coarser-level and finer-level feature aggregations in Hierarchical Anchor Coupling (HAC) by distinguishing anchors participating in valid octree parent-child containment relations from spatially isolated anchors. It reduces the influence of topologically isolated anchors using a binary containment mask or a soft-weighting scheme, complementing the cross-level gradient pathways of HAC.

### Wu INR Compression
Wu INR Compression is a method scaling INR compression to tera-scale data on consumer hardware using hash grids combined with small fully fused MLPs and three engineering optimizations [S2, unofficial]. It implements sample-streaming rendering for speedup, macro-cell acceleration for empty-space skipping, and out-of-core training for volumes exceeding GPU memory [S2, unofficial]. On a direct numerical simulation (DNS) volume of $10,240 \times 7,680 \times 1,536$ (approximately 950 GB in double precision), the method achieved 35.9 dB PSNR with a 1.5 GB model in under two hours on a single consumer GPU [S2, unofficial].

## Operating characteristics
- **Memory**: The hash grid size is fixed at $16 \times 2^{19} \times 8$ bytes $\approx 100$ MB for the grid alone, plus MLP weights [S1]. This is significantly smaller than the original 950 GB DNS volume [S2, unofficial].
- **Speed**: Hash lookups are $O(1)$ per level, making encoding fast. The method enables real-time or near-real-time rendering when combined with efficient sampling [S2, unofficial].
- **Scalability**: The configuration scales to tera-scale data when combined with out-of-core training and macro-cell acceleration [S2, unofficial].
- **Limitations**: Performance degrades on highly sparse data compared to methods that do not rely on hash collision averaging, such as those using explicit sparse structures [S2, unofficial].

## Background
The Neuralangelo hashgrid configuration was introduced as part of the Neuralangelo framework for high-fidelity neural surface reconstruction [S1]. It builds on the multi-level hashed grid construction method [3402-multi-level-hashed-grid-construction.md]. The specific parameters (16 levels, $2^{19}$ hashmap size, 8 features) were adopted from the Sdfstudio implementation [S1]. The configuration was later adapted for training efficiency, reducing epochs from 500k to 100k with scaled schedulers [S1]. It has been used in various neural field evaluations, including RING-NeRF, NeuS-facto, and Neuralangelo comparisons [S1].

## Sources
[S1] RING-NeRF : Rethinking Inductive Biases for Versatile and Efficient Neural Fields, https://doi.org/10.1007/978-3-031-72761-0_9, fetched 2026-10-30
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
