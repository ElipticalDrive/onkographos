# Attention Aggregation for 3D Points

Also called: Attention-based Point Feature Aggregation

## Overview
Attention Aggregation for 3D Points aggregates unordered, variable-sized sets of retrieved pixel features from multiple input views into a single unique feature vector for a query 3D point [S1]. It solves the problem of identifying which retrieved features are true descriptions of the query point in the presence of visual occlusions and arbitrary input counts [S1]. It sets itself apart from simple statistical aggregation (like mean pooling) by learning to weight views based on visual consistency and relative distance, rather than treating all views equally [S1][S2, unofficial].

## How it works
**Inputs:**
1. A query 3D point $p = [x_p, y_p, z_p]^T \in \mathbb{R}^3$.
2. A set of $M$ retrieved pixel features $\{F^1_p, \dots, F^M_p\}$, where each $F^m_p \in \mathbb{R}^d$ is the feature vector extracted from the $m$-th input view corresponding to the projection of $p$.

**Outputs:**
1. A single aggregated feature vector $\bar{F}_p \in \mathbb{R}^d$ for the query point.

**Data Structures:**
- The input set is treated as an unordered collection. The method requires permutation invariance with respect to the input set of feature vectors [S1].

**Steps:**
1. **Position-Aware Feature Transformation:** For each retrieved feature vector $F^m_p$, integrate the information of the query point $p$ to generate a position-aware feature vector $\hat{F}^m_p$. This is done using a shared Multi-Layer Perceptron (MLP) that concatenates the original feature with the 3D coordinates of $p$ [S1].
   $$ \hat{F}^m_p = \text{MLPs}(F^m_p \oplus [x_p, y_p, z_p]) $$
   Where $\oplus$ denotes concatenation. This step makes the features aware of the relative distance to the query point [S1].

2. **Attention Aggregation:** Compute a unique weight for all position-aware input features and aggregate them together using an attention function $A$ [S1]. The specific attention mechanism (e.g., AttSets or Slot Attention) must be permutation invariant [S1].
   $$ \bar{F}_p = A(\hat{F}^1_p, \dots, \hat{F}^M_p) $$
   The resulting $\bar{F}_p$ explicitly preserves general geometric patterns retrieved from 2D observations while being aware of the query point's 3D location [S1].

**Parameters:**
- The number of views $M$ is arbitrary [S1].
- The dimension $d$ of the feature vectors is determined by the feature extraction network [S1].
- The MLPs are shared across all views [S1].

**Edge Cases and Pitfalls:**
- **Occlusion:** The method is designed to handle occlusions by automatically selecting relevant features based on attention weights [S1].
- **Permutation Invariance:** The chosen attention mechanism must strictly maintain permutation invariance to handle the unordered nature of the input set [S1].

## Subtopics

### [View Transformer](14756-view-transformer.md)
The View Transformer is a module that aggregates multi-view features into a single projection feature to handle occlusions and ensure view consistency without assuming an order of source views. It addresses the limitation of mean/variance aggregation, which treats all views equally and fails to account for visibility and occlusion. By using self-attention, the model reasons about consistency across multiple views to produce a robust feature representation for a query point. It sets itself apart from simple statistical aggregation by learning to weight views based on visual consistency rather than fixed rules.

### [Variable Number of Conditioning Views Feature Aggregation](17954-variable-number-of-conditioning-views-feature-aggregation.md)
This method aggregates pixel-aligned features from an arbitrary number of input views into a single feature vector for a query 3D point, solving the problem of variable input dimensions at inference time. It differs from simple mean pooling by injecting camera parameters into the aggregation process to mitigate artifacts caused by depth ambiguity, and it differs from fixed-view concatenation by being permutation invariant to the number of conditioning views.

### Fixed Number of Conditioning Views Feature Aggregation
Aggregates features from a fixed number of conditioning images by concatenating them along the depth dimension to preserve feature information from all viewpoints [S2, unofficial].
**Formula:**
The feature extraction for the $i$-th image is:
$$ f_X^{(i)} = F(f^{(i)}; x_i) $$
Where $F$ is the feature extraction network, $f^{(i)}$ is the feature map of the $i$-th image, and $x_i$ is the 2D projection of 3D point $X$ onto the $i$-th image.
The projection is defined as:
$$ x_i = \Pi(X; K_i [R|t]_i) $$
Where $\Pi$ is the projection function using camera intrinsic $K_i$ and extrinsic $[R|t]_i$ of the $i$-th viewpoint.
The final aggregated feature is the concatenation:
$$ f_X = [f_X^{(1)} \dots f_X^{(n)}] $$
**Parameter:**
$\phi(X)$: positional encoding of $X \in \mathbb{R}^3$, mapping to $\mathbb{R}^{6 \times l}$ with $2 \times l$ different basis functions [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources. The complexity depends on the specific attention mechanism used (e.g., AttSets) and the number of views $M$.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
- **Strengths:** Handles visual occlusions by selecting relevant features; processes an arbitrary number of input views; permutation invariant [S1].
- **Limitations:** Requires careful selection of the attention mechanism to ensure permutation invariance [S1]. Simple mean aggregation fails to account for visibility and occlusion [S2, unofficial].

**When to use it:**
Use when dealing with multi-view inputs where occlusions are present and the number of views may vary or is large [S1].

**When not to use it:**
Not suitable when a fixed, simple concatenation of a known small number of views is sufficient and computational efficiency is paramount, as attention mechanisms add overhead [S2, unofficial].

## Background
Originating from the GRF (General Radiance Field) paper, this method formulates the aggregation of pixel features as an attention process to tackle issues of visual occlusions and arbitrary input counts [S1]. It builds on theoretical analysis from AttSets and Slot Attention to ensure permutation invariance [S1].

## Sources
[S1] GRF: Learning a General Radiance Field for 3D Representation and Rendering, https://doi.org/10.1109/iccv48922.2021.01490, fetched 2026-10-26
[S2] PVA: Pixel-aligned Volumetric Avatars, https://doi.org/10.48550/arxiv.2101.02697, fetched 2026-10-30 (unofficial)
