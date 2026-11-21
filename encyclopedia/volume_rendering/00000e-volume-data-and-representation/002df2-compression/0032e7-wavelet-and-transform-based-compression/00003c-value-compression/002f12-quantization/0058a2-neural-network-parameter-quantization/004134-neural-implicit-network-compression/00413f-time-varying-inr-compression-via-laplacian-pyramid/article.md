# Time-varying INR compression via Laplacian pyramid

Also called: Laplacian Pyramid INR, Tang and Wang, Tang and Wang Laplacian Pyramid INR

## Overview
This method compresses time-varying volumetric datasets by decomposing the 4D space-time domain into a Laplacian pyramid and representing each spatial block at each scale with a separate Implicit Neural Representation (INR) [S1, unofficial]. It sets itself apart from centralized INR approaches like [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md) by trading a single global model for multiple local models, which significantly accelerates decoding speeds while maintaining reconstruction quality [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A time-varying volumetric dataset $V(x, y, z, t)$ with spatial dimensions up to $1,728^3$ and a temporal dimension $T$ [S1, unofficial].
- Output: A set of trained MLPs, one for each block at each pyramid level, along with metadata for the pyramid structure and block assignments.

**Data Structures:**
1.  **Laplacian Pyramid:** A multi-scale decomposition of the 4D volume. The base level contains the low-frequency approximation of the volume. Higher levels contain the difference (residual) between the upsampled lower level and the current level's input.
2.  **Block Grid:** Each level of the pyramid is divided into a 3D grid of spatial blocks.
3.  **MLP Model:** A coordinate-based MLP (typically a SIREN or similar architecture) that maps 4D coordinates $(x, y, z, t)$ to the scalar value for that specific block and level.

**Steps:**
1.  **Laplacian Decomposition:** Apply a 4D Laplacian pyramid decomposition to the input volume $V$. This generates a set of residual volumes $L_0, L_1, \dots, L_k$, where $L_0$ is the coarsest approximation and $L_k$ contains the finest details [S1, unofficial].
2.  **Block Partitioning:** For each level $i$ in the pyramid, partition the spatial domain into a grid of non-overlapping blocks. The block size may vary by level or remain constant, but the partitioning is consistent across time [S1, unofficial].
3.  **Block-MLP Assignment:** Assign each spatial block at each level to a dedicated MLP. Unlike centralized methods that use one MLP for the entire volume, this method uses a distinct MLP for each block-level pair [S1, unofficial].
4.  **Training:** Train each MLP independently using the data from its assigned block at its specific pyramid level. The input to the MLP is the 4D coordinate $(x, y, z, t)$, and the output is the value of the residual at that location. The training data for a block at level $i$ is the subset of the residual volume $L_i$ corresponding to that block [S1, unofficial].
5.  **Compression:** Apply compression techniques to the trained MLPs. This typically includes weight quantization and entropy encoding of the MLP parameters [S1, unofficial]. The specific quantization method is not detailed in the provided notes, but it is implied to be part of the "deep compression techniques" mentioned in the context of similar methods like ECNR [S1, unofficial].

**Parameters:**
-   **Pyramid Levels:** Not specified in the notes.
-   **Block Size:** Not specified in the notes.
-   **MLP Architecture:** Not specified in the notes, but typically a small MLP given the local nature of the representation [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Boundary Artifacts:** Discontinuities at block boundaries may occur if the MLPs are not trained to ensure continuity. This is a common issue in block-based INR methods [G].
-   **Temporal Consistency:** Since each block is trained independently, temporal consistency within a block must be ensured by the MLP architecture and training data [G].

## Subtopics

### [Efficient Compressive Neural Representation (ECNR)](15130-efficient-compressive-neural-representation-ecnr.md)
ECNR is an adaptive neural representation method for compressing time-varying volumetric datasets by replacing a single global Implicit Neural Representation (INR) with multiple small local Multi-Layer Perceptrons (MLPs). It decomposes 4D space-time data into a multiscale block structure using a Laplacian pyramid, assigns blocks to MLPs via balanced clustering, and applies deep compression techniques including block-guided pruning, global quantization, and entropy encoding. Unlike [neurcomp](52-name.md), which uses one large MLP for the entire volume, ECNR achieves significantly faster encoding and decoding speeds while maintaining high reconstruction quality, making it suitable for archiving large datasets where speed is not the primary concern.

## Operating characteristics
-   **Time Complexity:** Decoding is significantly faster than centralized INR methods. Specifically, decoding speeds are 13–30× faster than [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md) on datasets up to $1,728^3$ [S1, unofficial].
-   **Memory Complexity:** The total memory usage is the sum of the compressed MLP parameters for all blocks and levels. This is generally lower than a single large MLP for very large volumes due to the ability to prune and quantize smaller models more effectively [G].
-   **Strengths:** High decoding speed, scalability to large datasets (up to $1,728^3$), and good reconstruction quality [S1, unofficial].
-   **Limitations:** Slower encoding speed compared to centralized methods due to the need to train many small MLPs [S1, unofficial]. Potential for block boundary artifacts [G].
-   **When to use:** When fast decoding is critical, such as in interactive visualization or archiving where retrieval speed is more important than training time [S1, unofficial].
-   **When not to use:** When encoding speed is the primary constraint or when the volume is small enough that a centralized INR is sufficient [G].

## Background
This method was proposed by Tang and Wang [141] as a way to address the scalability issues of centralized INR methods for large, time-varying scientific volumes [S1, unofficial]. It builds on the idea of decomposing the data into a multi-scale representation (Laplacian pyramid) and using local models to capture the details at each scale [S1, unofficial]. The method is part of a broader trend in INR compression that focuses on decomposition and partitioning to improve efficiency [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
