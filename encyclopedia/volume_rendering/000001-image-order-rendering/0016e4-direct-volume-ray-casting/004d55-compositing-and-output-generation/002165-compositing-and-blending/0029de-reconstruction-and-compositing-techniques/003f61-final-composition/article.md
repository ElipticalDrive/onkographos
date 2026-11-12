# Final Composition

Also called: Density-dependent interpolation, Segment compositing, Volume imaging, Volumetric Compositing

## Overview
Final Composition is a volume rendering method that integrates information from the entire volume into a rendered image using lighting to depict organ surfaces, rather than performing exact boundary determination [S1, unofficial]. It distinguishes itself from segmentation-based approaches by allowing intermediate voxel values to define regions, making it suitable for visualizing structures where precise edges are ambiguous or unnecessary [S1, unofficial]. It is primarily used in medical imaging, such as SPECT, to portray organ surfaces while being heavily dependent on chosen intensity ranges [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (voxel values), a set of intensity ranges defining specific organs or regions, and lighting parameters.
*   **Output:** A 2D rendered image where organ surfaces are depicted via integration and lighting.

**Data Structures:**
*   **Volume:** 3D array of voxel intensities.
*   **Transfer Function:** Maps intensity ranges to optical properties (opacity $\alpha$, color $c$). In this method, ranges define organs with allowance for intermediate values [S1, unofficial].

**Steps:**
1.  **Define Intensity Ranges:** Select intensity ranges to define organs. Intermediate values are allowed, meaning the transition between regions is not binary [S1, unofficial].
2.  **Sample the Ray:** For each pixel, cast a ray through the volume.
3.  **Evaluate Optical Properties:** For each sample point along the ray, determine opacity $\alpha_i$ and color $c_i$ based on the voxel value and the defined intensity ranges.
4.  **Integrate with Lighting:** Integrate the volume along the ray. Unlike standard front-to-back compositing which often assumes uniform lighting or simple attenuation, this method incorporates lighting information to depict surfaces [S1, unofficial]. The standard compositing equation is used as the base integration model:
    $$
    C_{acc}^{(i)} = C_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i C_i
    $$
    $$
    \alpha_{acc}^{(i)} = \alpha_{acc}^{(i-1)} + (1 - \alpha_{acc}^{(i-1)}) \alpha_i
    $$
    where $C_{acc}$ and $\alpha_{acc}$ are the accumulated color and opacity [S12, S43, S50, unofficial].
5.  **Apply Lighting:** Modify the contribution of each sample $c_i$ using lighting calculations (e.g., shading) to enhance the perception of surfaces [S1, unofficial].
6.  **Composite:** Accumulate the lit samples from front to back.
7.  **Background Composite:** Composite the final accumulated color over a background color $C_{bg}$:
    $$
    C_{final} = C_{acc} + (1 - \alpha_{acc}) C_{bg}
    $$

**Parameters:**
*   **Intensity Ranges:** User-defined or automatically selected thresholds that define the organs. These are critical as the result is heavily dependent on them [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Intensity Range Dependence:** The method is heavily dependent on the chosen intensity ranges. Incorrect ranges can lead to inaccurate portrayal of organs [S1, unofficial].
*   **Surface-Only Portrayal:** Only organ surfaces are portrayed, not the interior. This is a limitation if interior structure visualization is required [S1, unofficial].
*   **Aliasing:** In related implementations involving convolution solutions (e.g., [S2, unofficial]), density-dependent interpolation between low-density and high-density solutions is used to eliminate aliasing artifacts. While not explicitly detailed in [S1, unofficial], this suggests that abrupt changes in density ranges can cause visual artifacts.

## Operating characteristics
*   **Strengths:** Incorporates information from the entire volume, avoiding the need for exact segmentation. Effective for depicting organ surfaces in medical imaging (e.g., SPECT) [S1, unofficial].
*   **Limitations:** Heavily dependent on user-selected intensity ranges. Does not portray interior details, only surfaces [S1, unofficial].
*   **Performance:** Not explicitly quantified in the provided sources. Standard ray casting complexity applies, with additional cost for lighting calculations.

## Background
Final Composition, also referred to as Volume Imaging or Volumetric Compositing, was developed as an alternative to methods that require exact boundary determination or segmentation [S1, unofficial]. It is particularly noted in the context of three-dimensional display of SPECT (Single Photon Emission Computed Tomography) images [S1, unofficial]. The method allows for the integration of the entire volume's information, using lighting to enhance surface perception, addressing limitations of earlier techniques that relied on arbitrary thresholds or strict segmentation [S1, unofficial].

## Sources
[S1] "Volume rendering in three-dimensional display of SPECT images", http://jnm.snmjournals.org/cgi/pmidlookup?view=long&pmid=2384811, fetched 2026-10-29 (unofficial)
[S2] Diffusion and multiple anisotropic scattering for global illumination in clouds, https://digital.library.unt.edu/ark:/67531/metadc1410406/?q=multiple scattering rendering, fetched 2026-10-29 (unofficial)
