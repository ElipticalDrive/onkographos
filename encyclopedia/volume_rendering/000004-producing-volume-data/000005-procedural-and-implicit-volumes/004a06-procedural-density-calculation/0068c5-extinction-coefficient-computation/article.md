# Extinction Coefficient Computation

Also called: Binning surface area density from hierarchical vegetation model, Opacity calculation from atomic cross-sections

## Overview
Extinction coefficient computation calculates the volumetric opacity $\kappa$ required for radiative transport simulations by aggregating geometric surface densities or atomic cross-sections. It differs from procedural noise methods by deriving opacity from explicit physical geometry (vegetation models) or fundamental atomic physics (X-ray interactions) rather than stochastic field evaluation. This method serves as the foundational step for defining material interaction properties before ray marching or compositing.

## How it works
Two distinct physical regimes define this method: geometric binning for vegetation and atomic cross-section integration for X-ray radiography.

### Inputs and Outputs
*   **Geometric Input:** Hierarchical vegetation model with detailed polygonal descriptions; ranges for height $z$ and polar angle $\theta$.
*   **Atomic Input:** Tabulated atomic cross-sections $\sigma_E$ indexed by photon energy $E$; material composition (mass fractions $m_i$); density $\rho$.
*   **Output:** Extinction coefficient table indexed by spatial/angle bins (vegetation) or material number and energy group (atomic).

### Data Structures
*   **Vegetation Bins:** A 2D array indexed by $(z, \theta)$ bins. Stores summed clipped polygon areas.
*   **Atomic Table:** A 2D table indexed by material number and energy group number $g$. Stores constant approximation opacities $\mathbf{K}_{mat,g}$.

### Steps

#### 1. Geometric Binning (Vegetation)
1.  Define a 2D binning grid over the ranges of height $z$ and polar angle $\theta$ [S1, book].
2.  Traverse the hierarchical vegetation model using its detailed polygonal description [S1, book].
3.  For each polygon, clip it to the $(z, \theta)$ ranges it overlaps [S1, book].
4.  Sum the clipped areas of the polygon into the appropriate $(z, \theta)$ bins [S1, book].
5.  Compute the extinction coefficient $K(z, \omega)$ from the surface area density per unit volume in each bin. The specific conversion from bin area to $K$ is detailed in the source appendix [S1, book].

#### 2. Atomic Cross-Section Integration (X-Ray)
1.  **Interpolate Elemental Cross-Sections:** For a given photon energy $E$, interpolate the atomic cross-section $\sigma_E$ from tabulated data. The energy is defined as $E = h\nu$, where $h$ is Planck's constant and $\nu$ is frequency [S2, unofficial].
2.  **Compute Elemental Opacity:** Calculate the opacity $\kappa_E$ for a single element using:
    $$ \kappa_E = \frac{\sigma_E}{uA} $$
    where $\sigma_E$ is in barns/atom ($1 \text{ barn} = 10^{-24} \text{ cm}^2$), $u$ is the atomic mass unit ($1 \text{ amu} = 1.6605402 \times 10^{-24} \text{ g}$), and $A$ is the atomic mass in amu [S2, unofficial].
3.  **Group Constant Approximation:** Divide the energy spectrum into groups $g$ with bounds $E_g$ and $E_{g+1}$. Compute the constant opacity $\mathbf{k}_g$ for each group:
    $$ \mathbf{k}_g = \frac{1}{E_{g+1} - E_g} \int_{E_g}^{E_{g+1}} \mathbf{k}_{E'} dE' $$
    This approximates the varying opacity $\mathbf{k}_{E'}$ as a piecewise constant value per group [S2, unofficial].
4.  **Mixture Composition:** For a material composed of $N$ elements, compute the material opacity $\mathbf{K}_{mat,g}$ for group $g$ using atomic mixing:
    $$ \mathbf{K}_{mat,g} = \sum_{i=0}^{N-1} m_i \mathbf{K}_{i,g} $$
    where $m_i$ is the mass fraction of element $i$ and $\mathbf{K}_{i,g}$ is the opacity of element $i$ in group $g$ [S2, unofficial].
5.  **Final Extinction Coefficient:** Convert the mass opacity $\kappa$ to the volumetric extinction coefficient $K_\nu$ using the material density $\rho$:
    $$ K_\nu = \kappa \rho $$
    where $\kappa$ is in $\text{cm}^2/\text{g}$ and $\rho$ is the density [S2, unofficial].

### Parameters
*   **Planck's constant ($h$):** $6.6263 \times 10^{-27} \text{ erg/s}$ [S2, unofficial].
*   **Energy Conversion:** $1 \text{ eV} = 1.60206 \times 10^{-12} \text{ erg}$ [S2, unofficial].
*   **Atomic Mass Unit ($u$):** $1.6605402 \times 10^{-24} \text{ g}$ [S2, unofficial].
*   **Barn:** $10^{-24} \text{ cm}^2$ [S2, unofficial].

### Edge Cases and Pitfalls
*   **Absorption Edges:** At energies corresponding to electron shell binding energies, opacity exhibits sharp discontinuities (sawtooth shape). The piecewise constant approximation must resolve these edges sufficiently to avoid significant error [S2, unofficial].
*   **Precision:** Cross-sections are interpolated from tabulated data. Interpolation errors at low energies or near edges can propagate into the final radiograph [S2, unofficial].
*   **Geometry Clipping:** In the vegetation model, precise clipping of polygons to $(z, \theta)$ bins is required to avoid double-counting or gaps in the surface area density [S1, book].

## Operating characteristics
*   **Complexity:** Geometric binning depends on the number of polygons in the hierarchical model and the resolution of the $(z, \theta)$ bins. Atomic computation depends on the number of elements in the mixture and the number of energy groups.
*   **Performance:** The geometric method requires a full traversal of the model hierarchy. The atomic method is a lookup and summation operation, typically fast per material, but requires pre-computed cross-section tables.
*   **Scaling:** Geometric binning scales with scene complexity (polygon count). Atomic scaling is linear with the number of elements in a mixture.
*   **Strengths:** Physically accurate for specific geometries and materials.
*   **Limitations:** Requires detailed input data (polygons or cross-section tables). Not suitable for generic procedural noise-based volumes.

## Background
*   **Geometric Method:** Originated in the context of global illumination in vegetation, specifically for plane-parallel radiance transport. It was used to process hierarchical vegetation models (e.g., maple trees) to compute density of leaf and non-leaf surfaces [S1, book].
*   **Atomic Method:** Developed for hardware-accelerated simulated radiography of emissive objects. It utilizes publicly available tabulations of computed atomic cross-sections to define cold opacities for X-ray simulation [S2, unofficial].

## Sources
[S1] Plane parallel radiance transport for global illumination in vegetation, https://www.osti.gov/biblio/649229, fetched 2026-10-26 (book)
[S2] Hardware-Accelerated Simulated Radiography, https://digital.library.unt.edu/ark:/67531/metadc876978/?q=front-to-back compositing, fetched 2026-10-29 (unofficial)
