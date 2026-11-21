# Rigid Body Transformation (SE3)

Also called: SE(3) transformation, Rigid transformation, RigidXform, SE(3) Camera Transformation

## Overview
This family defines the mathematical and algorithmic framework for applying rigid body transformations—rotations and translations—to 3D geometry and camera poses within volume rendering pipelines. It solves the problem of mapping coordinates between different reference frames (e.g., local cell space to world space, or camera space to world space) while preserving distances and angles. It is distinguished from [Rotation Transformation](2279-rotation-transformation.md) and [Translation Transformation](2281-translation-transformation.md) by treating rotation and translation as a single coupled operation in the Special Euclidean group $SE(3)$, and from [Pose Recentering](1786-pose-recentering.md) by its general applicability to any rigid mapping rather than just centering.

## Shared principles
The family operates on the Special Euclidean group $SE(3)$, which consists of all rigid transformations in 3D space. A rigid transformation $T$ is defined by a rotation matrix $R \in SO(3)$ and a translation vector $t \in \mathbb{R}^3$.

To perform these operations linearly, points and vectors are represented in homogeneous coordinates. A 3D point $p = (x, y, z)$ is represented as $p_h = (x, y, z, 1)^T$, while a direction vector $v = (x, y, z)$ is represented as $v_h = (x, y, z, 0)^T$. The distinction is critical: appending 1 allows translation to affect the position, while appending 0 ensures translation does not affect the direction.

The transformation matrix $T$ is a $4 \times 4$ matrix constructed as:

$$
T = \begin{bmatrix}
R & t \\
0 & 1
\end{bmatrix}
$$

where $R$ is a $3 \times 3$ orthogonal matrix with determinant 1, and $t$ is a $3 \times 1$ column vector. The bottom row is $[0, 0, 0, 1]$ to preserve the homogeneous coordinate.

The transformation of a homogeneous point $p_h$ is given by matrix multiplication:

$$
p'_h = T \cdot p_h
$$

For a point, this yields:

$$
\begin{bmatrix}
x' \\
y' \\
z' \\
1
\end{bmatrix}
=
\begin{bmatrix}
R & t \\
0 & 1
\end{bmatrix}
\begin{bmatrix}
x \\
y \\
z \\
1
\end{bmatrix}
=
\begin{bmatrix}
R \begin{bmatrix} x \\ y \\ z \end{bmatrix} + t \\
1
\end{bmatrix}
$$

For a vector, the translation component $t$ is multiplied by 0, resulting in pure rotation:

$$
\begin{bmatrix}
x' \\
y' \\
z' \\
0
\end{bmatrix}
=
\begin{bmatrix}
R & t \\
0 & 1
\end{bmatrix}
\begin{bmatrix}
x \\
y \\
z \\
0
\end{bmatrix}
=
\begin{bmatrix}
R \begin{bmatrix} x \\ y \\ z \end{bmatrix} \\
0
\end{bmatrix}
$$

The inverse transformation $T^{-1}$ is also a rigid transformation, given by:

$$
T^{-1} = \begin{bmatrix}
R^T & -R^T t \\
0 & 1
\end{bmatrix}
$$

This property ensures that the conjugate of the rotation is the exact inverse, allowing for round-trip consistency [S1, unofficial].

## Subtopics

### [Camera Coordinate Homogenization and Rigid Transformation](4005-camera-coordinate-homogenization-and-rigid-transformation.md)
This method converts 3D points and direction vectors into homogeneous coordinates and applies a rigid body transformation (rotation and translation) to map them between coordinate systems, such as from camera space to world space. It differs from its sibling [Scene-Related Transforms](11646-scene-related-transforms.md) by explicitly handling the distinction between points and vectors via the homogeneous fourth component, ensuring translation affects positions but not directions, whereas siblings may handle scene-relative adjustments without this strict separation.

### [Scene-Related Transforms](11646-scene-related-transforms.md)
Scene-related transforms are a class of imaging transforms that operate primarily on scenes, as opposed to structure-related transforms which operate on objects extracted from scenes. They facilitate the transformation of information between scene, object, and view spaces, assuming the orientation and position relative to the scene or object coordinate system are known. This article builds on [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md).

### [Camera Pose Optimization via SE(3) Decomposition](17547-camera-pose-optimization-via-se-3-decomposition.md)
This method optimizes camera poses, represented as $SE(3)$ transformations, during the training of generative radiance fields by decomposing the transformation matrix into manageable components. It solves the problem of manifold escape, where direct optimization of rotation matrices causes them to violate orthogonality constraints ($RR^T \neq I$). Unlike standard parametric approaches that optimize Euler angles directly, this method parameterizes rotations using sine and cosine pairs, ensuring valid rotation matrices through normalization.

### RigidXform
#### definition
- [S1, unofficial] A transformation structure that maps a volume's local cell space to world metres by applying a uniform cell scale, a rotation, and a translation.
#### step
- [S1, unofficial] Construct RigidXform from a Pose by dequantising the quaternion, renormalising it, and rejecting if the orientation is zero.
#### parameter
- [S1, unofficial] cell_size_m: uniform scale factor converting local cells to world metres.
#### pitfall
- [S1, unofficial] The stored rotation is normalised, ensuring the conjugate is the exact inverse of the rotation.

### SE(3) Camera Transformation
#### definition
- [S2, unofficial] Applies an SE(3) transformation to a batch of camera-to-world matrices, preserving the scale of the rotation part.
#### step
- [S2, unofficial] Validate inputs: matrix must be 4x4, camtoworlds must be Nx4x4.
- [S2, unofficial] Apply transformation using einsum: camtoworlds = np.einsum('nij, ki -> nkj', camtoworlds, matrix).
- [S2, unofficial] Normalize the rotation part to ensure pure SE(3) (no scaling): compute scaling = np.linalg.norm(camtoworlds[:, 0, :3], axis=1), then divide camtoworlds[:, :3, :3] by scaling[:, None, None].

## Background
The mathematical foundation of rigid body transformations in 3D space is rooted in the Special Euclidean group $SE(3)$. In the context of volume rendering and computer graphics, this group is essential for defining camera poses and transforming geometric primitives between local and world coordinate systems.

The specific implementation details of `RigidXform` originate from the Spall project, a CPU-only volume rendering engine. The `RigidXform` structure was introduced to handle the mapping of local cell spaces to world metres, supporting deterministic ray traversal via DDA (Amanatides & Woo) [S1, unofficial].

The optimization technique using sine/cosine decomposition for $SE(3)$ poses was developed for training generative radiance fields, specifically within the `mrGSplat` project. This approach addresses the challenges of optimizing camera poses on the manifold of rotation matrices without violating orthogonality constraints [S2, unofficial].

## Sources
[S1] T03 — Fixture worlds, rays, and integer brushes, https://github.com/deranjer/spall/pull/4, fetched 2026-10-25 (unofficial)
[S2] meshroomHub/mrGSplat, https://github.com/meshroomHub/mrGSplat, fetched 2026-10-27 (unofficial)
