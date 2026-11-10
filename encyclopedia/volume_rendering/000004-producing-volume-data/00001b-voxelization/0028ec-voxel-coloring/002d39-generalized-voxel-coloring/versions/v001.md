# Generalized Voxel Coloring

Also called: GVC-IB, GVC-LDI

## Overview
Generalized Voxel Coloring (GVC) is a volumetric reconstruction algorithm that computes visibility exactly to determine color consistency, addressing the approximation errors inherent in earlier Space Carving methods [S1, unofficial]. Unlike its parent [Voxel Coloring](4418-voxel-coloring.md), which relies on approximate visibility checks, GVC ensures that a voxel is only retained if its color is consistent across all cameras for which it is strictly visible, resulting in numerically more consistent reconstructions [S1, unofficial].

## How it works
GVC operates by maintaining precise visibility information for every voxel in the volume. The core innovation lies in the data structures used to track which voxels are occluded by others from the perspective of each camera.

**Inputs and Outputs**
*   **Inputs:** A set of $N$ calibrated camera views (intrinsics and extrinsics), a bounding volume discretized into voxels, and the image data for each view.
*   **Outputs:** A set of color-consistent surface voxels.

**Data Structures**
GVC uses one of two data structures to record visibility per pixel:
1.  **Item Buffer (IB):** For every pixel in an image, stores the single surface voxel that is currently visible from that pixel [S1, unofficial].
2.  **Layered Depth Image (LDI):** For every pixel in an image, stores a depth-sorted list of *all* surface voxels that project to that pixel [S1, unofficial].

**Algorithm Steps**
1.  **Initialization:** Initialize the voxel grid. For each camera view, initialize the visibility structure (IB or LDI). In an IB, this may involve marking pixels as empty or storing a null reference. In an LDI, initialize empty lists for each pixel [S1, unofficial].
2.  **Visibility Update:** For each camera view $i$:
    *   Project all candidate voxels onto the image plane of camera $i$.
    *   Determine the depth of each projected voxel.
    *   **If using IB:** For each pixel, identify the voxel with the minimum depth (closest to camera). Update the IB entry for that pixel to store this voxel. This effectively performs a depth-buffer-like occlusion test [S1, unofficial].
    *   **If using LDI:** For each pixel, collect all voxels projecting to it. Sort them by depth. Store the sorted list in the LDI entry for that pixel [S1, unofficial].
3.  **Color Consistency Check:** Iterate through all voxels in the volume. For each voxel $v$:
    *   Determine the set of cameras $C_v$ for which $v$ is visible.
        *   **With IB:** $v$ is visible to camera $i$ if $v$ is stored in the IB entry for the pixel corresponding to $v$'s projection in camera $i$ [S1, unofficial].
        *   **With LDI:** $v$ is visible to camera $i$ if $v$ is present in the LDI list for the pixel corresponding to $v$'s projection in camera $i$ [S1, unofficial].
    *   Sample the color of voxel $v$ from the input images of all cameras in $C_v$.
    *   Check for color consistency. If the colors are consistent (e.g., variance below a threshold), mark $v$ as a surface voxel. Otherwise, carve (remove) $v$ [S1, unofficial].
4.  **Refinement (Optional):** Iteratively update visibility and re-check consistency to handle concavities or noise, as the visibility of a voxel may change if neighboring voxels are carved [S1, unofficial].

**Parameters**
*   **Color Consistency Threshold:** Not given in the sources.
*   **Voxel Resolution:** Not given in the sources.

**Edge Cases and Pitfalls**
*   **Memory Usage:** The LDI variant consumes considerably more memory than the IB variant because it stores all occluded voxels, not just the visible one [S1, unofficial].
*   **Calibration:** Requires accurately calibrated cameras. Approximate visibility extensions exist for handling calibration errors, but standard GVC assumes precision [S1, unofficial].
*   **Occlusion:** The exact visibility computation correctly handles complex occlusions that approximate methods might miss, but this increases computational cost [S1, unofficial].

## Subtopics

### Roxels
Roxels is a variant that attempts to reconstruct semi-transparent voxels, departing from the opaque object voxel assumption of standard GVC and other space carving algorithms [S1, unofficial].

### 6D Voxel Coloring
6D Voxel Coloring extends the method to time-varying scenes by linking two time-consecutive 3D voxel spaces together to form a 6D space (3D space + 1D time), allowing for the reconstruction of dynamic scenes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Higher than approximate space carving due to the exact visibility computation. The IB variant is more efficient than LDI as it avoids storing and sorting large lists of occluded voxels [S1, unofficial].
*   **Memory Complexity:** IB uses memory proportional to the number of pixels in the input images. LDI uses memory proportional to the number of voxels projected to each pixel, which can be significantly larger [S1, unofficial].
*   **Strengths:** Produces numerically more consistent reconstructions with input images compared to approximate methods. Handles complex occlusions correctly [S1, unofficial].
*   **Limitations:** Computationally expensive. LDI variant has high memory requirements. Requires accurate camera calibration [S1, unofficial].
*   **When to use:** When high fidelity and color consistency are critical, and computational resources allow for exact visibility checks.
*   **When not to use:** For very large scenes where memory is constrained (prefer IB over LDI, or approximate methods), or when camera calibration is unreliable (use approximate space carving extensions) [S1, unofficial].

## Background
Generalized Voxel Coloring was developed by Culbertson et al. in 1999 [CMS99], as cited in [S1, unofficial]. It was introduced to overcome the limitations of approximate visibility in Space Carving, which often failed to check color consistency over the entire set of visible images for a voxel [S1, unofficial]. The method was later extended with variants like Roxels for transparency [DebVio99] and 6D coloring for dynamic scenes [VBS*00]. Slabaugh also contributed to the field with multiresolution carving using GVC-IB and optimization techniques [Slabaugh00b, SMC00a].

## Sources
[S1] Accelerated volumetric reconstruction from uncalibrated camera views, http://doras.dcu.ie/157/, fetched 2026-10-26 (unofficial)
