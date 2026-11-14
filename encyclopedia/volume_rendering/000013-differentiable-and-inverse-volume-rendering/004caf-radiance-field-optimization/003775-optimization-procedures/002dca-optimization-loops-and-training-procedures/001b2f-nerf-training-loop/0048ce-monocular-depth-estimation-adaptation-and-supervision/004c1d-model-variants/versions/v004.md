# Model Variants

Also called: FWD-D, FWD-U

## Overview
This family defines three specific configurations for the Forward Warping Depth (FWD) framework, distinguished by their source of depth supervision and the training status of the Monocular Depth Estimation (MDE) module. They solve the problem of how to integrate depth priors into NeRF optimization when the availability of ground-truth depth or pre-trained MDE networks varies. FWD-U represents a fully unsupervised baseline where the depth network is trained from scratch, while FWD leverages a pre-trained MVS module that is fine-tuned end-to-end. FWD-D bypasses the MVS module entirely, using sensor depths as direct inputs.

## Shared principles
All variants operate within the FWD framework, which synthesizes novel views by warping source images using estimated depth maps rather than volumetric ray marching. The core mechanism involves estimating a depth map $D$ for each input view and using it to project pixels from source views to the target view. The optimization minimizes photometric and perceptual losses between the rendered target view and the ground-truth target view. The specific variant determines the initialization and gradient flow of the depth estimation network.

## Subtopics

### FWD-U
FWD-U is the unsupervised variant of the FWD model. It utilizes the same architecture as FWD but initializes the PatchmatchNet [77] depth estimation module with random weights. It does not use any pre-training or Multi-View Stereo (MVS) pre-training. The network learns depth estimations entirely in an unsupervised manner during the end-to-end training process, relying solely on photometric and perceptual losses from the novel view synthesis task [S1]. Choose this variant when no pre-trained MVS model or sensor depth data is available, and the goal is to learn depth representations purely from video consistency.

### FWD
FWD is the supervised variant that utilizes a pre-trained PatchmatchNet [77] as the MVS module for depth estimation. Unlike FWD-U, this module is not frozen; it is updated during the end-to-end training alongside the NeRF or warping parameters. The pre-training follows typical MVS settings using the same data splitting as the Novel View Synthesis (NVS) task. This variant benefits from the geometric priors learned during pre-training while allowing adaptation to the specific domain of the target dataset [S1]. Choose this variant when a pre-trained MVS model is available and end-to-end fine-tuning is feasible to improve depth accuracy for the specific scene or category.

### FWD-D
FWD-D is a variant that incorporates sensor depths as additional inputs during both training and inference. It does not use any MVS module because the sensor depths provide abundant geometry cues directly. By feeding sensor depths into the network, the model bypasses the need to estimate depth from monocular images or stereo pairs, relying instead on the provided dense depth maps. This variant is suitable for scenarios where accurate sensor depth data (e.g., from LiDAR or structured light) is available and can be aligned with the camera views [S1].

## Background
The FWD framework was introduced to enable real-time novel view synthesis by replacing expensive volumetric rendering with forward warping techniques. The model variants were evaluated on ShapeNet categories to assess category-agnostic synthesis performance. The evaluation protocol used 13 ShapeNet categories, with each instance containing 24 fixed views of $64 \times 64$ resolution. The variants were designed to test the impact of different depth supervision sources on the quality of novel view synthesis [S1].

## Sources
[S1] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
