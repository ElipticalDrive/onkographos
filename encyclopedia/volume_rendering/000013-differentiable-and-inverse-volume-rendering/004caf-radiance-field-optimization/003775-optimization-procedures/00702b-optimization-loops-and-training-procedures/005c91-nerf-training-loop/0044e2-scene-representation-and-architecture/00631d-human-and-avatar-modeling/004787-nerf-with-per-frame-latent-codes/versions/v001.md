# NeRF with Per-Frame Latent Codes

Also called: NeRFace, NeRF with Facial Expression Coefficients

## Overview
NeRF with Per-Frame Latent Codes (NeRFace) extends the static NeRF representation to handle dynamic, non-rigid scenes, specifically human faces, by conditioning the neural radiance field on explicit deformation parameters and per-frame latent codes [S1][S2, unofficial]. Unlike static NeRF, which assumes a rigid scene, NeRFace uses a morphable model to control facial expressions and learnable latent vectors to compensate for tracking errors and unmodeled regions (e.g., shoulders) [S2, unofficial]. It is distinguished from sibling methods like [NeuMan](12480-neuman.md) (skeletal body deformation) and [AR-NeRF](12493-ar-nerf.md) by its use of blendshape coefficients for implicit facial control [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images (monocular video), camera poses, and per-frame facial blendshape parameters (76-dimensional) derived from a face tracker [S2, unofficial].
*   **Outputs:** Novel view synthesis of the face with arbitrary expressions, and expression editing capabilities [S2, unofficial].

**Data Structures:**
1.  **Blendshape Coefficients ($\mathbf{b}$):** A 76-dimensional vector representing the facial expression state, derived from the Face2Face tracker [S1][S2, unofficial].
2.  **Per-Frame Latent Code ($\mathbf{z}$):** A learnable vector unique to each training image. Its dimension is not specified in the sources [G: typical implementations use 16-64 dimensions].
3.  **Neural Radiance Field ($F$):** Two coupled MLPs (Density $F_\sigma$, Color $F_c$) as defined in [Scene Representation and Architecture](11368-scene-representation-and-architecture.md).

**Network Topology Modification:**
The input to the MLPs is modified to include the deformation conditioning.
1.  **Positional Encoding:** Standard positional encoding $\gamma(\mathbf{x})$ is applied to the 3D position $\mathbf{x}$ [S1].
2.  **Conditioning Vector:** The input to the networks is augmented with the blendshape coefficients $\mathbf{b}$ and the per-frame latent code $\mathbf{z}$ [S2, unofficial].
    *   The density network $F_\sigma$ takes $\gamma(\mathbf{x})$ and $\mathbf{b}$ as input.
    *   The color network $F_c$ takes $\gamma(\mathbf{x})$, viewing direction $\mathbf{d}$, the intermediate feature from $F_\sigma$, $\mathbf{b}$, and $\mathbf{z}$ as input [S2, unofficial].
    *   *Note:* The exact concatenation order or projection layers for $\mathbf{b}$ and $\mathbf{z}$ are not detailed in the provided sources [G: typically concatenated or projected via linear layers before MLP input].

**Numbered Steps:**
1.  **Tracking:** Use a face tracker (Face2Face) to reconstruct face blendshape parameters $\mathbf{b}_t$ and camera poses for each training view $t$ in a monocular video sequence [S2, unofficial].
2.  **Assumption:** Assume a known static background. The radiance field only stores information about the face [S2, unofficial].
3.  **Initialization:** Initialize a learnable per-frame latent code $\mathbf{z}_t$ for each training image $t$ [S2, unofficial].
4.  **Training:**
    a. For each training image $t$, sample rays $\mathbf{r}$.
    b. Sample points $\mathbf{x}_i$ along rays.
    c. Compute $\gamma(\mathbf{x}_i)$.
    d. Query $F_\sigma(\gamma(\mathbf{x}_i), \mathbf{b}_t)$ to get density $\sigma_i$.
    e. Query $F_c(\gamma(\mathbf{x}_i), \mathbf{d}, \text{feature}, \mathbf{b}_t, \mathbf{z}_t)$ to get color $\mathbf{c}_i$.
    f. Render pixel color $C(\mathbf{r})$ using volumetric rendering [S1].
    g. Optimize network weights and latent codes $\mathbf{z}_t$ using photometric loss against ground truth images [S2, unofficial].
5.  **Inference/Editing:**
    a. To synthesize a novel view/expression, provide target blendshape coefficients $\mathbf{b}_{target}$ and a latent code $\mathbf{z}$ (either from a reference frame or optimized) [S2, unofficial].
    b. Query the network with $\mathbf{b}_{target}$ to deform the radiance field implicitly [S2, unofficial].

**Formulas:**
The rendering integral remains standard:
$$
C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)
$$
Where $\sigma_i = F_\sigma(\gamma(\mathbf{x}_i), \mathbf{b}_t)$ and $\mathbf{c}_i = F_c(\gamma(\mathbf{x}_i), \mathbf{d}, \dots, \mathbf{b}_t, \mathbf{z}_t)$ [S1][S2, unofficial].

**Parameters:**
*   Blendshape dimension: 76 [S1][S2, unofficial].
*   Background: Static [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Tracking Errors:** The per-frame latent code $\mathbf{z}$ is critical for compensating for missing tracking information (e.g., shoulders) and tracking errors [S2, unofficial].
*   **Monocular Ambiguity:** Using monocular video limits the ability to disambiguate depth and motion, relying heavily on the morphable model prior [S2, unofficial].

## Subtopics

### [NeuMan](12480-neuman.md)
Uses skeletal deformation graphs for full-body human avatars. Choose when body motion is dominant.

### [AR-NeRF](12493-ar-nerf.md)
Uses attention-based mechanisms for dynamic scenes. Choose when complex non-rigid dynamics beyond faces are required.

### [Neural Body](12176-neural-body.md)
Uses skeletal deformation for body. Choose for full-body human rendering.

## Operating characteristics
*   **Time Complexity:** Training time is increased due to the optimization of per-frame latent codes $\mathbf{z}_t$ for every image [S2, unofficial].
*   **Memory Complexity:** Requires storing one latent code $\mathbf{z}_t$ per training image [S2, unofficial].
*   **Strengths:** Enables expression editing and reenactment via blendshape control [S2, unofficial]. Compensates for incomplete tracking data [S2, unofficial].
*   **Limitations:** Limited to faces (due to morphable model) [S2, unofficial]. Requires accurate initial tracking [S2, unofficial]. Monocular setup limits geometric fidelity compared to multi-view [S2, unofficial].
*   **When to use:** When the goal is facial reenactment or expression editing from monocular video.
*   **When not to use:** For general dynamic scenes or when multi-view data is available and higher geometric accuracy is needed.

## Background
NeRFace was introduced in December 2020 as a concurrent work to Nerfies [S1]. It is the first approach to use a morphable model to implicitly control a neural radiance field for face reenactment and expression editing [S2, unofficial]. It builds on the Face2Face tracker [210] to extract blendshape parameters [S1][S2, unofficial].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
