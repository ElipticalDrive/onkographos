# Fourier Slice Theorem
Also called: Central Slice Theorem, FST, Théorème de Radon, Théorème de la Tranche Centrale

## Overview
The Fourier Slice Theorem (FST), also known as the Central Slice Theorem, establishes that the 1D Fourier transform of a projection of a function at angle $\theta$ is equal to a radial slice through the 2D Fourier transform of the original function along the same angle [S1, S2, unofficial]. It serves as the theoretical foundation for Filtered Back-Projection (FBP) by enabling the reconstruction of a 2D image from its projections via frequency-domain operations rather than direct spatial-domain algebraic inversion [S1, S2, unofficial]. Unlike iterative or algebraic methods that solve linear systems directly, FST allows for analytical reconstruction by filling the 2D frequency domain with 1D Fourier transforms of projections [S1, unofficial].

## How it works
**Inputs and Outputs**
- **Input**: A set of 1D projections $R_\theta(\rho)$ acquired at various angles $\theta$, representing line integrals of the original 2D function $f(x,y)$.
- **Output**: The reconstructed 2D function $f(x,y)$.

**Data Structures**
- **Projections**: Array of 1D signals $R_\theta(\rho)$ for discrete angles $\theta_i$.
- **Frequency Domain**: 2D grid representing the Fourier transform of $f$, denoted $F^{2D}$.
- **Coordinate Systems**: Cartesian $(x,y)$ for spatial domain; Polar $(r, \phi)$ for spatial domain; Cartesian $(\nu_x, \nu_y)$ or Polar $(\nu, \psi)$ for frequency domain.

**Mathematical Definitions**
Let $f(x,y)$ be the original 2D function. The Radon transform (projection) at angle $\theta$ is:
$$ R_\theta(\rho) = \iint f(x,y) \delta(\rho - x \cos \theta - y \sin \theta) \, dx \, dy $$
where $\delta$ is the Dirac delta function, $\rho$ is the distance from the origin, and $\theta$ is the angle of the projection normal.

The 1D Fourier transform of the projection $R_\theta(\rho)$ is:
$$ F_\theta(\nu) = \int_{-\infty}^{\infty} R_\theta(\rho) e^{-2i\pi\nu\rho} \, d\rho $$
where $\nu$ is the spatial frequency variable [S1, S2, unofficial].

The 2D Fourier transform of $f(x,y)$ in Cartesian coordinates is:
$$ \hat{f}(\nu_x, \nu_y) = \iint f(x,y) e^{-2i\pi(x\nu_x + y\nu_y)} \, dx \, dy $$
The theorem states that evaluating this 2D transform along a line passing through the origin at angle $\theta$ yields the 1D Fourier transform of the projection at that angle [S1, unofficial]:
$$ \hat{f}(\nu \cos \theta, \nu \sin \theta) = F_\theta(\nu) $$
where $\nu_x = \nu \cos \theta$ and $\nu_y = \nu \sin \theta$ [S1, unofficial].

**Derivation Steps**
1.  **Coordinate Transformation**: Express $f(x,y)$ in polar coordinates: $x = r \cos \phi$, $y = r \sin \phi$. Substitute into the Radon integral definition. The Jacobian of the transformation introduces a factor of $|r|$ [S2, unofficial].
    $$ R_\theta(\rho) = \int_{0}^{2\pi} \int_{-\infty}^{\infty} f(r \cos \phi, r \sin \phi) \delta(\rho - r \cos(\phi - \theta)) |r| \, dr \, d\phi $$
2.  **1D Fourier Transform of Projection**: Compute the 1D Fourier transform $F_\theta(\nu)$ of $R_\theta(\rho)$ [S2, unofficial].
    $$ F_\theta(\nu) = \int_{-\infty}^{\infty} R_\theta(\rho) e^{-2i\pi\nu\rho} \, d\rho $$
3.  **Substitution and Simplification**: Substitute the polar form of $R_\theta(\rho)$ into the 1D Fourier transform integral. Use the property of the Dirac delta function, which is non-zero only when $\rho = r \cos(\phi - \theta)$, to eliminate the $\rho$ integral [S2, unofficial].
    $$ F_\theta(\nu) = \int_{0}^{2\pi} \int_{-\infty}^{\infty} f(r \cos \phi, r \sin \phi) |r| e^{-2i\pi\nu r \cos(\phi - \theta)} \, dr \, d\phi $$
