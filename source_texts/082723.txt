# Joint-MinkUNet

This repository is reserved for the source code accompanying the manuscript:

**Fine-grained Semantic Segmentation of RMLS Shield Tunnel Point Clouds for Automated Structural Health Assessment: Joint-MinkUNet with Cylindrical Geometry Adaptation and Element Distribution-aware Learning**

## Code availability

**If the manuscript is accepted for publication, the source code will be publicly released in this repository.** The code is not publicly available at this stage.

The release will include the implementation of Joint-MinkUNet and its key components, together with supporting scripts and configuration files for training, inference, and evaluation.

## Overview

Joint-MinkUNet is a framework for fine-grained semantic segmentation of shield tunnel point clouds acquired by Railborne Mobile Laser Scanning (RMLS). Built on the MinkUNet backbone, it integrates:

- **SPCV:** Sparse Polar-coordinate Voxel representation adapted to cylindrical tunnel geometry.
- **CL-DS-Adapter:** Circumferential-Longitudinal Direction-Structure-Aware Adapter for capturing directional features of elongated tunnel elements.
- **ABC-TFE:** Adaptive Balanced Constraint mechanism for Tail Fine-grained Elements to improve learning under class imbalance.

## Dataset

For the dataset release announcement, see [RMLS-Tunnel17](https://github.com/ZelasYou/RMLS-Tunnel17).
