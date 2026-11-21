# Precomputed Radiance Transfer (PRT)
Also called: Directional surface prelighting
## Overview
Precomputed Radiance Transfer (PRT) is a technique that models light interaction with surfaces using transfer functions to map incoming light to a description of radiance distribution, rather than operating on actual radiance values directly [S1, unofficial]. Unlike full baking which is static and inflexible, PRT offers greater flexibility by allowing arbitrary lighting setups at runtime, though it incurs a higher runtime cost than full baking [S1, unofficial].
## Shared principles
The core mechanism of PRT relies on the linearity of light transport, which allows the contribution of each light to be computed independently and summed to achieve the final lighting value [S1, unofficial]. This process defines a vector space where the precomputed unit contributions of individual lights serve as the basis vectors [S1, unofficial].

The final radiance $L(p)$ for a point $p$ is calculated as:

$$L(p) = \sum_i L_i(p) W_i$$

where:
*   $L(p)$ is the final radiance for point $p$ [S1, unofficial].
*   $L_i(p)$ is the precomputed unit contribution of light $i$ [S1, unofficial].
*   $W_i$ is the current brightness of light $i$ [S1, unofficial].

The representation of the transfer function (e.g., spherical harmonics) is independent of the basis function used to represent the final lighting [S1, unofficial].
## Subtopics
## Background
The concept of precomputed radiance transfer was introduced to computer graphics by Sloan et al. [S1, unofficial]. While they originally formulated the method in terms of spherical harmonics, the approach does not strictly require their usage [S1, unofficial]. Sloan et al. stored the scene's response to light using a distribution defined by a spherical harmonic basis function, modeling the response for a number of spherical harmonic bands to enable rendering with any arbitrary lighting setup [S1, unofficial].
## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
