# Viewpoint Saliency

Also called: Saliency-based View Selection, Visibility-Weighted Saliency Metric

## Overview
Viewpoint Saliency is a metric used to select the best viewpoints for volume visualization by maximizing the visible saliency of the volume data [S1]. It builds on [Visibility-Weighted Saliency](6824-visibility-weighted-saliency.md) by aggregating voxel-level saliency scores weighted by their visibility from a specific viewpoint [S1].

## How it works
**Inputs:**
- A set of candidate viewpoints $V_{cand}$ [G].
- A precomputed saliency value $S(p)$ for every voxel $p$ in the volume dataset [S1].
- A visibility function $V(p, v)$ that determines if voxel $p$ is visible from viewpoint $v$ [S1].

**Outputs:**
- The viewpoint $v^*$ that maximizes the total visible saliency [S1].

**Data Structures:**
- $P$: The set of all voxels in the volume [S1].
- $S(p)$: Scalar saliency value for voxel $p$ [S1].
- $V(p)$: Visibility value of voxel $p$ from a viewpoint [S1].

**Steps:**
1. For each candidate viewpoint $v \in V_{cand}$:
    a. Initialize $S(v) \leftarrow 0$.
    b. For each voxel $p \in P$:
        i. Compute visibility $V(p)$ from $v$ [S1].
        ii. Accumulate weighted saliency: $S(v) \leftarrow S(v) + S(p) \times V(p)$ [S1].
2. Return the viewpoint $v^* = \arg\max_{v \in V_{cand}} S(v)$ [S1].

**Formula:**
The viewpoint saliency $S(v)$ is defined as:

$$S(v) = \sum_{p \in P} S(p)V(p)$$

where $v$ is a given viewpoint, $P$ is the set of voxels of the volume data, $S(p)$ is the saliency of voxel $p$, and $V(p)$ is the visibility of voxel $p$ from $v$ [S1].

**Parameters:**
- Saliency computation parameters: Not given in the sources.
- Visibility threshold: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Saliency Definition:** The source references [LVJ05] for mesh saliency but does not specify the exact voxel saliency algorithm used [S1].
- **Visibility Calculation:** The source does not specify whether $V(p)$ is binary or continuous; implementers must choose a visibility evaluation method [G].

## Subtopics

### [Information-Theoretic Viewpoint Estimation](8149-information-theoretic-viewpoint-estimation.md)
An algorithm that estimates optimal viewpoints for a set of features using an information-theoretic framework to maximize visual discrimination between focus features and context. It selects viewpoints that make specific objects "pop out" by ensuring dense rendering for focus features and sparse rendering for context, distinguishing it from geometric-only approaches by incorporating visibility probabilities into the selection metric.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Directly optimizes for perceptual importance and guides viewer attention to the most salient parts of the model [S1].
- **Limitations:** Requires precomputed voxel saliency; the exact visibility calculation method is not specified [S1].
- **When to use:** When the goal is to automatically generate views that highlight the most salient features of a volume [S1].
- **When not to use:** Not given in the sources.

## Background
The concept was introduced in the "Obscurance-based Volume Rendering Framework" paper, which proposed using saliency to select the best views and enhance volume visualization by modifying transfer functions [S1]. It builds on earlier work in mesh saliency [LVJ05], adapting the concept to volumetric data [S1]. The method was demonstrated on CT-body datasets to show the most salient and least salient views [S1].

## Sources
[S1] Obscurance-based Volume Rendering Framework, https://doi.org/10.2312/vg/vg-pbg08/113-120, fetched 2026-10-26
