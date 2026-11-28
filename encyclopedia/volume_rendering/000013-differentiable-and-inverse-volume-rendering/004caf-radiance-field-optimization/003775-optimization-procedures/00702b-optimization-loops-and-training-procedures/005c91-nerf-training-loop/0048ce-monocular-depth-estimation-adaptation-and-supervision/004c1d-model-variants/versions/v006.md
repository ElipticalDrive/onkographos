# Model Variants

Also called: FWD-D, FWD-U

## Overview
This family defines three architectural configurations for the FWD framework, distinguished by their depth supervision strategy and the initialization state of the depth estimation module [S1]. The variants evaluate different accessibility levels to depth information during training and inference [S1]. FWD-U serves as the unsupervised baseline, FWD provides a supervised baseline with pre-trained geometry, and FWD-D bypasses learned depth estimation entirely by using sensor data [S1].

## Shared principles
All variants in this family operate within the FWD framework for novel view synthesis [S1]. They share the core mechanism of using a PatchmatchNet architecture for depth estimation, with the primary divergence lying in how the depth map is obtained and whether the depth estimation network is trainable during the end-to-end optimization loop [S1].

## Subtopics

### FWD-U
FWD-U is the unsupervised variant of the model [S1]. It utilizes a PatchmatchNet architecture for depth estimation that is randomly initialized at the start of training [S1]. No pre-training on MVS datasets is performed [S1]. The network shares the same model and settings as FWD [S1].

### FWD
FWD is the supervised variant that employs a PatchmatchNet module pre-trained on standard Multi-View Stereo (MVS) settings [S1]. During the end-to-end training of the NeRF and rendering pipeline, the PatchmatchNet weights are updated jointly with the scene representation using photometric and perceptual loss [S1].

### FWD-D
FWD-D is a variant that eliminates the learned MVS module entirely [S1]. Instead of estimating depth from RGB images, it takes sensor depths as direct additional inputs during both training and inference [S1]. Because sensor depths provide abundant geometry cues, no MVS network is used [S1].

## Background
The model variants were evaluated on the ShapeNet dataset to assess performance under different accessibility levels to depth information [S1]. The study evaluates category-agnostic synthesis on 13 ShapeNet categories, with each instance containing 24 fixed views of 64 × 64 resolution [S1].

## Sources
[S1] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
