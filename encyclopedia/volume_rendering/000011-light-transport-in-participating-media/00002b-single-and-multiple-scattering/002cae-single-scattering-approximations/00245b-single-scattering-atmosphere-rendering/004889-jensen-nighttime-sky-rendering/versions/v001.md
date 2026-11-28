# Jensen Nighttime Sky Rendering

Also called: Jensen, Jensen et al. nighttime sky rendering, Jensen's Night Sky Rendering

## Overview
Jensen Nighttime Sky Rendering is a method for rendering the night sky by simulating illumination from the Moon, stars, zodiacal light, and the atmosphere itself [S1]. It sets itself apart by combining the atmospheric model and phase functions of Nishita with a hybrid transport simulation that uses distribution ray-tracing and ray-marching to approximate multiple scattering effects, specifically targeting the low-light conditions of nighttime [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A spherical atmospheric model, scattering phase functions (from Nishita), scene geometry, and light sources (Moon, stars, zodiacal light) [S1][S2, unofficial].
*   **Outputs:** The radiance of the night sky as seen from a viewpoint [S1].

**Data Structures:**
Not given in the sources. The method relies on the atmospheric model and phase functions defined by Nishita [S1].

**Steps:**
1.  **Initialize Atmospheric Model:** Load the spherical atmospheric model and the scattering phase functions as defined by Nishita [S1][S2, unofficial].
2.  **Hybrid Transport Simulation:** Simulate light transport using a combination of distribution ray-tracing and ray-marching [S1][S2, unofficial].
3.  **Ray Marching Integration:** As a ray traverses the atmosphere, use ray-marching to integrate optical-depth sampling [S1].
4.  **Random Position Sampling:** During the ray-marching process, sample in-scattered indirect radiance and direct illumination at random positions along the ray path [S1].
5.  **Accumulate Contributions:** Combine the contributions from the Moon, stars, zodiacal light, and atmospheric scattering to render the final image [S1][S2, unofficial].

**Formulas:**
Not given in the sources. The method uses the atmospheric model and scattering phase functions of Nishita [S1].

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.

**Measured Performance:**
Not given in the sources.

**Strengths and Limitations:**
The method effectively renders complex nighttime illumination sources including the Moon, stars, and zodiacal light [S1]. It approximates multiple scattering, which is more accurate than single-scattering-only models, though higher-order scattering decreases drastically in radiance [S1].

**When to Use:**
Use this method when rendering realistic nighttime skies that require illumination from multiple celestial bodies and atmospheric scattering effects [S1][S2, unofficial].

**When Not to Use:**
Not given in the sources.

## Background
**Origin:**
The method was proposed by Jensen et al. [38] [S1].

**History and Development:**
Jensen et al. built upon the atmospheric model and scattering phase functions established by Nishita [59] [S1]. The work is cited in surveys of participating media rendering techniques [S1] and reviews of light shaft rendering [S2, unofficial].

**Where it is Used:**
The method is used for rendering the nighttime sky, including illumination from the Moon, stars, zodiacal light, and the atmosphere [S1].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] A Review On Light Shafts Rendering For Indoor Scenes, https://doi.org/10.5281/zenodo.1125510, fetched 2026-10-26 (unofficial)
