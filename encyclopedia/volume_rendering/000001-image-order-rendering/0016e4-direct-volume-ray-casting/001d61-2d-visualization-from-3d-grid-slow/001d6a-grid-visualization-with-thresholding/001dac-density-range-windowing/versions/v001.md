# Density Range Windowing

Also called: Density Clipping, Volume Thresholding

## Overview
Density Range Windowing restricts rendering to voxels whose density falls within a specified $[min, max]$ range, allowing selective visualization of specific structures within a volume [S1, unofficial]. It sets itself apart from its parent method, Grid Visualization with Thresholding, by operating on a continuous density interval rather than a single binary opacity threshold or gradient magnitude, enabling finer control over semi-transparent vs. dense core separation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D scalar grid (volume data) where each voxel has a normalized density value $d \in [0, 1]$.
- Output: A 2D image where voxels outside the window are transparent (opacity 0) and those inside are rendered according to the transfer function.

**Data Structures:**
The method relies on two scalar parameters defining the window:
- $m\_densityMin$: The lower bound of the visible density interval.
- $m\_densityMax$: The upper bound of the visible density interval.

**Parameters:**
- $m\_densityMin$: Default $0.0f$.
- $m\_densityMax$: Default $1.0f$ [S1, unofficial].

**Steps:**
1. For each voxel in the volume grid, retrieve its scalar density value $d$.
2. Check if $d$ satisfies the condition:
   $$m\_densityMin \le d \le m\_densityMax$$
3. If the condition is true, the voxel is included in the rendering pipeline (opacity is determined by the transfer function). If false, the voxel's opacity is set to 0.0, effectively clipping it from view [S1, unofficial].

**Edge Cases and Pitfalls:**
- If $m\_densityMin > m\_densityMax$, the window is invalid, and no voxels are rendered [G].
- Raising $m\_densityMin$ drops semi-transparent structures (e.g., skin, foliage) while retaining dense cores (e.g., bone) [S1, unofficial].
- Lowering $m\_densityMax$ peels off the densest structures [S1, unofficial].

## Subtopics

### MIDA Visualization
A visualization mode that works with a settable range of visualized values, allowing different parts of the volume (inside/outside envelopes) to have different visible value ranges [S2, unofficial]. It enables "carving" or "peeling" by specifying an empty range (where minimum and maximum visualized values are identical) to examine the outline or shape of internal structures [S2, unofficial]. It uses visible range sliders to control the min/max visualized values for specific regions [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(N)$ where $N$ is the number of voxels, as each voxel requires a single comparison against the window bounds.
- Memory: $O(1)$ additional memory for the window parameters, assuming the volume data is already in memory.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
- Strengths: Simple to implement; effective for isolating specific tissue types or density ranges in medical imaging (e.g., CT scans) [S1, unofficial].
- Limitations: Cannot handle complex, non-contiguous density distributions without multiple windows; loses context of clipped structures [S1, unofficial].

**When to use it:**
Use when the goal is to isolate specific density ranges (e.g., bone vs. soft tissue) in a scalar volume.

**When not to use it:**
Not suitable when the volume requires complex, multi-range opacity mapping or gradient-based shading alone.

## Background
Originated in the context of GPU-based volume rendering applications, specifically within the GPUGemsVolumeRendering repository [S1, unofficial]. It is commonly used in radiology-style volume rendering presets (e.g., CT Bone, CT Soft Tissue) where window level/width (WL/WW) acts as a live contrast tweak [S1, unofficial]. MIDA Visualization, a variant, was developed in the context of context-preserving volume image exploration using a 3D painting metaphor [S2, unofficial].

## Sources
[S1] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S2] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
