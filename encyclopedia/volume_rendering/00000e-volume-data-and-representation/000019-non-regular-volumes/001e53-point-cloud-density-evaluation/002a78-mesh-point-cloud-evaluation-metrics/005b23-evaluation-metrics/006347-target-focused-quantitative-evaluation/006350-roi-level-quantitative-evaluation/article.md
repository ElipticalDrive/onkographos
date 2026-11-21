# ROI-Level Quantitative Evaluation

Also called: ROI-based Evaluation

## Overview
ROI-Level Quantitative Evaluation assesses the reconstruction quality of specific, semantically important regions within a rendered volume or image, rather than evaluating the entire scene uniformly. It addresses the limitation of global metrics by isolating local features—such as vegetation crowns, structural details, or fine surface textures—to verify visibility and fidelity where they matter most. It differs from full-image assessment by focusing on representative target features and from global target-focused evaluation by applying standard perceptual metrics within fixed spatial windows rather than aggregating over logical object masks.

## How it works
The method consists of defining spatial regions of interest and computing standard image quality metrics within those bounds.

**Inputs:**
- Reference image (ground truth).
- Rendered image (prediction).
- A set of ROIs, each defined by a location (top-left coordinate) and dimensions (width, height).

**Outputs:**
- Per-ROI metrics: Peak Signal-to-Noise Ratio (PSNR), Structural Similarity Index (SSIM), and Learned Perceptual Image Patch Similarity (LPIPS).

**Data Structures:**
- ROI Definition: A tuple or object containing `(x, y, w, h)` where `x, y` are the top-left pixel coordinates and `w, h` are the width and height in pixels.

**Steps:**
1. **Define ROIs:** Select representative regions that capture characteristic local structures or textures (e.g., object boundaries, surface deterioration, fine features) [S1].
2. **Extract Patches:** For each ROI $i$, extract the corresponding rectangular patch from the reference image $I_{ref}$ and the rendered image $I_{pred}$ using the defined location and dimensions.
3. **Compute Metrics:** Calculate PSNR, SSIM, and LPIPS for each patch pair $(I_{ref}^{(i)}, I_{pred}^{(i)})$ using the same procedures as full-image assessment [S1].
   - **SSIM:** Calculated using a Gaussian window (e.g., $11 \times 11$ pixels, $\sigma=1.5$), data range of 1.0, and population covariance normalization [S1]. The local error map is often defined as $1 - \text{SSIM}$.
   - **PSNR:** Standard calculation based on Mean Squared Error (MSE) of the patch.
   - **LPIPS:** Perceptual similarity using a learned network, applied to the patch.
4. **Aggregate (Optional):** Results are typically reported per ROI to analyze specific feature reconstruction quality [S1].

**Parameters:**
- ROI Location/Dimensions: Determined by the scene content and the specific features of interest (e.g., vegetation crowns, steel-girder details) [S1]. Not given in the sources as a fixed formula; determined empirically.
- SSIM Window: $11 \times 11$ Gaussian, $\sigma=1.5$ [S1].

**Edge Cases and Pitfalls:**
- **Boundary Effects:** Ensure ROI dimensions do not exceed image boundaries.
- **Subjectivity:** ROI selection is not an unbiased statistical sampling of all image regions but a targeted assessment of visibility and reconstruction quality [S1].

## Subtopics

### Attenuation Metrics
Computes metrics for attenuation simulation, specifically target fraction, tooth gap fraction, and Intersection over Union (IoU). It operates on boolean arrays derived from simulation data.
**Steps:**
1. Convert input data (`target`, `vial`, `tooth_gaps`, `internal_holes`, `tooth_profile`) to boolean arrays [S2, unofficial].
2. Compute `selected_negative` as the bitwise OR of `tooth_gaps` and `internal_holes` [S2, unofficial].
3. Compute `non_target` as the bitwise AND of `vial` and the negation of `target` [S2, unofficial].
4. Compute `predicted` as the bitwise AND of `(field >= threshold)` and `vial` [S2, unofficial].
5. Compute connected labels for `predicted & target` using a $3 \times 3$ ones structure [S2, unofficial].
**Parameters:**
- `field`: `np.ndarray`, the field to evaluate.
- `data`: `dict`, containing `target`, `vial`, `tooth_gaps`, `internal_holes`, `tooth_profile`.
- `threshold`: `float`, threshold for prediction [S2, unofficial].

### Chemical Window Metrics
Computes metrics for chemical window simulation, including target fraction at a required value and gelation status.
**Steps:**
1. Compute `target_fraction_at_or_above_required_percent` as $100.0 \times \text{mean}(\text{field}[\text{target}] \ge \text{required\_value})$ [S2, unofficial].
2. Compute `target_fraction_at_or_above_gelation_percent` as $100.0 \times \text{mean}(\text{field}[\text{target}] \ge 1.0)$ [S2, unofficial].
3. Compute `selected_negative_feature_fraction_above_gelation_percent` as $100.0 \times \text{mean}(\text{field}[\text{negative}] \ge 1.0)$ [S2, unofficial].
**Parameters:**
- `field`: `np.ndarray`, the field.
- `required_value`: `float`, the required value threshold [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the total number of pixels in all ROIs. SSIM and LPIPS computation depends on window size and network depth respectively, but remains local to the ROI.
- **Memory Complexity:** Linear with respect to the size of the ROIs. No global scene storage is required during metric calculation.
- **Strengths:** Provides granular insight into local reconstruction quality; identifies specific failure modes (e.g., loss of fine texture) that global metrics might average out.
- **Limitations:** ROI selection is subjective and not statistically representative of the entire scene [S1]. It does not provide a single scalar score for the whole volume.
- **When to use:** When specific features (e.g., structural integrity, fine details) are critical and global metrics are insufficient to validate their reconstruction.
- **When not to use:** When a single, unbiased statistical summary of overall image quality is required.

## Background
The concept of ROI-level evaluation in the context of 3D Gaussian Splatting and real-world scene reconstruction was detailed in a 2026 study evaluating standard 3D Gaussian Splatting [S1]. The method was used to complement whole-image metrics by assessing local reconstruction quality at representative target features such as vegetation crowns, structural details, and pavement textures. It was applied to scenes including forests, piled-pier undersides, steel-girder bridges, asphalt pavements, and sculptures [S1].

## Sources
[S1] Practical Evaluation of Standard 3D Gaussian Splatting for Real-World Scene Reconstruction., https://doi.org/10.3390/s26185754, fetched 2026-10-26
[S2] EPFL-LAPD/Antagonistic-Dual-Wavelength-Tomographic-Volumetric-Additive-Manufacturing, https://github.com/EPFL-LAPD/Antagonistic-Dual-Wavelength-Tomographic-Volumetric-Additive-Manufacturing, fetched 2026-10-26 (unofficial)
