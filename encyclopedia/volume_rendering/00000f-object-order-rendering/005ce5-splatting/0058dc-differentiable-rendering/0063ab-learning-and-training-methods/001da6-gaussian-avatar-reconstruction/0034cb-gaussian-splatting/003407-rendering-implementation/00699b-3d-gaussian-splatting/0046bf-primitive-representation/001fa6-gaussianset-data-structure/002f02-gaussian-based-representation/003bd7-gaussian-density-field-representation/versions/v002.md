# Gaussian Density Field Representation

Also called: Gaussian + Hashed Residual Field

## Overview
A compact representation of the density field $D$ as a weighted sum of Gaussians plus a hashed residual field $F$, allowing efficient modeling of fine density details [S1]. It sets itself apart from pure Gaussian splatting methods by explicitly separating coarse structure from high-frequency residuals, enabling precise reconstruction of complex media without requiring a single ultra-high-resolution representation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $n$ Gaussian parameters $\{w_j, c_j, r_j\}_{j=1}^n$ representing the coarse density field, and optionally a hashed residual field data structure [S1].
*   **Output:** The total density value $D(x)$ at any spatial point $x$ [S1].

**Data Structures:**
1.  **Gaussian List:** An array of $n$ Gaussians. Each Gaussian $j$ is defined by weight $w_j$ (scalar), center $c_j$ (3D vector), and radius $r_j$ (scalar) [S1].
2.  **Hashed Residual Field:** A hash table mapping spatial coordinates to residual values $R(x)$ [S1]. Hash table size and bucket configuration: Not given in the sources.

**Steps:**
1.  **Preprocessing:** Compute the sequence of Gaussians and residual fields for the target density distribution [S1].
2.  **Runtime Evaluation (Direct):** For a query point $x$, compute the Gaussian sum $\tilde{D}(x)$, retrieve $R(x)$ from the hashed residual field, set $F(x) = R(x)$, and return $D(x) = \tilde{D}(x) + F(x)$ [S1].
3.  **Runtime Evaluation (Splatting/Texture-based):** To construct the density field for rendering, initialize texture channels [S1]. For each Gaussian $j$, splat its weight $w_j$ into Channel 1 to accumulate $\tilde{D}(x)$, with no weight normalization [S1]. If a residual field hash table exists, retrieve $R(x)$ for each entry, multiply it by the current $\tilde{D}(x)$ at that location, and store the result in Channel 2 [S1]. After splatting, for each pixel/voxel, divide the value in Channel 2 by the value in Channel 1 to obtain $R(x)$, then add $R(x)$ to $\tilde{D}(x)$ to yield the final density $D(x)$ [S1].

**Formulas:**
The density field $D(x)$ is defined as the sum of the Gaussian approximation $\tilde{D}(x)$ and the residual field $F(x)$ [S1]:

$$D(x) = \tilde{D}(x) + F(x) = \sum_{j=1}^n w_j \exp\left(-\frac{\|x - c_j\|^2}{r_j^2}\right) + F(x)$$

Where:
*   $x$ is the query point in 3D space [S1].
*   $\|\cdot\|$ denotes the Euclidean norm [S1].
*   $F(x)$ is the value retrieved from the hashed residual field [S1]. If no residual field is used, $F(x) = 0$ [G].

**Parameters:**
*   $n$: Not given in the sources.
*   Hash table size: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Preprocessing Limitation:** Preprocessing is only for the representation of the density field and does not prevent run-time changes to media properties, lighting, or scene configuration [S1].

## Subtopics

### Density Field Construction via Splatting
A texture-based method for constructing the density field per frame by splatting Gaussian weights and residual products into separate color channels, then dividing to recover the residual field [S1]. It sets itself apart from direct per-point evaluation by leveraging parallel texture operations for field construction rather than sequential queries [S1]. It is chosen when the rendering pipeline supports texture-based splatting and requires fast, frame-by-frame reconstruction of the density field for ray marching or ray tracing [S1]. See [Gaussian-based Representation](5918-gaussian-based-representation.md).

## Operating characteristics
*   **Strengths:** Efficient modeling of fine density field details; allows run-time changes to lighting and media properties despite preprocessing [S1].
*   **Limitations:** Requires preprocessing for the density field representation; accuracy depends on the number of Gaussians and the resolution/coverage of the residual field [S1].
*   **When to use:** When high-frequency density details are needed and can be captured by a residual field, and when the rendering algorithm can accommodate rapid reconstruction (e.g., via splatting) [S1].
*   **When not to use:** Not given in the sources.

## Background
Originated from the work of Zhou et al. (2007b) and extended in the context of gradient-based interpolation and sampling for real-time rendering of inhomogeneous, single-scattering media [S1]. The Gaussian model was chosen for its ability to compactly represent density fields, while the hashed residual field was added to capture details that the Gaussian sum alone could not [S1]. The method is used in real-time rendering pipelines for volumetric media, particularly where animated sequences require efficient density field reconstruction per frame [S1].

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
