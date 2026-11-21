# EBAD-NeRF

Also called: Event-driven Bundle Adjustment for Deblurring Neural Radiance Fields

## Overview
EBAD-NeRF is a method for reconstructing sharp 3D radiance fields from motion-blurred images by jointly optimizing learnable camera poses and NeRF parameters using hybrid event-RGB data [S1]. It distinguishes itself from prior works like BAD-NeRF and E2 NeRF by modeling complex camera motion trajectories using multiple learnable poses (rather than linear interpolation or fixed poses) and explicitly supervising the motion blur via an intensity-change-metric event loss alongside a photo-metric blur loss [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of motion-blurred RGB images $B$, corresponding event data $E$ (intensity changes), and the exposure start/end times $t_{start}, t_{end}$ for each image [S1].
*   **Outputs:** A trained NeRF network $F_\Theta$ representing the sharp static scene, and the optimized camera poses $P(t)$ for each exposure interval [S1].

**Data Structures:**
*   **NeRF Network $F_\Theta$:** A Multi-Layer Perceptron (MLP) taking encoded 3D position $\mathbf{o}$ and view direction $\mathbf{d}$ as input, outputting volume density $\sigma$ and view-dependent color $\mathbf{c}$ [S1].
*   **Camera Poses:** For each view $v$, a sequence of $p$ learnable poses $\{P_i\}_{i=1}^p$ representing the camera trajectory during the exposure time, mapped to SE(3) space [S1].
*   **Sampling Points:** $N$ points sampled along a ray segment $[l_{near}, l_{far}]$ using stratified sampling [S1].

**Numbered Steps:**

1.  **Trajectory Modeling:** For each blurred image view, define $p$ learnable camera poses $\{P_i\}_{i=1}^p$ sampled evenly over the exposure time interval $[t_{start}, t_{end}]$. These poses represent the camera motion trajectory in SE(3) space [S1].
2.  **Ray Sampling:** For a pixel $\mathbf{x}$ in the image plane, generate a ray $\mathbf{r}$ based on the current camera pose. Divide the ray segment from $l_{near}$ to $l_{far}$ into $N$ equal parts and sample one point $\mathbf{o}_i$ in each part [S1].
3.  **Positional Encoding:** Encode the 3D coordinate $\mathbf{o}$ and view direction $\mathbf{d}$ using positional encoding $\gamma_K$:
    $$ \gamma_K(\mathbf{x}) = \{\sin(2^k \pi \mathbf{x}), \cos(2^k \pi \mathbf{x})\}_{k=0}^K $$
    This maps inputs to $K+1$ dimensions [S1].
4.  **NeRF Query:** Input the encoded coordinates and directions into the NeRF MLP $F_\Theta$ to obtain density $\sigma_i$ and color $\mathbf{c}_i$ for each sampled point:
    $$ (\mathbf{c}_i, \sigma_i) = F_\Theta(\gamma_o(\mathbf{o}_i), \gamma_d(\mathbf{d}_i)) $$
    [S1].
5.  **Volume Rendering (Sharp Image):** Compute the color $C(\mathbf{r}, \mathbf{x})$ for a sharp view using volume rendering:
    $$ C(\mathbf{r}, \mathbf{x}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    where $\delta_i = l_{i+1} - l_i$ is the distance between adjacent points, and $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transparency [S1].
6.  **Blurred Image Formation:** Model the motion blur by averaging the rendered sharp colors across the $p$ poses in the trajectory:
    $$ \hat{B}(\mathbf{x}) = \frac{1}{p} \sum_{i=1}^p C(P(t_i), \mathbf{x}) $$
    where $P(t_i)$ is the $i$-th pose in the trajectory [S1].
7.  **Loss Computation:**
    *   **Photo-metric Blur Loss:** Compare the rendered blurred image $\hat{B}$ with the observed blurred image $B$ [S1].
    *   **Intensity-change-metric Event Loss:** Supervise the pose changes during exposure using event data. The event data measures intensity changes with high temporal resolution, providing explicit connection between camera motion trajectory and the target sharp NeRF [S1].
8.  **Joint Optimization:** Update both the NeRF parameters $\Theta$ and the learnable camera poses $\{P_i\}$ using the combined loss [S1].

**Parameters:**
*   $N$: Number of stratified samples per ray. Not given in the sources.
*   $p$: Number of poses used to model the motion trajectory. Not given in the sources.
*   $K$: Degree of positional encoding. Not given in the sources.
*   $l_{near}, l_{far}$: Near and far clipping planes for ray sampling. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Complex Motion:** Linear interpolation of start and end poses (as in BAD-NeRF) fails to model complex motion blur; EBAD-NeRF addresses this by using multiple learnable poses [S1].
*   **Pose Initialization:** Unlike E2 NeRF which uses fixed poses from COLMAP, EBAD-NeRF learns poses, requiring careful initialization or regularization to avoid degenerate trajectories [S1].
*   **Event Data Alignment:** Requires spatially and temporally aligned event-RGB data. Misalignment can lead to incorrect intensity-change supervision [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples $N$, poses $p$, and network depth. Each iteration requires rendering $p$ views per pixel and backpropagating through the NeRF and pose parameters. Not given in the sources.
*   **Memory Complexity:** Stores the NeRF parameters and $p$ poses per view. Not given in the sources.
*   **Strengths:** Can recover accurate camera trajectories and sharper 3D representations from blurry images by leveraging event data for explicit motion supervision [S1].
*   **Limitations:** Requires event-RGB data, which is not always available. Joint optimization of poses and NeRF parameters can be computationally expensive and sensitive to initialization [S1].
*   **When to use:** When reconstructing 3D scenes from motion-blurred images and event data is available, especially for complex camera motions [S1].
*   **When not to use:** When only RGB images are available, or when the scene is static and sharp images are captured [S1].

## Background
EBAD-NeRF was proposed by Yunshan Qi, Lin Zhu, Yifan Zhao, Nan Bao, and Jia Li [S1]. It builds upon previous works like Deblur-NeRF, BAD-NeRF, and E2 NeRF, which attempted to address motion blur in NeRF using deformable kernels, linear pose interpolation, and fixed poses with event data, respectively [S1]. EBAD-NeRF introduces event-driven bundle adjustment to jointly optimize poses and NeRF parameters, leveraging the high temporal resolution of event cameras to model motion blur explicitly [S1].

## Sources
[S1] Deblurring Neural Radiance Fields with Event-driven Bundle Adjustment, https://doi.org/10.1145/3664647.3680569, fetched 2026-10-27
