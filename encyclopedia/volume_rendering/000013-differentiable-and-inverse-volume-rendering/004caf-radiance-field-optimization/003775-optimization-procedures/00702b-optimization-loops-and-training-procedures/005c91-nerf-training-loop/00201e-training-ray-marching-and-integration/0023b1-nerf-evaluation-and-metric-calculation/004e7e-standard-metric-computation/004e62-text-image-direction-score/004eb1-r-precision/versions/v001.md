# R-Precision
Also called: R-Precision Evaluation
## Overview
R-Precision measures how well a rendered view of a synthesis object aligns with the text caption used to generate it [S1]. It computes the precision of rendered views over a group of text captions using a retrieval model [S1]. It is distinct from pixel-based metrics (e.g., LPIPS) by evaluating semantic alignment rather than geometric fidelity [S1].
## How it works
**Inputs:**
- A set of $N$ text captions $\{c_1, \dots, c_N\}$ associated with a synthesis object [S1].
- A rendered view $I$ of the synthesis object [S1].
- A vision-language model backbone $M$ (CLIP or BLIP2) [S1].

**Data Structures:**
- Caption list: Array of strings.
- Image tensor: RGB image corresponding to the rendered view.

**Steps:**
1. **Dataset Construction:** Collect object-centric captions from the COCO dataset [S1]. Sample 20 captions for training [S1]. Use a total of 153 captions for evaluation [S1].
2. **View Selection:** Render the object at a 30° elevation [S2, unofficial]. This angle is chosen because it is not prone to overfitting issues seen in other methods [S2, unofficial].
3. **Retrieval:** For each caption $c_i$, compute the similarity score between the rendered image $I$ and the text $c_i$ using the backbone $M$ [S1].
4. **Precision Calculation:** Compute the precision of the rendered views over the group of text captions [S1]. Specifically, determine if the ground-truth caption is ranked within the top $k$ results (where $k$ is the number of relevant items, typically 1 for single-object retrieval) [S1].
5. **Averaging:** Average the metric over multiple azimuths to reduce variance [S2, unofficial].
6. **Seeding:** Evaluate all captions with 2 generation seeds unless otherwise noted [S2, unofficial].

**Parameters:**
- Elevation angle: 30° [S2, unofficial].
- Backbone models: CLIP [51] and BLIP2 [32] [S1].
- Caption count: 153 captions for evaluation [S1].
- Training captions: 20 captions sampled from COCO [S1].

**Edge Cases:**
- Overfitting: Evaluating at training viewpoints can lead to overfitting; 30° elevation mitigates this [S2, unofficial].
- Variance: Single-view evaluation may have high variance; averaging over azimuths reduces this [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
- Strengths: Evaluates semantic alignment with text prompts [S1]. The 30° elevation reduces overfitting issues [S2, unofficial].
- Limitations: Relies on the quality of the vision-language model backbone [S1].

**When to Use:**
Use when evaluating text-to-3D generation quality based on semantic alignment [S1].

## Background
R-Precision was introduced in the context of text-to-3D generation evaluation [S1]. It is inspired by retrieval metrics in information retrieval [S1]. The method uses CLIP and BLIP2 as backbones [S1].

## Sources
[S1] Blended-NeRF: Zero-Shot Object Generation and Blending in Existing Neural Radiance Fields, https://arxiv.org/abs/2306.12760v2, fetched 2026-10-26
[S2] DreamFusion: Text-to-3D using 2D Diffusion, https://doi.org/10.48550/arxiv.2209.14988, fetched 2026-10-27 (unofficial)
