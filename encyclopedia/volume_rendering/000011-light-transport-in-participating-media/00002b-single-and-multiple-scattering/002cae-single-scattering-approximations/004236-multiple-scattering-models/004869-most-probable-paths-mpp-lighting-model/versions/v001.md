# Most Probable Paths (MPP) Lighting Model

Also called: General Participating Media Lighting Model, MPP model, MPP-based lighting model, Most Probable Path (MPP) Multiple Scattering Model, Multiple Scattering approximation using MPP, Premože et al. 2004 model

## Overview
The Most Probable Paths (MPP) model is an approximation for the multiple scattering term ($L_{MS}$) in participating media, designed to handle general inhomogeneous media without relying on the diffusion approximation. It simplifies the complex integral of light transport by summing contributions only over paths close to the "Most Probable Path" (MPP), assuming strong forward scattering. Unlike previous methods that assume planar surfaces, uniform media, or optical thickness, this model handles inhomogeneous densities and avoids artifacts near boundaries and low-density areas where diffusion fails [S1].

## How it works
The model splits the total radiance $L(x, \vec{\omega})$ arriving at the eye from direction $\vec{\omega}$ at point $x$ into three components: the weighted object image, single-scattered light, and multiple-scattered light [S1]:

$$L(x,\vec{\omega}) = W \cdot L_{obj}(x, \vec{\omega}) + L_{SS}(x, \vec{\omega}) + L_{MS}(x, \vec{\omega})$$

### Inputs and Outputs
**Inputs:**
*   Medium density field $\rho(x)$, from which coefficients are derived.
*   Light source direction $\vec{\omega}_0$ and intensity $L_0$.
*   Camera position and viewing direction $\vec{\omega}$.
*   Material properties: scattering coefficient $b(x)$, absorption coefficient $a(x)$, extinction coefficient $c(x) = a(x) + b(x)$.
*   Phase function $P(\theta)$, assumed strongly forward-scattering with mean square scattering angle $\langle \theta^2 \rangle$.
*   Rate adjustment parameter $\gamma$ (typically 1.0 or 0.45) [S1].

**Outputs:**
*   Final radiance $L(x, \vec{\omega})$ for each sample point.

### Data Structures
1.  **Light Volume:** A 3D data structure storing $\ell_{light}$ values for every volume cell. This is oriented such that the z-axis aligns with the light direction $\vec{\omega}_0$. It acts as a variation of a deep shadow map [S1].
2.  **Object Image:** The scene rendered without participating media effects (except shadows cast by the medium onto objects), denoted $L_{obj}$ [S1].

### Parameters
*   $\ell$: Number of scattering events along a path, defined as $\ell = \int_{path} b(x) ds$ [S1].
*   $s$: Physical length (arclength) of the path segment [S1].
*   $\langle \theta^2 \rangle$: Mean square scattering angle, assumed constant for the medium [S1].
*   $\bar{b}$: Global average of the scattering coefficient, used for area estimation in blurred radiance calculation [S1].
*   $\gamma$: Rate adjustment parameter for multiple scattering attenuation. $\gamma=1$ is the original model; $\gamma=0.45$ provides better fits to Monte Carlo simulations for denser media [S1].

### Formulas and Definitions

**1. Single-Scattered Radiance ($L_{SS}$)**
The single-scattered contribution arriving at the eye from a directional light $\vec{\omega}_0$ at position $x$ along a segment $\Delta x$ is [S1]:

$$\Delta L_{SS}(x, \vec{\omega}) = L_{unscat}(x, \vec{\omega}_0) P(x, \vec{\omega}, \vec{\omega}_0) \exp\left(-\frac{c}{b}\ell_{eye}\right) b(x) \Delta x$$

Where $L_{unscat}$ is the unscattered radiance reaching point $x$ from the light:

$$L_{unscat}(x, \vec{\omega}_0) = L_0 \exp\left(-\frac{c}{b}\ell_{light}\right)$$

Here, $\ell_{eye}$ is the scattering events from $x$ to the eye, and $\ell_{light}$ is the scattering events from $x$ to the light source (stored in the Light Volume) [S1].

