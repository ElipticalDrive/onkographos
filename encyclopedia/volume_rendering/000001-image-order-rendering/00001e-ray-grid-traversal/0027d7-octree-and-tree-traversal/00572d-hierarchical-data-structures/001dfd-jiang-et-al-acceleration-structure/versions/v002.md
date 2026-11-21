# Jiang et al. Acceleration Structure
Also called: JRZ*16 method
## Overview
The Jiang et al. acceleration structure [JRZ*16] is a hierarchical data structure designed to efficiently locate active cells within a scalar volume for both direct and indirect visualization approaches [S1]. It solves the problem of redundant ray marching through inactive background regions by identifying and rendering only the relevant active cells [S1]. Unlike pure image-order methods, it is classified as a hybrid object-order and image-order approach, meaning its performance and structure are dependent on the rendering resolution [S1]. It builds on [Hierarchical Data Structures](4133-hierarchical-data-structures.md) and differs from [Hierarchical Spatial Enumeration Optimization](1663-hierarchical-spatial-enumeration-optimization.md) and [Silhouette Image Space Acceleration Structure (Mip Pyramid)](3731-silhouette-image-space-acceleration-structure-mip-pyramid.md).
## How it works
Inputs: A scalar volume dataset and a target rendering resolution [S1].
Outputs: A set of active cells identified for ray marching [S1].
Data structures: The specific internal layout, bit-packing, or hierarchical tree structure of the Jiang et al. acceleration structure is Not given in the sources [S1].
Steps:
1. Traverse the acceleration structure to identify active cells within the volume [S1].
2. Render the identified active cells using ray marching [S1].
Formulas: Not given in the sources.
Parameters: Not given in the sources.
Edge cases: As a resolution-dependent method, performance characteristics and active cell identification boundaries may vary with the chosen rendering resolution [S1].
Pseudocode: Not given in the sources.
## Subtopics

### IsoBAS Acceleration Structure
The IsoBAS acceleration structure [LCD15] is a variant introduced by Liu et al. for faster identification of active cells [S1]. It is distinguished by requiring less memory overhead than previous methods [S1]. It is chosen when memory efficiency is a priority alongside active cell identification speed [S1].

## Operating characteristics
Time and memory complexity: Not given in the sources.
Measured performance: Not given in the sources.
Scaling: The method is dependent on rendering resolution, implying that scaling the resolution affects the structure's utility or performance [S1].
Strengths and limitations: It efficiently locates active cells for both direct and indirect visualization [S1]. Its limitation is the dependency on rendering resolution, classifying it as a hybrid approach rather than a purely resolution-independent object-order method [S1].
When to use it: Use when efficient active cell location is required for ray marching in both direct and indirect visualization [S1].
When not: Avoid if a purely resolution-independent acceleration structure is required [S1].
## Background
Origin: Proposed by Jiang et al. [JRZ*16] [S1].
History: It follows earlier methods such as those by Liu et al. [LCD09, LCD15] which utilized CPU-based sorting of volume slices for front-to-back rasterization [S1]. The Jiang et al. method is considered part of the same family of hybrid object-order and image-order approaches [S1].
## Sources
[S1] Efficient Direct Isosurface Rasterization of Scalar Volumes, https://www.uni-weimar.de/en/media/chairs/computer-science-department/vr/research/real-time-rendering/efficient-direct-isosurface-rasterization-of-scalar-volumes/, fetched 2026-11-01
