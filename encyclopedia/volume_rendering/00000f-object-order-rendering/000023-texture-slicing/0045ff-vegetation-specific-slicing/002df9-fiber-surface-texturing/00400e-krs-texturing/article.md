# KRS Texturing
## Overview
KRS Texturing is a shading technique for Kernel-Reflection Sequences (KRS) that applies procedural or triplanar textures to kernel solids to eliminate observable repetitiveness [S1]. It addresses the visual artifact where applying identical coordinates to all kernel instances produces an extremely repetitive pattern that undesirably emphasizes symmetries [S1]. It sets itself apart from standard parametric texturing by handling complex or reflective kernels where simple $u,v$ parametrization is not feasible [S1].

## How it works
**Inputs and Outputs:** Not given in the sources.
**Data Structures:** Kernel solids are defined by a distance function $d(\mathbf{x})$ and its gradient $\nabla d(\mathbf{x})$ [S1]. Transformation chains maintain the composition of Möbius transformations ($\mathcal{L}_{G_i}$) and reflections ($\mathcal{L}_R$) to map world space to kernel space [S1].
**Steps:**
1. Determine parametrization feasibility: Check if the kernel solid allows simple $u,v$ parametrization. If the kernel is simplistic (e.g., infinite planes) or the parametrization is non-trivial, $u,v$ mapping is infeasible [S1].
2. Select texturing method: Apply standard $u,v$ parametrization for simple kernel solids, or procedural/triplanar texturing for complex or reflective kernels [S1].
3. Eliminate repetition: Apply procedural 3D or triplanar texturing to eliminate observable repetitiveness [S1]. Combine procedural geometry and procedural or wrapped textures to further eliminate repetitiveness [S1].
4. Sample texture: Evaluate the texture function at the transformed coordinate. [G]
5. Shading: Combine the texture color with lighting calculations. [G]
**Formulas:**
* Triplanar blending projects the world-space position onto the $xy$, $yz$, and $xz$ planes, samples each, and blends based on the surface normal's dominant axis:
  $$ C = \frac{C_{xy} |n_z| + C_{yz} |n_x| + C_{xz} |n_y|}{|n_x| + |n_y| + |n_z|} $$
  Where $C_{xy}, C_{yz}, C_{xz}$ are texture samples and $n_x, n_y, n_z$ are normal components [G].
* Gradient transformation: To shade in world space, transform the kernel gradient $\nabla d_{\text{kernel}}$ to world space $\nabla d_{\text{world}}$ using the Jacobian of the transformation chain $\mathcal{T}$, as detailed in Appendix C of the source [S1].
**Parameters:** Texture scale and triplanar blend factors are not given in the sources.
**Edge Cases and Pitfalls:** Texturing all kernel instances with the same coordinates produces an extremely repetitive pattern, emphasizing symmetries undesirably [S1]. If kernel separation is not upheld, surface normals may be discontinuous at mirror planes; these can be smoothed by interpolating between kernel-to-world-space matrices on the two sides of a mirror when the shaded point is near a mirror plane [S1].

## Subtopics

### MöKRS Modeling
A KRS variant where expansion levels are conformally transformed unions of kernel sets and reflected instances, using Möbius transformations to eliminate symmetry [S1]. It models natural features using conformal distortions to compensate for scale variation loss in isometries [S1]. To ray-trace, for every expansion level, the current ray point is transformed with $\mathcal{L}_{G_i}$ to symmetry space, then to kernel space with the identity or $\mathcal{L}_R$ based on a mirror side test [S1]. It uses speculative sphere tracing (Binary Sphere Tracing) for conformally mapped geometries [S1]. Spheres or infinite planes are sufficient kernel sets [S1]. The iteration formula is $L_{i+1} = \mathcal{L}_{\mathbf{G}_i}(K_{i+1} \cup L_i \cup \mathcal{L}_{\mathbf{R}}(L_i))$, where $\mathcal{L}_R$ is a Möbius transformation with matrix $\mathbf{R} = \begin{bmatrix} -i & 0 \\ 0 & i \end{bmatrix}$, representing reflection across the $x=0$ plane [S1]. Smooth shading and procedural or triplanar texturing are applied to enhance visual quality [S1].

## Operating characteristics
Time and memory complexity are not given in the sources. Measured performance, scaling, strengths, limitations, and usage guidelines are not given in the sources.

## Background
KRS Texturing was introduced as part of the Kernel-Reflection Sequences framework, which allows for the generation of complex, fractal-like geometries using simple kernel solids and reflection rules [S1]. The method was developed to address the visual limitations of early KRS rendering, specifically the repetitive patterns caused by uniform texturing across symmetric instances [S1]. It builds on the mathematical foundation of Möbius transformations and conformal geometry, enabling the modeling of natural features with scale-invariant properties [S1].

## Sources
[S1] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26
