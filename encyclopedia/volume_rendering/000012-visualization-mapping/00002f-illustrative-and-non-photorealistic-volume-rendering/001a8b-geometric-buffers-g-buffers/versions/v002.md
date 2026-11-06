# Geometric Buffers (G-buffers)
Also called: 2D Image Processing for NPR, G-buffer approach, Intermediate geometric buffers, Pixel-Based Geometric Buffer Approach

## Overview
Geometric Buffers (G-buffers) is a Non-Photorealistic Rendering (NPR) technique that decouples geometric analysis from shading by storing per-pixel geometric properties (e.g., depth, normals) in intermediate 2D images [S1, unofficial]. It solves the problem of efficiently generating illustrative effects for complex 3D volumes by leveraging fast 2D image processing algorithms instead of expensive 3D analytic computations [S1, unofficial]. Unlike sibling methods that may compute shading or illumination directly in 3D space, G-buffers provide hidden surface elimination for free and allow NPR parameters to be tweaked interactively without re-traversing the volume [S1, unofficial]. Builds on [Illustrative and non-photorealistic volume rendering](40-illustrative-and-non-photorealistic-volume-rendering.md).

## How it works
**Inputs:** A 3D volume dataset, camera position, and camera orientation [S1, unofficial].
**Outputs:** A final 2D NPR image [S1, unofficial].
**Data Structures:**
*   **G-buffers:** A set of 2D images (arrays) corresponding to the screen resolution [S1, unofficial]. Each buffer stores a single geometric property for every pixel [S1, unofficial]. Common buffers include:
    *   **Depth Buffer:** Stores the distance from the camera to the visible surface along the view ray [S1, unofficial].
    *   **Normal Buffer:** Stores the surface normal vector $(n_x, n_y, n_z)$ of the visible surface at each pixel [S1, unofficial].
**Steps:**
1.  **Single Volume Traversal:** Traverse the 3D volume once from the camera's perspective to project the geometric scene onto a viewplane [S1, unofficial]. For each pixel, determine the visible surface intersection, implicitly removing hidden surfaces [S1, unofficial].
2.  **Buffer Construction:** For each pixel where a surface is visible, compute and store the relevant geometric properties into the corresponding G-buffers [S1, unofficial].
    *   *Depth:* Store the ray-surface intersection distance [S1, unofficial].
    *   *Normal:* Store the surface normal vector encapsulated during the projection phase [S1, unofficial].
3.  **2D Image Processing:** Apply various 2D image processing techniques to the G-buffers to achieve different NPR effects [S1, unofficial].
    *   *Edge/Profile Extraction:* Extract features such as profiles and internal edges from the depth map using a first-order differential operator [S1, unofficial].
4.  **Compositing:** Apply image processing operators to the G-buffers to obtain the final rendering [S1, unofficial].
5.  **Reuse:** If NPR parameters change but the scene geometry and camera position do not change, skip the traversal and reuse the existing G-buffers to avoid re-traversal [S1, unofficial].
**Formulas:**
*   **Edge Detection:** A first-order differential operator $D$ applied to the depth buffer $D_{buf}$ yields the edge map $E$:
    $$ E = |D(D_{buf})| $$
    Pixels where $E$ exceeds a threshold are marked as edges [S1, unofficial].
**Parameters:**
*   **Thresholds:** Used for edge detection in G-buffers. Determining the most suitable combination of enhancement techniques depends on the scene, purpose, and viewer's preferences, requiring trial and error [S1, unofficial].
*   **Reconstruction Resolution:** High-resolution illustrations communicate more information, but source volume models are often smaller than final 2D images, necessitating consideration of reconstruction and interpolation issues [S1, unofficial]. Variable Gaussian blurring reduces visible artifacts from the Nyquist limit [S1, unofficial].
**Edge Cases and Pitfalls:**
*   **Aliasing:** Sharp surface borders may exhibit aliasing artifacts and digitization problems due to pixel discretization [S1, unofficial].
*   **Transparency/Reflection:** The basic method assumes all objects are opaque and cannot natively enhance reflected or transparent images without extensions [S1, unofficial].
*   **Parameter Tuning:** There is no automatic way to determine the optimal combination of enhancement techniques; it requires manual trial and error based on scene context and viewer preference [S1, unofficial].
*   **Resolution Mismatch:** If the source volume is significantly smaller than the target image, reconstruction artifacts appear due to the discrete sampled nature of the original volume models [S1, unofficial].

## Subtopics

### Reconstruction and Interpolation
Addresses the mismatch between low-resolution volume data and high-resolution NPR output. It is chosen when source volumes are significantly smaller than the target image to mitigate Nyquist limit artifacts. See [Reconstruction and Interpolation](link).

### Extensions for Transparency
Modifies the opaque assumption to handle semi-transparent or reflective surfaces. It is chosen when the scene requires rendering of non-opaque objects. See [Extensions for Transparency](link).

## Operating characteristics
*   **Time Complexity:** Dominated by the single 3D volume traversal [S1, unofficial]. Subsequent NPR effect generation is fast since much of the processing is done in 2D rather than 3D [S1, unofficial].
*   **Memory Complexity:** Scales with the number of pixels and geometric properties stored per pixel [S1, unofficial].
*   **Performance:** Fast enough for interactive user experimentation [S1, unofficial]. Pixel-based techniques handle large or complex datasets better than analytic alternatives [S1, unofficial].
*   **Strengths:**
    *   Reusability: G-buffers can be reused without recalculation if scene geometry and camera position do not change [S1, unofficial].
    *   Hidden Surface Elimination: Provided automatically by the traversal and buffering process [S1, unofficial].
    *   Scalability: Handles large or complex datasets better than analytic alternatives [S1, unofficial].
*   **Limitations:**
    *   Assumes opacity without extensions [S1, unofficial].
    *   Susceptible to aliasing and digitization artifacts [S1, unofficial].
    *   Subjective tuning required for comprehensibility [S1, unofficial].

## Background
The G-buffer approach was proposed by Saito and Takahashi [ST90] to enable line generation and NPR effects [S1, unofficial]. The authors claim these techniques help make hand-drawn illustrations more comprehensible, and apply them to various digital domains like edge enhancement, line drawing illustrations, topographical maps, medical imaging, and surface analysis [S1, unofficial]. The method is a foundational technique in pixel-based NPR for volume visualization, allowing for interactive user experimentation by decoupling geometry from style [S1, unofficial].

## Sources
[S1] Volume Visualisation Via Variable-Detail Non-Photorealistic Illustration, https://uwspace.uwaterloo.ca/bitstream/10012/1104/1/jlmckinl2002.pdf, fetched 2026-10-26 (unofficial)