**2. Blurred Incident Radiance ($L_{blurred}$)**
To approximate the multiple scattering integral, incident radiance is blurred using a Gaussian filter. The blurred radiance at a point $x_0$ with width $w$ is [S1]:

$$L_{blurred}(x_0, w) = \frac{\bar{b}}{\pi \langle \theta^2 \rangle} \int L_0(x) \exp\left(-\frac{(x-x_0)^2}{2w^2}\right) dA$$

Note: The factor $\frac{\bar{b}}{\pi \langle \theta^2 \rangle}$ accounts for the effective area of contributing paths, correcting the dimensionality from "radiance $\times$ area" to "radiance" [S1].

**3. Multiple-Scattered Radiance ($L_{MS}$)**
The multiple scattering radiance is approximated by integrating over MPPs. The elementary contribution for a path with $\ell$ scattering events is [S1]:

$$\Delta L_{MS}(x, \vec{\omega}) = L_{blurred}(x_0, w(\ell)) \cdot \ell \cdot \exp\left(-\gamma \frac{c}{b} \ell\right)$$

Where $x_0$ is the location on the incident plane corresponding to the MPP, and $w(\ell)$ is the Gaussian width [S1].

**4. Gaussian Width ($w$)**
The width of the Gaussian used to weight incident radiance depends on the number of scattering events $\ell$, the physical length $s$, and media parameters [S1]:

$$w(\ell, s)^2 = \frac{\langle \theta^2 \rangle \ell s^2}{24} \left(1 + \frac{\langle \theta^2 \rangle (a/b) \ell^2}{12}\right)$$

**5. Object Image Weight ($W$)**
The weight applied to the unfiltered object image $L_{obj}$ combines the direct object contribution and the multiple scattering contribution from the object to the eye [S1]:

$$W = \left(1 + \ell_{obj} + \frac{2 \bar{b}^2 w(\ell_{obj})^2 \ell_{obj}}{\langle \theta^2 \rangle}\right) \exp\left(-\frac{c}{b}\ell_{obj}\right)$$

Here, $\ell_{obj}$ is the number of scattering events from the object surface to the eye [S1].

### Algorithm Steps

1.  **Preprocessing (Light Volume Construction):**
    *   For every voxel in the volume, compute $\ell_{light}$, the number of scattering events from the voxel to the light source along the light direction.
    *   Store these values in the Light Volume. This can be done via ray marching from the light source or using a deep shadow map technique [S1].

2.  **Object Image Rendering:**
    *   Render the scene without participating media effects to generate $L_{obj}$. Shadows cast by the medium onto objects should be included [S1].

3.  **Volume Sweep (Ray Marching):**
    *   For each pixel, cast a ray from the eye into the scene.
    *   Initialize accumulators for $L_{SS}$ and $L_{MS}$ to zero.
    *   March along the ray in steps $\Delta x$. For each sample point $x$:
        a.  **Retrieve $\ell_{light}$:** Look up $\ell_{light}$ for point $x$ from the Light Volume.
        b.  **Compute $\ell_{eye}$:** Accumulate $\int b(x) ds$ from the eye to $x$ during the march.
        c.  **Compute $\ell_{obj}$:** This is the total scattering events from the object hit point (if any) to the eye. If the ray hits an object, $\ell_{obj}$ is the accumulated $\ell_{eye}$ at the hit point. If the ray misses all objects, $\ell_{obj} = 0$ for the object term, but $L_{MS}$ still contributes [S1].
        d.  **Single Scattering ($L_{SS}$):**
            *   Calculate $L_{unscat} = L_0 \exp\left(-\frac{c}{b}\ell_{light}\right)$.
            *   Calculate $\Delta L_{SS} = L_{unscat} P(x, \vec{\omega}, \vec{\omega}_0) \exp\left(-\frac{c}{b}\ell_{eye}\right) b(x) \Delta x$.
            *   Add $\Delta L_{SS}$ to the $L_{SS}$ accumulator.
        e.  **Multiple Scattering ($L_{MS}$):**
            *   Calculate the total scattering events for the MPP: $\ell = \ell_{eye} + \ell_{light}$.
            *   Estimate the physical length $s$ of the MPP segment. In the simplified two-segment approximation, $s$ can be approximated by the distance from the eye to the light source projection or derived from the geometry. The source notes $s$ is the physical length [S1].
            *   Calculate Gaussian width $w(\ell, s)$ using the formula in Section 4.
            *   Compute $L_{blurred}$. In a hardware implementation, this is often precomputed or approximated. The source suggests using the global average $\bar{b}$ and $\langle \theta^2 \rangle$ [S1].
            *   Calculate the MPP contribution: $\Delta L_{MS} = L_{blurred} \cdot \ell \cdot \exp\left(-\gamma \frac{c}{b} \ell\right)$.
            *   Add $\Delta L_{MS}$ to the $L_{MS}$ accumulator.
        f.  **Attenuation:** Apply extinction to the accumulated radiance as the ray continues.

