# Pixel-Planes 5 Volume Rendering Architecture

Also called: Pixel-Planes 5 Architecture

## Overview
A hardware architecture for real-time volume rendering using the Pixel-Planes 5 system, consisting of graphics processors and pixel processors, where pixel processors store voxel function values and gradients, and graphics processors handle ray tracing and compositing [S1, unofficial]. It separates the heavy classification and shading computations from the ray traversal logic to leverage massive parallelism in the pixel processor array [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D voxel dataset (e.g., $256 \times 256 \times 256$), classification mapping, shading parameters, and observer position.
*   **Output:** A color image in the frame buffer.

**Data Structures**
*   **Pixel Processor Backing Store:** Stores function value and gradient for several voxels. In the reference implementation, each pixel processor is assigned 64 voxels [S1, unofficial].
*   **Hierarchical Enumeration:** A pyramid of binary volumes installed in each graphics processor to encode coherence [S1, unofficial].
*   **Local Cache:** Retains voxels received by a graphics processor for reuse [S1, unofficial].

**Numbered Steps**
1.  **Voxel Storage:** Pixel processors store function value and gradient for their assigned voxels in their backing store [S1, unofficial].
2.  **Classification and Shading:** Pixel processors perform classification and shading calculations for all voxels in their backing store [S1, unofficial].
3.  **Ray Tracing Initiation:** Graphics processors trace rays to generate an image [S1, unofficial].
4.  **Voxel Request:** Graphics processors request sets of voxels from pixel processors as necessary [S1, unofficial].
5.  **Interpolation and Compositing:** Graphics processors perform tri-linear interpolation and compositing operations on the requested voxels [S1, unofficial].
6.  **Frame Buffer Update:** Graphics processors transmit resulting pixel colors to the frame buffer [S1, unofficial].

**Strategies for Optimization**
1.  **Hierarchical Enumeration:** Install a pyramid of binary volumes in each graphics processor to encode coherence and tell the graphics processor which voxels are interesting (non-transparent) and worth requesting [S1, unofficial].
2.  **Adaptive Sampling:** Use an adaptive sampling scheme to reduce the number of rays required to generate an initial image [S1, unofficial].
3.  **Caching:** Retain all voxels received by a graphics processor in a local cache. If the observer does not move, cached voxels drive successive refinement. If the observer moves, cached voxels may still be useful depending on movement distance [S1, unofficial].

**Formulas and Parameters**
*   **Shading Time:** Applying a monochrome Phong shading model at a single voxel using a pixel processor takes about 1 msec [S1, unofficial].
*   **Total Classification/Shading Time:** For a $256 \times 256 \times 256$ voxel dataset, with each pixel processor assigned 64 voxels, the time to classify and shade the entire dataset is about 64 msec [S1, unofficial].

**Pitfalls**
*   **Recomputation:** If the user changes the classification mapping such that the set of interesting voxels changes, the hierarchical enumeration must be recomputed, taking several seconds [S1, unofficial].

## Operating characteristics
*   **Hardware Layout:** 32 graphics processors (10-20 MFLOP each), 1/4 million pixel processors, 1024 x 768 pixel color frame buffer, 5 Gbit/sec ring network [S1, unofficial].
*   **Performance Estimate (Classification/Shading):** For a $256 \times 256 \times 256$ voxel dataset, the time to classify and shade the entire dataset is about 64 msec [S1, unofficial].
*   **Frame Rate Estimate (Observer Position Change):** 10 fps for slightly coarse images, 1 fps for high-quality images [S1, unofficial].
*   **Frame Rate Estimate (Shading/Classification Change):** 20 fps for coarse images, 2 fps for high-quality images, if the hierarchical enumeration is not invalidated [S1, unofficial].
*   **Strengths:** High parallelism for shading/classification; potential for real-time interaction.
*   **Limitations:** Recomputation of hierarchical enumeration is slow (several seconds) if classification mapping changes significantly [S1, unofficial].

## Background
The Pixel-Planes 5 architecture was developed under joint funding by DARPA and the NSF, with development starting in the late summer of 1986 [S1, unofficial]. It was expected to become operational during the summer or fall of 1989 [S1, unofficial]. Although not explicitly designed for volume rendering, its flexibility made it well suited to the task [S1, unofficial].

## Sources
[S1] DTIC ADA208091: Interactive Visualization of 3D Medical Data, https://archive.org/details/DTIC_ADA208091, fetched 2026-10-26 (unofficial)
