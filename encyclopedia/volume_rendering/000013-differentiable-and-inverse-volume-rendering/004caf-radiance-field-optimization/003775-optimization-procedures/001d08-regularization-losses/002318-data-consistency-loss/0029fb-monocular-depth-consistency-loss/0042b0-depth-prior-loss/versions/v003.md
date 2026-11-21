# Depth Prior Loss
## Overview
Depth Prior Loss is a regularization term that constrains the geometry of a radiance field by enforcing consistency between rendered depth and monocular depth priors [S1]. It is typically combined with cross-scale geometric adaptation losses (e.g., $L_{geo}$) to leverage their complementarity [S1].

## How it works
**Inputs and Outputs:** Rendered depth map $D_r$ and monocular depth prior $D_m$ for a given view, producing a scalar loss $L_d$ [S1].

**Data Structures:** $D_r$ and $D_m$ are 2D tensors of shape $(H, W)$ representing depth along each ray [S1].

**Steps:**
1. **Alignment:** Compute scale $s$ and shift $t$ parameters that minimize the least-squares error between $D_r$ and $D_m$ using a closed-form solution [G].
2. **Transformation:** Align the monocular prior to the rendered depth scale: $\hat{D}_m = s \cdot D_m + t$ [G].
3. **Loss Calculation:** Compute the mean squared error between the rendered depth and the aligned prior: $L_d = \frac{1}{N} \sum_{i,j} (D_r(i,j) - \hat{D}_m(i,j))^2$, where $N$ is the number of valid pixels [G].

**Formulas:** The closed-form least-squares alignment computes the means $\bar{D}_r$ and $\bar{D}_m$ over valid pixels, then $s = \frac{\sum (D_r - \bar{D}_r)(D_m - \bar{D}_m)}{\sum (D_m - \bar{D}_m)^2}$ and $t = \bar{D}_r - s \bar{D}_m$ [G].

**Parameters:** Scale and shift are computed per image or per batch to handle scale ambiguity [G].

**Edge Cases and Pitfalls:** Using $L_d$ in isolation initially offers good performance but later suffers from higher error due to scale mismatches between the depth prior and voxel resolutions [S1].

## Subtopics

### [Relative Depth Loss](12871-relative-depth-loss.md)
A regularization term that aligns rendered depth with monocular or MVS depth priors using a relative formulation ($D_{mvs}/D_r$), which exhibits high sensitivity to errors at closer depths. It is distinguished from its parent by rejecting erroneous MVS values via thresholding and weighting by rendering certainty. Choose it when relative geometric structure is more critical than absolute scale.

## Operating characteristics
* **Strengths:** Initially offers good performance due to strong geometric constraints [S1].
* **Limitations:** Suffers from higher error in later training stages due to scale mismatches between the depth prior and voxel resolutions when used in isolation [S1].
* **Performance:** Combining $L_d$ with $L_{geo}$ yields better PSNR, SSIM, and LPIPS scores than using either loss alone, with comparable model size and training time (~10 minutes for FrugalNeRF) [S1].
* **When to use:** Use as part of a combined regularization strategy with cross-scale geometric adaptation losses to leverage fast initial convergence while mitigating long-term scale drift [S1].

## Background
It is used in few-shot NeRF methods, specifically FrugalNeRF, to constrain geometry during training using monocular depth priors [S1]. The method addresses high-quality novel view synthesis with limited training views by providing an initial geometric constraint that complements cross-scale geometric adaptation [S1].

## Sources
[S1] FrugalNeRF: Fast Convergence for Extreme Few-shot Novel View Synthesis without Learned Priors, https://doi.org/10.1109/cvpr52734.2025.01049, fetched 2026-10-29
