# Occlusion-Eliminating NeRF Variant

Also called: Occlusion-Eliminating NeRF, Zhu et al. NeRF

## Overview
This method addresses novel view synthesis degradation caused by foreground occlusions obscuring background features during training [S1, S2]. It distinguishes itself from standard NeRF by explicitly modeling the scene and the occluding background as separate components, allowing for the recovery of occlusion-free background information [S1, S2]. Unlike siblings that rely on generic regularization or diffusion priors, this variant uses a dual-MLP architecture with specific cost volume and depth inconsistency losses to disentangle foreground objects from background artifacts [S1, S2].

## How it works
The method employs a neural network architecture composed of two distinct Multi-Layer Perceptrons (MLPs): a Scene MLP and a Background MLP [S1, S2].

**Inputs and Outputs:**
*   **Inputs:** Camera parameters (position, orientation), 3D coordinates $\mathbf{x}$, and viewing directions $\mathbf{d}$ [G].
*   **Outputs:** Volume density and color predictions for both the scene and background components, which are combined during rendering [G].

**Data Structures:**
*   **Scene MLP:** A standard NeRF-style MLP mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma_s, \mathbf{c}_s)$ [G].
*   **Background MLP:** A separate MLP mapping $(\mathbf{x}, \mathbf{d}) \to (\sigma_b, \mathbf{c}_b)$ [G].
*   **Cost Volume:** A 3D structure used to guide the joint optimization of scene parameters and camera poses [S1, S2].

**Training Procedure:**
1.  **Initialization:** Initialize the Scene MLP and Background MLP with random weights [G].
2.  **Cost Volume Loss:** Construct a cost volume to guide the joint optimization of the Scene MLP parameters and the camera parameters [S1, S2].
3.  **Depth Bidirectional Inconsistency Prior:** Apply a depth bidirectional inconsistency prior within the Background MLP to restore occlusion-free backgrounds via depth distance supervision [S1, S2].
4.  **Mask Learning:** Train the background MLP and remove occlusions from aggregated information from the scene MLP to determine if the output of the background NeRF matches the observed color, learning a mask from the ray's weights [S1].
5.  **Depth Constraint:** Include a depth constraint for probing occluded areas by comparing the depth of occlusion and background, based on the assumption that occlusions are foreground objects at a closer distance [S1].
6.  **Total Loss:** Minimize the sum of the photometric loss and the regularization terms derived from the cost volume and depth priors [G].

**Formulas:**
Exact loss formulas are not given in the sources.

**Parameters:**
Typical parameter values are not given in the sources.

**Edge Cases and Pitfalls:**
*   **Transparent Occluders:** When the occluding object is a light-loss transparent glass body, reflection artifacts introduce noise pollution into the training images [S1, S2]. This prevents the model from generalizing effectively, as the background MLP struggles to distinguish between true background geometry and reflection artifacts [S1, S2].

## Operating characteristics
*   **Strengths:** Capable of recovering background information that is occluded in training views, improving synthesis quality in cluttered scenes [S1, S2].
*   **Limitations:** Performance degrades significantly in the presence of transparent or reflective occluders (e.g., glass) due to reflection artifacts [S1, S2].
*   **Complexity:** Requires maintaining and optimizing two separate MLPs and additional depth-based loss computations, increasing memory and computational overhead compared to single-MLP NeRF [S1, S2].

## Background
The method was proposed by Zhu et al. in 2023 [S1, S2]. It was developed to solve the problem of feature damage in NeRF synthesis caused by foreground occlusions [S1, S2]. It is part of the broader effort to improve NeRF robustness in real-world scenarios where perfect unobstructed views are unavailable [S1, S2]. A follow-up model, NeRFReN by Guo et al., addresses the glass reflection limitation by utilizing geometric priors for depth smoothness and bidirectional depth consistency losses to separate reflection artifacts from physical entities [S1].

## Sources
[S1] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
