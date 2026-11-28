# EBAD-NeRF

Also called: Event-driven Bundle Adjustment for Deblurring Neural Radiance Fields

## Overview
EBAD-NeRF is a method for reconstructing sharp 3D radiance fields and accurate camera trajectories from motion-blurred images and hybrid event-RGB data [S1]. It solves the problem of modeling complex camera motion blur during exposure time, which standard NeRF methods fail to capture accurately [S1]. Unlike BAD-NeRF, which uses linear interpolation between start and end poses, EBAD-NeRF models the motion trajectory using multiple learnable poses, and unlike E2 NeRF, it jointly optimizes these poses and NeRF parameters rather than relying on fixed poses from pre-deblurred images [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Blurry RGB images $B$, corresponding event data (intensity changes), and exposure start/end times $t_{start}, t_{end}$ for each view.
*   **Outputs:** A sharp NeRF model $F_\Theta$ (mapping 3D coordinates and view directions to color and density) and a set of learnable camera poses modeling the motion trajectory during exposure.

**Data Structures:**
*   **NeRF MLP ($F_\Theta$):** A multi-layer perceptron taking encoded 3D position $\mathbf{o}$ and view direction $\mathbf{d}$ as input, outputting volume density $\sigma$ and view-dependent color $\mathbf{c}$ [S1].
*   **Learnable Poses:** For each view, a set of $p$ poses $\{P_i\}_{i=1}^p$ in SE(3) space, sampled evenly over the exposure interval $[t_{start}, t_{end}]$ [S1].
*   **Ray Sampling:** Rays are divided into $N$ equal parts between $l_{near}$ and $l_{far}$. One point is sampled per part [S1].

**Numbered Steps:**
1.  **Trajectory Modeling:** For each view, define $p$ learnable camera poses $\{P_i\}_{i=1}^p$ distributed evenly across the exposure time $[t_{start}, t_{end}]$ [S1].
2.  **Ray Casting:** For each pixel $\mathbf{x}$ in the blurry image, generate a ray $\mathbf{r}$ based on the camera projection. Divide the ray segment $[l_{near}, l_{far}]$ into $N$ equal intervals [S1].
3.  **Sampling:** Sample $N$ points along the ray. For each point $i$, determine its 3D world coordinate $\mathbf{o}_i$ and view direction $\mathbf{d}$ [S1].
4.  **Positional Encoding:** Encode the 3D coordinate $\mathbf{o}_i$ and view direction $\mathbf{d}$ using positional encoding $\gamma_K(\cdot)$:
    $$ \gamma_K(\mathbf{x}) = \{\sin(2^k \pi \mathbf{x}), \cos(2^k \pi \mathbf{x})\}_{k=0}^K $$
    where $K$ is the number of frequency bands [S1].
5.  **NeRF Query:** Pass the encoded coordinates and directions into the NeRF MLP $F_\Theta$ to obtain density $\sigma_i$ and color $\mathbf{c}_i$ for each sampled point [S1].
6.  **Volume Rendering (Sharpness):** Compute the sharp color $C(\mathbf{r}, \mathbf{x})$ for the ray using volume rendering:
    $$ C(\mathbf{r}, \mathbf{x}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    where $\delta_i = l_{i+1} - l_i$ is the distance between adjacent points, and $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transparency [S1].
7.  **Blur Simulation:** Model the motion blur by averaging the rendered sharp colors from the $p$ poses in the trajectory:
    $$ \hat{B}(\mathbf{x}) = \frac{1}{p} \sum_{i=1}^p C(P(t_i), \mathbf{x}) $$
    where $P(t_i)$ is the pose at time $t_i$ [S1].
8.  **Loss Computation:**
    *   **Photo-metric Blur Loss:** Compare the simulated blurry image $\hat{B}$ with the observed blurry image $B$ [S1].
    *   **Intensity-change-metric Event Loss:** Supervise the pose changes during exposure using event data to ensure the modeled trajectory matches the intensity changes captured by the event camera [S1].
9.  **Optimization:** Jointly optimize the NeRF parameters $\Theta$ and the learnable poses $\{P_i\}$ using the combined losses [S1].

**Parameters:**
*   $N$: Number of stratified samples per ray (Not given in the sources).
*   $p$: Number of learnable poses per view to model the trajectory (Not given in the sources).
*   $K$: Number of frequency bands in positional encoding (Not given in the sources).
*   $l_{near}, l_{far}$: Near and far clipping planes for ray sampling (Not given in the sources).

**Edge Cases and Pitfalls:**
*   **Precision:** The integration of light intensity over time requires accurate temporal sampling of poses. Insufficient $p$ may fail to model complex non-linear motion [S1].
*   **Boundaries:** The method assumes a static 3D scene; dynamic scenes are not explicitly handled by the core formulation [S1].

## Operating characteristics
*   **Time Complexity:** Training is computationally intensive due to the joint optimization of NeRF parameters and multiple camera poses per view, requiring multiple forward passes through the NeRF for each pixel to simulate blur [S1].
*   **Memory Complexity:** Requires storing $p$ poses per view in addition to the NeRF parameters [S1].
*   **Strengths:** Capable of recovering accurate camera trajectories and sharper 3D representations from blurry images by leveraging event data for explicit motion supervision [S1].
*   **Limitations:** Requires hybrid event-RGB data; performance depends on the quality and synchronization of event data [S1].
*   **When to use:** When reconstructing scenes from motion-blurred images where event camera data is available and accurate camera trajectory estimation is required [S1].
*   **When not:** When only RGB data is available (use BAD-NeRF or E2 NeRF) or when the scene is dynamic [S1].

## Background
EBAD-NeRF was proposed by Yunshan Qi, Lin Zhu, Yifan Zhao, Nan Bao, and Jia Li [S1]. It addresses limitations in prior works like BAD-NeRF (linear interpolation of poses) and E2 NeRF (fixed poses from pre-deblurred images) by introducing event-driven bundle adjustment [S1]. The method was published in 2024 [S1].

## Sources
[S1] Deblurring Neural Radiance Fields with Event-driven Bundle Adjustment, https://doi.org/10.1145/3664647.3680569, fetched 2026-10-27
