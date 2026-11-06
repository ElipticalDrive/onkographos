# Stop-the-pop
## Overview
Stop-the-pop is a technique to mitigate popping artifacts in volume splatting that are caused by the sorting procedure of Gaussians along the ray [S1][S4, unofficial]. It uses hierarchical or pixel-resorting to remove popping artifacts without sacrificing speed [S2, unofficial][S3, unofficial]. It is distinct from methods that modify opacity compensation terms [S1][S3, unofficial] or use ray-tracing-style intersections for accurate depth [S2, unofficial].
## How it works
Inputs: A set of 3D Gaussians (position, covariance, color, opacity) and a camera view [G].
Outputs: A rendered image with reduced popping artifacts [G].

Data Structures:
- Gaussian List: Array of Gaussian primitives [G].
- Pixel Bucket: For each pixel, a list of Gaussians sorted by depth [G].
- Hierarchical Structure: A structure allowing fast re-sorting of Gaussians within a local region (pixel or tile) [S2, unofficial].

Steps:
1. Project all 3D Gaussians to 2D screen space to determine their footprint and depth [G].
2. For each pixel (or tile), identify the Gaussians whose footprints cover it [G].
3. Sort these Gaussians by depth (front-to-back) [G].
4. Apply the Stop-the-pop algorithm:
   - Detect potential popping events: These occur when the relative depth order of Gaussians changes significantly between frames or when a Gaussian enters/leaves the pixel's view, causing abrupt changes in the accumulated color/opacity [S1].
   - Hierarchical/Pixel-resorting: Instead of a global sort, perform a localized sort or use a hierarchical structure to maintain a consistent depth ordering for Gaussians contributing to the same pixel or small region [S2, unofficial]. The method groups Gaussians into coarse depth bins and sorts only within the bin that covers the depth discontinuities [S2, unofficial].
   - The specific implementation details of the hierarchical structure (e.g., k-d tree, grid) are not provided in the sources [S2, unofficial]. The core idea is to prevent the "pop" caused by the discrete nature of the sorting process [S1].
5. Composite the sorted Gaussians using alpha blending [G].

Parameters:
- Sorting threshold: Not given in the sources.
- Hierarchical level: Not given in the sources.

Edge cases:
- Degenerate input: If Gaussians have identical depths, the sort order may be unstable [G].
- The hierarchical approach helps mitigate this by maintaining a consistent local order [S2, unofficial].
- Precision: Floating-point errors in depth calculation can cause sorting instability [G].
- The method aims to reduce the visual impact of these errors [S1].

## Operating characteristics
Time complexity: The method removes popping artifacts without sacrificing speed, maintaining the baseline sorting complexity [S2, unofficial][S3, unofficial].
Memory complexity: Additional memory is required for the hierarchical structure [S2, unofficial].
Performance: It removes popping artifacts without sacrificing speed [S2, unofficial].
Strengths: Reduces visual artifacts (popping) caused by sorting [S1][S3, unofficial].
Limitations: Not given in the sources.

## Background
Origin: Stop-the-pop (StopThePop) was introduced by Radl et al. at SIGGRAPH 2024 [S2, unofficial][S3, unofficial].
History: It addresses issues in 3D Gaussian Splatting where the sorting procedure can cause popping artifacts [S1][S3, unofficial].
Usage: It is used in 3D Gaussian Splatting pipelines to improve visual quality [S2, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S4] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
