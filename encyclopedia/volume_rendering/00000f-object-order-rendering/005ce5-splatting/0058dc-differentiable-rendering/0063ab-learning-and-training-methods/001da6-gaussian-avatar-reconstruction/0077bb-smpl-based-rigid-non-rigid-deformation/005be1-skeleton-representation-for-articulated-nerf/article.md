# Skeleton Representation for Articulated NeRF

Also called: Articulated Pose Encoding, Articulated Skeleton Pose Model, Bone Transformation, Forward Kinematics Mapping, Skeleton Representation

## Overview
Skeleton Representation for Articulated NeRF (A-NeRF) encodes human shape, appearance, and pose by mapping 3D query points relative to an articulated skeleton hierarchy rather than absolute world coordinates [S1, unofficial]. It solves the problem of modeling dynamic articulated human motion in neural radiance fields by replacing the parametric surface model of SMPL with a pure skeletal structure, allowing the method to be initialized with any external skeleton pose estimator [S1, unofficial]. Unlike sibling methods that rely on SMPL mesh topology for deformation, this approach explicitly incorporates domain knowledge of bone connectivity and static bone lengths to reparameterize query locations and view directions before density and color evaluation [S1, unofficial].

## How it works
**Inputs:**
*   Per-frame skeleton pose $\theta_k$ for image $I_k$ [S1, unofficial].
*   Rest pose joint locations $\{a_m\}_{m=1}^{24}$ [S1, unofficial].
*   Camera intrinsics [S1, unofficial].
*   Query point $q \in \mathbb{R}^3$ and view direction $d$ [S1, unofficial].

**Outputs:**
*   Density $\sigma$ and color $C$ at query point $q$ [S1, unofficial].

**Data Structures:**
*   **Skeleton Hierarchy:** Defined by joints indexed $0 \dots 24$ and $B$ bones, with joint 0 as the root [S1, unofficial].
*   **Rest Pose:** Static 3D joint locations $\{a_m\}_{m=1}^{24}$ encoding connectivity and bone lengths [S1, unofficial].
*   **Pose Parameters:** $\theta_k = [\omega_{k,0}, \dots, \omega_{k,24}]$ [S1, unofficial].
    *   $\omega_{k,m} \in \mathbb{R}^3$ or $\mathbb{R}^6$: Relative rotation of joint $m$ to its parent [S1, unofficial].
    *   $\omega_{k,0}$: Global position (special case) [S1, unofficial].
*   **Transformation Matrices:** $T(\theta_k, m) \in \mathbb{R}^{4 \times 4}$ for each bone $m$ [S1, unofficial].

**Formulas:**
The mapping from local bone coordinates $p_{k,m} \in \mathbb{R}^3$ to world coordinates $q \in \mathbb{R}^3$ is defined via forward kinematics using homogeneous coordinates [S1, unofficial]:

$$
\begin{bmatrix} q \\ 1 \end{bmatrix} = T(\theta_k, m) \begin{bmatrix} p_{k,m} \\ 1 \end{bmatrix}
$$

The transformation matrix $T(\theta_k, m)$ is computed as the product of transformations along the path from the root to joint $m$ [S1, unofficial]:

$$
T(\theta_k, m) = \prod_{l \in A(m)} \begin{bmatrix} R(\omega_{k,l}) & a_{l,l-1} \\ 0 & 1 \end{bmatrix} \in \mathbb{R}^{4 \times 4}
$$

Where:
*   $A(m)$ is the ordered set of joint ancestors of $m$ [S1, unofficial].
*   $R(\omega_{k,l})$ is the rotation matrix inferred from $\omega_{k,l}$ (using axis-angle or overparametrized representation from [78]) [S1, unofficial].
*   $a_{l,l-1} \in \mathbb{R}^3$ is the translation vector between joint $l$ and its child joint $l-1$ (derived from the rest pose) [S1, unofficial].

