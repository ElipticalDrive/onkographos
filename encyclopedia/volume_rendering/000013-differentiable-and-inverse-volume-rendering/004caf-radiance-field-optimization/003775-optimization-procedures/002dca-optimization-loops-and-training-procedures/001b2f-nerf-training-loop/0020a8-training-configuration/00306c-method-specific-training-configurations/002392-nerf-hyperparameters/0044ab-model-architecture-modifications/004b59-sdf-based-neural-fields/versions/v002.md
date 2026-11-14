# SDF-based Neural Fields

Also called: SDF NeRF, Signed Distance Function Neural Fields

## Overview
SDF-based Neural Fields replace the standard NeRF density prediction with a Signed Distance Function (SDF) to provide a more explicit and smooth formulation of surfaces, addressing the geometric inaccuracy and surface holes typical of density-based NeRFs [S1, unofficial]. Unlike standard NeRFs which model volume density $\sigma$, this method models the distance to the nearest surface, allowing for precise geometry extraction via the zero-level set.

## Shared principles
The core mechanism involves replacing the density output $\sigma(\mathbf{x})$ with an SDF value $s(\mathbf{x})$. The SDF is a continuous scalar field $s: \mathbb{R}^3 \rightarrow \mathbb{R}$ where $s(\mathbf{x})$ represents the distance to the closest surface for any 3D point $\mathbf{x} \in \mathbb{R}^3$ [S1, unofficial]. The sign convention is typically negative inside the object and positive outside [S1, unofficial]. The surface geometry is defined as the zero-level set:
$$
\{ \mathbf{x} \mid s(\mathbf{x}) = 0 \}
$$
To enable volumetric rendering, the predicted SDF value $s(\mathbf{x})$ must be converted into a density $\sigma$. While the exact conversion function varies by implementation, it generally involves a differentiable mapping from distance to opacity/density, often using a sigmoid or similar function parameterized by a sharpness term, to allow gradients to flow through the rendering equation [S1, unofficial]. This substitution allows the network to learn explicit surfaces rather than implicit volumetric blobs.

## Subtopics
Not given in the sources.

## Background
The approach emerged as a prominent research direction to improve surface quality in neural radiance fields, addressing limitations of density-based representations [S1, unofficial]. It builds on the foundational NeRF architecture [S6, unofficial] by substituting the density head with an SDF head. Key citations in this lineage include works referenced as [130, 196, 177, 95, 179] in the source literature, which focus on representing geometry by combining NeRF with SDFs [S1, unofficial].

## Sources
[S1] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