4.  **Final Computation:**
    *   For the final pixel, if the ray hit an object at distance corresponding to $\ell_{obj}$:
        *   Calculate $W$ using $\ell_{obj}$ and $w(\ell_{obj})$.
        *   $L_{final} = W \cdot L_{obj} + L_{SS} + L_{MS}$.
    *   If the ray did not hit an object (or for the volume contribution itself):
        *   $L_{final} = L_{SS} + L_{MS}$. (Note: The object term $W \cdot L_{obj}$ is zero if no object is hit) [S1].

### Edge Cases and Pitfalls
*   **Backscattering Underestimation:** The original model includes a curvature attenuation term $A_c$ that penalizes short paths with large angle changes. This severely underestimates backscattering. The modified model drops this term to correct the appearance [S1].
*   **Dimensionality Error:** Integrating weighted radiance in the incident plane yields "radiance $\times$ area". An effective area factor $\frac{\bar{b}}{\pi \langle \theta^2 \rangle}$ must be included in $L_{blurred}$ to ensure the result has the correct dimension of radiance [S1].
*   **Object Image Filtering:** Using a blurred version of the object image would be more accurate but requires runtime filtering. The model uses the unfiltered $L_{obj}$ to avoid this penalty, which is visually acceptable [S1].
*   **Homogeneous vs. Inhomogeneous:** The model assumes $\langle \theta^2 \rangle$ is constant. For highly inhomogeneous media, local variations in phase function behavior are not captured [S1].

## Operating characteristics
*   **Time Complexity:** The preprocessing step (Light Volume) is $O(V)$ where $V$ is the number of volume cells. The rendering step is $O(R \cdot S)$ where $R$ is the number of rays and $S$ is the number of steps per ray. It allows a single volume sweep for both single and multiple scattering, unlike earlier methods requiring separate blurring passes [S1].
*   **Memory Complexity:** Requires storage for the medium density, the Light Volume ($O(V)$), and the object image. The Light Volume is the primary auxiliary storage [S1].
*   **Strengths:** Handles general inhomogeneous media; avoids diffusion artifacts near boundaries and low-density regions; incorporates volumetric shadows and object-medium interactions; suitable for hardware implementation with fixed-step marching [S1].
*   **Limitations:** Assumes strong forward scattering; accuracy degrades if the phase function is not strongly forward-peaked; relies on global averages ($\bar{b}$) for area estimation which may be inaccurate in highly variable media; underestimates backscattering if the curvature term is not dropped [S1].
*   **When to use:** Real-time or interactive rendering of participating media with inhomogeneous density and strong forward scattering (e.g., fog, smoke, skin).
*   **When not to use:** Media with isotropic or backward-scattering dominant phase functions; offline rendering requiring high physical accuracy for complex multiple scattering paths [S1].

## Background
The MPP model was introduced by Premože et al. in 2004 [S1]. It builds on pioneering work by Tessendorf (1987) and Premože et al. (2003), who introduced the concept of the Most Probable Path (MPP) for light transport in strongly forward-scattering inhomogeneous media. The 2004 paper, "A lighting model for general participating media," extended this by approximating curved MPPs with two straight line segments and deriving a practical formulation for general inhomogeneous media. It was designed to overcome the limitations of diffusion-based methods and previous ad-hoc models that assumed uniform media or planar surfaces [S1].

## Sources
[S1] A lighting model for general participating media, https://doi.org/10.1145/1053427.1053446, fetched 2026-11-02
