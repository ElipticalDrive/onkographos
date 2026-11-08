# Semantic and Label-Based Classification

Also called: Discrete Label Mapping, Categorical Rendering

## Overview
Semantic and Label-Based Classification methods map discrete integer identifiers (labels) or semantic categories directly to visual properties, bypassing the continuous scalar interpolation used in standard transfer functions. This approach solves the problem of visualizing segmented data, mesh elements, or enumerated types where intermediate values are meaningless [S1, 7160-categorical-color-maps.md]. It differs from [Data Feature-Based Classification](5934-data-feature-based-classification.md) by operating on pre-computed discrete IDs rather than deriving classification from continuous field gradients, and from [Style Transfer Function (1D)](777-style-transfer-function-1d.md) by treating the input domain as a set of distinct classes rather than a continuous range.

## Shared principles
All methods in this family share the core mechanism of evaluating a classification function $C(\mathbf{x})$ for each sample point $\mathbf{x}$ in the volume to produce optical properties [G].

**Input Domain:**
The input is a discrete identifier $k \in \mathbb{Z}$ or a semantic class label, rather than a continuous scalar intensity [S1, 7160-categorical-color-maps.md].

**Output Range:**
The output is a deterministic color vector $(r, g, b)$ and often a binary opacity mask (fully opaque for valid labels, transparent or NaN-colored for invalid ones) [S1, 4667-semantic-label-colormap-generation.md].

**Classification Types:**
1.  **Exact Match:** The classification function returns a specific color only if the input label matches a defined key exactly [S1, 7160-categorical-color-maps.md].
2.  **Fallback Handling:** Undefined or unseen labels are mapped to a specific "NaN" color or a deterministic algorithmic color to ensure visual consistency [S1, 4667-semantic-label-colormap-generation.md].

**Evaluation Order:**
Classification is applied per-voxel or per-element based on its discrete ID. Interpolation is generally avoided for the classification logic itself to prevent "color bleeding" between distinct semantic regions, though spatial interpolation may be used for rendering the final image if the labels are treated as a texture [S1, 4667-semantic-label-colormap-generation.md].

**Implementation Structures:**
Methods typically use direct lookup tables (LUTs) mapping integer IDs to RGB tuples, or deterministic hash functions for generating colors for arbitrary unseen IDs [S1, 4667-semantic-label-colormap-generation.md].

## Subtopics

### [Semantic Label Colormap Generation](4667-semantic-label-colormap-generation.md)
Maps 2D semantic label images to RGB by assigning predefined colors to known classes and using a deterministic HSV-based hash for unknown classes. Chosen when handling arbitrary or evolving class IDs without user-defined palettes.

### [Categorical Color Maps](7160-categorical-color-maps.md)
Renders discrete enumerations (e.g., element types) using exact value-to-color matching and a NaN fallback. Chosen for non-continuous data where interpolation is invalid.

### [Semantic Transfer Function Design](9493-semantic-transfer-function-design.md)
A framework for designing high-level transfer functions using semantic parameters derived from PCA on control points from multiple data instances. Chosen for expert-driven, high-dimensional transfer function design.

## Background
The distinction between continuous scalar fields and discrete categorical data is fundamental in visualization. While early volume rendering focused on scalar fields [S1], the need to visualize segmented medical data, finite element meshes, and semantic segmentation masks necessitated label-based classification. The concept of categorical color maps was formalized in visualization toolkits to handle discrete enumerations where traditional interpolated transfer functions were cumbersome [S1]. Semantic label colormaps, particularly those using deterministic hue generation for unknown classes, emerged from computer vision and segmentation evaluation standards to ensure consistent visual comparison across different labeling schemes [4667-semantic-label-colormap-generation.md].

## Sources
[S1] Categorical Color Maps documentation, URL not provided in notes, date fetched: 2023-10-01
