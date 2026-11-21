# Space-Time Image Formation Model
Also called: 4D density function model
## Overview
The Space-Time Image Formation Model represents a continuously deforming 3D object as a 4D density function $h(x, y, z, t)$, discretized into a sequence of $N_k$ static frames [S1, unofficial]. It solves the tomographic reconstruction problem for dynamic scenes by treating each time step as an independent static volume, linked only by the temporal ordering of the projection data [S1, unofficial]. Unlike warp-based methods that separate geometry and appearance into a canonical form and a dynamic transformation, this model uses a single unified representation to directly regress dynamic properties, making it tolerant of topology changes but prone to time inconsistency [S1, unofficial]. It builds on [Physics-Based Image Formation Models](8344-physics-based-image-formation-models.md) by defining the specific forward operator $\mathcal{P}$ for a block-diagonal system.

## How it works
**Inputs:**
- $N_p$: Total number of captured projection images.
- $N_k$: Number of reconstructed frames (time steps), where $N_k \le N_p$ [S1, unofficial].
- $N_\theta$: Number of projections contributing to each frame. Typically $N_\theta = N_p / N_k$ [S1, unofficial].
- $M$: Number of pixels in a single projection image.
- $N_v$: Number of voxels in the 3D grid for each frame.

**Outputs:**
- $\mathbf{f}$: The reconstructed 4D density volume, represented as a vector in $\mathbb{R}^{N_v N_k}$ [S1, unofficial].

**Data Structures:**
The 4D density volume is described by a column vector $\mathbf{f}$ composed of $N_k$ sub-vectors $\mathbf{f}_t$, where each $\mathbf{f}_t \in \mathbb{R}^{N_v}$ represents the 3D volume at the $t$-th time step [S1, unofficial]:
$$
\mathbf{f} = \{\mathbf{f}_0, \mathbf{f}_1, \dots, \mathbf{f}_{N_k-1}\}
$$
The measured data is a vector $\mathbf{p}$ containing the sinogram data for all frames [S1, unofficial]:
$$
\mathbf{p} = \{\mathbf{p}_0, \mathbf{p}_1, \dots, \mathbf{p}_{N_k-1}\}
$$
where $\mathbf{p}_t \in \mathbb{R}^{N_\theta M}$ is the vector of measured data for frame $t$ [S1, unofficial].

**Algorithm:**
The tomography problem is expressed as a linear system $\mathbf{A} \mathbf{f} = \mathbf{p}$ [S1, unofficial]. The matrix $\mathbf{A}$ is a block diagonal matrix where each block $\mathbf{A}_t \in \mathbb{R}^{N_\theta M \times N_v}$ models the Radon transform operator for the $N_\theta$ projections contributing to the $t$-th frame [S1, unofficial]:

$$
\mathbf{A} = \begin{pmatrix} 
\mathbf{A}_0 & & & \\ 
& \ddots & & \\ 
& & \mathbf{A}_t & \\ 
& & & \ddots \\ 
& & & & \mathbf{A}_{N_k-1} 
\end{pmatrix}
$$

**Steps:**
1.  **Discretization:** Divide the temporal domain into $N_k$ frames. Assume each frame represents an approximately static geometry [S1, unofficial].
2.  **Projection Assignment:** Assign $N_\theta$ unique projection angles to each frame. The total number of projections $N_p$ is distributed such that $N_\theta = N_p / N_k$ [S1, unofficial]. Note that all projection angles are unique across the entire sequence [S1, unofficial].
3.  **Forward Projection:** For each frame $t$, compute the projected measurements $\hat{\mathbf{p}}_t$ by applying the Radon transform matrix $\mathbf{A}_t$ to the current volume estimate $\mathbf{f}_t$:
    $$
    \hat{\mathbf{p}}_t = \mathbf{A}_t \mathbf{f}_t
    $$
    Concatenate all $\hat{\mathbf{p}}_t$ to form the full predicted sinogram $\hat{\mathbf{p}}$ [S1, unofficial].
4.  **Optimization:** Minimize the loss function $\mathcal{L}$ between the measured sinogram $\mathbf{p}$ and the rendered projection $\hat{\mathbf{p}}$. Since the system is block-diagonal, the optimization for each frame $\mathbf{f}_t$ can be performed independently or jointly if regularization couples them [S1, unofficial]. The general form is:
    $$
    \min_{\mathbf{f}} \mathcal{L}(\mathbf{A}\mathbf{f}, \mathbf{p}) + \lambda R(\mathbf{f})
    $$
    where $R(\mathbf{f})$ is a regularizer [S1, unofficial].
5.  **Reconstruction:** Update $\mathbf{f}_t$ using an iterative solver (e.g., SGD, Adam) until convergence [S1, unofficial].

**Parameters:**
- $N_k$: Number of reconstructed frames. Constraint: $N_k \le N_p$ [S1, unofficial].
- $N_\theta$: Number of projections per frame. Typical value: $N_\theta = N_p / N_k$ [S1, unofficial].
- $N_{min}$: Minimum number of projections for acceptable quality. Values such as $N_{min} = 10$ are noted for achieving results [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Ill-posedness:** The problem is ill-posed, especially when few projections are used per frame ($N_\theta$ is small) [S1, unofficial].
- **Time Inconsistency:** Because frames are reconstructed independently (or with weak coupling), the model is prone to time inconsistency [S1, unofficial].
- **Sampling Strategy:** The sampling strategy is decoupled from the magnitude of deformation. The number of reconstructed frames $N_k$ is independent of the number of rounds and can be chosen post-capture [S1, unofficial].
- **Static Assumption:** The model assumes each frame is approximately static. If deformation is too rapid within a frame's time window, reconstruction quality degrades [S1, unofficial].

## Subtopics

### [Space-time representations](6885-space-time-representations.md)
Models dynamic scenes by extending the query space to include time as a direct input, omitting explicit deformation networks. Unlike warp-based methods, it uses a single unified function to regress dynamic properties, tolerating topology changes but suffering from time inconsistency.

## Operating characteristics
- **Time Complexity:** Depends on the number of frames $N_k$ and the cost of the Radon transform $\mathbf{A}_t$. Since the system is block-diagonal, frames can be processed in parallel if no temporal regularization is used.
- **Memory Complexity:** $O(N_v N_k)$ to store the 4D volume vector $\mathbf{f}$ [S1, unofficial].
- **Strengths:** Tolerant of topology changes; sampling strategy is decoupled from deformation magnitude; $N_k$ can be chosen post-capture [S1, unofficial].
- **Limitations:** Prone to time inconsistency; ill-posed with few projections per frame; assumes approximate static geometry per frame [S1, unofficial].
- **When to use:** When topology changes are expected, or when post-capture choice of temporal resolution is desired [S1, unofficial].
- **When not to use:** When high temporal consistency is required without additional regularization, or when deformation is too rapid for the "static frame" assumption [S1, unofficial].

## Background
The Space-Time Image Formation Model was introduced in the context of space-time tomographic reconstruction of deforming objects [S1, unofficial]. It addresses the challenge of reconstructing 3D objects that undergo continuous deformation over time by discretizing the 4D density function into static frames [S1, unofficial]. The approach allows for the reconstruction of dynamic scenes using a sequence of projections, where each frame is reconstructed from a subset of the total projections [S1, unofficial].

## Sources
[S1] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
