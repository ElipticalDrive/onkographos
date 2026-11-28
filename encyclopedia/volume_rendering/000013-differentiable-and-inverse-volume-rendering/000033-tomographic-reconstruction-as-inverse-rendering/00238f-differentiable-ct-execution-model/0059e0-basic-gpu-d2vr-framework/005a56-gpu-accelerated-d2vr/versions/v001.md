# GPU-Accelerated D2VR

Also called: GPU D2VR with occlusion culling

## Overview
A GPU-accelerated implementation of Direct 2D Volume Rendering (D2VR) that incorporates occlusion culling and empty space skipping to achieve interactive or near-interactive rendering speeds [S1, unofficial]. It builds on the RapidCT framework but generalizes it to allow reconstruction of arbitrary oriented volumes and incorporates acceleration techniques to limit reconstruction effort to visible/non-occluded samples [S1, unofficial].

## How it works
The method extends the GPU-accelerated filtered backprojection approach used in RapidCT [S1, unofficial]. It performs ray advancement in a single fragment shader loop, allowing for non-linear ray effects [S1, unofficial].

**Inputs and Outputs:**
*   **Inputs:** 2D projection images (sinogram data), camera orientation parameters defining an arbitrary image plane.
*   **Outputs:** Rendered 2D volume image.

**Data Structures:**
*   Projection images stored in GPU texture memory.
*   Volume data structure aligned with the arbitrary image plane.
*   Depth correction factors applied during backprojection.

**Algorithm Steps:**
1.  **Filtering:** Filter acquired images along their columns using a ramp filter (e.g., Shepp-Logan, RamLak) in the frequency domain [S1, unofficial].
2.  **Backprojection:** Backproject filtered data into the volume. For cone-beam geometry, multiply weights $w_{ij}$ by a depth correction factor:
    $$ w_{ij}(d) = w_{ij} \frac{a^2}{d^2} $$
    where $a$ is a constant and $d$ is the depth [S1, unofficial].
3.  **Volume Alignment:** Generate a matrix of samples exactly aligned with the arbitrary image plane, generalizing the CT framework to support arbitrary orientations [S1, unofficial].
4.  **Ray Casting:** Run the entire ray advancement in a single fragment shader loop [S1, unofficial].
5.  **Acceleration:** Apply occlusion culling and empty space skipping to limit reconstruction effort to only visible and non-occluded samples [S1, unofficial].

**Parameters:**
*   Filter type: Shepp-Logan, RamLak [S1, unofficial].
*   Target framerate: 2 frames/s and more for realistic dataset sizes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Arbitrary orientation requires precise alignment of the sample matrix with the image plane [S1, unofficial].
*   Depth correction factor must be correctly applied for cone-beam geometry [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Performance:** Enables framerates of 2 frames/s and more for realistic dataset sizes [S1, unofficial].
*   **Strengths:** Interactive or near-interactive speed; supports arbitrary volume orientations; efficient via occlusion culling and empty space skipping [S1, unofficial].
*   **Limitations:** Original D2VR method was comparatively slow; this implementation addresses that but performance depends on dataset size and acceleration effectiveness [S1, unofficial].

## Background
The method builds on the RapidCT framework for GPU-accelerated CT reconstruction [S1, unofficial]. It was proposed to address the slow rendering speed of the original D2VR method, which was a shortcoming for clinical deployment [S1, unofficial]. The work is part of the Volume Graphics 2006 conference proceedings [S1, unofficial].

## Sources
[S1] GPU-Accelerated D2VR, https://doi.org/10.2312/vg/vg06/023-030, fetched 2026-10-26 (unofficial)
