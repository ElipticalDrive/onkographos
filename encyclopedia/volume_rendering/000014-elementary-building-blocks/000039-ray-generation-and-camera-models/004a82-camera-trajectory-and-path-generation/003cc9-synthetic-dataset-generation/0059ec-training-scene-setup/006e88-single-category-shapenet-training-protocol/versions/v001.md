# Single-category ShapeNet Training Protocol

Also called: ShapeNet Training Protocol

## Overview
This protocol defines the training and evaluation configuration for single-category ShapeNet datasets, specifically chairs and cars, within the context of few-shot neural rendering. It addresses the problem of optimizing a neural radiance field from one or two input views by structuring the training schedule to balance convergence speed and generalization. It differs from the category-agnostic protocol by focusing on specific object classes and using fixed evaluation views (64 and 128) rather than random sampling for baseline comparisons [S1].

## How it works
The method consists of a two-phase training schedule and a fixed evaluation protocol.

**Inputs and Outputs:**
- **Input:** A set of training images for a specific ShapeNet category (e.g., chairs or cars).
- **Output:** A trained neural radiance field model capable of rendering novel views from 1 or 2 input images.

**Data Structures:**
- **Training Set:** Images associated with a specific ShapeNet category.
- **Evaluation Views:** Fixed indices from the camera trajectory. View 64 is used for single-view evaluation; Views 64 and 128 are used for two-view evaluation [S1].

**Parameters:**
- **Total Iterations:** 400,000 [S1].
- **Phase 1 Duration:** First 300,000 iterations [S1].
- **Phase 2 Duration:** Remaining 100,000 iterations [S1].
- **Bounding Box:** A tight bounding box around the object, active only in Phase 1 [S1].

**Steps:**
1. **Initialize** the neural radiance field model.
2. **Phase 1 (Iterations 1–300,000):**
   - Sample rays strictly within the tight bounding box surrounding the object [S1].
   - Use exactly 2 input views for encoding [S1].
   - Update model parameters for 300,000 iterations [S1].
3. **Phase 2 (Iterations 300,001–400,000):**
   - Remove the bounding box constraint; sample rays from the full scene volume to avoid background artifacts [S1].
   - Randomly choose to use either 1 or 2 input views for each training step to encourage robustness to varying input counts [S1].
   - Update model parameters for the remaining 100,000 iterations [S1].
4. **Evaluation:**
   - For single-view evaluation, use View 64 as the sole input [S1].
   - For two-view evaluation, use Views 64 and 128 as inputs [S1].

**Edge Cases and Pitfalls:**
- **Car Dataset Bug:** The original SRN car dataset contained a camera sampling bug. An updated car dataset provided by the SRN author must be used for training to ensure valid results [S1].
- **Background Artifacts:** Failing to remove the bounding box after 300,000 iterations may result in background artifacts in the rendered output [S1].

## Subtopics

### SRN Baseline Training Protocol
This variant adapts the Single-category ShapeNet Protocol for the SRN baseline, requiring test-time training (latent inversion) to generate results. For ShapeNet chairs, the model is trained for 1 million iterations, followed by latent inversion for 170,000 iterations for both 1-view and 2-view cases [S1]. For ShapeNet cars, the model is trained for 400,000 iterations, followed by latent inversion for 100,000 iterations for each of the 1-view and 2-view cases [S1]. Note that quantitative results for cars may be slightly lower than those reported in prior works due to the dataset bug correction, but are substantially higher than the original SRN paper's results which used the bugged renderings [S1].

### Category-agnostic ShapeNet Training Protocol
This variant trains on the entire ShapeNet training set rather than a single category. The model is trained for 800,000 iterations, with rays sampled from within a tight bounding box for the first 400,000 iterations [S1]. For evaluation, a random input view is sampled for each object to encode, and this same view is used for all baseline comparisons to ensure fairness [S1]. For the SRN baseline in this setting, training lasts 1 million iterations with latent inversion for 260,000 iterations [S1].

## Operating characteristics
- **Time Complexity:** Training the single-category protocol takes roughly 6 days on a single Titan RTX GPU [S1]. The category-agnostic protocol takes about 6 days on an RTX 2080Ti [S1]. SRN baseline training is significantly longer, taking about 14 days on a T-class GPU for the category-agnostic case [S1].
- **Memory Complexity:** Not given in the sources.
- **Strengths:** The two-phase training (bounding box then full volume) accelerates initial convergence while ensuring final quality. The random view selection in Phase 2 improves robustness to input count variations.
- **Limitations:** The protocol is specific to ShapeNet categories. The car dataset results are affected by the historical bug in the original dataset, requiring careful dataset selection [S1].

## Background
The protocol originates from the pixelNeRF paper, which introduced neural radiance fields from one or few images [S1]. It was developed to establish a consistent evaluation benchmark for few-shot novel view synthesis on the ShapeNet dataset, specifically addressing the challenges of training stability and fair comparison with baselines like SRN, DVR, and Soft-Ras [S1].

## Sources
[S1] pixelNeRF: Neural Radiance Fields from One or Few Images, https://doi.org/10.1109/cvpr46437.2021.00455, fetched 2026-10-26
