# Semantic and Label-Based Classification


## Overview
Semantic and Label-Based Classification methods map discrete integer identifiers (labels) or semantic categories directly to visual properties, bypassing the continuous scalar interpolation used in standard transfer functions [G]. This approach solves the problem of visualizing segmented data, mesh elements, or enumerated types where intermediate values are meaningless [G]. It differs from [Data Feature-Based Classification](5934-data-feature-based-classification.md) by operating on pre-computed discrete IDs rather than deriving classification from continuous field gradients, and from [Style Transfer Function (1D)](777-style-transfer-function-1d.md) by treating the input domain as a set of distinct classes rather than a continuous range [G].

## Shared principles
All methods in this family share the core mechanism of evaluating a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

**Input Domain:**
Discrete integer identifiers or semantic class labels, rather than continuous scalar intensities [G].

**Output Range:**
Deterministic color vectors $(r, g, b)$ and often binary opacity masks (fully opaque for valid labels, transparent or NaN-colored for invalid ones) [G].

**Classification Logic:**
Exact matching against defined keys for known classes, with deterministic algorithmic color generation (e.g., hash-based HSV mapping) or fallback NaN colors for unseen labels [G].

**Interpolation Handling:**
Interpolation is avoided during classification to prevent "color bleeding" between distinct semantic regions, though nearest-neighbor interpolation is required during ray casting to maintain label integrity [G].

**Implementation Structures:**
Direct lookup tables (LUTs) mapping integer IDs to RGB tuples, or deterministic hash functions for generating colors for arbitrary unseen IDs [G].

## Subtopics

### [Semantic Label Colormap Generation](4667-semantic-label-colormap-generation.md)
Maps 2D semantic label images to RGB by assigning predefined colors to known classes and using a deterministic HSV-based hash for unknown classes. It differs from [Categorical Color Maps](7160-categorical-color-maps.md) by using a deterministic hash-based hue generation for unknown labels rather than a fixed palette, and from [Style Transfer Function (1D)](777-style-transfer-function-1d.md) by operating on discrete integer labels rather than continuous scalar values [G]. Choose it when handling arbitrary or evolving class IDs without user-defined palettes.

### [Categorical Color Maps](7160-categorical-color-maps.md)
Renders discrete enumerations (e.g., element types) using exact value-to-color matching and a NaN fallback. Unlike continuous methods, this approach uses exact matching against a lookup table to assign deterministic colors, ensuring visual consistency for semantic labels [G]. Choose it for non-continuous data where interpolation is invalid.

### [Semantic Transfer Function Design](9493-semantic-transfer-function-design.md)
Addresses the complexity of manually adjusting low-level transfer function parameters by mapping high-level semantic concepts (e.g., "sharpness," "visibility") to TF instances using Principal Component Analysis (PCA) on expert-designed TF models [G]. It differs from direct classification methods by using a low-dimensional semantic parameter space controlled via sliders [G]. Choose it for expert-driven, high-dimensional transfer function design.

### [Label-Scoped Volume Rendering](12015-label-scoped-volume-rendering.md)
Renders labeled volumes using a single 1D transfer function by packing label rows into distinct scalar bands, allowing per-class opacity scaling without multi-dimensional transfer functions [G]. It differs from general semantic classification by relying on a fixed scalar offset to isolate classes within a 1D lookup, ensuring transfer function ramps apply strictly to their designated semantic region [G]. Choose it when rendering anatomical or segmented volumes with a single 1D VTK transfer function.

### Semantic Voxel Labelling
**Definition:** Predicting class labels for each voxel in an input grid, where every 3D voxel is assigned an integer class label indicating its semantic category [S1].
**Formula:** $S_{i,j,k} \in \{1, 2, \dots, n\}$, for $1 \le i \le W$, $1 \le j \le H$, $1 \le k \le D$ [S1].
**Parameter:** $n$: The number of semantic categories [S1].

## Background
The distinction between continuous scalar fields and discrete categorical data is fundamental in visualization [G]. Semantic voxel labelling predicts class labels for 3D grids to support downstream tasks like 3D object detection in NeRFs [S1]. Semantic transfer function design emerged to allow non-expert users to intuitively control visualization properties via high-level semantic parameters mapped through PCA [S2]. The concept of categorical color maps was formalized in visualization toolkits to handle discrete enumerations where traditional interpolated transfer functions were cumbersome [G].

## Sources
[S1] NeRF-MAE: Masked AutoEncoders for Self-Supervised 3D Representation Learning for Neural Radiance Fields, https://arxiv.org/abs/2404.01300v3, fetched 2026-10-26
