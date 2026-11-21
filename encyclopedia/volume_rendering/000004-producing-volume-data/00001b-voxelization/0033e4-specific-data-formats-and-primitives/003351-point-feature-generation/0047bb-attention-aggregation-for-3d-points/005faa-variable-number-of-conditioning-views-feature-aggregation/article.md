# Variable Number of Conditioning Views Feature Aggregation

Also called: Camera-aware Feature Aggregation, Permutation Invariant Feature Aggregation with Viewpoint Info

## Overview
This method aggregates pixel-aligned features from an arbitrary number of input views into a single feature vector for a query 3D point, solving the problem of variable input dimensions at inference time. It differs from simple mean pooling by injecting camera parameters into the aggregation process to mitigate artifacts caused by depth ambiguity, and it differs from fixed-view concatenation by being permutation invariant to the number of conditioning views.

## How it works
**Inputs:**
- A set of $n$ conditioning views.
- For each view $i \in \{1, \dots, n\}$:
  - A feature map $f^{(i)}$.
  - Camera parameters $c_i$, encoded as a 4D rotation quaternion and 3D camera position (a 7D vector) [S1, unofficial].
  - Camera intrinsic $K$ and extrinsic parameters $R, t$ [S1, unofficial].
- A query 3D point $X \in \mathbb{R}^3$.

**Outputs:**
- A single aggregated feature vector $f_X \in \mathbb{R}^d$.

**Data Structures:**
- The aggregation function $G$ maps $\mathbb{R}^{n \times d} \to \mathbb{R}^d$.
- The camera-aware summarization network $N_{cf}: \mathbb{R}^{d+7} \to \mathbb{R}^d$ [S1, unofficial].

**Steps:**
1. **Projection and Sampling:** For each conditioning view $i$, project the 3D point $X$ onto the 2D feature image plane using the camera intrinsic $K$ and extrinsic parameters $R, t$ to obtain 2D coordinates. Sample the feature vector $f_X^{(i)}$ from the feature map $f^{(i)}$ at these coordinates [S1, unofficial].
2. **Camera-Aware Summarization:** For each sampled feature $f_X^{(i)}$, concatenate the camera information $c_i$ and pass them through the network $N_{cf}$ to produce a camera-summarized feature $f_X^{0(i)}$:
   $$ f_X^{0(i)} = N_{cf}(f_X^{(i)}, c_i) $$
   This step allows the network to account for likely occlusions before averaging [S1, unofficial].
3. **Aggregation:** Compute the final feature vector by averaging the camera-summarized features from all $n$ views:
   $$ f_X = \frac{1}{n} \sum_{i=1}^{n} f_X^{0(i)} $$

**Pitfalls:**
- Simple mean aggregation (without camera awareness) produces artifacts due to inherent depth ambiguity when points are projected onto the feature image before sampling [S1, unofficial].
- The method must handle $n$ varying at inference time; simple concatenation is insufficient as it requires a fixed number of views [S1, unofficial].

## Subtopics

### Camera-Aware Feature Summarization
This variant explicitly injects viewpoint information into the feature aggregation. It uses a network $N_{cf}$ to transform features conditioned on camera parameters (quaternion and position) before averaging. This is chosen when depth ambiguity causes artifacts in simple mean pooling, as the camera info helps the network distinguish between visible and occluded content [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of conditioning views $n$, as it requires one forward pass through $N_{cf}$ per view and one summation.
- **Memory Complexity:** Linear with respect to $n$ to store intermediate features before aggregation.
- **Strengths:** Handles arbitrary numbers of input views at inference; reduces artifacts caused by depth ambiguity by leveraging camera geometry.
- **Limitations:** Requires training a separate network $N_{cf}$; performance depends on the accuracy of the camera parameter encoding.

## Background
Originated in the PVA (Pixel-aligned Volumetric Avatars) paper [S1, unofficial]. The work addresses the limitation of previous methods that required a fixed number of conditioning views or suffered from artifacts when using simple mean pooling in multi-view settings with depth ambiguity.

## Sources
[S1] PVA: Pixel-aligned Volumetric Avatars, https://doi.org/10.48550/arxiv.2101.02697, fetched 2026-10-30 (unofficial)
