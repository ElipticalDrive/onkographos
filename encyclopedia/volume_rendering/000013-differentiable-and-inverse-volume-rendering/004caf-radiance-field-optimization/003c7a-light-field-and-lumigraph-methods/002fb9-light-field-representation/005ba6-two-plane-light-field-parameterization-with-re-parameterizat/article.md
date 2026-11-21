# Two-plane light field parameterization with re-parameterization

Also called: Local affine transform embedding, Ray-space embedding

## Overview
This method parameterizes a light field using two planes, $\pi_{xy}$ and $\pi_{uv}$, and learns a re-parameterization (local affine transforms) to align the neural network's interpolation kernels with the object surface $\pi_{st}$ [S1, unofficial]. It solves the problem of misaligned interpolation kernels in standard two-plane parameterizations, which degrades view synthesis quality when the distance between the view plane and the object surface is large [S1, unofficial]. It sets itself apart from standard two-plane methods by explicitly optimizing the embedding space to minimize the discrepancy between the parameterized coordinates and the true surface coordinates [S1, unofficial].

## How it works
**Inputs:**
- A set of input images with known camera poses.
- A scene geometry (implicitly or explicitly defined, e.g., via a textured plane $\pi_{st}$ or a voxel grid).
- Two initial parameterization planes: $\pi_{xy}$ (view plane) and $\pi_{uv}$ (viewpoint plane).

**Outputs:**
- A re-parameterized light field representation where rays are mapped to a latent space aligned with the object surface.
- A neural network (MLP) that maps the re-parameterized ray coordinates to color and density.

**Data Structures:**
- **Ray Representation:** A ray $r$ is defined by its origin on $\pi_{xy}$ and its intersection with $\pi_{uv}$.
- **Voxel Grid:** For subdivided models, a 3D grid of voxels $\upsilon$ is used to cover the scene space (e.g., NDC space).
- **Embedding Network:** An MLP that takes the re-parameterized ray coordinates as input and outputs the ray's color and density.

**Steps:**
1. **Initial Parameterization:**
   - Define $\pi_{xy}$ and $\pi_{uv}$. For the Stanford Light Field dataset, $\pi_{xy}$ is set to $z = -1$ and $\pi_{uv}$ is set to $z = 0$ [S1, unofficial].
   - Scale camera positions to lie between $[-0.25, 0.25]$ in $x$ and $y$ [S1, unofficial].
   - Scale pixel coordinates on $\pi_{uv}$ to lie between $[-1, 1]$ [S1, unofficial].
   - For each ray $r$ originating at $(\hat{x}, \hat{y})$ on $\pi_{xy}$ and passing through a point $(\hat{s}, \hat{t})$ on the object surface $\pi_{st}$, calculate its intersection $(\hat{u}, \hat{v})$ with $\pi_{uv}$ using similar triangles [S1, unofficial]:
     $$ \hat{u} - \hat{x} = (\hat{s} - \hat{x}) \frac{z_{uv}}{z_{st}} $$
     $$ \hat{v} - \hat{y} = (\hat{t} - \hat{y}) \frac{z_{uv}}{z_{st}} $$
     where $z_{st}$ and $z_{uv}$ are the depths of $\pi_{st}$ and $\pi_{uv}$ respectively, and the depth of $\pi_{xy}$ is assumed to be 0 [S1, unofficial].

2. **Re-parameterization (Subdivided Model):**
   - For a subdivided model, re-parameterize each ray $r$ within a voxel $\upsilon$ [S1, unofficial].
   - Transform space so that the voxel center lies at the origin [S1, unofficial].
   - Intersect the transformed ray with the voxel’s front and back planes [S1, unofficial].
   - Take the $xy$ coordinates of these intersections as the parameterization, which lie within the range $[-\text{voxel width}, +\text{voxel width}]$ in all dimensions [S1, unofficial].

3. **Learning Re-parameterization:**
   - Learn a re-parameterization that moves $\pi_{uv}$ towards $\pi_{st}$, reducing the distance $z_{st} - z_{uv}$ [S1, unofficial].
   - The finite capacity of the light field MLP drives the embedding network to map rays intersecting the same 3D point to the same point in latent space [S1, unofficial].
   - This process aligns the color network's interpolation kernels with the object surface $\pi_{st}$ [S1, unofficial].

**Parameters:**
- **Stanford Light Field Dataset:** $\pi_{xy}$ at $z = -1$, $\pi_{uv}$ at $z = 0$ [S1, unofficial]. Camera positions scaled to $[-0.25, 0.25]$ in $x$ and $y$ [S1, unofficial]. Pixel coordinates on $\pi_{uv}$ scaled to $[-1, 1]$ [S1, unofficial].
- **Real Forward-Facing and Shiny Datasets:** Experiments performed in NDC space [S1, unofficial]. A $32^3$ voxel grid covers all NDC space, except for CD/Lab scenes which use a coarser $4^3$ grid [S1, unofficial].

**Pitfalls:**
- If the distance $z_{st} - z_{uv}$ is large, the difference between $(\hat{u}, \hat{v})$ and $(\hat{s}, \hat{t})$ increases, causing the network's interpolation kernels to become misaligned with the object surface, degrading reconstruction quality [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This method originates from the paper "Learning Neural Light Fields with Ray-Space Embedding Networks" [S1, unofficial]. It addresses the limitations of standard two-plane parameterization by introducing a learnable re-parameterization to improve view synthesis quality.

## Sources
[S1] Learning Neural Light Fields with Ray-Space Embedding Networks, https://doi.org/10.48550/arxiv.2112.01523, fetched 2026-10-26 (unofficial)
