# Correction Texture for Surface Albedo

Also called: Correction Texture, Ground Reflection Correction, Surface Albedo Correction Texture

## Overview
This method corrects the main scattering texture for light reflected off the ground, which is precomputed assuming a homogeneous mean surface albedo $\bar{a}$ [S1]. It allows the renderer to remove these homogeneous contributions from the main texture and evaluate surface albedos and reflection characteristics separately at runtime for surfaces with varying properties [S1].

## How it works
**Inputs:**
- Main scattering texture (precomputed).
- Mean surface albedo $\bar{a}$ used during the main scattering precomputation.
- Surface reflection function $R(P_O, V, L, \lambda)$.
- Multiple scattering terms $A_{\{R,M\}}^{(k)}(P_O, V, L, \lambda)$ for orders $k=1$ to $K-1$.
- Maximum scattering order $K$.

**Outputs:**
- Correction texture $I_C$ with the same resolution and parameterization as the main scattering texture.

**Data Structures:**
- **Correction Texture**: A 4D texture (or 2D if viewing direction is neglected, though this loses directional phase function information [S1]) storing intensity values $I_C$. The parameterization matches the main scattering texture: position $P_O$, viewing direction $V$, light direction $L$, and wavelength $\lambda$.

**Steps:**
1. Construct a correction texture with the exact same resolution and parameterization as the main scattering texture [S1].
2. For each texel in the correction texture, corresponding to parameters $(P_O, V, L, \lambda)$, calculate the correcting light intensity $I_{C_{\{R,M\}}}$ using Equation 15 [S1]:
   $$I_{C_{\{R,M\}}}(P_O, V, L, \lambda) = R(P_O, V, L, \lambda) + \sum_{k=1}^{K-1} A_{\{R,M\}}^{(k)}(P_O, V, L, \lambda)$$
   where:
   - $R(P_O, V, L, \lambda)$ is the surface reflection at point $P_O$ for view $V$ and light $L$ at wavelength $\lambda$ [S1].
   - $A_{\{R,M\}}^{(k)}(P_O, V, L, \lambda)$ are the multiple scattering terms up to order $K-1$ [S1].
   - The subscript $\{R,M\}$ indicates the term applies to both Rayleigh and Mie scattering components [S1].
3. Fill the correction texture with the calculated intensity values $I_C$ [S1].
4. During rendering, subtract the intensity values of the correction texture from the main scattering texture on a per-texel basis [S1].

**Pitfalls:**
- If the view vector $V$ does not intersect the planet, the correction intensity $I_C$ is zero [S1].
- Neglecting the viewing direction $V$ (using a 2D table for $I_C$) loses directional information from the phase function, though the error is small due to spherical integration averaging [S1].

## Operating characteristics
Not given in the sources.

## Background
This correction addresses considerations omitted by earlier works such as [Bruneton and Neyret 2008] and [Elek 2009], which assumed homogeneous surface albedo [S1]. It is part of the precomputation procedure for real-time spectral scattering in large-scale natural participating media [S1].

## Sources
[S1] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
