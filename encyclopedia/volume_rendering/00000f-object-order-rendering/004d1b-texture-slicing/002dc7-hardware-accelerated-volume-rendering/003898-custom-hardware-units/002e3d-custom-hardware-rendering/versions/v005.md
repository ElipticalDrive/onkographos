# Custom Hardware Rendering


## Overview
Custom Hardware Rendering refers to direct volume rendering techniques implemented via dedicated, non-standard graphics hardware [S2, unofficial]. It was developed to address the demand for real-time volume rendering systems when general-purpose CPUs and GPUs lacked sufficient processing power [S1]. Unlike its siblings, which leverage existing graphics pipelines or geometry units, this method relies on bespoke silicon designed specifically for the highly regular computations of volume rendering [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Regular 3D volume data (e.g., $256^3$ grids) [S1].
*   **Output:** Rendered 2D image at interactive frame rates [S1].

**Data Structures:**
*   **Voxel Processors:** Traverse data slice-by-slice in memory order [S1].
*   **On-chip Buffers:** Store intermediate voxel data for subsequent processing [S1].
*   **Sample Processors:** Responsible for illumination, filtering, and compositing [S1].

**Steps:**
1.  **Voxel Processing:** Voxel processors traverse the volume data slice-by-slice in memory order [S1].
2.  **Buffering:** Processed voxels are stored in on-chip buffers [S1].
3.  **Sample Processing:** Sample processors traverse these buffers to perform illumination, filtering, and compositing [S1].
4.  **Hybrid Approach (Cube):** In the Cube system, the shear-step was computed on-board, while the graphics board handled warping and image rendering [S1].

**Parameters:**
*   **Cube:** Eight identical rendering pipelines render a $256^3$ volume at 30 fps [S1].
*   **VolumePro:** Scalability was enhanced over Cube, but perspective rendering remained unsupported in earlier generations [S1].

**Edge Cases and Pitfalls:**
*   **Memory Limitations:** Like GPUs, custom hardware suffers from limited on-chip memory [S1].
*   **Bandwidth:** Out-of-core solutions are generally not viable due to the high bandwidth requirements [S1].
*   **Flexibility:** Changing or extending custom hardware is tedious and costly [S1].
*   **Perspective:** Early systems like Cube and VolumePro did not support perspective rendering [S1].

## Operating characteristics
*   **Performance:** Achieves real-time frame rates allowing for interactive rendering due to highly regular computation [S1].
*   **Scalability:** VolumePro enhanced scalability compared to Cube [S1].
*   **Limitations:** Limited memory capacity and lack of perspective support in early generations [S1].
*   **Cost:** High development and modification costs due to custom silicon [S1].

## Background
The need for custom graphics hardware arose from the demand for real-time volume rendering systems when neither GPUs nor CPUs were fast enough [S1]. Most systems were developed for rendering regular data, including Cube [KK99], Vizard [KS97], and VolumePro [PHK* 99] [S1]. Cube, based on a hybrid-order Shear-Warp algorithm, was the predecessor to VolumePro [S1]. VolumePro evolved to include separate sample and voxel processing pipelines [S1]. Never commercially realized, Cube paved the way for VolumePro, which enhanced scalability but still lacked perspective rendering in its earlier forms [S1].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
