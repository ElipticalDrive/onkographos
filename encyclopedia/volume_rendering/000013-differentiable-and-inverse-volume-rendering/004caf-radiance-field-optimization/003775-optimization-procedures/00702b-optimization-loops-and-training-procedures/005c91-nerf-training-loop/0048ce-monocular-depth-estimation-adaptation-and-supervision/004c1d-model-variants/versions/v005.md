# Model Variants

Also called: FWD-D, FWD-U

## Overview
This family defines three specific architectural configurations for the FWD framework, distinguished by their depth supervision strategy and the initialization state of the depth estimation module. They address the trade-off between training stability, convergence speed, and reliance on external geometric priors. FWD-U serves as the unsupervised baseline, FWD provides a supervised baseline with pre-trained geometry, and FWD-D bypasses learned depth estimation entirely by using sensor data.

## Shared principles
All variants in this family operate within the FWD (Forward Warping and Depth) framework for real-time novel view synthesis. They share the core mechanism of forward warping color features from source views to target views using estimated depth maps [S1]. The primary divergence lies in how the depth map is obtained and whether the depth estimation network is trainable during the end-to-end optimization loop.

## Subtopics

### FWD-U
FWD-U is the unsupervised variant of the model. It utilizes a PatchmatchNet architecture for depth estimation that is randomly initialized at the start of training [S1]. No pre-training on MVS datasets is performed, and the network learns depth representations solely through the photometric and perceptual losses inherent to the novel view synthesis task [S1]. This variant is chosen when no external depth priors or pre-trained models are available, serving as a baseline for unsupervised learning capabilities.

### FWD
FWD is the supervised variant that employs a PatchmatchNet module pre-trained on standard Multi-View Stereo (MVS) settings [S1]. During the end-to-end training of the NeRF and rendering pipeline, the PatchmatchNet weights are updated jointly with the scene representation [S1]. This approach leverages strong geometric priors from pre-training while allowing the depth estimator to adapt to the specific domain of the target scene. It is chosen when faster convergence and higher geometric accuracy are required compared to the unsupervised variant.

### FWD-D
FWD-D is a variant that eliminates the learned MVS module entirely. Instead of estimating depth from RGB images, it takes sensor depths (e.g., from LiDAR or structured light sensors) as direct additional inputs during both training and inference [S1]. Because sensor depths provide abundant and accurate geometry cues, no MVS network is used. This variant is chosen when high-quality sensor data is available, offering the most robust geometric supervision without the error propagation risks of learned depth estimation.

## Background
The FWD framework was introduced to enable real-time novel view synthesis by combining forward warping with depth-based feature transfer [S1]. The model variants were evaluated on the ShapeNet dataset to assess performance under different accessibility levels to depth information [S1]. The study highlights how the initialization of the depth estimator and the source of depth data impact the final synthesis quality and training dynamics.

## Sources
[S1] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
