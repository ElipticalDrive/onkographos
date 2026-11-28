# Time-varying INR compression via Laplacian pyramid

Also called: Laplacian Pyramid INR, Tang and Wang, Tang and Wang Laplacian Pyramid INR

## Overview
This method compresses time-varying volumetric datasets by decomposing the 4D space-time domain into a Laplacian pyramid and representing each spatial block at each scale with a separate Implicit Neural Representation (INR) [S1, unofficial]. It sets itself apart from centralized INR approaches like [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md) by trading a single global model for multiple local models, which significantly accelerates decoding speeds [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A time-varying volumetric dataset $V(x, y, z, t)$ with spatial dimensions up to $1,728^3$ [S1, unofficial].
- Output: Not given in the sources.

**Data Structures:**
- Laplacian Pyramid: Not given in the sources.
- Block Grid: Not given in the sources.
- MLP Model: Not given in the sources.

**Numbered Steps:**
1. Decompose the input volume $V$ into a Laplacian pyramid [S1, unofficial].
2. Partition the spatial domain of each pyramid level into a grid of non-overlapping blocks [S1, unofficial].
3. Assign a dedicated MLP to each spatial block at each pyramid level [S1, unofficial].
4. Train each MLP independently to map 4D coordinates $(x, y, z, t)$ to the scalar values of its assigned block at its specific pyramid level [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
- Pyramid Levels: Not given in the sources.
- Block Size: Not given in the sources.
- MLP Architecture: Not given in the sources.

**Edge Cases and Pitfalls:**
- Boundary Artifacts: Discontinuities at block boundaries may occur if the MLPs are not trained to ensure continuity [G].
- Temporal Consistency: Independent training of blocks may require architectural choices or regularization to maintain temporal smoothness [G].

**Pseudocode:**
Not given in the sources.

## Subtopics

### [Efficient Compressive Neural Representation (ECNR)](15130-efficient-compressive-neural-representation-ecnr.md)
ECNR is an adaptive neural representation method for compressing time-varying volumetric datasets by replacing a single global Implicit Neural Representation (INR) with multiple small local Multi-Layer Perceptrons (MLPs). It decomposes 4D space-time data into a multiscale block structure using a Laplacian pyramid, assigns blocks to MLPs via balanced clustering, and applies deep compression techniques including block-guided pruning, global quantization, and entropy encoding. Unlike [neurcomp](52-name.md), which uses one large MLP for the entire volume, ECNR achieves significantly faster encoding and decoding speeds while maintaining high reconstruction quality, making it suitable for archiving large datasets where speed is not the primary concern.

## Operating characteristics
- Time complexity: Decoding is 13–30× faster than [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md) on datasets up to $1,728^3$ [S1, unofficial].
- Memory complexity: Not given in the sources.
- Measured performance: Decoding speeds of 13–30× faster than [Neural Implicit Network Compression](10414-neural-implicit-network-compression.md) on datasets up to $1,728^3$ [S1, unofficial].
- Scaling: Not given in the sources.
- Strengths: High decoding speed and scalability to large datasets (up to $1,728^3$) [S1, unofficial].
- Limitations: Not given in the sources.
- When to use: Not given in the sources.
- When not to use: Not given in the sources.

## Background
This method was proposed by Tang and Wang [141] to address the scalability issues of centralized INR methods for large, time-varying scientific volumes [S1, unofficial]. It builds on the idea of decomposing the data into a multi-scale representation (Laplacian pyramid) and using local models to capture the details at each scale [S1, unofficial]. The method is part of a broader trend in INR compression that focuses on decomposition and partitioning to improve efficiency [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
