# Benchmark Performance Numbers

Also called: Average PSNR/LPIPS, M360 Metrics

## Overview
Benchmark Performance Numbers provide standardized quantitative baselines for evaluating radiance field optimization methods, primarily using the Mip-NeRF 360 dataset metrics (average PSNR and LPIPS) [S1, unofficial]. They serve as a reference point to compare the reconstruction quality of different variants (e.g., 3DGS, Mip-Splatting) against established baselines [S1, unofficial]. Unlike raw metric calculation, these numbers aggregate results across a specific scene set to highlight trade-offs between visual fidelity, geometric accuracy, and resource efficiency [S1, unofficial].

## How it works
Inputs: Standardized evaluation datasets (e.g., Mip-NeRF 360 room-scale scenes) and trained model checkpoints [S1, unofficial].
Outputs: A set of scalar metrics (PSNR, LPIPS) and comparative statistics (primitive count, training time, inference speed) [S1, unofficial][S3].

Data Structures:
- Metric Record: A tuple or object containing:
  - `method_name`: String identifier.
  - `psnr`: Float (dB).
  - `lpips`: Float (0-1 range, lower is better).
  - `ssim`: Float (0-1 range, optional) [S3].
  - `training_time`: Float (minutes) [S3].
  - `inference_fps`: Float (frames per second) [S3].
  - `primitive_count`: Integer or relative ratio [S1, unofficial].

Steps:
1. Load the trained model for the method being evaluated [S1, unofficial][S3].
2. Render images for the held-out test views of the benchmark dataset (e.g., Mip-NeRF 360) [S1, unofficial].
3. Compute Per-Pixel Mean Squared Error (MSE) between rendered and ground truth images [S1, unofficial].
4. Convert MSE to PSNR using the formula:
   $$ \text{PSNR} = 10 \cdot \log_{10}\left(\frac{255^2}{\text{MSE}}\right) $$
   where 255 is the maximum pixel value for 8-bit images [S1, unofficial].
5. Compute LPIPS (Learned Perceptual Image Patch Similarity) scores using a pre-trained network (typically VGG or AlexNet) to measure perceptual difference [S1, unofficial].
6. Average PSNR and LPIPS across all test scenes in the benchmark set [S1, unofficial].
7. Record auxiliary metrics: training time, inference FPS, and model size/primitive count if available [S1, unofficial][S3].

Parameters:
- Dataset: Mip-NeRF 360 (room-scale) is the primary reference for these specific numbers [S1, unofficial].
- Metric Weights: Not specified; standard averaging is used [S1, unofficial].

Edge Cases and Pitfalls:
- Overfitting: Train PSNR is easy to overfit and does not reflect generalization; test PSNR on held-out trajectories is the critical metric [S1, unofficial].
- Metric Disagreement: LPIPS often disagrees with PSNR; a model may have lower PSNR but better perceptual quality (lower LPIPS) [S1, unofficial].
- Cherry-picking: Beware of metrics that ignore inference speed, memory footprint, or primitive count inflation [S1, unofficial].
- Failure Modes: Floaters, popping, aliasing, and over-smoothing can distort these numbers if not properly handled by the method [S1, unofficial].

## Subtopics

### Compression Regularization
Applies regularization during training to mitigate artifacts introduced by compressing multilevel hash encodings and MLP architectures [S2]. It uses a strong Laplacian regularizer and a lower learning rate to improve reconstruction quality under compression constraints [S2].

### Sparsity Loss
A regularization term applied during training to force the radiance field representation to be sparser, accelerating rendering speed by reducing the number of active voxels/features evaluated during ray marching [S3]. It uses an occupancy grid bitmap (binary mask: 0 for empty, 1 for non-empty) to visualize and enforce sparsity [S3]. Adopting this loss can accelerate rendering speed (e.g., 23% for Ours-B8) with minor decreases in PSNR (e.g., 33.60 to 33.59) and unchanged SSIM [S3].

## Operating characteristics
Time and Memory Complexity:
- Training Time: Varies by method and sparsity. For example, Ours-B8 training time decreased from 14.7 min to 13.9 min with sparsity loss [S3].
- Inference Speed: Can be significantly improved by sparsity. Ours-B8 inference speed increased from 2.2 fps to 2.7 fps (23% acceleration) with sparsity loss [S3].
- Memory/Size: Compression regularization reduces storage size (e.g., TiNeuVox-B size reduced from 47.51 MB to 3.88 MB with binary feature encoding) [S3].

Measured Performance (Mip-NeRF 360, Room Scale):
- Vanilla 3DGS: 27.4 PSNR / 0.18 LPIPS [S1, unofficial].
- Mip-Splatting: 27.7 PSNR / 0.17 LPIPS [S1, unofficial].
- Scaffold-GS: 27.9 PSNR / 0.18 LPIPS [S1, unofficial].
- 2DGS: ~27.0 PSNR / similar LPIPS (but better geometry) [S1, unofficial].
- GES: Comparable PSNR with ~30% fewer primitives [S1, unofficial].
- Hierarchical 3DGS: Comparable performance with LoD streaming [S1, unofficial].

Strengths and Limitations:
- Strengths: Provides a common ground for comparing diverse methods (Gaussian Splatting, NeRF, Hash-based) [S1, unofficial]. Highlights trade-offs between quality and efficiency (FPS, size) [S1, unofficial][S3].
- Limitations: Metrics can be misleading if cherry-picked; PSNR may not correlate with perceptual quality (LPIPS) [S1, unofficial]. Benchmarks may not cover all failure modes (e.g., dynamic scenes, glossy materials) [S1, unofficial][S2].

When to Use:
- Use for initial comparison of new radiance field methods against SOTA baselines [S1, unofficial].
- Use to evaluate the impact of regularization techniques (sparsity, compression) on efficiency and quality [S2, S3].

When Not to Use:
- Do not rely solely on PSNR for perceptual quality assessment; always check LPIPS and visual inspection [S1, unofficial].
- Do not use for dynamic/deformable scenes unless the benchmark specifically supports it (e.g., D-NeRF, HyperNeRF) [S1, unofficial].

## Background
Origin: The specific numbers for Mip-NeRF 360 room-scale scenes are compiled from various SOTA Gaussian Splatting and radiance field papers, summarized in an unofficial cheat sheet [S1, unofficial]. The concept of using PSNR/LPIPS benchmarks for NeRF/3DGS is standard in the field [S1, unofficial]. Sparsity loss and compression regularization are techniques developed to improve efficiency and reduce artifacts in hash-based and voxel-based radiance fields [S2, S3].

History and Development:
- Early NeRF evaluations relied heavily on PSNR on synthetic datasets (Synthetic-NeRF) [S3].
- Mip-NeRF 360 became a standard for real-world scene evaluation [S1, unofficial].
- 3D Gaussian Splatting (3DGS) and its variants (Mip-Splatting, Scaffold-GS, 2DGS, GES, Hierarchical 3DGS) established new baselines for real-time rendering and quality [S1, unofficial].
- Techniques like Sparsity Loss and Compression Regularization were introduced to address the high memory and computational costs of dense radiance fields [S2, S3].

Where it is Used:
- Academic research for comparing new radiance field optimization methods [S1, unofficial][S2, S3].
- Game development and real-time rendering applications where efficiency and quality trade-offs are critical [S1, unofficial][S2].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Real-time Rendering with a Neural Irradiance Volume, https://doi.org/10.48550/arxiv.2602.12949, fetched 2026-10-26
[S3] Binary Radiance Fields, https://doi.org/10.52202/075280-2438, fetched 2026-10-26
