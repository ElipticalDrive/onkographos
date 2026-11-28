# Inverse LBS Motion Field

Also called: Inverse Linear Blend Skinning

## Overview
Inverse LBS Motion Field is a module that transforms a query point from posed space to unposed (canonical) space using inverse Linear Blend Skinning (LBS), serving as the base for residual deformation in neural volumetric representations of dynamic humans [S1]. It differs from forward skinning methods by explicitly computing the inverse of the weighted bone transformation matrices to map points back to a canonical reference frame, which is then used to condition occupancy networks or predict residuals [S1, S2].

## How it works
**Inputs and Outputs:**
- Input: A query point $x$ in posed space, blend weights $w$ (associated with the point), and human pose parameters $\rho$ which define the bone transformations.
- Output: The coordinate $x_{can}$ in canonical (unposed) space.

**Data Structures:**
- $x$: A 3D vector representing the query point in posed space.
- $w$: A vector of blend weights $w_{k,j}$, where $k$ indexes the point and $j$ indexes the bone. The weights sum to 1 for each point.
- $G_j$: A $4 \times 4$ transformation matrix for bone $j$, derived from the pose parameters $\rho$. These matrices represent the rigid transformation of each bone in the posed skeleton.
- $J$: The total number of bones in the skeleton.

**Formula:**
The transformation is defined as [S1]:

$$
\Phi_{LBS} (x, w, \rho) = \left( \sum_{j=1}^{J} w_{k,j} G_j \right)^{-1} x
$$

Where:
- $\Phi_{LBS}$ is the inverse LBS motion field function.
- $x$ is the query point (treated as a homogeneous coordinate vector, typically with $w=1$ for the last component, though the formula notation implies matrix-vector multiplication).
- $w_{k,j}$ is the weight of bone $j$ for point $k$.
- $G_j$ is the transformation matrix of bone $j$.
- The term $\sum_{j=1}^{J} w_{k,j} G_j$ computes the weighted sum of bone matrices, representing the forward skinning transformation.
- The inverse of this sum is applied to $x$ to obtain the canonical coordinate.

**Steps:**
1. Retrieve the pose parameters $\rho$ and compute the bone transformation matrices $G_j$ for all $J$ bones.
2. For a given query point $x$, retrieve its associated blend weights $w_{k,j}$ for all bones $j=1 \dots J$.
3. Compute the weighted sum of the bone transformation matrices: $M = \sum_{j=1}^{J} w_{k,j} G_j$.
4. Compute the inverse of the matrix $M$: $M^{-1}$.
5. Apply the inverse matrix to the query point: $x_{can} = M^{-1} x$.

**Pitfalls:**
- Matrix inversion can be numerically unstable if the weighted sum of matrices is singular or near-singular. This may occur if the pose configuration leads to degenerate bone orientations or if weights are poorly distributed.
- The method assumes that the inverse of the weighted sum of matrices exists. In cases where the forward skinning transformation is not invertible (e.g., extreme deformations causing self-intersection or collapse), this method fails.

## Subtopics

### Inverse Linear Blend Skinning for Occupancy Networks
This variant, used in methods like LEAP, integrates inverse LBS into occupancy networks. It sends transformation matrices of all bones into shape, structure, and pose encoders as global features, obtains local features through a projection module for each bone, and uses the transformation matrix of each bone to learn inverse LBS. The local features of all bones are then weighed using inverse LBS to condition the occupancy network [S2]. A key pitfall is that adopting reverse skinning makes the learning of the canonical space pose dependent on the specific pose, making it difficult to handle unseen poses [S2].

### SNARF
SNARF avoids the direct learning of the reverse skinning function. Instead, it maps the posed shape into canonical space through reverse skinning but solves for the inverse of the forward skinning according to geometry cycle consistency [S2]. It uses an iterative root-seeking method to handle situations where the solution is not closed when calculating forward skinning inversely [S2]. This approach aims to mitigate the issues of pose dependency found in direct inverse LSS methods.

## Operating characteristics
- **Time Complexity:** The core operation involves matrix multiplication and inversion. For $J$ bones, computing the weighted sum takes $O(J)$ matrix multiplications and additions. Inverting a $4 \times 4$ matrix is $O(1)$ but involves a fixed number of operations. Thus, the complexity per query point is $O(J)$.
- **Memory Complexity:** Requires storing $J$ bone transformation matrices $G_j$ and the blend weights $w$ for each query point.
- **Strengths:** Provides a direct, differentiable mapping from posed to canonical space, which is essential for training neural fields on dynamic data. It leverages the well-established LBS framework.
- **Limitations:** Numerical instability in matrix inversion can occur. The method is pose-dependent, which can lead to difficulties with unseen poses if not regularized or handled carefully (as noted in the Occupancy Networks variant) [S2].
- **When to use:** When a direct, explicit inverse mapping is required for neural volumetric representations of humans, and the pose space is well-covered during training.
- **When not to use:** When dealing with highly degenerate poses where the skinning matrix might be singular, or when computational resources are extremely limited and iterative methods (like SNARF) are preferred for robustness despite higher per-step cost.

## Background
Inverse LBS Motion Field originates from the need to represent dynamic human avatars in a canonical space for neural volumetric rendering. The specific formulation $\Phi_{LBS} (x, w, \rho) = \left( \sum_{j=1}^{J} w_{k,j} G_j \right)^{-1} x$ is cited from [S1], which references [35] for the inverse LBS module. The concept is further explored in surveys of implicit neural representations for human modeling, where variants like LEAP and SNARF are discussed [S2]. LEAP adopts reverse skinning to condition occupancy networks [S2], while SNARF uses geometry cycle consistency to solve the inverse [S2].

## Sources
[S1] Learning Neural Volumetric Representations of Dynamic Humans in Minutes, https://doi.org/10.1109/cvpr52729.2023.00846, fetched 2026-10-30
[S2] Human 3D Avatar Modeling with Implicit Neural Representation: A Brief Survey, https://doi.org/10.1109/icsps58776.2022.00148, fetched 2026-10-26