**Steps:**
1.  **Initialize Skeleton:** Define the rest pose joint locations $\{a_m\}_{m=1}^{24}$ and the bone connectivity graph. This structure is equivalent to SMPL joints but lacks the parametric surface mesh [S1, unofficial].
2.  **Compute Forward Kinematics:** For a given frame $k$ and pose $\theta_k$, compute the world-space transformation matrix $T(\theta_k, m)$ for each bone $m$ using the product of ancestor transformations [S1, unofficial].
3.  **Reparameterize Query:** To evaluate the NeRF at world coordinate $q$, transform $q$ into the local coordinate system of the relevant bone $m$ using the inverse transformation:
    $$
    \begin{bmatrix} p_{k,m} \\ 1 \end{bmatrix} = T(\theta_k, m)^{-1} \begin{bmatrix} q \\ 1 \end{bmatrix}
    $$
    Similarly, transform the view direction $d$ into the local bone frame [S1, unofficial].
4.  **NeRF Evaluation:** Pass the local position $p_{k,m}$ and local view direction into the neural network to predict density $\sigma$ and color $C$ [S1, unofficial].
5.  **Rendering:** Render the image using standard volume rendering integration over the sampled points [S1, unofficial].

**Parameters:**
*   **Rotation Representation:** Can use traditional axis-angle ($\omega_{k,m} \in \mathbb{R}^3$) or overparametrized representation ($\omega_{k,m} \in \mathbb{R}^6$) [S1, unofficial].
*   **Joint Count:** 24 joints (plus root) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Initialization:** Since the method does not use an SMPL surface, it requires an external skeleton pose estimator for initialization. Accuracy depends on the quality of this estimator [S1, unofficial].
*   **Multi-view Consistency:** When using multiple views, global position and orientation are view-specific, while relative rotations are averaged across views to refine pose [S1, unofficial].

## Subtopics

### [Neural Skinning via Blend Weight Field](20672-neural-skinning-via-blend-weight-field.md)
A method for animating a learned template Neural Radiance Field under novel human poses by computing a deformation field conditioned on a new latent code and SMPL parameters. It differs from standard rigid skinning by learning a residual neural blend weight field that corrects the linear blend skinning approximation, allowing for non-rigid deformations such as muscle bulging and clothing movement. It builds on [Skeleton Representation for Articulated NeRF](17005-skeleton-representation-for-articulated-nerf.md) by using the skeleton to regularize the deformation field learning. Choose this variant when non-rigid soft-tissue dynamics are required beyond rigid bone transformations.

### Kinematic Model
Represents an articulated object with $P+1$ joints and $P$ bones in a tree structure, where one joint is the root and others are linked to a single parent by a bone of fixed length [S2]. The global rigid transformation for joint $i$ is obtained by multiplying local transformation matrices along the kinematic tree from the root to joint $i$ [S2]. It serves as a general mathematical formulation for the forward kinematics used in the primary method [S2].

## Operating characteristics
*   **Time Complexity:** Rendering one $512 \times 512 \times 3$ image takes approximately 1-4 seconds on a single Nvidia Tesla V100 GPU [S1, unofficial]. Training takes approximately 60 hours [S1, unofficial].
*   **Memory Complexity:** Not given in the sources [S1, unofficial].
*   **Strengths:** Decouples the skeletal structure from the surface mesh, allowing initialization with any skeleton pose estimator [S1, unofficial].
*   **Limitations:** Relies on accurate initial pose estimation. Does not inherently model non-rigid deformation (e.g., muscle bulge) unless combined with other techniques, as it focuses on rigid bone transformations [S1, unofficial].

## Background
A-NeRF was introduced in "A-NeRF: Articulated Neural Radiance Fields for Learning Human Shape, Appearance, and Pose" [S1, unofficial]. It builds on the principles of NeRF by incorporating articulated human motion [S1, unofficial]. The skeleton representation is equivalent to SMPL joints but removes the parametric surface model, providing a more flexible prior for pose estimation [S1, unofficial].

## Sources
[S1] A-NeRF: Articulated Neural Radiance Fields for Learning Human Shape, Appearance, and Pose, https://doi.org/10.48550/arxiv.2102.06199, fetched 2026-10-30 (unofficial)
[S2] Neural Articulated Radiance Field, https://doi.org/10.1109/iccv48922.2021.00571, fetched 2026-10-26
