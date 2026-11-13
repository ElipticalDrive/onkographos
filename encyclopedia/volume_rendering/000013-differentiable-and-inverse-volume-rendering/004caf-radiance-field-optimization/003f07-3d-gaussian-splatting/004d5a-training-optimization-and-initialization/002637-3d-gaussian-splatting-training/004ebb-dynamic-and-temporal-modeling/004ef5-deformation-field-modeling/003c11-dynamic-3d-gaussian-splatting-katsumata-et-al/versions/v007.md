# Dynamic 3D Gaussian Splatting (Katsumata et al.)
Also called: Dynamic 3DGS, Fourier-based Dynamic 3DGS, Katsumata et al

## Overview
Dynamic 3D Gaussian Splatting models dynamic scenes by maintaining static Gaussian properties (opacity, scale, spherical harmonics) while parameterizing position and rotation as continuous functions of time [S1, S2]. It distinguishes itself from implicit deformation networks by using explicit, time-independent Fourier series for position and linear functions for rotation, ensuring temporal continuity and robustness against overfitting [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\{G_i\}$ with static attributes (base position $\mathbf{c}_0$, base rotation $\mathbf{q}_0$, scale $\mathbf{s}$, opacity $\alpha$, spherical harmonics coefficients $\mathbf{c}_{SH}$) and a sequence of images with known camera poses and timestamps $t$.
*   **Output:** Time-dependent Gaussian parameters $\mathbf{c}_i(t)$ and $\mathbf{q}_i(t)$, optimized to minimize rendering and flow losses.

**Data Structures:**
Each Gaussian $i$ stores:
1.  **Static Attributes:** $\mathbf{c}_0 \in \mathbb{R}^3$, $\mathbf{q}_0 \in \mathbb{R}^4$ (normalized quaternion), $\mathbf{s} \in \mathbb{R}^3$, $\alpha \in \mathbb{R}$, $\mathbf{c}_{SH}$.
2.  **Dynamic Parameters:**
    *   **Position:** Fourier series coefficients $\{\mathbf{a}_k, \mathbf{b}_k\}_{k=0}^{K}$ for each dimension $x, y, z$. Total $3 \times (2K+1)$ parameters.
    *   **Rotation:** Linear coefficients $\mathbf{m}_q \in \mathbb{R}^4$ (rate of change of quaternion components). Total 4 parameters.

**Formulas:**
The position $\mathbf{c}(t)$ and rotation $\mathbf{q}(t)$ at time $t$ are computed as:
$$ \mathbf{c}(t) = \mathbf{c}_0 + \sum_{k=0}^{K} \left( \mathbf{a}_k \cos(k \omega t) + \mathbf{b}_k \sin(k \omega t) \right) $$
$$ \mathbf{q}(t) = \mathbf{q}_0 + \mathbf{m}_q t $$
where $\omega$ is the base frequency. The quaternion $\mathbf{q}(t)$ must be normalized after computation [S2].

**Optimization Steps:**
1.  **Initialization:** Initialize Gaussians from static reconstruction. Set dynamic coefficients $\mathbf{a}_k, \mathbf{b}_k, \mathbf{m}_q$ to zero or small random values.
2.  **Forward Pass:** For each training image at time $t$:
    *   Compute $\mathbf{c}(t)$ and $\mathbf{q}(t)$ using the formulas above.
    *   Rasterize the Gaussians using standard 3DGS rasterization with the time-dependent position and rotation.
    *   Render the image $\hat{I}$.
3.  **Loss Calculation:**
    *   **Photometric Loss:** $L_{img} = L1(\hat{I}, I_{gt})$ or LPIPS [S2].
    *   **Optical Flow Loss:** Compute bidirectional optical flow between rendered frames or between rendered and ground truth frames. Add $L_{flow}$ to penalize inconsistencies [S2].
    *   **Total Loss:** $L = L_{img} + \lambda_{flow} L_{flow}$.
4.  **Backpropagation:** Update Fourier coefficients and linear rotation parameters using gradient descent. Static attributes are typically frozen or updated separately in a joint optimization scheme [S2].

**Parameters:**
*   $K$: Number of Fourier harmonics. Not given in the sources.
*   $\omega$: Base frequency. Not given in the sources.
*   $\lambda_{flow}$: Weight for optical flow loss. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Quaternion Normalization:** The linear approximation $\mathbf{q}_0 + \mathbf{m}_q t$ may result in non-unit quaternions. Normalization is required before rotation matrix construction [S2].
*   **Periodicity Assumption:** Fourier series assumes periodic motion. Non-periodic or transient motions may require higher $K$ or hybrid models (e.g., Gaussian-Flow) [S2].
*   **Overfitting:** While less prone than implicit networks, high $K$ can lead to overfitting noise. Regularization on coefficients may be needed [S2].

## Operating characteristics
*   **Time Complexity:** Optimization time is independent of the number of time steps, as parameters are time-independent [S2]. Rendering time is similar to static 3DGS per frame.
*   **Memory Complexity:** Storage increases by $O(K)$ per Gaussian for Fourier coefficients and $O(1)$ for linear rotation. Total memory is $O(N \cdot K)$ where $N$ is the number of Gaussians [S2].
*   **Strengths:** Fast training due to explicit parameterization; temporal continuity ensured by continuous functions; robust to overfitting compared to implicit deformation fields [S2].
*   **Limitations:** Assumes smooth or periodic motion; may struggle with abrupt, non-periodic changes without high-frequency terms [S2].

## Background
Dynamic 3D Gaussian Splatting was proposed by Katsumata et al. [S2]. It was introduced as an explicit deformation method to address the overfitting and computational cost of implicit deformation networks in dynamic scene reconstruction. It leverages the observation that human and articulated object motion often exhibits periodic characteristics, justifying the use of Fourier series for position modeling [S2].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
