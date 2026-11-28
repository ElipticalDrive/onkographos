# Neural Rendering


## Overview
Neural rendering learns the rendering process directly from data rather than relying on handcrafted analytical differentiation of physical rendering equations [S1, unofficial]. It jointly trains a scene generation network and a rendering network by minimizing image reconstruction error, producing high-quality images indistinguishable from real-world captures [S1, unofficial]. Unlike traditional differentiable rendering that explicitly computes gradients through geometric primitives, neural rendering bypasses explicit differentiation by learning the input-output mapping, though it faces limitations in generalization, multi-object scaling, and human editability [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of images representing the target scene or domain, optionally with camera poses [S1, unofficial].
*   **Outputs:** A trained pair of neural networks: a scene generation network (encoding the scene) and a rendering network (mapping the representation to pixel colors) [S1, unofficial].

**Data Structures:**
*   **Scene Generation Network:** Outputs a neural scene representation for given 3D coordinates and viewing directions [S1, unofficial].
*   **Rendering Network:** Maps the scene representation and camera parameters to predicted pixel colors [S1, unofficial].
*   The specific architecture (e.g., MLP, CNN, or voxel grid) is not fixed by the method definition but is selected based on applied inductive biases [S1, unofficial].

**Steps:**
1.  **Scene Encoding:** For a given ray, sample points and pass them through the scene generation network to obtain features or field values [G].
2.  **Rendering:** Pass the features through the rendering network to predict color contributions [G].
3.  **Integration:** Combine the color contributions into a final image pixel [G].
4.  **Loss Calculation:** Compute the reconstruction error between the rendered image and the ground truth image [G].
5.  **Optimization:** Update the parameters of both networks jointly using gradient descent to minimize the loss [S1, unofficial].

**Pitfalls:**
*   **Generalization:** Struggles to generalize to scenes significantly different from the training data [S1, unofficial].
*   **Scaling:** Scaling to scenes composed of multiple distinct objects is limited [S1, unofficial].
*   **Editability:** Human modification of the generated scene is restricted due to the lack of explicit semantic or geometric structure [S1, unofficial].

## Subtopics

### Deferred Neural Rendering
Deferred Neural Rendering uses textures containing learnable view-dependent features (neural textures) [S2, unofficial]. A coarse mesh serves as the underlying 3D representation to rasterize these neural textures, and a neural network interprets the rasterized features in image space [S2, unofficial]. The network can be a pixel-wise MLP, in which case the neural texture represents surface radiance [S2, unofficial].

### [Rendering-Invariant State Predictor (RISP)](15076-rendering-invariant-state-predictor-risp.md)
RISP is a neural network $N_\theta$ that acts as an inverse renderer, mapping an input image $I$ to a predicted state $\hat{s}$ that is invariant to rendering parameters $\psi$. It solves the problem of optimizing physical system parameters from video by replacing sensitive image-space losses with robust state-space losses, allowing gradient-based optimization to succeed even when source and target rendering domains (e.g., lighting, materials) differ vastly. Unlike direct image-space optimization which traps gradient descent in local minima under domain shift, RISP enables optimization in a rendering-invariant state space.

### [Equivariant Neural Rendering](15993-equivariant-neural-rendering.md)
Equivariant Neural Rendering is a framework for learning neural scene representations directly from 2D images without requiring 3D ground truth or camera pose supervision at inference time. It solves the problem of novel view synthesis by enforcing that the learned implicit scene representation transforms like a real 3D scene, specifically satisfying the equivariance relation $T_X g(z) = g(T_Z z)$. Unlike sibling methods such as Rendering-Invariant State Predictor (RISP) which may rely on different invariance assumptions, this method explicitly models 3D rotation equivariance using deep voxel grids, allowing the model to generalize across camera transformations without absolute pose alignment.

## Operating characteristics
*   **Strengths:** Capable of generating high-quality images indistinguishable from real-world ones by learning from data, bypassing the need for accurate physical modeling [S1, unofficial].
*   **Limitations:** Limited generalization to unseen scenes, poor scalability to multi-object scenes, and low editability by humans [S1, unofficial].
*   **Performance:** Efficiency can be improved with coarse-to-fine sampling and importance sampling near boundaries [S1, unofficial].
*   **Usage:** Novel view synthesis, semantic photo manipulation, facial/body reenactment, relighting, free-viewpoint video, and photo-realistic avatar creation [S1, unofficial].

## Background
The concept was introduced by Eslami et al. as an alternative to handcrafting rendering differentiation, proposing to learn the rendering process from data [S1, unofficial]. It has become a prominent area of research, often combined with inductive biases for 3D scenes to improve performance [S1, unofficial].

## Sources
[S1] Differentiable Rendering: A Survey, https://doi.org/10.48550/arxiv.2006.12057, fetched 2026-10-26 (unofficial)
[S2] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
