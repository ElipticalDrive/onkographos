# AnySplat

Also called: Pose-free feed-forward 3D Gaussian Splatting

## Overview
AnySplat is a pose-free feed-forward network that reconstructs 3D Gaussian Splatting scenes from uncalibrated image collections without requiring camera pose annotations [S1, unofficial]. It predicts 3D Gaussian primitives and per-image intrinsics and extrinsics in a single forward pass [S1, unofficial]. Unlike sibling methods such as [InstantSplat](6136-instantsplat.md) and [pixelSplat](6464-pixelsplat.md), AnySplat scales to casual dense or sparse multi-view captures, matches pose-aware baselines, and surpasses prior pose-free methods like NoPoSplat in zero-shot evaluations at real-time rendering latency [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A collection of uncalibrated images (sparse or dense) [S1, unofficial].
- **Output:** 3D Gaussian primitives (position, covariance, opacity, spherical harmonics) and per-image camera intrinsics and extrinsics [S1, unofficial].

**Data Structures:**
The network leverages MASt3R or VGGT geometry priors to process the input image collection [S1, unofficial]. The output consists of standard 3D Gaussian Splatting primitives and camera parameters [S1, unofficial]. Specific internal feature map dimensions or token layouts are not given in the sources [S1, unofficial].

**Method Steps:**
1. **Input Ingestion:** The network receives a set of uncalibrated images [S1, unofficial].
2. **Geometry & Feature Extraction:** MASt3R or VGGT geometry priors extract geometric cues and features from the input images [S1, unofficial].
3. **Joint Prediction:** A single forward pass simultaneously predicts 3D Gaussian primitives and per-image intrinsics and extrinsics [S1, unofficial].
4. **Differentiable Rendering:** The predicted Gaussians are rendered to images using the predicted poses, and gradients flow back to update network weights and Gaussian parameters [G].

**Formulas:**
Not given in the sources.

**Parameters:**
A vanilla 3DGS scene averages 1–2 M Gaussians and 200–800 MB on disk [S1, unofficial]. Network hyperparameters are not given in the sources [S1, unofficial].

**Edge Cases and Pitfalls:**
The method handles uncalibrated inputs, avoiding pitfalls associated with failed explicit pose estimation [S1, unofficial]. Precision issues related to feed-forward pose and Gaussian prediction are not given in the sources [S1, unofficial].

## Subtopics

### [FreeSplat](15163-freesplat.md)
FreeSplat is a feed-forward framework that reconstructs geometrically consistent global 3D Gaussian Splatting from long, sparse image sequences to support free-view synthesis across wide view ranges. It differs from pixelSplat and MVSplat, which are limited to narrow-range interpolation due to heavy transformer backbones, by using pure CNN-based backbones and a Pixel-aligned Triplet Fusion (PTF) module to handle arbitrary view counts and wide baselines.

### [ZipSplat](17331-zipsplat.md)
ZipSplat is a feed-forward method that reconstructs scenes from unposed, uncalibrated images as compact, observation-grounded scene tokens, each derived from a source patch and decoded into freely placed Gaussians. It solves the problem of redundant, misaligned Gaussian predictions across views by aggregating multi-view observations into a fixed set of tokens, while allowing the reconstruction budget to be adjusted at inference without retraining. It differs from pixel-to-Gaussian mapping methods by deriving tokens directly from image patches rather than learned queries, and by using a cone loss to ground decoded Gaussians to their source observations, preventing spatial drift.

### [SHARE (Pose-free 3D Gaussian Splatting via Shape-Ray Estimation)](19346-share-pose-free-3d-gaussian-splatting-via-shape-ray-estimati.md)
SHARE is a pose-free, feed-forward Gaussian splatting framework that jointly estimates relative camera poses and reconstructs accurate 3D geometry by operating in latent space rather than explicit 3D space, mitigating pose-induced geometry misalignment. Unlike methods that transform 3D geometries from each viewpoint into a reference camera coordinate using predicted poses, SHARE constructs a unified cost volume in latent space, bypassing erroneous 3D space transformation. It sets itself apart from siblings like [FreeSplat](15163-freesplat.md) and [ZipSplat](17331-zipsplat.md) by using Plücker rays to guide multi-view feature fusion directly, avoiding the sequential estimation of pose and shape that leads to significant geometric errors.

## Operating characteristics
**Time and Memory Complexity:**
The method operates in a single forward pass, enabling real-time rendering latency [S1, unofficial]. Specific time and memory complexity metrics are not given in the sources [S1, unofficial].

**Measured Performance:**
AnySplat matches pose-aware baselines and surpasses prior pose-free methods (NoPoSplat-class) in zero-shot evaluations [S1, unofficial]. It trains in seconds [S1, unofficial].

**Scaling:**
The method scales to casual dense or sparse multi-view captures [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** No pose annotations required; real-time rendering latency; handles sparse and dense captures; zero-shot evaluation performance; fast training [S1, unofficial].
- **Limitations:** Not given in the sources.

**When to Use:**
Use AnySplat when camera poses are unknown or unavailable, and real-time rendering is required [S1, unofficial].

## Background
AnySplat is a pose-free feed-forward method within the 3D Gaussian Splatting family [S1, unofficial]. It addresses the limitations of prior pose-free approaches by enabling robust reconstruction from uncalibrated image collections [S1, unofficial]. It appears in the 2026 SOTA update for Gaussian Splatting variants [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
