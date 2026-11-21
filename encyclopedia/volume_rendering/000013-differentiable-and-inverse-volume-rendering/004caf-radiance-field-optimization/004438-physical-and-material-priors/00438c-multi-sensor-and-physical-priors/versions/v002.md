# Multi-Sensor and Physical Priors
## Overview
This family adapts differentiable volume rendering to non-ideal imaging conditions and multi-modal sensor data by replacing or augmenting the standard emission-absorption model with physics-based signal formation equations [DP-NeRF](8273-dp-nerf.md)[Radar Fields](8904-radar-fields.md).
## Shared principles
Both methods optimize implicit scene representations by backpropagating gradients through a modified rendering operator tailored to specific sensor physics [DP-NeRF](8273-dp-nerf.md)[Radar Fields](8904-radar-fields.md). They replace the standard emission-absorption integration with physics-specific accumulation (rigid motion deformation for optical blur, azimuth-plane summation for SAR) and minimize reconstruction losses against observed multi-view inputs [DP-NeRF](8273-dp-nerf.md)[Radar Fields](8904-radar-fields.md).
## Subtopics

### [DP-NeRF](8273-dp-nerf.md)
DP-NeRF reconstructs sharp 3D scenes from blurry inputs (motion/defocus blur) using a Rigid Blurring Kernel (RBK) and Adaptive Weight Proposal (AWP) [DP-NeRF](8273-dp-nerf.md). It ensures 3D consistency by modeling blur as rigid camera motion, unlike 2D pixel-wise deblurring [DP-NeRF](8273-dp-nerf.md). Choose when inputs are calibrated but blurry [DP-NeRF](8273-dp-nerf.md).

### [Radar Fields](8904-radar-fields.md)
Radar Fields renders Synthetic Aperture Radar (SAR) imagery by integrating signal contributions across rays in an azimuth plane sharing the same range distance [Radar Fields](8904-radar-fields.md). It enables 3D geometry learning from SAR data, unlike optical fields [Radar Fields](8904-radar-fields.md).

### NeuRAD
NeuRAD is a neural rendering method for Autonomous Driving (AD) that integrates camera and LiDAR sensor models [S1, unofficial].

## Background
Not given in the sources.
## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
