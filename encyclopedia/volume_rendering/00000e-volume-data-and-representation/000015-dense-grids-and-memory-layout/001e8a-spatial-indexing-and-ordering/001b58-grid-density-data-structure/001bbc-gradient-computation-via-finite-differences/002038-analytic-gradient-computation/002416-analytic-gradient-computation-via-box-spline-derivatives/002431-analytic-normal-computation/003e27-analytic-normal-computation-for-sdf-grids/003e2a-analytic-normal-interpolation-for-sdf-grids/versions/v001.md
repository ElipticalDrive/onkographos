# Analytic Normal Interpolation for SDF Grids

Also called: Proposed normal method, Dual Voxel Normal Interpolation

## Overview
This method computes continuous surface normals for ray marching through Signed Distance Function (SDF) grids by evaluating analytic derivatives at the ray hit point within a "dual voxel" framework, rather than interpolating normals defined at voxel corners. It solves the discontinuity artifacts inherent in standard trilinear normal interpolation or single-voxel analytic gradients by trading strict geometric accuracy for visual smoothness across voxel boundaries. Unlike its sibling [Analytic Normal Computation for SDF Grids](9645-analytic-normal-computation-for-sdf-grids.md), which computes normals strictly within the local voxel domain, this method evaluates normals from neighboring voxels at the global hit point to ensure $C^0$ continuity of the normal field.

## How it works
**Inputs:**
- Ray hit point $\mathbf{p} = (x, y, z)$ in world space.
- Voxel grid dimensions and spacing.
- SDF values $s_{ijk}$ for a $4 \times 4 \times 4$ neighborhood centered on the voxel containing the hit point (or sufficient neighbors to cover the dual voxel influence).

**Outputs:**
- Interpolated surface normal $\mathbf{n}$.

**Data Structures:**
- SDF Grid: Stores $s_{ijk}$ values. To support this method efficiently, the implementation must store $4 \times 4 \times 4$ SDF values per voxel to avoid excessive runtime lookups [S1, unofficial].

**Step 1: Define the Dual Voxel**
A "dual voxel" is defined as a voxel shifted by half the voxel dimensions in all axes relative to the standard grid. Any hit point $\mathbf{p}$ falls inside exactly one dual voxel. This dual voxel overlaps $2 \times 2 \times 2$ standard voxels. Let the indices of these 8 surrounding standard voxels be $(i, j, k)$ where $i, j, k \in \{0, 1\}$ relative to the dual voxel's local coordinate system.

**Step 2: Compute Local Coordinates**
Calculate the position $(u, v, w)$ of the hit point $\mathbf{p}$ within the dual voxel. These coordinates are normalized to the range $[0, 1]^3$.

**Step 3: Evaluate Analytic Normals**
For each of the 8 surrounding voxels $(i, j, k)$, compute the analytic normal $\mathbf{n}_{ijk}$ at the hit point $\mathbf{p}$. This involves computing the gradient of the SDF polynomial defined by that voxel's data, evaluated at $\mathbf{p}$, and normalizing the result. The gradient components are computed using bilinear interpolation of distance differences. For the x-component:

$$
\frac{\partial f}{\partial x} = (1-v)(1-w)(s_{100} - s_{000}) + v(1-w)(s_{110} - s_{010}) + (1-v)w(s_{101} - s_{001}) + vw(s_{111} - s_{011})
$$

Similar formulas apply for $\frac{\partial f}{\partial y}$ and $\frac{\partial f}{\partial z}$, using the appropriate SDF values from the $4 \times 4 \times 4$ neighborhood associated with voxel $(i,j,k)$. The normal is then:

$$
\mathbf{n}_{ijk} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|}
$$

Note that for 7 out of 8 voxels, the hit point $\mathbf{p}$ lies outside the standard domain of that voxel, meaning the analytic normal is computed "outside" the voxel's local bounds [S1, unofficial].

**Step 4: Trilinear Interpolation of Normals**
Interpolate the 8 computed normals $\mathbf{n}_{ijk}$ using trilinear interpolation weights based on $(u, v, w)$. The final normal $\mathbf{n}$ is:

$$
\mathbf{n} = \sum_{i=0}^{1} \sum_{j=0}^{1} \sum_{k=0}^{1} w_{ijk}(u,v,w) \mathbf{n}_{ijk}
$$

where the weights $w_{ijk}$ are the standard trilinear basis functions:
$$
w_{ijk} = u^i (1-u)^{1-i} v^j (1-v)^{1-j} w^k (1-w)^{1-k}
$$

**Pitfalls:**
- **Domain Extrapolation:** Normals are computed outside the usual domain of a voxel in seven out of eight cases. This extrapolation is necessary for continuity but means the normals do not correspond exactly to the original discontinuous geometric normals at voxel boundaries [S1, unofficial].
- **Storage Requirement:** The method requires storing $4 \times 4 \times 4$ SDF values per voxel. Reducing this to $2 \times 2 \times 2$ makes the runtime substantially larger due to increased lookup overhead [S1, unofficial].
- **Geometric Accuracy:** The resulting normals are smooth but do not strictly correspond to the mathematical normals of the surface. The method trades geometric accuracy for visual smoothness [S1, unofficial].

## Subtopics

### Dual Voxel Normal Interpolation
This is the core mechanism of the method. It defines the dual voxel as a voxel shifted by half the voxel dimensions in all axes. Any hit point falls inside a single dual voxel, which overlaps $2 \times 2 \times 2$ neighboring voxels. The method evaluates the analytic normal in each of the 8 surrounding voxels at the hit point's coordinates and interpolates the 8 normals using trilinear interpolation weights based on the hit point's position $(u, v, w)$ within the dual voxel. Unlike PN triangles, which compute normals corresponding to the surface, this method computes continuous normals that appear reasonable but do not strictly correspond to the mathematical normals of the surface [S1, unofficial]. Simple trilinear interpolation of normals shared at voxel corners suffers from quality issues and is insufficient for reasonable normals in complex scenes [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The method is more expensive than single-voxel analytic normals. It uses 1.1–4.2% more time per frame on average for GST-A and 1.0–7.3% more time per frame on average for SVS-A compared to the baseline method [S1, unofficial].
- **Memory Complexity:** Requires storing $4 \times 4 \times 4$ SDF values per voxel. Without this storage, the runtime was substantially larger [S1, unofficial].
- **Performance:** Tested on an Intel Core i9 10980XE (3.00 GHz) with 128 GB DDR4 (2.4 GHz) and NVIDIA RTX 3090. The method provides superior results for closeups compared to Falcao normals and standard analytic normals [S1, unofficial].
- **Strengths:** Delivers continuous normals across voxel boundaries, avoiding the visual artifacts of discontinuous normals.
- **Limitations:** Does not correspond exactly to the original geometric normals; requires significant memory overhead for SDF storage.

## Background
This method was proposed in the context of ray tracing SDF grids to address the discontinuity of normals at voxel boundaries. It is similar in spirit to the approach used for PN triangles [Vlachos et al. 2001], where normals are interpolated to achieve smooth shading. The method was introduced in "Ray tracing of signed distance function grids" [S1, unofficial].

## Sources
[S1] Ray tracing of signed distance function grids, https://www.jcgt.org/published/0011/03/06/paper-lowres.pdf, fetched 2026-10-25 (unofficial)
