# CompGS
Also called: Efficient 3D Scene Representation via Compressed Gaussian Splatting
## Overview
CompGS is a compression method for 3D Gaussian Splatting that reduces storage and transmission costs by replacing standard independent primitives with a hybrid structure of anchor and coupled primitives [S1]. It differs from sibling methods like [Mini-Splatting](7850-mini-splatting.md) and [C3DGS](9184-c3dgs.md) by explicitly modeling dependencies between primitives to predict coupled attributes from anchors, rather than relying solely on quantization or pruning of independent Gaussians [S1].
## How it works
**Inputs and Outputs:**
Input: A set of $N$ standard 3D Gaussians $\{G_i\}$ with attributes $\{\mathbf{p}_i, \mathbf{q}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_{i, lm}\}$ [S1].
Output: A compressed representation consisting of anchor primitives and residual data for coupled primitives [S1].

**Data Structures:**
The scene is partitioned into two sets:
1.  **Anchor Primitives ($A$):** A subset of primitives whose full attributes are stored explicitly.
2.  **Coupled Primitives ($C$):** The remaining primitives whose attributes are predicted from anchors.

For each coupled primitive $j \in C$, the method stores:
*   An index or spatial reference to its associated anchor(s) $k \in A$.
*   Residual attributes $\Delta \mathbf{p}_j, \Delta \mathbf{q}_j, \Delta \mathbf{s}_j, \Delta \alpha_j, \Delta \mathbf{c}_{j, lm}$ relative to the predicted values.

**Steps:**
1.  **Hybrid Structure Construction:** Partition the original Gaussian set into anchors and coupled primitives. The selection strategy is not detailed in the provided sources [G].
2.  **Attribute Prediction:** For each coupled primitive $j$, predict its attributes $\hat{\mathbf{p}}_j, \hat{\mathbf{q}}_j, \hat{\mathbf{s}}_j, \hat{\alpha}_j, \hat{\mathbf{c}}_{j, lm}$ based on the attributes of its associated anchor(s) [S1]. The specific prediction function (e.g., direct copy, linear interpolation) is not defined in the sources [G].
3.  **Residual Calculation:** Compute the residuals between the original attributes and the predicted attributes:
    $$ \Delta \mathbf{x}_j = \mathbf{x}_j - \hat{\mathbf{x}}_j $$
    where $\mathbf{x}_j$ represents any attribute of coupled primitive $j$ [S1].
4.  **Rate-Constrained Optimization:** Optimize the representation by jointly minimizing rendering distortion and bit rate [S1]. The optimization problem is formulated as:
    $$ \min_{\text{params}} \mathcal{L}_{distortion} + \lambda \mathcal{R} $$
    where $\mathcal{L}_{distortion}$ is the rendering error, $\mathcal{R}$ is the estimated bit rate, and $\lambda$ is the Lagrange multiplier [S1].
5.  **Entropy Estimation:** Model the bit rate $\mathcal{R}$ for both anchor and coupled primitives using entropy estimation [S1]. The rate for anchors is determined by the precision of their stored attributes. The rate for coupled primitives is determined by the entropy of the residuals $\Delta \mathbf{x}_j$ [S1].
6.  **Encoding:** Encode the anchor attributes and the residuals using the estimated entropy models [S1].

**Parameters:**
*   $\lambda$: Rate-distortion trade-off parameter. Value not given in the sources.
*   Anchor/Coupled ratio: Not specified in the sources.

**Edge Cases and Pitfalls:**
*   Prediction Accuracy: If the prediction of coupled attributes from anchors is poor, residuals may be large, increasing bit rate [S1].
*   Precision: Residuals must be quantized appropriately to balance rate and distortion [S1].
## Subtopics

### [PCGS (Progressive Compression of 3D Gaussian Splatting)](20013-pcgs-progressive-compression-of-3d-gaussian-splatting.md)
PCGS is a progressive compression framework for 3D Gaussian Splatting (3DGS) that jointly and adaptively controls both the quantity and quality of anchor primitives to enable effective progressivity for on-demand applications.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.
**Measured Performance:**
Not given in the sources.
**Strengths and Limitations:**
*   Strength: Reduces redundancy by exploiting correlations between primitives [S1].
*   Limitation: Requires a prediction mechanism and entropy modeling, adding complexity compared to simple quantization [S1].
## Background
**Origin:**
CompGS was proposed in the paper "CompGS: Efficient 3D Scene Representation via Compressed Gaussian Splatting" [S1].
**History:**
It is part of the broader effort to compress 3D Gaussian Splatting representations, building on the standard 3DGS format [S1].
## Sources
[S1] 3DGS.zip: A survey on 3D Gaussian Splatting Compression Methods, https://arxiv.org/abs/2407.09510v5, fetched 2026-10-26
