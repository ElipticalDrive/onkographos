# Reprojection Consistency Loss

Also called: Reprojection Error Loss, Static reprojection loss

## Overview
Reprojection Consistency Loss regularizes differentiable volume rendering by enforcing geometric consistency between adjacent frames using 2D optical flow. It projects volume-rendered 3D surface points from a source frame to a neighbor frame and minimizes the distance to the corresponding pixels estimated by a pre-trained optical flow network (e.g., RAFT) [S1]. Unlike sibling methods that may rely on monocular depth priors or optimal transport, this method directly leverages dense 2D correspondence to constrain the rendered geometry, particularly effective for static scenes or as a component in dynamic field regularization [S1].

## How it works
**Inputs:**
- Volume rendering network parameters.
- Camera poses for source frame $i$ and neighbor frame $j$.
- Pre-computed 2D optical flow field $F_{i \rightarrow j}$ from a network like RAFT [S1].

**Outputs:**
- Scalar loss value $L_{s\_reproj}$.

**Data Structures:**
- Ray sampling points: Set of 3D points $\mathbf{X} = \{ \mathbf{x}_k \}$ sampled along rays in frame $i$.
- Rendered depth: For each ray, the accumulated depth $d$ to the surface point $\mathbf{x}_{surf}$ is obtained via volume rendering [S1].

**Steps:**
1. **Volume Rendering:** For each ray in frame $i$, sample 3D points and volume render to identify the surface point $\mathbf{x}_{surf} = (x, y, z)$ and its depth $d$ [S1].
2. **Reprojection:** Transform $\mathbf{x}_{surf}$ from frame $i$'s camera coordinate system to frame $j$'s camera coordinate system using the relative pose. Project the transformed 3D point onto the 2D image plane of frame $j$ to obtain the predicted pixel coordinates $(u_{pred}, v_{pred})$ [S1].
3. **Flow Lookup:** Retrieve the optical flow vector $\mathbf{f} = (f_u, f_v)$ from the pre-computed flow field $F_{i \rightarrow j}$ at the source pixel location $(u_i, v_i)$ corresponding to $\mathbf{x}_{surf}$ [S1].
4. **Error Calculation:** Compute the Euclidean distance between the reprojected pixel $(u_{pred}, v_{pred})$ and the flow-estimated target pixel $(u_i + f_u, v_i + f_v)$ [S1].

**Formula:**
The loss is the mean squared error over all rays:
$$ L_{s\_reproj} = \frac{1}{N} \sum_{k=1}^{N} \| (u_{pred}^{(k)}, v_{pred}^{(k)}) - (u_i^{(k)} + f_u^{(k)}, v_i^{(k)} + f_v^{(k)}) \|^2 $$
where $N$ is the number of rays, and $(u, v)$ are pixel coordinates [S1].

**Parameters:**
- Flow network: RAFT [S1].
- Neighbor frame selection: Typically adjacent frames in the sequence [S1].

**Edge Cases:**
- **Occlusions:** The flow field may be unreliable in occluded regions. The method assumes the flow network provides robust correspondences [S1].
- **Pure Rotation:** The static reprojection loss alone struggles with pure rotating cameras, as depth ambiguity is not resolved by 2D flow alone [S1].

## Subtopics

### Dynamic Reprojection Loss
Used in dynamic radiance fields where scene motion exists. It induces 2D flow using camera poses, rendered depth, and estimated 3D scene flow from a dedicated Scene Flow MLP, rather than relying solely on pre-computed 2D flow. The loss compares this induced flow against the RAFT-estimated flow [S1]. The 3D scene flow $\mathbf{S}_{i \rightarrow i+1}$ for a point $(x, y, z)$ at time $t_i$ is predicted by an MLP: $\mathbf{S}_{i \rightarrow i+1} = \text{MLP}_{\theta_{sf}}(x, y, z, t_i)$ [S1]. This allows handling of non-rigid or dynamic scenes where static reprojection fails [S1].

## Operating characteristics
- **Time Complexity:** Dominated by the volume rendering step and the reprojection transformation. The flow lookup is $O(1)$ per ray.
- **Memory Complexity:** Requires storing the pre-computed flow field for the sequence, which scales with image resolution and frame count [S1].
- **Strengths:** Provides strong geometric constraints using off-the-shelf flow networks; effective for static or slowly moving scenes [S1].
- **Limitations:** Fails under pure rotation without additional constraints (e.g., monocular depth loss); dependent on the accuracy of the pre-trained flow network [S1].
- **When to use:** When high-quality 2D optical flow is available and the scene is largely static or rigid.

## Background
Introduced in "Robust Dynamic Radiance Fields" (CVPR 2023) to address training instability in fast-moving or pure rotating camera trajectories [S1]. It builds on the observation that 2D optical flow provides dense correspondence cues that can regularize the 3D reconstruction process [S1].

## Sources
[S1] Robust Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00010, fetched 2026-10-26
