# NeuralVDB

Also called: Neural VDB

## Overview
NeuralVDB is a hybrid data structure that replaces the lower nodes of a shallow, wide VDB tree with hierarchical neural networks to encode topology and value information, significantly reducing memory footprints while maintaining spatial adaptivity [S1]. It differs from statistical compression methods like PCA or auto-encoders by deliberately "over-fitting" to the input volume to memorize it, ensuring flexibility for arbitrary inputs rather than learning generalizable statistical priors [S1]. Unlike JPEG or MPEG which operate on dense domains, NeuralVDB handles inherently sparse 3D data, offering compression ratios an order of magnitude higher than NanoVDB and two orders of magnitude higher than OpenVDB for in-core representations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse volumetric dataset represented by a standard VDB tree structure (typically OpenVDB configuration) containing scalar, vector, or binary mask values [S1].
*   **Output:** A compressed representation consisting of the retained upper VDB tree nodes and a set of trained Multi-Layer Perceptrons (MLPs) that encode the values and topologies of the lower tree levels [S1].

**Data Structures:**
The structure retains the standard VDB tree hierarchy but modifies the content of the lower levels [S1]:
1.  **Upper Tree Nodes:** Retained explicitly as in standard VDB [S1].
    *   Level 1 (Top): $32^3$ nodes.
    *   Level 2: $16^3$ nodes.
    *   Level 3: $8^3$ nodes.
    *   Level 4 (Leaf): Replaced by neural networks [S1].
2.  **Neural Networks:** A collection of MLPs partitioning the coordinate space [S1]. Each MLP maps floating-point voxel coordinates $(x, y, z)$ to relevant value types [S1].
    *   **Regressor MLPs:** Used for scalar and vector values [S1].
    *   **Classifier MLPs:** Used for binary masks indicating active value/child status [S1].

**Algorithm Steps:**
1.  **Construct VDB Tree:** Represent the input sparse volume using a standard VDB tree structure [S1].
2.  **Define Hierarchy:** Determine the tree depth and fanout factors ($32^3 \to 16^3 \to 8^3$) [S1].
3.  **Partition Space:** Divide the coordinate span of the sparse volume into partially overlapping domains for the MLPs [S1].
4.  **Train Regressors:** For each domain, train an MLP to map coordinates $(x,y,z)$ to scalar/vector values using MSE loss [S1].
5.  **Train Classifiers:** For each domain, train an MLP to map coordinates to binary masks (active/inactive) [S1].
6.  **Store Structure:** Retain the upper VDB nodes explicitly and store the trained MLP weights as the representation for the lower levels [S1].
7.  **Decompression:** Access values by querying the relevant MLP based on coordinate domains, supporting both out-of-core and in-core decompression [S1].

**Formulas:**
The networks are trained to minimize reconstruction error [S1]. For scalar and vector values, the network minimizes Mean Squared Error (MSE) [S1]:
$$L_{MSE}(f, \hat{f}) = \frac{1}{N} \sum_{i=1}^N (f - \hat{f}_i)^2$$
where $f$ is the target value and $\hat{f}_i$ is the predicted value from the network [S1]. For binary masks, a classifier MLP is used [S1].

**Parameters:**
*   **VDB Tree Depth:** Typically four levels [S1].
*   **VDB Fanout Factors:** $32^3$ nodes at the top level, $16^3$ at the second, and $8^3$ at the third level (from top to bottom) [S1].
*   **Optimized Configurations:** Two versions are defined based on which levels are replaced by neural networks [S1]:
    *   `[Hash, 5, 4, NN(3)]`: Applies neural networks only to leaf nodes (Level 3). Favors fast random access [S1].
    *   `[Hash, 5, NN(4), NN(3)]`: Applies neural networks heuristically to the two lower levels (Levels 2 and 3). Achieves smaller memory footprint at the cost of slower access [S1].

**Edge Cases and Pitfalls:**
*   **Temporal Coherency:** For animated volumes, warm-starting training from previous frames is required to accelerate compression and improve temporal coherency [S1].
*   **Access Speed:** The `[Hash, 5, NN(4), NN(3)]` variant has slower random access times compared to `[Hash, 5, 4, NN(3)]` due to the deeper hierarchy of neural lookups [S1].
*   **Precision:** The method incurs small, user-controlled compression errors; visual artifacts are minimal for SDFs and density volumes [S1].

## Operating characteristics
*   **Compression Ratio:**
    *   **Disney Cloud Dataset:** NeuralVDB file size is 25 MB compared to 1.5 GB for OpenVDB (16-bit quantization, Blosc compression), a reduction factor of 60 [S1].
    *   **Space Ship Breaching Simulation:** NeuralVDB total footprint is 1.2 MB compared to 22.7 GB for OpenVDB (16-bit, Blosc), a reduction factor of 18 [S1].
    *   **General:** Compression ratios for sparse signed distance fields and density volumes range from 10x to more than 100x from already compressed VDB inputs [S1].
    *   **Comparison:** Offers an order of magnitude higher compression ratio than NanoVDB and two orders of magnitude higher than OpenVDB for in-core representations [S1].
*   **Memory Footprint:** Significantly reduced by orders of magnitude compared to explicit VDB storage [S1].
*   **Performance:**
    *   **Rendering Speed:** Improving rendering speed is a non-goal; the method may sacrifice rendering speed for compression [S1].
    *   **Access:** Fast random access is supported, particularly in the `[Hash, 5, 4, NN(3)]` configuration [S1].
*   **Strengths:** High memory efficiency, compatibility with existing VDB pipelines, support for arbitrary input types (scalar, vector, SDF, density), and support for both out-of-core and in-core decompression [S1].
*   **Limitations:** Slower access times for deeper neural hierarchies; training time for compression (mitigated by warm-starting for animations) [S1].
*   **When to Use:** Static sparse volumes and animated volumes where memory footprint is critical and compatibility with VDB pipelines is required [S1].
*   **When Not to Use:** When maximum rendering speed is the primary constraint and memory is not a bottleneck [S1].

## Background
NeuralVDB was introduced to improve upon the industry-standard VDB data structure [Museth 2013] by leveraging machine learning advancements [S1]. It addresses the limitation that VDB's memory footprint is never smaller than that incurred by sparse non-constant voxel values [S1]. The method was developed to provide better memory efficiency and compatibility with VDB compared to other hybrid methods like Neural Geometric Level of Detail [Takikawa et al. 2021], Variable Bitrate Neural Fields [Takikawa et al. 2022a], and Instant Neural Graphics Primitives [Müller et al. 2022] [S1]. It is designed to be an open framework that does not require a dedicated network architecture, allowing any purely neural or hybrid methods to be used as a black box submodule [S1].

## Sources
[S1] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks, https://www.emergentmind.com/papers/2208.04448, fetched 2026-10-27
