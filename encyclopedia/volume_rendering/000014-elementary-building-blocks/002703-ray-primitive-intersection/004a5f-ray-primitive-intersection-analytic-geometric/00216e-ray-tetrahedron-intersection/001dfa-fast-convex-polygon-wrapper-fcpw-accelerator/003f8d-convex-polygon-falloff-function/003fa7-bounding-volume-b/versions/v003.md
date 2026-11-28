# Bounding Volume {B}
Also called: AO Bounding Polyhedron, Polyhedral Bounding Volume for AO [S1, unofficial]

## Overview
The Bounding Volume {B} method defines the spatial extent of the ambient occlusion volume (AOV) for a convex polygon $P$ by constructing a closed polyhedron $V$ bounded by $k+2$ planes [S1, unofficial]. It enables efficient rasterization-based occlusion queries by explicitly computing vertex displacement vectors to extrude the polygon into a 3D volume [S1, unofficial].

## How it works
**Inputs:**
- Convex polygon $P$ with $k$ vertices $\{\vec{p}_0, \dots, \vec{p}_{k-1}\}$, no three collinear [S1, unofficial].
- Maximum occlusion distance $\delta$ [S1, unofficial].
- Edge normals $\hat{m}_i$ for $i \in [0, k-1]$, pointing inward relative to the polygon projection [S1, unofficial].
- Face normal $\hat{m}_k$ (normal to the plane containing $P$) [S1, unofficial].

**Outputs:**
- A polyhedron $V$ defined by $2k$ vertices $\{\vec{v}_0, \dots, \vec{v}_{2k-1}\}$ [S1, unofficial].
- The bounding planes $\{B_0, \dots, B_{k+1}\}$ [S1, unofficial].

**Data Structures:**
- **Vertices:** Array of $2k$ 3D vectors [S1, unofficial]. Indices $0$ to $k-1$ lie in the plane of $P$ [S1, unofficial]. Indices $k$ to $2k-1$ are displaced along $\hat{m}_k$ [S1, unofficial].
- **Normals:** Array of $k+2$ 3D unit vectors [S1, unofficial]. $\hat{m}_0 \dots \hat{m}_{k-1}$ are edge normals; $\hat{m}_k$ is the face normal; $\hat{m}_{k+1}$ is the normal for the "above" plane [S1, unofficial].

**Steps:**
1. **Define Plane Normals and Reference Points:** Set $\hat{m}_0, \dots, \hat{m}_{k-1}$ as the inward-facing normals of the polygon edges [S1, unofficial]. Set $\hat{m}_k$ as the unit normal to the plane containing $P$ [S1, unofficial]. Define $\vec{p}_k = \vec{p}_0$ and $\vec{p}_{k+1} = \vec{p}_0 + \delta \hat{m}_{k+1}$, where $\hat{m}_{k+1}$ is the normal of the plane $B_{k+1}$ containing $P$ [S1, unofficial].
2. **Compute Extension Vectors $\vec{e}_i$ for Base Vertices ($0 \le i < k$):** The extension vector $\vec{e}_i$ displaces polygon vertex $\vec{p}_i$ to volume vertex $\vec{v}_i$ [S1, unofficial]. It is constrained by the geometry of the bounding planes:
   $$ \vec{e}_i \cdot -\hat{m}_i = \delta \quad (9) $$
   $$ \vec{e}_i \cdot -\hat{m}_{(i-1+k) \bmod k} = \delta \quad (10) $$
   $$ \vec{e}_i \cdot \hat{m}_k = 0 \quad (11) $$
   Solve for $\vec{e}_i$ using the linear system:
   $$ \vec{e}_i = \begin{bmatrix} \hat{m}_i \\ \hat{m}_{(i-1+k) \bmod k} \\ \hat{m}_k \end{bmatrix}^{-1} \begin{bmatrix} -\delta \\ -\delta \\ 0 \end{bmatrix} \quad (12) $$
   [S1, unofficial].
3. **Clamp Extension Vectors:** To prevent sliver polygons from creating excessively long volumes with negligible occlusion, clamp the magnitude of each extension vector: $\|\vec{e}_i\| \le 2\delta$ for all $i < k$ [S1, unofficial].
4. **Compute Top Extension Vectors ($k \le i < 2k$):** The vertices in the "top" plane are displaced from the base vertices along the face normal:
   $$ \vec{e}_i = \vec{e}_{i-k} - \delta \hat{m}_k \quad (13) $$
   [S1, unofficial].
5. **Construct Volume Vertices:** For $0 \le i < k$: $\vec{v}_i = \vec{p}_i + \vec{e}_i$ [S1, unofficial]. For $k \le i < 2k$: $\vec{v}_i = \vec{p}_{i-k} + \vec{e}_i$ [S1, unofficial].
6. **Define Bounding Planes:** Planes $B_0, \dots, B_{k-1}$ correspond to polygon edges with normal $\hat{m}_i$ [S1, unofficial]. Plane $B_k$ is above $P$ [S1, unofficial]. Plane $B_{k+1}$ contains $P$ [S1, unofficial]. The plane equation is given by $(\vec{x} - \vec{p}_i) \cdot \hat{m}_i = \delta$ [S1, unofficial].

**Simplified Falloff Evaluation:** When evaluating the falloff function $g$ for a point at the origin, the formula simplifies to:
$$ g(\vec{0}) = \bar{\alpha} \prod_{i=0}^k \max(0, \min(1, 1 - \vec{p}_{i \bmod k} \cdot \hat{m}_i / \delta)) \quad (8) $$
Here, $\bar{\alpha}$ is the average alpha of the polygon [S1, unofficial].

**Pitfalls:** Sliver polygons produce long, thin volumes that are computationally expensive to rasterize but contribute little occlusion; the clamp $\|\vec{e}_i\| \le 2\delta$ is critical [S1, unofficial]. Incorrect normals invert the volume or create empty intersections [G]. Solving the $3 \times 3$ linear system may suffer from numerical instability if the polygon is nearly degenerate [G].

## Operating characteristics
Not given in the sources.

## Background
The Bounding Volume {B} method was introduced in the paper "Ambient Occlusion Volumes" [S1, unofficial]. It was developed to enable efficient rasterization-based ambient occlusion for complex geometry, particularly foliage and fences, by converting polygonal occluders into volumetric primitives that can be processed by standard graphics pipelines [S1, unofficial]. The method relies on the Convex Polygon Falloff Function to define the occlusion intensity within the volume [S1, unofficial].

## Sources
[S1] Ambient Occlusion Volumes, https://research.nvidia.com/sites/default/files/pubs/2010-06_Ambient-Occlusion-Volumes/McGuire10AOV.pdf, fetched 2026-10-28 (unofficial)
