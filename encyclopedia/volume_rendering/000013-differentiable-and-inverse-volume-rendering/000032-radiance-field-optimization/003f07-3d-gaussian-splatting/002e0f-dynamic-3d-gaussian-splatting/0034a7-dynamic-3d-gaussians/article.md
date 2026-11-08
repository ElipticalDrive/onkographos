# Dynamic 3D Gaussians

Also called: Dynamic 3D Gaussian Splatting, 4D Gaussian Splatting, Dynamic Gaussian Splatting

## Overview
Dynamic 3D Gaussians is a method for persistent dynamic view synthesis that models time-varying scenes by linking the motion of each Gaussian primitive to the movement of corresponding 3D points in the scene [S1, unofficial]. It solves the problem of reconstructing and rendering dynamic 3D content from multi-view video by extending static 3D Gaussian Splatting with temporal tracking and deformation constraints. It differs from its parent [Dynamic 3D Gaussian Splatting](5703-dynamic-3d-gaussian-splatting.md) by explicitly defining the tracking mechanism (linking Gaussians to 3D point motion) and from [GaussianFlow](6149-gaussianflow.md) by its reliance on persistent synthesis rather than flow-based deformation fields.

## How it works
The method maintains a set of 3D Gaussian primitives $\mathcal{G} = \{g_i\}$, where each primitive $g_i$ is defined at time $t$ by its position $\boldsymbol{\mu}_i^t$, orientation $q_i^t$, scale $s_i^t$, opacity $\alpha_i^t$, and spherical harmonic coefficients $c_i^t$. The core innovation is the temporal update of $\boldsymbol{\mu}_i^t$ and $q_i^t$ via tracking.

**Inputs:**
- Multi-view video frames $\{I_t\}$ for $t \in [0, T]$.
- Camera poses $\{C_t\}$ for each frame.
- Initial static 3D Gaussians $\mathcal{G}_0$ (from [3D Gaussian Splatting](52-3d-gaussian-splatting.md)).

**Outputs:**
- Time-varying Gaussian parameters $\mathcal{G}_t$ for all $t$.
- Rendered dynamic views.

**Data Structures:**
- **Gaussian Primitive:** A struct containing $\boldsymbol{\mu} \in \mathbb{R}^3$, $q \in \mathbb{R}^4$ (normalized quaternion), $s \in \mathbb{R}^3$, $\alpha \in \mathbb{R}$, $c \in \mathbb{R}^K$ (SH coefficients).
- **Anchor Points:** A sparse subset of Gaussians selected via Farthest Point Sampling (FPS) [S2].
- **Embedded Deformation Graph (EDG):** A graph connecting each Gaussian to its $K$ nearest anchor points, with blend weights $w_{ij}$ based on spatial distance [S2].

**Steps:**
1. **Initialization:** Initialize $\mathcal{G}_0$ using standard 3DGS from the first frame [S1, unofficial].
2. **2D Projection:** For each Gaussian $g_i$ at time $t'$, compute its 2D projection $p_i^{t'}$ across $N$ input views using camera poses $C_{t'}$ [S2].
3. **2D Tracking:** Run a pre-trained 2D point tracking model on the projected points $p_i^{t'}$ to obtain corresponding 2D pixel positions $p_i^t$ in the current frame $t$ [S2].
4. **Triangulation:** Recover the 3D position $\boldsymbol{\mu}_i^t$ by finding the point closest to the $N$ camera rays originating from $p_i^t$ in frame $t$ [S2].
5. **Motion Propagation:**
   - Select anchor points from $\mathcal{G}_t$ using FPS [S2].
   - For each non-anchor Gaussian $g_i$, identify its $K$ nearest anchor points $\{a_j\}_{j=1}^K$ [S2].
   - Compute blend weights $w_{ij}$ based on spatial distance between $g_i$ and $a_j$ [S2].
   - Update $\boldsymbol{\mu}_i^t$ and $q_i^t$ by interpolating the motion of the anchor points using the EDG [S2].
6. **Keyframe Management:** Maintain primitives only from a set of past keyframes. Smoothly adjust contributions as older keyframes are replaced by newer ones to ensure seamless temporal transitions [S2].

**Parameters:**
- $K$: Number of nearest anchor points for EDG. Not given in the sources.
- $N$: Number of input views. Not given in the sources.
- Tracking Model: Pre-trained 2D point tracking model. Not given in the sources.

**Edge Cases:**
- **Occlusion:** 2D tracking may fail if points are occluded. The EDG helps propagate motion from visible anchors to occluded regions [S2].
- **Precision:** Triangulation accuracy depends on the baseline between views and the precision of 2D tracking [S2].

## Subtopics

### 3D Warping
A method to estimate the 3D position of a Gaussian primitive at a current frame $t$ from its position at a previous frame $t'$ using 2D point tracking and triangulation [S2]. It sets itself apart by explicitly using a pre-trained 2D point tracker and EDG for motion propagation, rather than learning deformation fields directly. Choose this when you have multi-view video and want to leverage existing 2D tracking models for robust motion estimation [S2].

## Operating characteristics
- **Time Complexity:** Depends on the 2D tracking model and the number of Gaussians. Triangulation is $O(N)$ per Gaussian per frame. EDG propagation is $O(K)$ per Gaussian. Not given in the sources.
- **Memory Complexity:** Proportional to the number of Gaussians and keyframes. Not given in the sources.
- **Performance:** Real-time 4DGS achieves interactive frame rates by fusing static background Gaussians with a small, fast-updating set of dynamic foreground Gaussians [S1, unofficial].
- **Strengths:** Persistent dynamic view synthesis; leverages robust 2D tracking; seamless temporal transitions via keyframe management [S1, unofficial][S2].
- **Limitations:** Relies on the accuracy of the pre-trained 2D point tracker; may struggle with fast motion or severe occlusion [S2].

## Background
Dynamic 3D Gaussians was introduced as part of the broader 3D Gaussian Splatting family to handle dynamic scenes. It builds on [3D Gaussian Splatting](52-3d-gaussian-splatting.md) and [Dynamic 3D Gaussian Splatting](5703-dynamic-3d-gaussian-splatting.md). The method was developed to achieve persistent dynamic view synthesis by linking Gaussian motion to 3D point motion [S1, unofficial]. It is used in applications requiring real-time 4D reconstruction and dynamic view synthesis [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
