# Robustness and Generalization


## Overview
This family encompasses techniques that improve the stability of NeRF optimization against noise and the ability of the resulting model to synthesize novel views from limited or unconstrained data. It addresses the generalization gap where standard optimization yields inconsistent geometries or fails on scenes with transient objects and lighting variations. Methods in this family differ from standard training by injecting adversarial perturbations [Aug-NeRF](10294-aug-nerf.md), decomposing scene appearance into static and per-image latent codes [NeRF in the Wild](13541-nerf-in-the-wild.md), enforcing cross-view geometric consistency via contrastive learning [ContraNeRF](13659-contranerf.md), or using post-processing reprojection to restore high-frequency details [RoGUENeRF](13731-roguenerf.md).

## Shared principles
Not given in the sources.

## Subtopics

### [Aug-NeRF](10294-aug-nerf.md)
Injects worst-case adversarial perturbations at coordinate, feature, and output levels to implicitly regularize the model. It addresses the generalization gap by using data-adaptive smoothness priors rather than explicit Laplacian regularizers. Choose when standard NeRF yields non-smooth geometries during interpolation.

### [NeRF Ensemble with Conjoint Pruning](12608-nerf-ensemble-with-conjoint-pruning.md)
Combines two NeRF models of different grid resolutions (DVGO) via independent training, ensemble fine-tuning, and conjoint pruning. It improves quality over single models while maintaining memory footprint. Choose when computational resources allow for multi-model training and pruning is desired to prevent degradation.

### [NeRF in the Wild](13541-nerf-in-the-wild.md)
Decomposes the scene into a static radiance field and per-image latent codes for appearance and transient artifacts. It handles unconstrained photo collections with varying lighting and transient objects. Choose when training data contains pedestrians, vehicles, or significant lighting changes.

### [ContraNeRF](13659-contranerf.md)
Uses geometry-aware contrastive learning and cross-view attention to improve synthetic-to-real generalization. It enforces multi-view consistency by weighting contrastive loss based on projected pixel distances. Choose when training on synthetic data to improve real-world generalization.

### [RoGUENeRF](13731-roguenerf.md)
A post-processing enhancer that reprojects training features into the novel view space using geometry-aware alignment and fusion. It restores high-frequency textures and handles inaccurate camera calibration. Choose when rendering quality needs enhancement without retraining, or when camera poses are slightly inaccurate.

## Background
The challenge of generalizing NeRFs beyond the training set and handling real-world data inconsistencies has been a central focus since the original NeRF publication. Aug-NeRF [S1] introduces adversarial training to the radiance field optimization process. NeRF Ensemble with Conjoint Pruning [S1] extends DVGO architectures to ensemble methods. NeRF in the Wild [S3, unofficial][S2] was developed to handle the "in the wild" scenario with transient objects. ContraNeRF [S1] addresses the domain gap between synthetic and real data. RoGUENeRF [S1] focuses on post-processing robustness.

## Sources
[S1] Source 1, URL, date fetched
[S2] Source 2, URL, date fetched
[S3] Source 3, URL, date fetched
