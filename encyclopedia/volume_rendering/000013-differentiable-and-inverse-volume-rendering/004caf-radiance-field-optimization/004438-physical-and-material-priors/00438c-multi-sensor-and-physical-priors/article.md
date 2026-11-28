# Multi-Sensor and Physical Priors
## Overview
This family adapts differentiable volume rendering to non-ideal imaging conditions and multi-modal sensor data by replacing or augmenting the standard emission-absorption model with physics-based signal formation equations [G].
## Shared principles
Both methods optimize scene representations by backpropagating gradients through a modified rendering operator tailored to specific sensor physics [G]. They replace standard emission-absorption integration with physics-specific accumulation: rigid motion deformation for optical blur [DP-NeRF](8273-dp-nerf.md) and azimuth-plane summation for SAR [Radar Fields](8904-radar-fields.md), minimizing reconstruction losses against observed inputs [G].
## Subtopics

### [DP-NeRF](8273-dp-nerf.md)
Reconstructs sharp 3D scenes from blurred inputs using 3D physical priors instead of 2D pixel-wise optimization, ensuring geometric and appearance consistency across views. Choose when inputs have known camera poses but suffer from motion or defocus blur.

### [Radar Fields](8904-radar-fields.md)
Extends radiance fields to Synthetic Aperture Radar imagery by integrating radar signal contributions across rays in an azimuth plane sharing the same range distance. Choose when reconstructing volumetric surface models from multiple SAR measurements.

### NeuRAD
NeuRAD is a neural rendering method for Autonomous Driving (AD) that integrates camera and LiDAR sensor models [S1, unofficial].

## Background
Not given in the sources.
## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
