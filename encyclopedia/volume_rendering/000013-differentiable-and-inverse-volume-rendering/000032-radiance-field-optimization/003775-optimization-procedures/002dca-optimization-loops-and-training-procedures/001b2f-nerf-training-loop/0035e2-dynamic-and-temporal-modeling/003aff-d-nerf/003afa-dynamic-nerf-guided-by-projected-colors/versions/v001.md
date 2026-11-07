# Dynamic NeRF Guided by Projected Colors

Also called: Projected Color Conditioning, Space-Color Dynamic NeRF

## Overview
This method is a dynamic Neural Radiance Field (NeRF) variant that replaces the explicit temporal coordinate $t$ with multi-view projected color statistics to improve point correspondence estimation during on-the-fly training [S1]. Unlike standard space-time dynamic NeRFs that optimize irregular temporal deformation fields, this approach maps inputs to a "space-color" coordinate system where correspondence is easier to track due to color invariance under motion [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Spatial coordinates $x \in \mathbb{R}^3$, viewing direction $d \in \mathbb{R}^3$, projected color mean $\bar{c}_p(x, k)$, and projected color variance $Var(c_p(x, k))$ at frame $k$.
*   **Outputs:** Density $\sigma \in \mathbb{R}$ and color $c \in \mathbb{R}^3$.

**Data Structures**
The model input vector consists of:
1.  $x$: 3D spatial position.
2.  $d$: Normalized viewing direction.
3.  $\bar{c}_p(x, k)$: Scalar mean of projected colors.
4.  $Var(c_p(x, k))$: Scalar variance of projected colors.

**Projected Color Calculation**
For a 3D point $x$ at frame $k$, the set of projected colors $c_p(x, k)$ is derived from all training cameras $M$. Let $C_{k, cam}$ be the pixel color at the projection of $x$ in camera $cam$'s image at frame $k$, and $P_{cam}$ be the camera projection matrix [S1]:

$$
c_p(x, k) = \{ C_{k, cam}(P_{cam} \cdot x) \mid cam \in M \}
$$

The scalar inputs are the mean and variance of this set [S1]:
$$
\bar{c}_p(x, k) = \text{mean}(c_p(x, k))
$$
$$
Var(c_p(x, k)) = \text{variance}(c_p(x, k))
$$

**Algorithm Steps**
1.  **Precompute Projections:** For each 3D point $x$ sampled during training and each frame $k$, project $x$ into the image plane of every camera $cam \in M$ using $P_{cam}$.
2.  **Sample Colors:** Retrieve the pixel colors $C_{k, cam}$ at the projected coordinates. Handle out-of-bounds projections by excluding them or using a default value [G].
3.  **Compute Statistics:** Calculate the mean $\bar{c}_p(x, k)$ and variance $Var(c_p(x, k))$ of the collected colors.
4.  **Model Forward Pass:** Construct the input vector $[x, d, \bar{c}_p(x, k), Var(c_p(x, k))]$. Pass this through the neural network $F_k$ to predict density $\sigma$ and color $c$ [S1]:
    $$
    F_k: (x, d, \bar{c}_p(x, k), Var(c_p(x, k))) \mapsto (\sigma, c)
    $$
5.  **Render and Optimize:** Use volume rendering to synthesize images and optimize network parameters using gradient descent [S1].

**Pitfalls**
*   **Color Invariance Assumption:** Projected colors are not perfectly invariant due to reflections, lighting changes, and occlusions. However, they remain largely similar across consecutive frames [S1].
*   **Projection Errors:** Incorrect camera matrices $P_{cam}$ or misaligned frames will corrupt the projected color statistics [G].

## Operating characteristics
*   **Complexity:** Reduces optimization time compared to space-time dynamic NeRFs by avoiding the need to learn irregular temporal deformation fields [S1].
*   **Strengths:** Better point correspondence estimation on-the-fly; easier extrapolation to untrained frames [S1].
*   **Limitations:** Relies on multi-view consistency; performance degrades if lighting or reflections vary significantly between frames [S1].

## Background
This method was introduced in OD-NeRF [S1]. It addresses the difficulty of estimating point correspondence in dynamic scenes where temporal trajectories are irregular [S1]. By shifting from a space-time to a space-color coordinate system, it leverages the invariance of point color under motion to simplify the learning task [S1].

## Sources
[S1] OD-NeRF: Efficient Training of On-the-Fly Dynamic Neural Radiance Fields, https://doi.org/10.1109/3dv66043.2025.00053, fetched 2026-10-26
