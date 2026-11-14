# Scene-Related Transforms


## Overview
Scene-related transforms are a class of imaging transforms that operate primarily on scenes, as opposed to structure-related transforms which operate on objects extracted from scenes [S1]. They facilitate the transformation of information between scene, object, and view spaces, assuming the orientation and position relative to the scene or object coordinate system are known [S1]. This article builds on [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md).

## Shared principles
The family operates on the Special Euclidean group SE(3), which consists of all rigid transformations in 3D space. A rigid transformation $T$ is defined by a rotation matrix $R \in SO(3)$ and a translation vector $t \in \mathbb{R}^3$.

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

This property ensures that the conjugate of the rotation is the exact inverse, allowing for round-trip consistency [S1].

## Subtopics

### Camera Coordinate Homogenization and Rigid Transformation
This sibling transform handles the specific homogenization and rigid transformation steps required for camera coordinates. It is chosen when converting between camera-specific coordinate systems and the global scene space. See [Camera Coordinate Homogenization and Rigid Transformation](4005-camera-coordinate-homogenization-and-rigid-transformation.md).

## Background
The classification of imaging transforms into scene-related and structure-related categories originates from the work of Udupa and Goncalves [S1]. These transforms are applicable to both grey-level and binary scenes, although their applicability to binary scenes is sometimes trivial [S1]. The transforms described are used to map information among scene, object, and view spaces [S1].

## Sources
[S1] Imaging transforms for visualizing surfaces and volumes, https://doi.org/10.1007/bf03168529, fetched 2026-10-29
