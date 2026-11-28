# Custom Hardware Rendering


## Overview
Custom Hardware Rendering refers to direct volume rendering techniques implemented via dedicated, non-standard graphics hardware [S2, unofficial]. It emerged to address the computational bottlenecks of software-based rendering when neither CPUs nor GPUs were fast enough to achieve real-time frame rates [S1]. Unlike general-purpose graphics accelerators, these systems utilize highly specialized, fixed-function pipelines optimized for the regular computation patterns of volume data [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Regular 3D volume data (e.g., $256^3$ grids).
*   **Output:** Real-time rendered images (typically 30 fps or higher) [S1].

**Data Structures:**
Specific memory layouts vary by implementation (e.g., Cube, Vizard, VolumePro), but generally involve on-chip buffers to store intermediate voxel or sample data to minimize external memory bandwidth [S1].

**Algorithm Steps:**
The specific implementation depends on the hardware generation. The following describes the hybrid-order approach used in the Cube system, a predecessor to VolumePro [S1]:

1.  **Shear Computation:** The custom hardware computes the shear transformation step on-board. This involves transforming the volume data to align with the viewing direction, typically based on a Shear-Warp algorithm [S1].
2.  **Offloading:** The hardware does not perform the final warping or rasterization. Instead, it passes the sheared data to the standard graphics board [S1].
3.  **Warp and Render:** The graphics board performs the warp transformation and renders the image [S1].

**Parameters:**
*   **Pipelines:** Systems like Cube utilize eight identical rendering pipelines [S1].
*   **Performance:** These pipelines can render a $256^3$ volume at 30 fps [S1].

**Edge Cases and Pitfalls:**
*   **Blurring:** Images may blur during zoom-in operations because the re-sampling of the warp matrix is not adaptive [S1]. Enhanced versions exist but increase computational cost [S1].
*   **Perspective:** Early custom hardware (like Cube and early VolumePro) did not support perspective rendering [S1].
*   **Memory:** Limited on-chip memory restricts the size of the volume that can be processed without external bandwidth bottlenecks [S1].

## Operating characteristics
*   **Performance:** Achieves real-time frame rates suitable for interactive rendering due to highly regular computation [S1].
*   **Scalability:** Scalability was enhanced in later generations (e.g., VolumePro), but perspective rendering remained a limitation in early designs [S1].
*   **Limitations:**
    *   **Cost and Flexibility:** Changing or extending custom hardware is tedious and costly [S1].
    *   **Memory Constraints:** Like GPUs, these systems suffer from limited memory [S1].
    *   **Bandwidth:** Out-of-core solutions are generally not viable due to the high bandwidth requirements [S1].

## Background
The need for custom graphics hardware arose from the demand for real-time volume rendering systems when standard CPUs and GPUs were insufficient [S1]. Notable systems include Cube [KK99], Vizard [KS97], and VolumePro [PHK* 99] [S1]. Cube was based on a hybrid-order algorithm using Shear-Warp and served as the predecessor to the well-known VolumePro board [S1]. While Cube was never commercially realized, VolumePro evolved to include separate sample and voxel processing pipelines, where voxel processors traverse data slice-by-slice and store them in on-chip buffers, which are then traversed by sample processors for illumination, filtering, and compositing [S1].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
