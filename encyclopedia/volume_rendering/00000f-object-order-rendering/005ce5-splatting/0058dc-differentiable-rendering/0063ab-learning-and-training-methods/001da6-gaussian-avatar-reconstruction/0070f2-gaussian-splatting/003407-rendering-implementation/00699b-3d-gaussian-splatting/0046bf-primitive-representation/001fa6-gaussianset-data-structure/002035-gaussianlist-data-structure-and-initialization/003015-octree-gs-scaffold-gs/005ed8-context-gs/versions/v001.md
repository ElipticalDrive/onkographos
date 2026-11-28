# Context-GS

Also called: CompGS, ContextGS

## Overview
Context-GS is a structured compression method for 3D Gaussian Splatting that enhances representational capacity by explicitly modeling hierarchical relationships among anchor primitives [S1, unofficial][S2]. Unlike flat compression approaches, it employs a context-aware encoding scheme where higher-level anchors rely on context from coarser-level anchors for both encoding and decoding [S2]. This builds on the anchor-based representation introduced in [Octree-GS / Scaffold-GS](6142-octree-gs-scaffold-gs.md) by using sparse anchors as reference points to predict attributes of dependent Gaussians, thereby reducing redundancy and improving coding efficiency [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\mathcal{G}$, typically organized into a hierarchical anchor structure (e.g., from Scaffold-GS or Octree-GS).
*   **Output:** A compressed bitstream representing the Gaussian attributes (position, scale, rotation, color, opacity) using entropy coding guided by the context model.

**Data Structures:**
The method relies on a three-level hierarchical encoding scheme for anchors [S2].
1.  **Coarse Level:** Sparse anchor primitives that serve as reference points.
2.  **Intermediate/Fine Levels:** Dependent Gaussians whose attributes are predicted based on the context of previously decoded anchors.
*   *Note:* The exact bit-packing layout for the entropy-coded residuals is not specified in the sources [G: Standard arithmetic coding or probability estimation quantization (PEQ) is implied by the reference to autoregressive models in variants, but specific bit-depths or residual formats are "Not given in the sources."].

**Algorithm Steps:**
1.  **Hierarchical Organization:** Organize anchors into a hierarchy based on their spatial positions [S2].
2.  **Coarse-to-Fine Encoding:** Encode anchor attributes starting from the coarsest level to the finest level [S2].
3.  **Contextual Prediction:** For each anchor at a finer level, use the attributes of previously decoded anchors (from coarser levels or neighboring spatial positions) to predict its current attributes [S2].
4.  **Residual Coding:** Encode the difference (residual) between the predicted attributes and the actual attributes using entropy coding [S2].
5.  **Decoding:** Reverse the process: decode coarse anchors first, use them to predict and decode finer anchors, reconstructing the full Gaussian set [S2].

**Formulas:**
Let $A_i$ be the attributes of the $i$-th anchor. The probability distribution $P(A_i | \text{Context}_i)$ is modeled using the context from previously decoded anchors [S2]. The context $\text{Context}_i$ includes attributes of coarser-level anchors and spatially neighboring decoded anchors [S2].
*   *Note:* Specific mathematical formulations for the context model or the prediction function are not provided in the sources [G: The context model likely involves a neural network or statistical estimator, but details are "Not given in the sources."].

**Parameters:**
*   **Hierarchy Levels:** Three levels are specified [S2].
*   **Anchor Sparsity:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Boundary Conditions:** The method relies on spatial proximity for context; anchors at scene boundaries may have fewer neighbors, potentially affecting prediction accuracy [G: Standard handling involves padding or limiting context to available neighbors].
*   **Precision:** Quantization of residuals is implied but not detailed [G: Standard fixed-point or floating-point quantization is assumed].

## Subtopics

### CAT-3DGS
A variant that introduces a context-adaptive triplane-based hyperprior [S2]. It incorporates both spatial and channel-wise autoregressive (AR) models to push contextual modeling to a finer granularity [S2].
*   **Spatial AR Model:** Captures inter-anchor dependencies in the projected triplane space [S2].
*   **Channel-wise AR Model:** Leverages intra-feature correlations for coding latent attributes [S2].
*   **When to choose:** When finer granularity in contextual modeling is required, specifically leveraging spatial and channel-wise correlations via triplane projections [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enhances representational capacity through hierarchical context modeling; improves coding efficiency by exploiting spatial and hierarchical redundancies [S1, unofficial][S2].
*   **Limitations:** Complexity of hierarchical context modeling may increase encoding/decoding overhead compared to flat methods [G: Inferred from the complexity of the approach].
*   **When to use:** For compressing 3DGS scenes where hierarchical structure is available and high compression ratios are needed [S1, unofficial][S2].
*   **When not:** When real-time decoding speed is critical and hierarchical context overhead is prohibitive [G: Inferred].

## Background
Context-GS and its variant CompGS were introduced concurrently as context-aware approaches for 3DGS compression [S1, unofficial][S2]. They build upon anchor-based representations (such as those in Scaffold-GS) to improve convergence quality and storage efficiency for large scenes [S1, unofficial]. The approach is inspired by autoregressive models commonly used in learned image compression (LIC) [S2]. CompGS specifically exploits contextual information through a hybrid primitive structure [S2]. CAT-3DGS further extends this line of work by introducing triplane-based hyperpriors [S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Compression in 3D Gaussian Splatting: A Survey of Methods, Trends, and Future Directions, https://doi.org/10.1109/TCSVT.2026.3676048, fetched 2026-10-26
