# Volumetric Representation via Depth Map Registration

Also called: Sparse Multi-View Volumetric Representation, Sparse Non-Lambertian Multi-view Volume Construction, Volume Initialization

## Overview
This method constructs a 3D volumetric representation from a sparse set of calibrated multi-view images of a non-Lambertian scene by registering estimated depth maps into a voxel grid [S1, S2]. It solves the problem of representing complex geometry with specular or non-Lambertian reflections, where global depth thresholds fail, by using individual minimum cost per view for depth estimation and aggregating voxel occupancy counts [S1]. It distinguishes itself from static voxelization methods by incorporating an iterative refinement loop that removes isolated noise voxels and smooths sharp corners based on color consistency with original reference views, ensuring the volume accurately supports virtual view synthesis [S1, S2].

## How it works
**Inputs:** A set of $N$ calibrated reference views (images and camera parameters) of a non-Lambertian scene.
**Outputs:** A sparse 3D voxel grid where each active voxel stores a depth value, a view-count weight, and a color.

**Data Structures:**
*   **Voxel Grid:** A 3D array or sparse hash map indexed by $(i, j, k)$.
*   **Voxel Entry:**
    *   `weight` ($\delta_{ijk}$): Integer count of views mapping to this voxel.
    *   `depth` ($d$): The registered depth value for the voxel.
    *   `color` ($C$): The assigned color for the voxel.
    *   `active`: Boolean flag indicating if the voxel is occupied.

**Algorithm Steps:**

1.  **Depth Estimation:** For each reference view, estimate a depth map by searching along the view's geometry. Use an individual minimum cost function per view to handle non-Lambertian reflections, avoiding global thresholds [S1].

2.  **Volume Initialization (Registration):** Register the depth maps into the 3D volume. For each voxel at location $(i, j, k)$, determine if any depth map projects to it.
    *   If a depth is registered, set the voxel weight $\delta_{ijk} = n$, where $n$ is the number of views mapping to that voxel ($n \in \{1, \dots, N\}$) [S1, S2].
    *   If no depth maps to the voxel, set $\delta_{ijk} = 0$ [S1, S2].
    $$ \delta_{ijk} = \begin{cases} n & \text{if registering depth} \\ 0 & \text{otherwise} \end{cases} $$

3.  **Noise Removal and Depth Aggregation:**
    *   Detect and remove isolated voxels that have no neighbors, as these are likely errors [S2].
    *   Project the remaining volume back to the reference view geometry to build new depth maps. This step incorporates depth information from other views to correct errors [S1, S2].
    *   If a pixel in the reference view has multiple projected depth values (a "front cluster" of depth $K$), compute the proper depth $d$ for that pixel:
        *   If only one depth value $d_n$ has the maximum weight $\delta_n = \max(\delta)$, select $d = d_n$ [S1, S2].
        *   Otherwise, compute a weighted average of the front cluster depths:
        $$ d = \frac{\sum_{k=1}^{K} \delta_k d_k}{\sum_{k=1}^{K} \delta_k} $$
        where $d_k$ is the depth value and $\delta_k$ is the voxel weight for the $k$-th depth in the cluster [S1, S2].

4.  **Volume Reconstruction and Interpolation:** Reconstruct the volume using the aggregated depths. Interpolate voxels to connect the surface among near voxels, addressing resolution differences between reference views [S1, S2].

5.  **Iterative Volume Refinement:** Refine the shape of the volume to smooth sharp corners and remove scattering voxels [S1, S2].
    *   **Temporary Coloring:** Assign temporary colors to voxels using the median of color values from all visible views. The median is used because it is less sensitive to extreme values than the mean [S2].
    *   **Error Calculation:** Iterate through reference views (e.g., left to right). For each view, calculate the Mean Square Error (MSE) between the reconstructed view (using current voxel colors) and the original reference image. Use a window-based computation (e.g., $3 \times 3$ pixels) to account for imperfect calibration and resolution mismatches [S2].
    *   **Depth Replacement:** If the MSE exceeds a threshold:
        *   Remove the old depth and its associated voxel [S2].
        *   Investigate a new depth derived from neighboring pixels, assuming a smooth surface [S2].
        *   If the new depth satisfies color constraints for all visible views, generate the new associated voxel [S2].
    *   **Re-evaluation:** If voxels are removed or generated, iteratively reconstruct the images again, as voxel position changes affect visibility (making previously invisible points visible and vice versa) [S2].

6.  **Color Selection:** Post-process colors for each voxel to handle non-Lambertian reflectance.
    *   For two views 1 and 2, blend colors $C_1$ and $C_2$ based on angles $\alpha_1$ and $\alpha_2$ between the voxel $x$ and points on lines $L_1$ and $L_2$ respectively:
        $$ C = \frac{\alpha_1 C_2 + \alpha_2 C_1}{\alpha_1 + \alpha_2} $$
        if the condition is met; otherwise $C = C_1$ [S2].
    *   **Blur Handling:** If the difference between $C_1$ and $C_2$ is significant, the average method causes blur. Classify voxels into "blur" or "clear" groups. For voxels in the blur group, replace the color with that of a neighboring voxel from the clear group that has the closest color value [S2].

**Parameters:**
*   **Threshold:** MSE threshold for volume refinement. Not given in the sources.
*   **Window Size:** Size of the window for error calculation (e.g., $3 \times 3$ pixels) [S2].

**Edge Cases and Pitfalls:**
*   **Non-Lambertian Reflections:** Standard global depth thresholds fail; individual view minimum cost is required [S1].
*   **Calibration Errors:** Imperfect camera calibration and unequal resolutions cause pixel displacement. Window-based MSE calculation mitigates this [S2].
*   **Color Blur:** Averaging colors from views with significantly different intensities/components causes object blur. The blur/clear group classification prevents this [S2].
*   **Visibility Changes:** Removing or adding voxels changes the visibility of other points, requiring iterative reconstruction [S2].

## Subtopics

### [Iterative Volume Refinement](6473-iterative-volume-refinement.md)
A post-initialization algorithm that cleans and smooths a sparse voxel cloud by removing scattering voxels and smoothing sharp corners to increase coding gain. It distinguishes itself by dynamically regenerating voxels based on a smooth surface assumption and iteratively re-evaluating visibility.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The iterative refinement step involves repeated reconstruction and error calculation, suggesting high computational cost relative to static voxelization.
*   **Memory Complexity:** Sparse voxel cloud. Memory usage depends on the number of registered voxels and the number of reference views $N$.
*   **Strengths:** Handles non-Lambertian scenes effectively; produces smooth, realistic synthesized views; robust to calibration errors via window-based error calculation.
*   **Limitations:** Computationally intensive due to iterative refinement; requires calibrated multi-view input; color averaging can cause blur if not handled.
*   **When to use:** When synthesizing views from a sparse set of calibrated cameras for non-Lambertian scenes where surface smoothness and coding gain are important.
*   **When not to use:** For dense, Lambertian scenes where simpler projection methods suffice; when real-time performance is critical and iterative refinement is too slow.

## Background
This method was proposed by N. Anantrasirichai, C. Nishan Canagarajah, David W. Redmill, and David R. Bull from the University of Bristol [S1, S2]. It was introduced in 2006 for sparse multi-view image synthesis and further detailed in 2010 for color volumetric compression [S1, S2]. It is used in 3D visual environments for communication and computer vision, specifically for virtual view synthesis of non-Lambertian scenes [S2].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
[S2] Volumetric Representation for Sparse Multi-Views, https://doi.org/10.1109/icip.2006.312545, fetched 2026-10-26
