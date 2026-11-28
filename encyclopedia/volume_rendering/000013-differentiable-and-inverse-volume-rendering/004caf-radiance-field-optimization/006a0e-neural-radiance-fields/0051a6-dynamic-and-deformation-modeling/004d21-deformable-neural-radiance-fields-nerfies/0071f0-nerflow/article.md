# NeRFlow

Also called: Neural Radiance FLow, NeRF-Flow

## Overview
NeRFlow is a dynamic Neural Radiance Field method that models non-rigid scene deformation by predicting a continuous 3D scene flow field rather than explicit displacement offsets [S2, unofficial]. It distinguishes itself from sibling methods like Nerfies by using infinitesimal velocity vectors integrated via Neural Ordinary Differential Equations (Neural ODE) to determine point trajectories, enabling temporally consistent novel view synthesis [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of images with known camera poses and timestamps $t$. Spatial coordinates $\mathbf{x} \in \mathbb{R}^3$ and time $t$.
*   **Outputs:** Volume rendering of radiance $c$ and density $\sigma$ for novel viewpoints and time steps.

**Data Structures:**
1.  **Radiance Field Network ($\mathcal{N}_r$):** A multi-layer perceptron (MLP) taking 3D position $\mathbf{x}$ and viewing direction $\mathbf{v}$ as input, outputting density $\sigma$ and color $c$. This is identical to the static NeRF representation [S2, unofficial].
2.  **Flow Field Network ($\mathcal{N}_f$):** A 4D function taking spatial coordinates $\mathbf{x}$ and time $t$ as input, outputting an instantaneous 3D velocity vector $\mathbf{v}_{flow} \in \mathbb{R}^3$ [S2, unofficial].

**Algorithm Steps:**
1.  **Flow Prediction:** For a query point $\mathbf{x}$ at time $t$, the flow field network predicts the instantaneous velocity:
    $$ \mathbf{v}_{flow} = \mathcal{N}_f(\mathbf{x}, t) $$
    This vector describes how the point moves over time [S2, unofficial].

2.  **Trajectory Integration:** To obtain the displacement offset from a reference time (e.g., $t_0$) to the target time $t$, the velocity field is integrated using a Neural ODE solver [S1, S2, unofficial]. The position $\mathbf{x}(t)$ evolves according to:
    $$ \frac{d\mathbf{x}(t)}{dt} = \mathcal{N}_f(\mathbf{x}(t), t) $$
    The solution $\mathbf{x}(t)$ provides the deformed position corresponding to the canonical or reference space [S1, S2, unofficial].

3.  **Radiance Sampling:** The deformed position $\mathbf{x}(t)$ is passed to the radiance field network $\mathcal{N}_r$ along with the viewing direction to sample density and color [S2, unofficial].

4.  **Volume Rendering:** Standard volume rendering is performed along the camera ray, accumulating color and opacity based on the sampled densities and colors [S2, unofficial].

**Parameters:**
*   Neural ODE solver tolerance and step size (Not given in the sources).
*   Network architecture depth/width for $\mathcal{N}_r$ and $\mathcal{N}_f$ (Not given in the sources).

**Edge Cases and Pitfalls:**
*   **Static Backgrounds:** The method struggles to preserve static backgrounds, as the flow field may incorrectly predict motion for stationary regions [S1, unofficial].
*   **Complex Deformations:** Performance degrades on scenes with non-piecewise-rigid deformations and complex motions [S1, unofficial].
*   **Novel View Synthesis:** Rendering quality drops significantly for camera trajectories substantially different from the input training views [S1, unofficial].
*   **Depth Supervision Artifacts:** If supervised by monocular depth maps, artifacts from the depth estimator may appear in novel views [S1, unofficial].

## Subtopics

### Neural ODE Integration
Uses continuous-time modeling to integrate the flow field. Sets it apart from discrete warping methods. Choose when smooth temporal consistency is required. [Link to Neural ODE](link)

### Scene Flow Prediction
Predicts 3D velocity vectors. Sets it apart from displacement-based methods. Choose when explicit deformation fields are too rigid. [Link to Scene Flow](link)

## Operating characteristics
*   **Time Complexity:** Depends on the Neural ODE solver steps and the number of sampling points along the ray. Generally higher than static NeRF due to the additional flow network evaluation and ODE integration [S2, unofficial].
*   **Memory Complexity:** Requires storing parameters for both the radiance field and the flow field [S2, unofficial].
*   **Strengths:** Provides temporally consistent 4D view synthesis by propagating radiance fields across time using flow vectors [S2, unofficial].
*   **Limitations:** Difficulty with static backgrounds, complex non-rigid motions, and extreme novel view synthesis [S1, unofficial]. Accuracy may depend on monocular depth estimation if used for supervision [S1, unofficial].

## Background
NeRFlow was introduced by Du et al. [DZY* 21]. It is part of the broader effort to model dynamic scenes using neural radiance fields, moving from explicit deformation fields to implicit flow-based modeling [S1, S2, unofficial]. It is used for dynamic scene reconstruction, novel view synthesis, and can be applied to de-noising and super-resolving views of pre-trained scenes [S1, unofficial].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Dynamic Scene Reconstruction: Recent Advance in Real-time Rendering and Streaming, https://doi.org/10.48550/arXiv.2503.08166, fetched 2026-10-29 (unofficial)
