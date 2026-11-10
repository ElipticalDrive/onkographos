# KRS Texturing


## Overview
KRS Texturing is a shading technique for Kernel-Reflection Sequences (KRS) that applies procedural or triplanar textures to kernel solids to eliminate observable repetitiveness [S1, unofficial]. It addresses the visual artifact where applying identical coordinates to all kernel instances produces an extremely repetitive pattern that undesirably emphasizes symmetries [S1, unofficial]. It sets itself apart from standard parametric texturing by handling complex or reflective kernels where simple $u,v$ parametrization is not feasible [S1, unofficial].

## How it works
**Inputs:** KRS geometry definition (kernel sets, expansion levels, reflection rules); texture definition (procedural or image-based); ray intersection point; surface normal.
**Outputs:** Final color value for the pixel.

**Data Structures:**
*   **Kernel Solid:** Defined by a distance function $d(\mathbf{x})$ and its gradient $\nabla d(\mathbf{x})$.
*   **Transformation Chain:** A matrix or quaternion stack representing the composition of Möbius transformations ($\mathcal{L}_{G_i}$) and reflections ($\mathcal{L}_R$) applied to reach the current kernel space from world space [S1, unofficial].

**Steps:**
1.  **Determine Parametrization Feasibility:** Check if the kernel solid allows simple $u,v$ parametrization. If the kernel is simplistic (e.g., infinite planes) or the parametrization is non-trivial, $u,v$ mapping is infeasible [S1, unofficial].
2.  **Select Texturing Method:**
    *   If feasible: Apply standard $u,v$ parametrization based on the kernel's local coordinates [S1, unofficial].
    *   If not feasible: Apply procedural 3D texturing or triplanar texturing [S1, unofficial].
3.  **Eliminate Repetition:**
    *   Combine procedural geometry with procedural or wrapped textures [S1, unofficial].
    *   Use triplanar projection to blend textures along world-space axes, ensuring that mirrored or transformed instances do not share identical texture coordinates [S1, unofficial].
4.  **Sample Texture:** Evaluate the texture function at the transformed coordinate. For triplanar texturing, project the world-space position onto the $xy$, $yz$, and $xz$ planes, sample each, and blend based on the surface normal's dominant axis [G].
5.  **Shading:** Combine the texture color with lighting calculations (local or global) [S1, unofficial].

**Formulas:**
*   **Triplanar Blending:**
    $$ C = \frac{C_{xy} |n_z| + C_{yz} |n_x| + C_{xz} |n_y|}{|n_x| + |n_y| + |n_z|} $$
    Where $C_{xy}, C_{yz}, C_{xz}$ are texture samples projected onto the respective planes, and $n_x, n_y, n_z$ are components of the surface normal [G].
*   **Gradient Transformation:** To shade in world space, transform the kernel gradient $\nabla d_{kernel}$ to world space $\nabla d_{world}$ using the Jacobian of the transformation chain $\mathcal{T}$:
    $$ \nabla d_{world} = (\mathcal{T}')^{-T} \nabla d_{kernel} $$
    Where $\mathcal{T}'$ is the derivative of the transformation [S1, unofficial].

**Parameters:**
*   **Texture Scale:** Controls the frequency of procedural noise or image repetition.
*   **Triplanar Blend Factor:** Weighting for normal-based blending.

**Edge Cases and Pitfalls:**
*   **Repetitive Patterns:** Using the same $u,v$ coordinates for all kernel instances creates visible repetition [S1, unofficial].
*   **Infeasible Parametrization:** $u,v$ parametrization fails for infinite planes or complex kernels where details are determined by reflection transformations [S1, unofficial].
*   **Normal Discontinuities:** If kernel separation is not upheld, surface normals may be discontinuous at mirror planes. Smooth shading can be achieved by interpolating between kernel-to-world-space matrices on either side of a mirror plane when the shaded point is near a mirror plane [S1, unofficial].

## Subtopics

### MöKRS Modeling
A variant of KRS where expansion levels are conformally transformed unions of kernel sets and reflected instances, using Möbius transformations to eliminate symmetry [S1, unofficial]. It models natural features by using conformal distortions to compensate for scale variation loss in isometries [S1, unofficial]. To ray-trace, for every expansion level, the current ray point is transformed with $\mathcal{L}_{G_i}$ to symmetry space, then to kernel space with identity or $\mathcal{L}_R$ based on a mirror side test [S1, unofficial]. It uses speculative sphere tracing (Binary Sphere Tracing) for conformally mapped geometries [S1, unofficial]. Spheres or infinite planes are sufficient kernel sets [S1, unofficial]. The iteration formula is $L_{i+1} = \mathcal{L}_{\mathbf{G}_i}(K_{i+1} \cup L_i \cup \mathcal{L}_{\mathbf{R}}(L_i))$, where $\mathcal{L}_R$ is a Möbius transformation with matrix $\mathbf{R} = \begin{bmatrix} -i & 0 \\ 0 & i \end{bmatrix}$, representing reflection across the $x=0$ plane [S1, unofficial]. Smooth shading and procedural or triplanar texturing are applied to enhance visual quality [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Texturing is $O(1)$ per shading point, dependent on the complexity of the procedural texture or triplanar blending. Sphere tracing dominates the overall cost.
*   **Memory Complexity:** Low, as textures are procedural or sampled on the fly. No large texture atlases required for KRS instances.
*   **Strengths:** Eliminates repetitive artifacts in fractal-like KRS geometries; handles complex reflective kernels where parametric texturing fails; integrates with global illumination techniques [S1, unofficial].
*   **Limitations:** Requires careful selection of transformation chains to avoid normal discontinuities; procedural texturing can be computationally expensive if high-frequency details are needed [S1, unofficial].
*   **When to Use:** When rendering KRS geometries with high symmetry or reflection complexity where standard parametric texturing fails or produces visible repetition [S1, unofficial].
*   **When Not to Use:** For simple, non-reflective kernels where standard $u,v$ parametrization is sufficient and cheaper [S1, unofficial].

## Background
KRS Texturing was introduced as part of the Kernel-Reflection Sequences framework, which allows for the generation of complex, fractal-like geometries using simple kernel solids and reflection rules [S1, unofficial]. The method was developed to address the visual limitations of early KRS rendering, specifically the repetitive patterns caused by uniform texturing across symmetric instances [S1, unofficial]. It builds on the mathematical foundation of Möbius transformations and conformal geometry, enabling the modeling of natural features with scale-invariant properties [S1, unofficial]. The technique is used in real-time rendering contexts where procedural geometry generation is preferred over triangle meshes for dynamic or complex structures [S1, unofficial].

## Sources
[S1] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26 (unofficial)
