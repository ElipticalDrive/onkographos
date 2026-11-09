# Atmospheric Parameterization

Also called: Atmospheric LUT Parameterization, UVWR Remapping

## Overview
Atmospheric Parameterization is the coordinate remapping function used to store precomputed single-scattering data into a 3D texture, enabling efficient runtime lookup for sky and atmosphere rendering. It reduces the four-dimensional input space (altitude, view-zenith angle, sun-zenith angle, sun-view azimuth) into a 3D texture coordinate space (U, V, W) with an emulated fourth dimension (R) via tiling. It sets itself apart from generic linear parameterizations by using non-linear mappings for altitude and angles to better distribute sampling density according to the physical behavior of light scattering in spherical atmospheres, building on [Sky View LUT with Angular Compression](6648-sky-view-lut-with-angular-compression.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** Altitude $h \in [0, H_{max}]$, view-zenith angle $\vartheta \in [0, \pi]$, sun-zenith angle $\delta \in [0, \pi]$, sun-view azimuth $\phi \in [0, \pi]$.
*   **Constants:** Planet radius $r_P$, atmospheric shell radius $r_A$, maximum altitude $H_{max}$.
*   **Outputs:** Texture coordinates $(U, V, W, R) \in [0, 1]^4$.

**Data Structures:**
The scattering data is stored in a 3D texture of size $N_U \times N_V \times N_W$. The fourth dimension $R$ is handled by tiling the texture or manual interpolation during rendering.

**Formulas:**
The remapping function $f: [0, H_{max}] \times [0, \pi]^3 \rightarrow [0, 1]^4$ is defined as follows:

1.  **Intermediate Constants:**
    $$ Z_1 = \sqrt{h^2 - r_P^2} $$
    $$ Z_2 = \sqrt{r_A^2 - r_P^2} $$
    $$ Z_3 = r^2 \cos^2(\vartheta) - Z_1 $$
    *Note: $r$ in $Z_3$ refers to the observer's distance from the planet center, i.e., $r = r_P + h$. [S1]*

2.  **U Coordinate (Altitude):**
    $$ U(h) = \frac{Z_1}{Z_2} $$

3.  **V Coordinate (View-Zenith):**
    $$ V(\vartheta) = \begin{cases} 
    \frac{1}{2} + \frac{h \cos(\vartheta) + \sqrt{Z_3}}{2Z_1} & \text{if } h \cos(\vartheta) < 0 \text{ and } Z_3 > 0 \\
    \frac{1}{2} - \frac{h \cos(\vartheta) - \sqrt{Z_3 + Z_2^2}}{2Z_1 + 2Z_2} & \text{otherwise}
    \end{cases} $$

4.  **W Coordinate (Sun-Zenith):**
    $$ W(\delta) = \frac{1 - e^{-2.8 \cos(\delta) - 0.8}}{1 - e^{-3.6}} $$
    *Note: This is a modified version of the original Bruneton and Neyret [S1] function to account for all sun angles with nonzero scattering in denser/higher atmospheres, as the original was designed specifically for Earth's atmosphere [S1].*

5.  **R Coordinate (Sun-View Azimuth):**
    $$ R(\phi) = \frac{1 + \cos(\phi)}{2} $$

**Parameters:**
*   $H_{max}$: Upper atmosphere boundary altitude. Typically $65$ km. For denser atmospheres, $100$ km or more may be used [S1].
*   $r_P$: Planet radius.
*   $r_A$: Atmospheric shell radius ($r_P + H_{max}$).

**Edge Cases and Pitfalls:**
*   **Precision:** The condition $h \cos(\vartheta) < 0$ and $Z_3 > 0$ must be evaluated carefully to avoid numerical instability near the horizon or zenith.
*   **Boundary Conditions:** The function maps inputs to $[0, 1]$. Ensure texture coordinates are clamped or wrapped appropriately during sampling.
*   **Assumptions:** The method assumes parallel light rays, a spherical planet, altitude-only density variation, and spherical shell symmetry around the plane between the light source and zenith [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def remap_atmosphere(h, theta, delta, phi, r_p, r_a):
    """
    Remaps atmospheric parameters to UVWR texture coordinates.
    
    Args:
        h: Altitude above planet surface [0, H_max]
        theta: View-zenith angle [0, pi]
        delta: Sun-zenith angle [0, pi]
        phi: Sun-view azimuth [0, pi]
        r_p: Planet radius
        r_a: Atmospheric shell radius
        
    Returns:
        (u, v, w, r): Texture coordinates in [0, 1]
    """
    # Constants
    r = r_p + h  # Observer distance from planet center
    
    # Intermediate constants
    z1 = sqrt(h**2 - r_p**2)
    z2 = sqrt(r_a**2 - r_p**2)
    z3 = (r * cos(theta))**2 - z1
    
    # U coordinate
    u = z1 / z2
    
    # V coordinate
    if h * cos(theta) < 0 and z3 > 0:
        v = 0.5 + (h * cos(theta) + sqrt(z3)) / (2 * z1)
    else:
        v = 0.5 - (h * cos(theta) - sqrt(z3 + z2**2)) / (2 * z1 + 2 * z2)
        
    # W coordinate
    w = (1 - exp(-2.8 * cos(delta) - 0.8)) / (1 - exp(-3.6))
    
    # R coordinate
    r_coord = (1 + cos(phi)) / 2
    
    return u, v, w, r_coord
```

## Subtopics

### Water Parameterization
Water Parameterization adapts the LUT approach for underwater scenes. It requires a 5D table (depth $D$, observer depth $d$, and 3 angles) but excludes the sun-view azimuth $\phi$ because scattered light intensity varies negligibly with $\phi$ due to the strong locality of light transport in water (light traveling 100m retains only $0.6 \cdot 10^{-20}\%$ to $1.5\%$ of intensity) [S1]. It uses a simpler linear parameterization: $U(D) = D/D_{max}$, $V(d) = d/D$, $W(\vartheta) = (1 + \cos(\vartheta))/2$, $R(\delta) = (1 + \cos(\delta))/2$ [S1]. $D_{max}$ is the peak depth, and should not exceed $100$ m due to low transmissivity [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per pixel for texture lookup and interpolation. The remapping function involves several trigonometric and square root operations, which are relatively expensive but constant time.
*   **Memory Complexity:** $O(N_U \times N_V \times N_W)$ for the 3D texture. The 4th dimension is emulated, so no additional memory is needed for it, but runtime interpolation may be required.
*   **Strengths:** Efficient real-time rendering of atmospheric scattering. Non-linear parameterization improves visual quality by concentrating samples where scattering changes rapidly.
*   **Limitations:** Assumes spherical symmetry and parallel light rays. Not suitable for terrain-dependent density variations or local light sources. The emulated 4th dimension may introduce artifacts if not interpolated correctly.
*   **When to Use:** Real-time sky and atmosphere rendering for games and simulations where performance is critical and the assumptions hold.
*   **When Not to Use:** When high-fidelity multiple scattering is required without precomputation, or when the atmosphere has significant lateral density variations.

## Background
The method was introduced in "Real-time spectral scattering in large-scale natural participating media" (2010) by Schafhitzel et al., building on ideas from Schafhitzel et al. (2007) and Bruneton and Neyret (2008) [S1]. The W function was modified from the original Bruneton and Neyret formulation to better handle denser atmospheres and all sun angles with nonzero scattering [S1].

## Sources
[S1] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
