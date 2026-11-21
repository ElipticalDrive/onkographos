# Dynamic 3D Gaussian Splatting (Katsumata et al.)
Also called: Dynamic 3DGS, Fourier-based Dynamic 3DGS, Katsumata et al
## Overview
Dynamic 3D Gaussian Splatting models dynamic scenes by keeping Gaussian properties (scale, opacity, spherical harmonics) unchanged over time while parameterizing position and orientation as continuous functions of time [S1, S2]. It distinguishes itself from implicit deformation networks by using explicit, time-independent Fourier series for position and linear functions for rotation, ensuring temporal continuity and reducing overfitting [S2].
## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians with static attributes and a sequence of training images with known camera poses and timestamps $t$ [S1, S2].
*   **Output:** Optimized Fourier series coefficients for position and linear coefficients for rotation, minimizing rendering and flow losses [S2].

**Data Structures:**
Each Gaussian $i$ stores:
1.  **Static Attributes:** Base position $\mathbf{c}_0 \in \mathbb{R}^3$, base rotation $\mathbf{q}_0 \in \mathbb{R}^4$, scale $\mathbf{s} \in \mathbb{R}^3$, opacity $\alpha \in \mathbb{R}$, and spherical harmonics coefficients $\mathbf{c}_{SH}$ [S1, S2].
2.  **Dynamic Parameters:**
    *   **Position:** Fourier series coefficients $\{\mathbf{a}_k, \mathbf{b}_k\}_{k=0}^{K}$ for each spatial dimension [S2].
    *   **Rotation:** Linear coefficients $\mathbf{m}_q \in \mathbb{R}^4$ representing the rate of change of quaternion components [S2].
The total number of dynamic parameters per Gaussian is independent of time [S2].

**Formulas:**
The position $\mathbf{c}(t)$ and rotation $\mathbf{q}(t)$ at time $t$ are computed as:
$$ \mathbf{c}(t) = \mathbf{c}_0 + \sum_{k=0}^{K} \left( \mathbf{a}_k \cos(k \omega t) + \mathbf{b}_k \sin(k \omega t) \right) $$
$$ \mathbf{q}(t) = \mathbf{q}_0 + \mathbf{m}_q t $$
where $\omega$ is the base frequency [S2].

**Optimization Steps:**
1.  **Initialization:** Initialize Gaussians from a static reconstruction [G]. Set dynamic coefficients $\mathbf{a}_k, \mathbf{b}_k, \mathbf{m}_q$ to zero or small random values [G].
2.  **Forward Pass:** For each training image at time $t$:
    *   Compute $\mathbf{c}(t)$ and $\mathbf{q}(t)$ using the formulas above [G].
    *   Rasterize the Gaussians using standard 3DGS rasterization with the time-dependent position and rotation [G].
    *   Render the image $\hat{I}$ [G].
3.  **Loss Calculation:**
    *   **Image Loss:** Compute image losses between the rendered image $\hat{I}$ and the ground truth $I_{gt}$ [S2].
    *   **Optical Flow Loss:** Compute bidirectional optical flow between rendered frames or between rendered and ground truth frames, and add $L_{flow}$ to penalize temporal inconsistencies [S2].
    *   **Total Loss:** $L = L_{img} + \lambda_{flow} L_{flow}$ [G].
4.  **Backpropagation:** Update Fourier coefficients and linear rotation parameters using gradient descent to minimize the total loss [S2].

**Parameters:**
*   $K$: Number of Fourier harmonics. Not given in the sources.
*   $\omega$: Base frequency. Not given in the sources.
*   $\lambda_{flow}$: Weight for optical flow loss. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Periodicity Assumption:** The Fourier series assumes periodic motion, which aligns with the observation that human and articulated object motion often exhibits periodic characteristics [S2]. Non-periodic motions may require higher $K$ or hybrid models [S2].
*   **Temporal Continuity:** The parametric functions are continuous with respect to time, which ensures temporal continuity and robustness for novel view synthesis [S2].
## Operating characteristics
*   **Time Complexity:** Optimization time is independent of the number of time steps because the parameterization is time-independent [S2]. Rendering time per frame is comparable to static 3DGS [G].
*   **Memory Complexity:** Storage increases by $O(K)$ per Gaussian for Fourier coefficients and $O(1)$ for linear rotation parameters [G].
*   **Strengths:** Fast training due to explicit parameterization; temporal continuity ensured by continuous functions; robust against overfitting compared to implicit deformation fields [S2].
*   **Limitations:** Assumes smooth or periodic motion; may struggle with abrupt, non-periodic changes without high-frequency terms [S2].
## Background
Dynamic 3D Gaussian Splatting was proposed by Katsumata et al. [S2]. It was introduced as an explicit deformation method to address the overfitting and high parameter count associated with implicit deformation modeling in NeRF-based approaches [S2]. The method leverages the observation that human and articulated object motion often exhibits periodic characteristics, justifying the use of Fourier series for position modeling [S2].
## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
