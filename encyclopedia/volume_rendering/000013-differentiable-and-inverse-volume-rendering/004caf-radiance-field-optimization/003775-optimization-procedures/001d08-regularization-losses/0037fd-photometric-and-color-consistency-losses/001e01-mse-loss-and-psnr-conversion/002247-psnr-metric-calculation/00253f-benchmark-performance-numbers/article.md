# Benchmark Performance Numbers

Also called: Average PSNR/LPIPS, M360 Metrics

## Overview
Benchmark Performance Numbers provide standardized quantitative comparisons of novel view synthesis methods, primarily using the Mip-NeRF 360 dataset's room-scale scenes. They report average Peak Signal-to-Noise Ratio (PSNR) and Learned Perceptual Image Patch Similarity (LPIPS) to evaluate photometric accuracy and perceptual quality. Unlike raw metric calculation, this method contextualizes scores against specific baselines (e.g., Vanilla 3DGS, Mip-Splatting) to highlight trade-offs between visual fidelity, geometric correctness, and computational efficiency.

## How it works
The method aggregates evaluation metrics across a fixed set of test scenes (typically the Mip-NeRF 360 room-scale subset) and compares them against established baseline implementations.

**Inputs:**
- Rendered images from a novel view synthesis method.
- Ground truth images from the Mip-NeRF 360 dataset.
- Reference baseline scores (e.g., Vanilla 3DGS, Mip-Splatting) for context.

**Outputs:**
- Average PSNR (dB) and LPIPS (0–1) across the test set.
- Comparative notes on geometric quality, primitive count, or memory usage if available.

**Data Structures:**
- A table mapping method names to their respective PSNR and LPIPS scores.
- Metadata fields for scene scale (e.g., "room scale"), metric type, and any additional qualitative assessments (e.g., "better geometry").

**Steps:**
1. Render novel views for each scene in the Mip-NeRF 360 room-scale subset using the target method.
2. Compute per-pixel PSNR and LPIPS between rendered and ground truth images.
3. Average the metrics across all scenes and pixels to obtain the final Average PSNR and LPIPS.
4. Compare the results against reference baselines (e.g., Vanilla 3DGS: 27.4 PSNR / 0.18 LPIPS; Mip-Splatting: 27.7 PSNR / 0.17 LPIPS; Scaffold-GS: 27.9 PSNR / 0.18 LPIPS) [S1, unofficial].
5. Document additional qualitative factors such as geometric accuracy, primitive count, or inference speed if they deviate from standard photometric metrics [S1, unofficial].

**Parameters:**
- Dataset: Mip-NeRF 360 (room-scale scenes) [S1, unofficial].
- Metrics: PSNR (higher is better), LPIPS (lower is better) [S1, unofficial].

**Edge Cases and Pitfalls:**
- Overfitting to training data can inflate PSNR; test PSNR on held-out trajectories is critical [S1, unofficial].
- LPIPS may disagree with PSNR; a model with lower PSNR might appear perceptually better [S1, unofficial].
- Cherry-picked metrics or inflated Gaussian counts (e.g., 3× increase) can misrepresent efficiency [S1, unofficial].
- Floaters, popping artifacts, and aliasing can degrade perceived quality despite high PSNR [S1, unofficial].

## Subtopics

### Compression Regularization
Applies regularization during training to mitigate artifacts introduced by compressing multilevel hash encodings and MLP architectures. Uses a strong Laplacian regularizer and a lower learning rate to improve reconstruction quality [S2].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** On Mip-NeRF 360 room-scale scenes, Vanilla 3DGS achieves 27.4 PSNR / 0.18 LPIPS, Mip-Splatting achieves 27.7 PSNR / 0.17 LPIPS, and Scaffold-GS achieves 27.9 PSNR / 0.18 LPIPS [S1, unofficial]. 2DGS achieves approximately 27.0 PSNR with similar LPIPS but better geometry [S1, unofficial]. GES achieves comparable PSNR with ~30% fewer primitives [S1, unofficial]. Hierarchical 3DGS achieves comparable performance with LoD streaming [S1, unofficial].
- **Strengths:** Provides a standardized, comparable metric for novel view synthesis methods. Highlights trade-offs between photometric accuracy and geometric quality.
- **Limitations:** PSNR and LPIPS may not fully capture perceptual quality or geometric fidelity. Results can be sensitive to implementation details and hyperparameters.
- **When to Use:** When comparing novel view synthesis methods on standard datasets like Mip-NeRF 360.
- **When Not to Use:** When evaluating methods on non-standard datasets or when geometric fidelity is the primary concern over photometric accuracy.

## Background
Benchmark Performance Numbers are widely used in the computer graphics and vision communities to evaluate novel view synthesis methods. The Mip-NeRF 360 dataset has become a standard benchmark for room-scale scenes. Recent works like 3D Gaussian Splatting and its variants have established new baselines for these metrics [S1, unofficial]. Compression regularization techniques have also been introduced to address artifacts in compressed neural representations [S2].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Real-time Rendering with a Neural Irradiance Volume, https://doi.org/10.48550/arxiv.2602.12949, fetched 2026-10-26
