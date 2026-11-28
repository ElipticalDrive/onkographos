# Specific Implicit Primitives


## Overview
This family provides closed-form analytical expressions for the scalar field $f(\vec{x})$ of specific geometric primitives, primarily yielding exact signed Euclidean distances [G]. Unlike generic implicit surface definitions that may rely on approximations or piecewise kernels, these functions offer precise evaluation for standard shapes like tori, shells, and periodic surfaces, which is critical for robust marching cubes and ray marching in volume rendering [G].

## Shared principles
Builds on [Implicit Surface Primitives](1800-implicit-surface-primitives.md). All methods evaluate a scalar field function $f$ over a domain $\Omega \subseteq \mathbb{R}^3$ on-demand without persistent data structures [G]. The geometry is computed directly via mathematical formulas, typically returning the exact signed distance to ensure $\|\nabla f(\vec{x})\| \approx 1$ for stable gradient-based rendering [G].

## Subtopics

### [Torus Implicit Function](1322-torus-implicit-function.md)
Defines a torus as the zero-level set of a scalar field, providing the exact signed Euclidean distance from a query point to the surface. It is distinguished from approximate quadric representations by its analytical closed-form solution, which avoids the gradient singularities and approximation errors inherent in first-order Taylor expansions of implicit quadrics. Choose it when exact distance queries are required for ray marching or CSG operations.

### [Soft Objects](20725-soft-objects.md)
A class of implicit surface primitives defined by a bounded, exponentially-decaying field function attributed to the Wyvill brothers. They solve the problem of creating smooth, organic shapes that can be easily morphed and deformed by adjusting control primitive attributes. Unlike Meta balls, which require square root computations for distance normalization, Soft Objects rely solely on squared distances, offering a slight computational advantage. Choose them for organic modeling and morphable volume fields.

### Gyroid Surface Evaluation
*   **Definition:** Evaluates the scalar field for a Triply Periodic Minimal Surface (TPMS) Gyroid at point $(x,y,z)$ [S1, unofficial].
*   **Formula:** $f(x,y,z) = \sin(sx)\cos(sy) + \sin(sy)\cos(sz) + \sin(sz)\cos(sx)$, where $sx = x \cdot scale, sy = y \cdot scale, sz = z \cdot scale$ [S1, unofficial].
*   **Parameter:** `scale`: The scaling factor for the gyroid coordinates. Default value is 4.5 [S1, unofficial].

### Thin Shell Implicit Function
*   **Definition:** Implicit function for a thin spherical shell: $f(p) = |x^2+y^2+z^2 - r^2| - \epsilon$ [S2, unofficial].
*   **Parameter:** `radius`: Sphere radius for the shell, default 1.0 [S2, unofficial].
*   **Parameter:** `epsilon`: Half-thickness of the shell, default 1e-3 [S2, unofficial].

### Rectangular Prism Generation
*   **Definition:** Generates a rectangular prism centered at the origin [S3, unofficial].
*   **Parameter:** `x, y, z`: Prism dimensions. Default: 5x5x5. Even dimensions round up to the next odd number [S3, unofficial].
*   **Layout:** Returns a data.frame of voxel coordinates [S3, unofficial].
*   **Pitfall:** Only odd side lengths are generated. Even dimensions are rounded up to the next odd number due to integer coordinate quantization [S3, unofficial].

### Meta Balls
*   **Definition:** A bounded field function where control primitives have no influence after a maximum distance 'b', solving the computational expense of infinite-range functions [S4, unofficial].
*   **Step:** Osaka University graduate students implemented a piecewise quadratic approximation to the Gaussian for faster ray-surface intersection testing, avoiding iterative root finders by solving a quadratic equation [S4, unofficial].
*   **Parameter:** 'a' is a scaling factor; 'b' is the maximum distance a control primitive contributes to the field [S4, unofficial].
*   **History:** Term 'metaballs' (or misspelled 'meatballs') was used by people at Osaka University and Toyo Links in Japan [S4, unofficial].

### Blobby Molecules
*   **Definition:** A field function attributed to Jim Blinn, modeled after electron density fields, using a Gaussian curve centered at the origin [S4, unofficial].
*   **Pitfall:** The field function extends to infinity, requiring summation of contributions from all control primitives for any point calculation, which becomes computationally expensive as the number of primitives increases [S4, unofficial].
*   **History:** Proposed by Jim Blinn for animating DNA for Carl Sagan's COSMOS TV Series, where each atom was approximated by a Gaussian potential [S4, unofficial].

## Background
The term "metaballs" (or misspelled "meatballs") was used by researchers at Osaka University and Toyo Links in Japan for blobby models [S4, unofficial]. Jim Blinn proposed approximating atoms with Gaussian potentials for DNA animation in Carl Sagan's COSMOS TV series, which became known as blobby molecules [S4, unofficial]. The Wyvill brothers introduced Soft Objects, a bounded field function using a truncated exponential series expansion that avoids square roots by relying on squared distances [S1, unofficial]. The Gyroid surface is a Triply Periodic Minimal Surface (TPMS) evaluated via a trigonometric scalar field [S1, unofficial]. Rectangular prism generation is implemented in voxel visualization tools like the `isocubes` R package, which quantizes coordinates to integer values for isometric rendering [S3, unofficial].

## Sources
[S1] nff747/marching-cubes-wgsl, https://github.com/nff747/marching-cubes-wgsl, fetched 2026-10-27 (unofficial)
[S2] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S3] isocubes: Voxel Data Visualization with Isometric Cubes, https://doi.org/10.32614/cran.package.isocubes, fetched 2026-10-26 (unofficial)
[S4] Implicit surfaces - Paul Bourke, https://paulbourke.net/geometry/implicitsurf/, fetched 2026-10-28 (unofficial)
