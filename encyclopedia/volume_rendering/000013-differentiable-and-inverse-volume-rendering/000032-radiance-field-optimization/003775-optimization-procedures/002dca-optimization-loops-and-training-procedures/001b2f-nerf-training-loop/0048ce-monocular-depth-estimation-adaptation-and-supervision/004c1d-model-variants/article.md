# Model Variants

Also called: FWD, FWD-D, FWD-U

## Overview
This family defines three specific architectural configurations for the Forward Warping Depth (FWD) framework, distinguished by their depth supervision source and the training dynamics of the depth estimation module [S1]. While the parent article establishes the general mechanism of aligning rendered NeRF depth with pseudo-depth from a Monocular Depth Estimation (MDE) network, these variants determine whether that MDE network is pre-trained, randomly initialized, or bypassed entirely in favor of sensor data [S1].

## Shared principles
All variants in this family operate within the FWD framework, which utilizes forward warping for novel view synthesis [S1]. They share the core objective of optimizing a neural radiance field $\mathcal{N}$ and a depth estimation network (specifically PatchmatchNet) to produce consistent geometry and appearance [S1]. The fundamental difference lies in the initialization and update rules of the depth estimation component and the availability of ground-truth depth data during training [S1].

## Subtopics

### FWD
FWD represents the supervised configuration of the FWD framework [S1]. It utilizes a PatchmatchNet module that is pre-trained using typical Multi-View Stereo (MVS) settings and data splitting protocols before the main NeRF optimization begins [S1]. During the end-to-end training of the NeRF, the PatchmatchNet weights are updated jointly with the radiance field using photometric and perceptual losses [S1]. This variant is chosen when high-quality pre-training data for depth estimation is available and the goal is to leverage supervised geometric cues while allowing the depth estimator to adapt to the specific NeRF representation [S1].

### FWD-U
FWD-U denotes the unsupervised variant of the framework [S1]. It shares the identical model architecture and training settings as FWD, but the PatchmatchNet module is randomly initialized without any prior MVS pre-training [S1]. Consequently, the depth estimation is learned entirely from the NeRF training data without external depth supervision or pre-trained weights [S1]. This variant is appropriate when pre-training data is unavailable or when the goal is to learn depth representations purely from the consistency of the novel view synthesis task itself [S1].

### FWD-D
FWD-D is a variant that bypasses the MVS-based depth estimation module entirely [S1]. Instead of relying on a learned depth network, it takes sensor depths (e.g., from RGB-D cameras) as additional inputs during both training and inference [S1]. Because sensor depths provide abundant and direct geometry cues, no MVS module is used in this configuration [S1]. This variant is selected when precise, real-world depth measurements are available for the input views, eliminating the need for learned depth priors or pseudo-depth alignment [S1].

## Background
The FWD framework was introduced to enable real-time novel view synthesis using forward warping and depth [S1]. The specific variants (FWD, FWD-U, FWD-D) were evaluated on ShapeNet benchmarks to assess the impact of depth supervision sources and pre-training on category-agnostic synthesis tasks [S1]. The evaluation protocol used fixed inputs and outputs across all variants to ensure fair comparison [S1].

## Sources
[S1] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
