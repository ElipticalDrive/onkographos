# Gaussian Density Field Representation

Also called: Gaussian + Hashed Residual Field

## Overview
A compact representation of the density field $D$ as a weighted sum of Gaussians plus a hashed residual field $F$, allowing efficient modeling of fine density details [S1]. It sets itself apart from pure Gaussian splatting methods by explicitly separating coarse structure (Gaussians) from high-frequency residuals (hashed field), enabling precise reconstruction of complex media without the aliasing or smoothing artifacts common in single-resolution approaches [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $n$ Gaussian parameters $\{w_j, c_j, r_j\}_{j=1}^n$ representing the coarse density field, and optionally a hashed residual field data structure.
*   **Output:** The total density value $D(x)$ at any spatial point $x$.

**Data Structures:**
1.  **Gaussian List:** An array of $n$ Gaussians. Each Gaussian $j$ is defined by:
    *   $w_j$: Weight (scalar).
    *   $c_j$: Center (3D vector).
    *   $r_j$: Radius (scalar).
2.  **Hashed Residual Field:** A hash table mapping spatial coordinates (or grid cells) to residual values $R(x)$. The specific hash function and bucket size are not detailed in the sources [S1].

**Formulas:**
The density field $D(x)$ is defined as the sum of the Gaussian approximation $\tilde{D}(x)$ and the residual field $F(x)$ [S1]:

$$D(x) = \tilde{D}(x) + F(x) = \sum_{j=1}^n w_j \exp\left(-\frac{\|x - c_j\|^2}{r_j^2}\right) + F(x)$$

Where:
*   $x$ is the query point in 3D space.
*   $\| \cdot \|$ denotes the Euclidean norm.
*   $F(x)$ is the value retrieved from the hashed residual field. If no residual field is used, $F(x) = 0$.

**Steps:**
1.  **Preprocessing:** Compute the sequence of Gaussians and residual fields for the target density distribution. This is done offline or as a preparatory step [S1].
2.  **Runtime Evaluation (Direct):** For a query point $x$:
    a. Compute $\tilde{D}(x) = \sum_{j=1}^n w_j \exp\left(-\frac{\|x - c_j\|^2}{r_j^2}\right)$.
    b. Retrieve $R(x)$ from the hashed residual field.
    c. Set $F(x) = R(x)$.
    d. Return $D(x) = \tilde{D}(x) + F(x)$.
3.  **Runtime Evaluation (Splatting/Texture-based):** To construct the density field for rendering (e.g., into a texture):
    a. Initialize texture channels. Channel 1 stores $\tilde{D}(x)$, Channel 2 stores $R(x)\tilde{D}(x)$.
    b. For each Gaussian $j$, splat its weight $w_j$ into Channel 1. No weight normalization is performed [S1].
    c. If a residual field hash table exists, for each entry $(x, R(x))$, multiply $R(x)$ by the current value of $\tilde{D}(x)$ at that location and store the result in Channel 2 [S1].
    d. After all splatting is complete, for each pixel/voxel:
        i. Let $V_1$ be the value in Channel 1 ($\tilde{D}(x)$).
        ii. Let $V_2$ be the value in Channel 2 ($R(x)\tilde{D}(x)$).
        iii. Compute $R(x) = V_2 / V_1$ (handling division by zero if $V_1 \approx 0$) [S1].
        iv. Compute $D(x) = V_1 + R(x)$ [S1].

**Parameters:**
*   $n$: Number of Gaussians. Not given in the sources.
*   Hash table size: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Preprocessing Limitation:** Preprocessing is only for the representation of the density field. It does not prevent run-time changes to media properties, lighting, or scene configuration [S1].
*   **Division by Zero:** In the splatting method, dividing $V_2$ by $V_1$ requires handling cases where $\tilde{D}(x)$ is zero or near-zero [S1].
*   **Residual Field Absence:** If no residual field is used, the method degrades to pure Gaussian sum, potentially losing fine details [S1].

## Subtopics

### Density Field Construction via Splatting
A method for constructing the density field into a texture buffer for efficient rendering. It splats Gaussian weights into one channel and the product of residual and Gaussian density into another, then divides to recover the residual field per-pixel. It is chosen when the rendering pipeline supports texture-based splatting and requires fast, parallel construction of the density field for ray marching or ray tracing. It differs from direct evaluation by leveraging GPU parallelism for field construction rather than per-point query [S1].

## Operating characteristics
*   **Time Complexity:** Evaluation is $O(n)$ per point for the Gaussian sum, plus $O(1)$ for hash lookup if a residual field is present. Splatting construction is $O(n + m)$ where $m$ is the number of residual entries, assuming constant-time hash operations.
*   **Memory Complexity:** $O(n)$ for Gaussians, plus $O(m)$ for the residual hash table.
*   **Strengths:** Efficient modeling of fine density field details; allows run-time changes to lighting and media properties despite preprocessing [S1].
*   **Limitations:** Requires preprocessing for the density field representation; accuracy depends on the number of Gaussians and the resolution/coverage of the residual field [S1].
*   **When to use:** When high-frequency density details are needed and can be captured by a residual field, and when the rendering algorithm can accommodate rapid reconstruction (e.g., via splatting) [S1].
*   **When not to use:** When memory is extremely constrained and $n$ is large, or when the density field is static and simple enough to be represented by Gaussians alone without residuals [S1].

## Background
Originated from the work of Zhou et al. (2007b) and extended in the context of gradient-based interpolation and sampling for real-time rendering of inhomogeneous, single-scattering media [S1]. The Gaussian model was chosen for its ability to compactly represent density fields, while the hashed residual field was added to capture details that the Gaussian sum alone could not [S1]. The method is used in real-time rendering pipelines for volumetric media, particularly where animated sequences require efficient density field reconstruction per frame [S1].

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
