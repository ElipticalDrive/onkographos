# Viewpoint Saliency

Also called: Saliency-based View Selection, Visibility-Weighted Saliency Metric

## Overview
Viewpoint Saliency is a metric used to select optimal viewpoints for volume visualization by maximizing the visible importance of the data. It builds on [Visibility-Weighted Saliency](6824-visibility-weighted-saliency.md) by aggregating voxel-level saliency scores weighted by their visibility from a specific camera position. Unlike geometric-only view selection methods, it incorporates perceptual distinctiveness (saliency) to guide the viewer's attention to the most significant features [S1].

## How it works
**Inputs:**
- A set of candidate viewpoints $V_{cand}$.
- A precomputed saliency value $S(p)$ for every voxel $p$ in the volume dataset.
- A visibility function $V(p, v)$ that determines if voxel $p$ is visible from viewpoint $v$.

**Outputs:**
- The viewpoint $v^*$ that maximizes the total visible saliency.

**Data Structures:**
- $P$: The set of all voxels in the volume.
- $S(p)$: Scalar saliency value for voxel $p$.
- $V(p, v)$: Binary or continuous visibility value (1 if visible, 0 if occluded, or a probability).

**Steps:**
1. For each candidate viewpoint $v \in V_{cand}$:
    a. Initialize total saliency $S(v) = 0$.
    b. For each voxel $p \in P$:
        i. Determine visibility $V(p, v)$. This typically involves ray casting or depth buffer analysis to check if $p$ is occluded by other voxels closer to the camera.
        ii. Add the weighted saliency to the total: $S(v) \leftarrow S(v) + S(p) \times V(p, v)$.
2. Select the viewpoint $v^*$ with the maximum $S(v)$.

**Formula:**
The viewpoint saliency $S(v)$ is defined as:

$$S(v) = \sum_{p \in P} S(p)V(p)$$

where:
- $v$ is the given viewpoint.
- $P$ is the set of voxels of the volume data.
- $S(p)$ is the saliency of voxel $p$.
- $V(p)$ is the visibility of voxel $p$ from $v$ [S1].

**Parameters:**
- Saliency computation parameters (e.g., for mesh or voxel-based saliency): Not given in the sources.
- Visibility threshold: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Occlusion:** If $V(p)$ is strictly binary, small misalignments in ray casting may cause flickering or incorrect visibility. Continuous visibility probabilities may be more robust [G].
- **Performance:** Summing over all voxels for every viewpoint is expensive. Optimization via hierarchical data structures (e.g., bounding volume hierarchies) is implied but not detailed in the source [G].
- **Saliency Definition:** The source references [LVJ05] for mesh saliency but does not specify the exact voxel saliency algorithm used. Implementers must define $S(p)$ independently [S1].

## Subtopics

### [Information-Theoretic Viewpoint Estimation](8149-information-theoretic-viewpoint-estimation.md)
An algorithm that estimates optimal viewpoints for a set of features using an information-theoretic framework to maximize visual discrimination between focus features and context. It selects viewpoints that make specific objects "pop out" by ensuring dense rendering for focus features and sparse rendering for context, distinguishing it from geometric-only approaches by incorporating visibility probabilities into the selection metric.

## Operating characteristics
- **Time Complexity:** $O(|V_{cand}| \cdot |P|)$ if visibility is checked per voxel. This is computationally expensive for large volumes.
- **Memory Complexity:** $O(|P|)$ to store voxel saliency values.
- **Strengths:** Directly optimizes for perceptual importance; guides viewer attention effectively.
- **Limitations:** Requires precomputed voxel saliency; visibility calculation can be slow; sensitive to the accuracy of the visibility function.
- **When to use:** When the goal is to automatically generate views that highlight the most "important" or distinct features of a volume.
- **When not to use:** When real-time interaction is required without precomputation, or when geometric features are more important than perceptual saliency.

## Background
The concept was introduced in the "Obscurance-based Volume Rendering Framework" paper, which proposed using saliency to select the best views and enhance volume visualization by modifying transfer functions [S1]. It builds on earlier work in mesh saliency [LVJ05], adapting the concept to volumetric data. The method was demonstrated on CT-body datasets to show the most salient and least salient views [S1].

## Sources
[S1] Obscurance-based Volume Rendering Framework, https://doi.org/10.2312/vg/vg-pbg08/113-120, fetched 2026-10-26
