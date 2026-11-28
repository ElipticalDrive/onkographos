# Principal Curvature Direction Estimation

Also called: Discrete Curvature Estimation (Monga et al.), Monga's algorithm

## Overview
A method to compute principal curvature directions in 3D scalar fields faster than extremal value quadratic forms, by computing an orthonormal basis for the gradient and solving for the angle theta [S1, unofficial]. It sets itself apart from sibling methods like [Lambertian Shading with Gradient Normals](1229-lambertian-shading-with-gradient-normals.md) by providing directional curvature information rather than just surface orientation, and from [Discrete Gaussian Curvature Estimation](14915-discrete-gaussian-curvature-estimation.md) by offering continuous directional vectors rather than topological classification.

## How it works
**Inputs:**
- Scalar field $F(x, y, z)$.
- Gradient vector $G = \nabla F = (g_0, g_1, g_2)^T$.
- Hessian matrix $D^2 F$ (second derivatives of $F$).

**Outputs:**
- Principal curvature directions in the plane orthogonal to $G$.
- Angle $\theta$ defining the direction within that plane.

**Data Structures:**
- Gradient vector $G$: 3 floats.
- Scalar $\gamma$: 1 float.
- Orthonormal matrix $P$: 3x3 floats.
- Angle $\theta$: 1 float.

**Steps:**
1. Compute the gradient magnitude components:
   $$ \gamma = \sqrt{g_0^2 + g_1^2} $$
   $$ \|G\| = \sqrt{g_0^2 + g_1^2 + g_2^2} $$

2. Handle degenerate cases:
   - If $\gamma = 0$, the matrix construction fails. Use a standard basis instead [S1, unofficial].
   - If $\|G\| = 0$, the point is a critical point; curvature is undefined.

3. Construct the orthonormal basis vectors $h$ and $f$ in the plane orthogonal to $G$:
   $$ h = \begin{pmatrix} \frac{g_1}{\gamma} \\ -\frac{g_0}{\gamma} \\ 0 \end{pmatrix} $$
   $$ f = \begin{pmatrix} \frac{g_2 \cdot g_0}{\gamma \cdot \|G\|} \\ \frac{g_2 \cdot g_1}{\gamma \cdot \|G\|} \\ \frac{\gamma}{\|G\|} \end{pmatrix} $$
   The first column of $P$ is $G/\|G\|$. The matrix $P$ rotates the first basis vector into the direction of $G$ [S1, unofficial].

4. Express any vector $v$ in the orthogonal complement of $G$ as:
   $$ v = h \cdot \cos \theta + f \cdot \sin \theta $$

5. Solve for $\theta$ using the derivative of the curvature $\kappa_v$ with respect to $\theta$ vanishing:
   $$ \tan 2\theta = \frac{2 \cdot h'(D^2 F)f}{h'(D^2 F)h - f'(D^2 F)f} $$
   where $h'(D^2 F)h$ denotes the second derivative of the scalar field $F$ in the direction of the basis vectors [S1, unofficial].

6. Resolve orientation flips during path line tracing using a heuristic with a globally defined reference field (e.g., cylindrical) to align directions consistently [S1, unofficial].

**Pitfalls:**
- If $\gamma = 0$, the matrix construction fails; use a standard basis instead [S1, unofficial].
- At umbilical points (locally flat/spherical surfaces), the denominator vanishes. Handle by increasing the filter width of the differentiation operator [S1, unofficial].
- The algorithm only solves for $\theta$ in $(-\pi/2, +\pi/2)$, causing curvature directions to flip 180 degrees along field lines, preventing consistent orientation for tracing [S1, unofficial].

## Subtopics

### [Discrete Gaussian Curvature Estimation](14915-discrete-gaussian-curvature-estimation.md)
This method classifies vertices on a triangular mesh as elliptic (local extrema) or hyperbolic (saddle/blending regions) by evaluating the sign of the discrete Gaussian curvature. It distinguishes itself from continuous curvature estimation by providing a robust Boolean classification based on the angular defect, which is computationally cheaper and less sensitive to noise than second-derivative approximations. It builds on [Principal Curvature Direction Estimation](6492-principal-curvature-direction-estimation.md) by offering a topological classification rather than directional magnitude.

## Operating characteristics
**Time Complexity:** Faster than extremal value quadratic forms [S1, unofficial].
**Memory Complexity:** Not given in the sources.
**Measured Performance:** Not given in the sources.
**Scaling:** Not given in the sources.
**Strengths:** Considerably faster method for computing principal directions in three-dimensional scalar fields [S1, unofficial].
**Limitations:** Only solves for $\theta \in (-\pi/2, +\pi/2)$, resulting in curvature fields that are not oriented consistently. Curvature directions might flip about 180 degrees along a particular field line. Tracing these fields in a globally consistent way becomes impossible without a heuristic [S1, unofficial].

## Background
Monga et al. derived this method for computing principal directions in three-dimensional scalar fields [S1, unofficial]. It is used in interactive volume illustration and surface hatching [S1, unofficial].

## Sources
[S1] Interactive volume illustration, https://openalex.org/W2147354080, fetched 2026-10-27 (unofficial)
