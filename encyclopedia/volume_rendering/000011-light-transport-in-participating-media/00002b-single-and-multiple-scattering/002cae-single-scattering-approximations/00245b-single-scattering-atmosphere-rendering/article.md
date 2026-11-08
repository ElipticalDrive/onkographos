# Single-Scattering Atmosphere Rendering

Also called: Nishita Physical Sky Model, Rayleigh-Mie Single Scattering, Single Scattering Atmospheric Model, Single-Scattering Sky Rendering

## Overview
Renders the sky and atmospheric scattering by numerically integrating single scattering along a ray through a spherical atmosphere shell, accumulating Rayleigh and Mie contributions weighted by view and light transmittance [S1, unofficial]. It solves the Radiative Transfer Equation (RTE) for a single distant light source (the sun), accounting for Rayleigh scattering (gas molecules), Mie scattering (aerosols), and Ozone absorption [S5, unofficial]. Unlike [Single-scattering homogeneous volume rendering](1135-single-scattering-homogeneous-volume-rendering.md) which assumes constant density, this method models the exponential density decay of a planetary atmosphere and the resulting wavelength-dependent color shifts (e.g., red sunsets) [S5, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Camera position $\mathbf{p}_{cam}$ (typically on or near the planet surface), view direction $\boldsymbol{\omega}$, sun direction $\boldsymbol{\omega}_{sun}$, planet radius $R_p$, atmosphere radius $R_a$, and physical coefficients for Rayleigh, Mie, and Ozone [S5, unofficial].
*   **Outputs:** Final pixel color (radiance), optionally including the direct sun disk and background transmittance [S5, unofficial].

**Data Structures**
*   **Atmosphere Constants:** Precomputed scale heights ($H_R, H_M, H_O$) and sea-level coefficients ($\sigma_{s,0}^{Rayleigh}, \sigma_{s,0}^{Mie}, \sigma_{a,0}^{Ozone}$) [S5, unofficial].
*   **Integration State:** Accumulators for optical depth and scattered radiance (vectors of size 3 for RGB) [S1, unofficial].

**Formulas**
1.  **Density Profiles:** The number density of particles decreases exponentially with height $h$ above the planet surface [S5, unofficial]:
    $$ \sigma_s^{Rayleigh}(h) = \sigma_{s,0}^{Rayleigh} e^{-h/H_R} $$
    $$ \sigma_s^{Mie}(h) = \sigma_{s,0}^{Mie} e^{-h/H_M} $$
    $$ \sigma_a^{Ozone}(h) = \sigma_{a,0}^{Ozone} e^{-h/H_O} $$
    Typical values: $H_R = 8000$ m, $H_M = 1200$ m, $H_O = 8000$ m [S5, unofficial]. Sea-level coefficients: $\sigma_{s,0}^{Rayleigh} = (5.8, 13.5, 33.1) \times 10^{-6} \text{ m}^{-1}$, $\sigma_{s,0}^{Mie} = 21 \times 10^{-6} \text{ m}^{-1}$, $\sigma_{a,0}^{Ozone} = (3.426, 8.298, 0.356) \times 0.06 \times 10^{-5} \text{ m}^{-1}$ [S5, unofficial].

2.  **Extinction Coefficient:**
    $$ \sigma_t(h) = \sigma_s^{Rayleigh}(h) + 1.11 \cdot \sigma_s^{Mie}(h) + \sigma_a^{Ozone}(h) $$
    The factor 1.11 accounts for Mie extinction being slightly higher than scattering [S5, unofficial].

3.  **Phase Functions:**
    *   **Rayleigh:** $P_R(\cos\theta) = \frac{3}{16\pi}(1 + \cos^2\theta)$ [S5, unofficial].
    *   **Mie (Henyey-Greenstein):** $P_M(\cos\theta) = \frac{1 - g^2}{4\pi (1 + g^2 - 2g\cos\theta)^{1.5}}$ with asymmetry factor $g=0.76$ [S5, unofficial].
    *   $\cos\theta = \boldsymbol{\omega} \cdot \boldsymbol{\omega}_{sun}$ [S5, unofficial].

4.  **Transmittance:**
    The transmittance between two points $a$ and $b$ is:
    $$ Tr(a \rightarrow b) = \exp\left(-\int_a^b \sigma_t(t) dt\right) $$
    This integral is approximated numerically by sampling $\sigma_t$ at midpoints of $N$ segments along the ray [S5, unofficial].

5.  **Source Term $j(x)$:**
    The in-scattered light source at position $x$ from the sun is:
    $$ j(x) = L_{outerspace} \cdot Tr(outerspace \rightarrow x) \cdot \Omega_{Sun} \cdot \left( \sigma_s^{Rayleigh}(x)P_R + \sigma_s^{Mie}(x)P_M \right) $$
    where $L_{outerspace}$ is the solar luminance at the top of the atmosphere, and $\Omega_{Sun} \approx 0.0000664$ sr is the sun's solid angle [S5, unofficial].

6.  **Final Radiance:**
    $$ L(\mathbf{p}_{cam}, \boldsymbol{\omega}) = \int_{atmosphere}^{camera} Tr(x \rightarrow \mathbf{p}_{cam}) j(x) dx $$
    This is solved via numerical integration along the view ray [S5, unofficial].

**Parameters**
*   **Integration Steps ($N$):** Fixed at 64 in one implementation [S1, unofficial], and 8 in another [S5, unofficial].
*   **Sun Angular Radius:** $\approx 0.0046$ rad (0.53 degrees diameter) [S5, unofficial].
*   **Ground Illuminance:** $\approx 120,000$ lux (used to derive $L_{outerspace}$) [S5, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants from [S5, unofficial]
PI = 3.14159265359
R_EARTH = 6360e3
R_ATMO = 6420e3
H_R = 8000.0
H_M = 1200.0
H_O = 8000.0

SIGMA_S_RAY_SEA = [5.8e-6, 13.5e-6, 33.1e-6]
SIGMA_S_MIE_SEA = [21e-6, 21e-6, 21e-6]
SIGMA_A_OZONE_SEA = [c * 0.06 * 1e-5 for c in [3.426, 8.298, 0.356]]

STEPS = 8
G_MIE = 0.76
SUN_ANGULAR_RADIUS = math.radians(0.53 / 2.0)
SUN_SOLID_ANGLE = 2 * PI * (1 - math.cos(SUN_ANGULAR_RADIUS))
ILLUMINANCE_GROUND = 120000.0

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def length(v):
    return math.sqrt(v[0]**2 + v[1]**2 + v[2]**2)

def smoothstep(edge0, edge1, x):
    t = max(0.0, min(1.0, (x - edge0) / (edge1 - edge0)))
    return t * t * (3.0 - 2.0 * t)

def get_height(pos):
    return length(pos) - R_EARTH

def sigma_s_rayleigh(pos):
    h = get_height(pos)
    return [c * math.exp(-h / H_R) for c in SIGMA_S_RAY_SEA]

def sigma_s_mie(pos):
    h = get_height(pos)
    return [c * math.exp(-h / H_M) for c in SIGMA_S_MIE_SEA]

def sigma_a_ozone(pos):
    h = get_height(pos)
    return [c * math.exp(-h / H_O) for c in SIGMA_A_OZONE_SEA]

def sigma_t(pos):
    sr = sigma_s_rayleigh(pos)
    sm = sigma_s_mie(pos)
    so = sigma_a_ozone(pos)
    return [sr[i] + 1.11 * sm[i] + so[i] for i in range(3)]

def integrate_sigma_t(from_pos, to_pos, steps):
    ds = [(to_pos[i] - from_pos[i]) / steps for i in range(3)]
    acc = [0.0, 0.0, 0.0]
    dist = length(ds)
    for i in range(steps):
        s = [from_pos[j] + (i + 0.5) * ds[j] for j in range(3)]
        st = sigma_t(s)
        for j in range(3):
            acc[j] += st[j]
    return [a * dist for a in acc]

def transmittance(from_pos, to_pos, steps):
    integral = integrate_sigma_t(from_pos, to_pos, steps)
    return [math.exp(-i) for i in integral]

def rayleigh_phase(cos_theta):
    return (3.0 / (16.0 * PI)) * (1.0 + cos_theta**2)

def mie_phase(cos_theta):
    denom = 1.0 + G_MIE**2 - 2.0 * G_MIE * cos_theta
    return (1.0 - G_MIE**2) / (4.0 * PI * (denom ** 1.5))

def intersect_ray_sphere_inside(ray_origin, ray_dir, radius):
    b = dot(ray_origin, ray_dir)
    c = dot(ray_origin, ray_origin) - radius**2
    disc = b*b - c
    if disc < 0.0:
        return -1.0
    return -b + math.sqrt(disc)

def compute_louterspace():
    h0 = 1.0
    hZ = R_ATMO - R_EARTH
    tr_z = [0.0, 0.0, 0.0]
    for i in range(3):
        tau_r = SIGMA_S_RAY_SEA[i] * H_R * (math.exp(-h0/H_R) - math.exp(-hZ/H_R))
        tau_m = SIGMA_S_MIE_SEA[i] * H_M * (math.exp(-h0/H_M) - math.exp(-hZ/H_M))
        tau_o = SIGMA_A_OZONE_SEA[i] * H_O * (math.exp(-h0/H_O) - math.exp(-hZ/H_O))
        tr_z[i] = math.exp(-(tau_r + tau_m + tau_o))
    l_out = [0.0, 0.0, 0.0]
    for i in range(3):
        l_out[i] = ILLUMINANCE_GROUND / (SUN_SOLID_ANGLE * tr_z[i])
    return l_out

def j_source(pos, view_dir, sun_dir, l_outerspace, steps):
    dist_sun = intersect_ray_sphere_inside(pos, sun_dir, R_ATMO)
    pos_out = [pos[i] + sun_dir[i] * dist_sun for i in range(3)]
    tr_to_sun = transmittance(pos, pos_out, steps)
    cos_theta = dot(view_dir, sun_dir)
    sr = sigma_s_rayleigh(pos)
    sm = sigma_s_mie(pos)
    pr = rayleigh_phase(cos_theta)
    pm = mie_phase(cos_theta)
    js = [0.0, 0.0, 0.0]
    for i in range(3):
        js[i] = l_outerspace[i] * tr_to_sun[i] * SUN_SOLID_ANGLE * (sr[i] * pr + sm[i] * pm)
    return js

def compute_luminance(origin_view, view_dir, sun_dir, l_outerspace, steps):
    dist_view = intersect_ray_sphere_inside(origin_view, view_dir, R_ATMO)
    view_out = [origin_view[i] + view_dir[i] * dist_view for i in range(3)]
    ds = [(view_out[i] - origin_view[i]) / steps for i in range(3)]
    dist_total = length(ds)
    acc = [0.0, 0.0, 0.0]
    for i in range(steps):
        s = [origin_view[j] + (i + 0.5) * ds[j] for j in range(3)]
        tr_cam = transmittance(s, origin_view, steps)
        js = j_source(s, view_dir, sun_dir, l_outerspace, steps)
        for j in range(3):
            acc[j] += tr_cam[j] * js[j]
    return [a * dist_total for a in acc]

def direct_sun(view_dir, origin_view, sun_dir, l_outerspace, steps):
    cos_theta = dot(view_dir, sun_dir)
    angle = math.acos(cos_theta)
    disk = 1.0 - smoothstep(SUN_ANGULAR_RADIUS * 0.95, SUN_ANGULAR_RADIUS * 1.05, angle)
    dist_view = intersect_ray_sphere_inside(origin_view, view_dir, R_ATMO)
    view_out = [origin_view[i] + view_dir[i] * dist_view for i in range(3)]
    tr_cam = transmittance(origin_view, view_out, steps)
    sun_color = [0.0, 0.0, 0.0]
    for i in range(3):
        sun_color[i] = disk * l_outerspace[i] * tr_cam[i]
    return sun_color

def main(origin_view, view_dir, sun_dir, steps=STEPS):
    l_outerspace = compute_louterspace()
    luminance = compute_luminance(origin_view, view_dir, sun_dir, l_outerspace, steps)
    sun = direct_sun(view_dir, origin_view, sun_dir, l_outerspace, steps)
    return [luminance[i] + sun[i] for i in range(3)]
```

**Edge Cases and Pitfalls**
*   **Ray Inside Planet:** If the camera is inside the planet, the ray intersection with the atmosphere sphere must be handled carefully. The start point should be advanced to the exit intersection, and the ray length reduced [S1, unofficial].
*   **Large Luminance Values:** Physical values ($L_{outerspace}$) can exceed $10^9$, causing overflow in non-physical renderers. A common fix is to normalize ground illuminance to $2\pi$ instead of $120,000$ lux [S5, unofficial].
*   **Ray Missing Atmosphere:** If the ray does not intersect the atmosphere sphere, return black or background color [S1, unofficial].

## Subtopics

### [Sky View LUT with Angular Compression](6648-sky-view-lut-with-angular-compression.md)
A technique for rendering sky and atmosphere using a precomputed Look-Up Table (LUT) with angular compression, ported from a reference implementation.

### [Spectral Scattering Model for Atmosphere](9399-spectral-scattering-model-for-atmosphere.md)
A physically-based model for simulating light transport in planetary atmospheres and large water bodies, accounting for direct illumination, indirect scattered light, interreflections between surface and volume, and seabed reflections. It uses fully spectral data precomputed into lookup tables for real-time GPU rendering.

## Operating characteristics
*   **Time Complexity:** $O(STEPS)$ per pixel for the luminance integral and $O(STEPS)$ for transmittance calculations, where STEPS is the number of integration segments (typically 8–64) [S5, unofficial].
*   **Memory Complexity:** $O(1)$ per pixel for the direct integration method [G].
*   **Strengths:** Physically accurate colors (Rayleigh blue, Mie white/grey, ozone absorption); efficient enough for real-time if step count is low or LUTs are used [S5, unofficial].
*   **Limitations:** Ignores multiple scattering, which causes inaccuracies in deep atmospheres or high optical depths (e.g., near the horizon at sunset) [S5, unofficial]. Single-step approximations (like [S2, unofficial]) remove the integration loop for a condensed calculation, trading accuracy for speed [S2, unofficial].

## Background
Originating from the physical sky model by Nishita et al. (1993), this method has been adapted for real-time graphics [S6, unofficial]. It is widely used in real-time sky rendering, specifically referenced in Sébastien Hillaire's 2016 course and 2020 paper, and Scratch a Pixel tutorial [S6, unofficial]. The model relies on the conservation of energy and the Radiative Transfer Equation (RTE) [S1, unofficial].

## Sources
[S1] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
[S2] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S4] Depersonalizc/volumetric-clouds, https://github.com/Depersonalizc/volumetric-clouds, fetched 2026-10-27 (unofficial)
[S5] al-ro/volumetrics, https://github.com/al-ro/volumetrics, fetched 2026-10-27 (unofficial)
[S6] Sky and Atmosphere rendering: A physical approach, https://cpp-rendering.io/sky-and-atmosphere-rendering/, fetched 2026-10-28 (unofficial)
[S7] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