4.  **2D Fourier Transform in Polar Coordinates**: Define the 2D Fourier transform $F^{2D}(\psi, \nu)$ in polar frequency coordinates, where $X = \nu \cos \psi$ and $Y = \nu \sin \psi$ [S2, unofficial].
    $$ F^{2D}(\psi, \nu) = \int_{0}^{2\pi} \int_{-\infty}^{\infty} f(r \cos \phi, r \sin \phi) |r| e^{-2i\pi\nu r \cos(\phi - \psi)} \, dr \, d\phi $$
5.  **Equivalence**: Comparing the results of Step 3 and Step 4, it is evident that:
    $$ F_\theta(\nu) = F^{2D}(\psi, \nu) \big|_{\psi=\theta} $$
    This confirms that the 1D Fourier transform of the projection at angle $\theta$ is exactly the slice of the 2D Fourier transform of the object at angle $\theta$ [S2, unofficial].

**Reconstruction Algorithm**
1.  For each projection angle $\theta_i$:
    a. Compute the 1D Fourier transform of the projection $R_{\theta_i}(\rho)$ to obtain $F_{\theta_i}(\nu)$.
2.  Map the values $F_{\theta_i}(\nu)$ into the 2D frequency domain grid $F^{2D}(\nu_x, \nu_y)$ at coordinates $(\nu_x, \nu_y) = (\nu \cos \theta_i, \nu \sin \theta_i)$.
    *   *Note*: Since projections provide data along radial lines in polar frequency coordinates, interpolation (e.g., linear or nearest-neighbor) is required to map these values onto a Cartesian grid [S2, unofficial].
3.  Compute the 2D Inverse Fourier Transform of the filled frequency domain grid $F^{2D}(\nu_x, \nu_y)$ to recover the spatial domain function $f(x,y)$ [S2, unofficial].

**Parameters**
- $\nu$: Spatial frequency variable. Maps to coordinates $(\nu \cos \theta, \nu \sin \theta)$ in the 2D frequency plane [S1, unofficial].
- $\theta$: Projection angle.

**Edge Cases and Pitfalls**
- **Discretization**: The theorem is exact in the continuous domain. In practice, finite number of projections and discrete sampling introduce artifacts (e.g., streaking) if not handled with appropriate interpolation and filtering (such as the ramp filter in FBP) [S2, unofficial].
- **Aliasing**: Insufficient sampling in the frequency domain (too few projections or low resolution) leads to aliasing artifacts in the reconstructed image [S2, unofficial].
- **Boundary Effects**: The assumption of infinite support for the Fourier transform can cause edge artifacts if the object is not zero-padded sufficiently [G].

## Operating characteristics
- **Time Complexity**: Dominated by the Fast Fourier Transform (FFT) operations. Computing 1D FFTs for $N_p$ projections of length $N$ takes $O(N_p \cdot N \log N)$. The 2D inverse FFT on an $N \times N$ grid takes $O(N^2 \log N)$.
- **Memory Complexity**: Requires storage for the projections ($O(N_p \cdot N)$) and the 2D frequency domain grid ($O(N^2)$).
- **Strengths**: Analytical solution; computationally efficient for large datasets due to FFTs; well-understood error characteristics.
- **Limitations**: Sensitive to noise and missing data (limited angle tomography); requires interpolation for polar-to-Cartesian mapping; assumes infinite support or requires windowing to reduce artifacts.
- **When to use**: Standard CT reconstruction with sufficient angular coverage and high signal-to-noise ratio.
- **When not to use**: Highly sparse data, limited angle tomography, or when iterative regularization is preferred for noise suppression.

## Background
The theorem was demonstrated by the Austrian mathematician Johann Radon in 1917 [S1, unofficial]. It provides the essential link between projections and the pixel values of the slice to be reconstructed. Historically, 2D reconstruction methods were classified into analytical methods (continuous approach, seeking analytical inversion formulas) and algebraic methods (discrete approach, treating the problem as linear system inversion) [S1, unofficial]. The Fourier Slice Theorem is the cornerstone of the analytical approach, specifically Filtered Back-Projection [S1, S2, unofficial].

## Sources
[S1] Élaboration d'une méthode tomographique de reconstruction 3D en vélocimétrie par image de particules basée sur les processus ponctuels marqués, https://hal.science/tel-01292656v3, fetched 2026-10-26 (unofficial)
[S2] Précision et qualité en reconstruction tomographique : algorithmes et applications, https://doi.org/10.70675/d2bc4875z6a5az46afz9289z2fe16b3a43f3, fetched 2026-10-30 (unofficial)
