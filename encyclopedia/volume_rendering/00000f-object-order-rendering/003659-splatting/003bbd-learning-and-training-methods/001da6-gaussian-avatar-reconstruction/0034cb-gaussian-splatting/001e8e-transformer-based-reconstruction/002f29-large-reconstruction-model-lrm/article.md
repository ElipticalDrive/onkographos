# Large Reconstruction Model (LRM)
Also called: LRM (Large Reconstruction Model)
## Overview
Large Reconstruction Model (LRM) is a feedforward neural network for single-image to 3D reconstruction [S1, unofficial]. It solves the problem of generating 3D representations from a single input image by directly predicting scene parameters in a single inference step, avoiding the iterative optimization required by methods like NeRF [S1, unofficial]. It is distinguished from its siblings [Flash3D](1481-flash3d.md) and [GS-LRM](1483-gs-lrm.md) by its specific architectural choices for handling single-view inputs, though it shares the general transformer-based feedforward paradigm of its parent [Transformer-based Reconstruction](1794-transformer-based-reconstruction.md).
## How it works
Inputs and outputs: The input is a single RGB image. The output is a set of 3D scene parameters (typically NeRF fields or mesh vertices, though the specific output format is not detailed in the provided sources) [S1, unofficial].
Data structures: Not given in the sources.
Numbered steps:
1. The input image is processed by the feedforward model [S1, unofficial].
2. The model directly predicts the 3D reconstruction [S1, unofficial].
Formulas: Not given in the sources.
Parameters: Not given in the sources.
Edge cases and pitfalls: Not given in the sources.
Pseudocode: Not given in the sources.
## Subtopics

### [Large Gaussian Reconstruction Model (GRM)](5961-large-gaussian-reconstruction-model-grm.md)
A variant that replaces the output format with Gaussian maps and supports multi-view inputs.

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance: Not given in the sources.
Scaling: Not given in the sources.
Strengths and limitations: It provides immediate 3D representations without iterative optimization [S1, unofficial].
When to use it: When single-image to 3D reconstruction is required via a feedforward approach [S1, unofficial].
## Background
Origin: LRM was introduced in the paper "LRM: Large Reconstruction Model for Single Image to 3D" at ICLR 2024 [S1, unofficial].
History and development: It is part of a lineage of feedforward models for 3D reconstruction, following methods like ReconFusion which used diffusion priors [S1, unofficial].
Where it is used: Single image to 3D reconstruction [S1, unofficial].
## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
