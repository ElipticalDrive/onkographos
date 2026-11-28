# View-Dependent Prompting

Also called: Directional View Suffixing

## Overview
View-Dependent Prompting augments text prompts with directional suffixes based on the camera's position relative to the scene to guide diffusion-based 3D generation [S1, unofficial][S2]. It differs from sibling methods like [TexGS-VolVis Representation and Optimization](11105-texgs-volvis-representation-and-optimization.md) by explicitly conditioning the text embedding on the viewing angle rather than relying solely on volumetric or geometric consistency [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A base text prompt $P$, the sampled camera azimuth $\theta_{cam}$, and the camera elevation $\phi_{cam}$.
*   **Output:** An augmented text prompt $P'$ containing a view-dependent suffix.

**Data Structures:**
*   **Prompt Suffixes:** A set of discrete strings: "front view", "side view", "overhead view", "bottom view".

**Steps:**
1.  **Determine Suffix:** Based on $\theta_{cam}$ and $\phi_{cam}$, select a suffix $S$ according to the rules below [S2]:
    *   If $\phi_{cam} > 60^\circ$: $S = \text{"overhead view"}$ [S2].
    *   If $\phi_{cam} < 0^\circ$: $S = \text{"bottom view"}$ [S2].
    *   If $|\theta_{cam}| \le 30^\circ$: $S = \text{"front view"}$ [S2].
    *   If $30^\circ < |\theta_{cam}| \le 90^\circ$: $S = \text{"side view"}$ [S2].
    *   If $90^\circ < |\theta_{cam}| \le 180^\circ$: $S = \text{"bottom view"}$ [S2].
    *   *Note:* Source [S1, unofficial] mentions interpolating between front/side/back augmentations based on the quadrant of $\theta_{cam}$, but explicitly states that taking the text embedding closest to the sampled azimuth worked better [S1, unofficial]. Source [S2] uses the discrete suffix rules above [S2].

2.  **Augment Prompt:** Append $S$ to the base prompt $P$ to form $P'$ [S2].

3.  **Encode:** Pass $P'$ to the frozen text encoder (e.g., CLIP) to obtain the view-dependent text embedding [S1, unofficial][S2].

**Parameters:**
*   **Elevation Thresholds:** $60^\circ$ (overhead), $0^\circ$ (bottom) [S2].
*   **Azimuthal Bands:** $\pm 30^\circ$ (front), $\pm 30^\circ-90^\circ$ (side), $\pm 90^\circ-180^\circ$ (bottom) [S2].

**Edge Cases and Pitfalls:**
*   **Disagreement:** [S1, unofficial] suggests interpolation of embeddings, while [S2] uses discrete suffix selection. Implementers should prefer the discrete suffix method if following [S2], or the closest-embedding method if following [S1, unofficial] [S1, unofficial][S2].
*   **Ambiguity:** The term "bottom view" is used for both low elevation ($\phi_{cam} < 0^\circ$) and high azimuth ($|\theta_{cam}| > 90^\circ$) in [S2] [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for suffix selection; dominated by the text encoder inference.
*   **Memory Complexity:** $O(1)$ additional memory for the suffix string.
*   **Strengths:** Improves geometric coherence by providing explicit directional context to the diffusion model [S1, unofficial][S2].
*   **Limitations:** Discrete suffixes may introduce abrupt changes in the text embedding as the camera moves across boundaries [S1, unofficial].

## Background
Originated in text-to-3D diffusion methods to mitigate view inconsistency [S1, unofficial][S2]. [S1, unofficial] (DreamFusion) explored interpolation but found nearest-neighbor embedding selection effective [S1, unofficial]. [S2] (RealFusion) formalized the discrete suffix approach based on camera quadrants [S2].

## Sources
[S1] DreamFusion: Text-to-3D using 2D Diffusion, https://doi.org/10.48550/arxiv.2209.14988, fetched 2026-10-27 (unofficial)
[S2] RealFusion 360° Reconstruction of Any Object from a Single Image, https://doi.org/10.1109/cvpr52729.2023.00816, fetched 2026-10-30
