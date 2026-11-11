# NeRF Editing and Inpainting Procedures


## Overview
This family encompasses optimization procedures that modify a pre-trained Neural Radiance Field (NeRF) to achieve specific editing goals, such as object removal, semantic segmentation, or content insertion, without retraining the entire scene from scratch. Unlike standard NeRF training which assumes multi-view consistency of the original data, these methods introduce external guidance (e.g., 2D inpainting, semantic features, or text prompts) and optimize the radiance field parameters $\Theta$ to align with this guidance while preserving unedited regions. They differ from sibling methods like [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md) by focusing on post-hoc modification of an existing representation rather than incremental scene expansion.

## Shared principles
All methods in this family operate on a pre-trained NeRF model $F_\Theta$ that maps 3D coordinates $\mathbf{x}$ and viewing directions $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$. The rendering equation remains the standard volume rendering integral [S1, 12372]:
$$
\hat{C}(r) = \sum_{i=1}^{N} T(t_i)(1 - \exp(-\sigma(t_i)\delta_i))c(t_i)
$$
where $T(t_i) = \exp(-\sum_{j=1}^{i-1} \sigma(t_j)\delta_j)$ [S1, 12372].

The core mechanism involves defining a loss function $L$ that combines photometric consistency with an editing-specific term. The optimization updates $\Theta$ to minimize this loss. Common editing terms include:
1.  **Inpainting Guidance:** Using 2D images inpainted by external models (e.g., Stable Diffusion) as target colors or depths for the NeRF to match [S1, 12372].
2.  **Feature Distillation:** Matching rendered features to those from a 2D teacher network (e.g., LSeg, DINO) to enable semantic editing [S1].
3.  **Text/CLIP Guidance:** Minimizing distance between rendered regions and text embeddings to guide content generation [S1].
4.  **Geometric Constraints:** Using depth or transmittance losses to maintain volumetric structure during editing [S1].

## Subtopics

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
A procedure for expanding a single-view NeRF into a complete 3D representation by iteratively synthesizing novel views and refining geometry. It solves the problem of geometry conflicts arising from inpainting invisible regions with external generative models, which may violate the consistency constraints of the already-learned visible geometry. Unlike standard NeRF training, it explicitly detects and erases conflicting regions from the supervision loss.

### [DFF (Decomposing NeRF for Editing Via Feature Field Distillation)](12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md)
Enables 3D semantic segmentation and editable novel-view synthesis by distilling features from a pre-trained 2D teacher network (LSeg or DINO) into a neural radiance field. It solves the problem that standard NeRF feature spaces are entangled with low-level factors like color or spatial adjacency rather than semantic similarity. Unlike sibling methods that rely on explicit masks, DFF achieves multi-view consistent decomposition through feature distillation.

### [Blended-NeRF](12273-blended-nerf.md)
A framework for editing a specific region of interest (ROI) in an existing NeRF scene based on text prompts and a user-defined 3D ROI box. It solves the problem of local 3D editing by initializing a generator NeRF with the weights of the original scene and blending the synthesized content with the original scene using volumetric blending. Unlike methods that synthesize objects from scratch, it operates on any region using a 3D box without learning a new feature space.

### [NeRF-In Inpainting Optimization](12372-nerf-in-inpainting-optimization.md)
A framework for removing unwanted objects from a pre-trained NeRF by optimizing model parameters $\Theta$ using color and depth guidance derived from 2D images, without requiring category-specific training data. It sets itself apart by explicitly formulating a joint optimization problem that minimizes both color-guiding and depth-guiding losses against inpainted RGB-D priors, ensuring geometric consistency and view coherence.

### NeRF Editing Procedure
A generic editing procedure based on the DFF framework that allows for deletion, colorization, and geometric transformation of semantic segments [S1].
1.  Sample points $[\dots, x_i, \dots]$ on a ray as usual in NeRF [S1].
2.  Query the DFF for each point to calculate probability $p$ of coordinate matching a set of queries [S1].
3.  Define 'coordinate selected by query' if $p$ is above user-defined threshold, otherwise 'not selected' [S1].
4.  If not selected, calculate density $\sigma(x)$ and color $c(x)$ at $x$ via vanilla NeRF [S1].
5.  If selected, apply transforms:
    *   **Deletion:** Sets density $\sigma(x)$ to zero [S1].
    *   **Color editing:** Queries NeRF for $\sigma(x)$ and $c(x)$, then transforms color to $b(c(x))$ via colorization function $b$ [S1].
    *   **Translation/rescaling:** Computes deformed point coordinate $x_0 = g^{-1}(x)$ where $g$ is inverse editing transformation [S1]. If $x_0$ is selected by query, use $x_0$ instead of $x$ for calculating color and density [S1]. If both $x$ and $x_0$ are selected and have non-zero density, mix colors $c(x)$ and $c(x_0)$ in ratio of their alphas at the point for simplicity [S1].
6.  Perform volume rendering with series of (density, color) tuples [S1].

## Background
The concept of editing NeRFs emerged as the technology matured from static scene reconstruction to interactive 3D content creation. Early work focused on simple object removal using inpainting priors [12372]. The introduction of feature distillation, specifically Decomposing NeRF for Editing Via Feature Field Distillation (DFF), marked a significant shift by enabling semantic-aware editing through 2D foundation models like LSeg and DINO [S1]. Subsequent methods like Blended-NeRF and Progressive Inpainting-and-Erasing (PERF) addressed specific limitations such as local editing consistency and single-view expansion, respectively [S1]. These procedures are widely used in 3D content creation pipelines, virtual reality asset modification, and digital twin maintenance.

## Sources
[S1] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
