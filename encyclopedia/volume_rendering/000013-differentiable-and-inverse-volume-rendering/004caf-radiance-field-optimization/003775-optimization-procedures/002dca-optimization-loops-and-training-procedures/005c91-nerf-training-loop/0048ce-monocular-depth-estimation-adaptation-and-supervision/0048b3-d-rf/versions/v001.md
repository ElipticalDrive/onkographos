# DäRF

Also called: Monocular Depth Adaptation for boosting Radiance Fields from Sparse Input Views

## Overview
DäRF is a framework for robust few-shot Neural Radiance Field (NeRF) reconstruction that combines NeRF with monocular depth estimation (MDE) through online complementary training [S1, unofficial]. It solves the problem of poor generalization and geometric ambiguity in sparse-view NeRF by injecting MDE's geometry prior into the NeRF representation at both seen and unseen viewpoints [S1, unofficial]. Unlike SCADE [50], which applies pretrained MDE only to seen views, DäRF jointly optimizes NeRF and MDE at a specific scene, adapting MDE to NeRF's absolute scaling and multiview consistency while using MDE to regularize NeRF [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Inputs: A set of sparse input images $S = \{I_i\}$ with known ground truth camera poses (typically obtained via COLMAP [43]), and a pretrained MDE network.
- Outputs: A NeRF representation $F_\theta(\cdot)$ and an adapted MDE network that produces multiview-consistent depth maps.

**Data Structures:**
- NeRF Network $F_\theta$: A multi-layer perceptron (MLP) parameterized by $\theta$. It takes a 3D coordinate $\mathbf{x} \in \mathbb{R}^3$ and a viewing direction $\mathbf{d} \in \mathbb{R}^2$ as input, outputting color $\mathbf{c}$ and volume density $\sigma$ [S1, unofficial].
- MDE Network: A pretrained neural network that estimates depth from a single image.

**Numbered Steps:**
1. **Ray Sampling:** For each training image, sample rays $\mathbf{r}$ originating from the camera. Sample 3D points $\mathbf{x}$ along these rays [S1, unofficial].
2. **NeRF Forward Pass:** Query $F_\theta(\mathbf{x}, \mathbf{d})$ to obtain density $\sigma$ and color $\mathbf{c}$ for each point. Render the image using volume rendering [S1, unofficial].
3. **Monocular Depth Estimation:** Pass the input image through the MDE network to obtain a predicted depth map $D_{pred}$ [S1, unofficial].
4. **Patch-wise Scale-Shift Fitting:** To map local depths from MDE to NeRF geometry, apply a novel patch-wise scale-shift fitting technique. Global scale-shift fitting is insufficient due to poor MDE performance on depth differences between multiple objects. This step computes local affine transformations (scale and shift) for patches to align $D_{pred}$ with NeRF's geometry [S1, unofficial].
5. **Confidence Modeling:** Apply a confidence modeling technique to verify accurate depth information. This handles ambiguity problems such as relative depths not being multiview-consistent and convexity issues [S1, unofficial].
6. **Joint Optimization (Complementary Training):**
   - **NeRF Adaptation:** Use the adapted MDE depth as a geometric prior to regularize the NeRF optimization, removing artifacts and improving robustness in both seen and unseen viewpoints [S1, unofficial].
   - **MDE Adaptation:** Leverage the NeRF representation to adapt the MDE network. By finetuning MDE with the canonical geometry captured by NeRF, ambiguity present in the initial MDE output is directly removed, ensuring multiview consistency [S1, unofficial].
7. **Loss Computation:** Compute losses based on the rendered image, the ground truth image, and the depth consistency between the adapted MDE and NeRF geometry [S1, unofficial].
8. **Update:** Update parameters $\theta$ for NeRF and the MDE network weights jointly [S1, unofficial].

**Formulas:**
- NeRF Rendering: Standard volume rendering equation is used, where the expected color $C(\mathbf{r})$ for a ray $\mathbf{r}$ is computed from sampled points $\mathbf{x}_i$ with densities $\sigma_i$ and colors $\mathbf{c}_i$ [S1, unofficial].
- Patch-wise Scale-Shift: For each patch, a local scale $s$ and shift $t$ are fitted to minimize the difference between the transformed MDE depth and the NeRF-derived depth [S1, unofficial].

**Parameters:**
- Not given in the sources.

**Edge Cases and Pitfalls:**
- **Global Scale-Shift Failure:** Global fitting fails when there are large depth differences between multiple objects; patch-wise fitting is required [S1, unofficial].
- **MDE Ambiguity:** Monocular depth suffers from relative depth ambiguity and convexity problems; confidence modeling is needed to verify accuracy [S1, unofficial].
- **Multiview Inconsistency:** MDE outputs may not be consistent across views; joint optimization with NeRF resolves this [S1, unofficial].

## Subtopics

### [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md)
General techniques for using MDE in NeRF training. DäRF specifically uses online complementary training and patch-wise fitting.

### [SCADE](50-scade.md)
A related method that distills depths from pretrained MDE only at seen views and uses probabilistic modeling for uncertainty. DäRF differs by jointly optimizing MDE and NeRF and using patch-wise distillation.

## Operating characteristics
- **Time Complexity:** Joint optimization of NeRF and MDE increases training time compared to standard NeRF or methods that only use pretrained MDE as a static prior.
- **Memory Complexity:** Requires storing parameters for both NeRF and the MDE network.
- **Measured Performance:** Achieves state-of-the-art results quantitatively and qualitatively on real-world indoor and outdoor scene datasets for few-shot NeRF optimization [S1, unofficial].
- **Strengths:** Robust geometry reconstruction, artifact removal in unseen viewpoints, multiview consistency, and effective handling of sparse inputs.
- **Limitations:** Relies on the quality of the pretrained MDE network; joint optimization may be computationally intensive.
- **When to use:** When reconstructing scenes from a small number of input images (few-shot) where standard NeRF fails due to lack of geometric constraints.
- **When not to use:** When abundant input views are available, or when real-time training is required without the benefit of MDE adaptation.

## Background
DäRF was proposed to address the weaknesses of methods that use 3D priors extracted from a few input images only, which prevents guidance from encompassing the entire scene [S1, unofficial]. It builds on the idea that pretrained monocular depth estimation (MDE) networks have strong generalization capability [39, 38, 5] and can inject an additional 3D prior into NeRF [S1, unofficial]. Unlike SCADE [50], which distills depths by applying pretrained MDE to NeRF at seen views only, DäRF fully exploits MDE by jointly optimizing NeRF and MDE at a specific scene [S1, unofficial]. The method was introduced in the paper "DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation" [S1, unofficial].

## Sources
[S1] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
