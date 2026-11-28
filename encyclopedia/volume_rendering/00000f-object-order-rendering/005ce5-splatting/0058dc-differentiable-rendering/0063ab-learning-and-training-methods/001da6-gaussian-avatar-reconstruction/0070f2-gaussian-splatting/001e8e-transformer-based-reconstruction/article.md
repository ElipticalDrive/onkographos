# Transformer-based Reconstruction


## Overview
Transformer-based Reconstruction methods replace the iterative optimization loop of standard 3D Gaussian Splatting (3DGS) with a feed-forward neural network that directly predicts Gaussian parameters from input images [G]. This approach solves the computational bottleneck of per-scene optimization, enabling real-time reconstruction from single or sparse views [G]. It differs from iterative siblings by eliminating the need for gradient descent on scene parameters, instead relying on learned priors within the transformer architecture to generalize across scenes [G].

## Shared principles

## Subtopics

### [Flash3D](1481-flash3d.md)
Flash3D is a feed-forward method for monocular 3DGS reconstruction that estimates depth from a single view to generate Gaussians. It differs from [GS-LRM](1483-gs-lrm.md) and [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md) by relying on monocular depth estimation rather than multi-view feature aggregation or tokenized multi-image inputs. It is chosen when only a single image is available and a depth prior is acceptable. See [Flash3D](1481-flash3d.md).

### [GS-LRM](1483-gs-lrm.md)
GS-LRM is a feed-forward transformer method that predicts 3DGS parameters directly from a small set of input images (typically 2-4). It distinguishes itself from [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md) by specifically targeting Gaussian parameter prediction rather than mesh or NeRF fields, and from [Flash3D](1481-flash3d.md) by its explicit handling of multi-view camera pose tokens. It is chosen for sparse-view reconstruction where camera poses are known. See [GS-LRM](1483-gs-lrm.md).

### [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md)
LRM is a feed-forward neural network designed for single-image to 3D reconstruction, establishing the foundational transformer-based architecture for direct parameter prediction in this family. It sets itself apart from [Flash3D](1481-flash3d.md) and [GS-LRM](1483-gs-lrm.md) by its generalizable architecture that can be adapted to various 3D representations, though here it predicts Gaussians. It is chosen as a baseline for single-view direct reconstruction. See [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md).

### [Reconstruction-based Generation](11811-reconstruction-based-generation.md)
Reconstruction-based Generation decomposes 3D generation into two stages: multi-view image synthesis followed by 3D reconstruction. Unlike direct prediction methods, it leverages pre-trained diffusion models to generate posed multi-view images, which then serve as supervision for a Gaussian Splatting reconstructor. This approach reuses mature reconstruction pipelines, making system-level engineering straightforward, but inherits the consistency limitations of the multi-view generator. It is chosen when text-to-3D or single-image-to-3D generation is required without training a dedicated reconstruction network from scratch. See [Reconstruction-based Generation](11811-reconstruction-based-generation.md).

## Background
The family originates from the need to accelerate 3D Gaussian Splatting, which traditionally requires per-scene optimization [G]. [Large Reconstruction Model (LRM)](5959-large-reconstruction-model-lrm.md) introduced the transformer-based feed-forward approach for 3D reconstruction [G]. [GS-LRM](1483-gs-lrm.md) and [Flash3D](1481-flash3d.md) adapted this paradigm specifically for 3DGS, with [GS-LRM](1483-gs-lrm.md) focusing on sparse multi-view inputs and [Flash3D](1481-flash3d.md) on monocular depth estimation [G]. [Reconstruction-based Generation](11811-reconstruction-based-generation.md) extends this by integrating diffusion-based multi-view synthesis [G].

## Sources
