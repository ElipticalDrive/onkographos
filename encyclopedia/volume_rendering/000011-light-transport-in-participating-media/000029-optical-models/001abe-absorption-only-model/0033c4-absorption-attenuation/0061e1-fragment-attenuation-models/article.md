# Fragment Attenuation Models
Also called: Distance Attenuation
## Overview
Fragment Attenuation Models compute a scalar attenuation factor $A(d)$ as a function of the distance $d$ traveled by a ray through a volume segment, using empirical uniform float parameters [S1, unofficial]. They build on [Absorption Attenuation](7104-absorption-attenuation.md) by replacing physical coefficient derivation with distance-based empirical falloff [G]. They are distinct from [Transparency and Opacity](8074-transparency-and-opacity.md) in that they explicitly model distance-dependent signal decay rather than binary or alpha-blended surface states [G].
## Shared principles
All models in this family compute a scalar attenuation factor $A(d)$ based on the distance $d$ the ray has traveled within the current volume segment, using uniform float coefficients [S1, unofficial]. The input $d$ is the distance along the ray, and the output is a scalar value bounded within $[0, 1]$ [S1, unofficial].
## Subtopics

### Basic Quadratic Model
A polynomial model providing a smooth, non-linear distance falloff [S1, unofficial].
- **Formula**: $A(d) = \min\left(\frac{1}{c_1 + c_2 d + c_3 d^2}, 1.0\right)$ [S1, unofficial].
- **Parameters**: $c_1, c_2, c_3$ are uniform float coefficients [S1, unofficial].
- **When to choose**: Not given in the sources.

### Ranged Linear Model
A linear interpolation between two distance bounds, creating a visibility window [S1, unofficial].
- **Formula**: $A(d) = \text{clamp}\left(\frac{d_{\text{max}} - d}{d_{\text{max}} - d_{\text{min}}}, 0.0, 1.0\right)$ [S1, unofficial].
- **Parameters**: $d_{\text{min}}$ and $d_{\text{max}}$ are uniform float bounds [S1, unofficial].
- **When to choose**: Not given in the sources.

### Pixar Exponential/Decay Model
A model combining exponential and power-law decay terms for artistic control [S1, unofficial].
- **Formula**: $A(d) = \text{mix}\left(f_{\text{max}} e^{k_0 (d/d_C)^{-k_1}}, f_C (d_C/d)^{\text{exponentSE}}, \text{step}(d_C, d)\right)$ [S1, unofficial].
- **Parameters**: $f_{\text{max}}, f_C, d_C, k_0, k_1, \text{exponentSE}$ are uniform floats [S1, unofficial].
- **When to choose**: Not given in the sources.

### Attenuation
This section defines the physical basis that fragment models often approximate or replace [G].
#### definition
Attenuation (or extinction) is the combined effect of absorption and out scattering, representing the total reduction in radiance [S2, book].
#### formula
The attenuation coefficient $\sigma_t(p, \omega)$ is the sum of the absorption coefficient $\sigma_a(p, \omega)$ and the scattering coefficient $\sigma_s(p, \omega)$ [S2, book]:
$$\sigma_t(p, \omega) = \sigma_a(p, \omega) + \sigma_s(p, \omega)$$
Two related values are:
1. **Single-scattering albedo** $\rho(p, \omega)$: The probability of scattering versus absorption at a scattering event, always between 0 and 1 [S2, book]:
$$\rho(p, \omega) = \frac{\sigma_s(p, \omega)}{\sigma_t(p, \omega)}$$
2. **Mean free path**: The average distance a ray travels before interacting with a particle [S2, book]:
$$\frac{1}{\sigma_t(p, \omega)}$$
#### When to choose
Not given in the sources.

## Background
Not given in the sources.
## Sources
[S1] AlfonsoLRz/tpc-anomalies, https://github.com/AlfonsoLRz/tpc-anomalies, fetched 2026-10-27 (unofficial)
[S2] 11.1 Volume Scattering Processes, https://pbr-book.org/4ed/Volume_Scattering/Volume_Scattering_Processes, fetched 2026-10-29 (book)
