# Spectral CT Simulation

Also called: Polychromatic Projection, Polychromatic spectral simulation, Spectralize

## Overview
Spectral CT Simulation generates polychromatic X-ray projections by integrating monochromatic attenuation components weighted by an effective X-ray spectrum, applying material attenuation via the Beer-Lambert law, and adding Poisson noise to simulate detector statistics [S1, unofficial]. It differs from monochromatic simulation by explicitly modeling the energy dependence of attenuation and the spectral shape of the source, which is critical for accurate beam-hardening artifacts and material decomposition studies [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume array `vol` (dtype float32, dimensions `[vrt, mag, hrz]`), geometry description `geom`, a list of material dictionaries `mats`, an energy grid `E`, an effective spectrum `S`, and a photon count parameter `n_phot` [S1, unofficial].
*   **Output:** A projection counts array `counts` (dtype float32), which must be converted to projection data using the negative logarithm: `proj = -numpy.log(counts)` [S1, unofficial].

**Data Structures:**
*   `mats`: A list of dictionaries. Each dictionary contains keys `'material'` (string), `'density'` (float, g/cm³), and optionally other properties corresponding to labels in the phantom [S1, unofficial].
*   `E`: 1D array of energy values in keV [S1, unofficial].
*   `S`: 1D array of spectrum intensities corresponding to `E` [S1, unofficial].

**Parameters:**
*   `kv`: Tube voltage in kV. Default: 90 [S1, unofficial].
*   `n_phot`: Total photon count for noise scaling. Default: 1e8 (typical forward projection value 1e6) [S1, unofficial].
*   `filtr`: Filter configuration (dict) containing material, density, and thickness [S1, unofficial].
*   `detector`: Detector configuration (dict) containing material, density, and thickness [S1, unofficial].

**Method Steps:**
1.  **Generate Effective Spectrum:** Compute the effective spectrum $S(E)$ based on the tube voltage `kv`, filtration `filtr`, and detector response [S1, unofficial]. The spectrum represents the intensity distribution of X-ray photons across energies [S1, unofficial].
2.  **Compute Linear Attenuation:** For each material in `mats`, compute the linear attenuation coefficient $\mu(E)$ for each energy bin in $E$ [S1, unofficial]. This is done by retrieving the mass attenuation coefficient and multiplying by the material density $\rho$ [S1, unofficial]:
    $$ \mu(E) = \rho \cdot \frac{\mu_{\text{mass}}(E)}{10} $$
    where $\mu_{\text{mass}}$ is in cm²/g and the factor 10 converts to mm⁻¹ [S1, unofficial].
3.  **Ray Integration (Monochromatic):** For each ray in the projection geometry, integrate the attenuation along the path through the volume [S1, unofficial]. For a specific energy $E$, the transmitted intensity $I_{\text{mono}}(E)$ is calculated using the Beer-Lambert law:
    $$ I_{\text{mono}}(E) = S(E) \cdot \exp\left(-\sum_i \mu_i(E) \cdot t_i\right) $$
    where $t_i$ is the path length through material $i$ [S1, unofficial].
4.  **Poisson Noise Application:** Apply Poisson statistics to simulate detector noise [S1, unofficial]. The noisy count $I_{\text{noisy}}$ is derived from the expected signal scaled by photon count $N_{\text{ph}}$:
    $$ I_{\text{noisy}} = \frac{\text{Poisson}(I_{\text{mono}} \cdot N_{\text{ph}})}{N_{\text{ph}}} $$
    where $\text{Poisson}(\lambda)$ draws a random integer from a Poisson distribution with mean $\lambda$ [S1, unofficial].
5.  **Detector Response Accumulation:** Sum the contributions across all energy bins [S1, unofficial]. Assuming detector response is proportional to energy, the total detected signal is:
    $$ I_{\text{total}} = \sum_E E \cdot I_{\text{noisy}}(E) $$
    The source notes describe $I_{\text{total}} = \sum_E E \cdot I_{\text{noisy}}(E)$, implying noise is added per energy bin before summation [S1, unofficial].
6.  **Logarithmic Conversion:** Convert the noisy counts to projection data (optical depth) for reconstruction:
    $$ \text{proj} = -\ln(I_{\text{noisy}}) $$
    *Pitfall:* If $I_{\text{noisy}}$ is zero, $\ln(0)$ is undefined [G]. A small epsilon or clamping is required in implementation [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def spectral_ct_simulation(vol, geom, mats, E, S, n_phot, kv=90, filtr=None, detector=None):
    """
    Simulate spectral CT projections.
    
    Args:
        vol: numpy array [vrt, mag, hrz], float32
        geom: geometry description providing ray paths
        mats: list of dicts with 'material', 'density'
        E: numpy array of energies in keV
        S: numpy array of spectrum intensities
        n_phot: float, total photon count
        kv: float, tube voltage
        filtr: dict, filter properties
        detector: dict, detector properties
        
    Returns:
        proj: numpy array, projection data
    """
    # 1. Precompute linear attenuation coefficients for all materials
    mu_map = {}
    for mat_info in mats:
        mat_name = mat_info['material']
        rho = mat_info['density'] # g/cm^3
        # Get mass attenuation from standard X-ray tables (cm^2/g)
        mass_atten = get_mass_attenuation(E, mat_name) 
        # Convert to linear attenuation in 1/mm: rho * mass / 10
        mu_map[mat_name] = rho * mass_atten / 10.0

    # 2. Initialize output counts array
    counts = np.zeros(geom.num_rays, dtype=np.float32)
    
    # 3. Trace rays to get path lengths per material per ray
    # ray_paths: list of arrays, ray_paths[r] is array of (material_label, path_length_mm)
    ray_paths = trace_volume_rays(vol, geom)
    
    # 4. Iterate over energy bins
    for i, energy in enumerate(E):
        # Compute monochromatic intensity for this energy
        I_mono = np.zeros(geom.num_rays, dtype=np.float32)
        
        for ray_idx, ray_data in enumerate(ray_paths):
            attenuation = 0.0
            for mat_label, path_len in ray_data:
                if mat_label in mu_map:
                    attenuation += mu_map[mat_label][i] * path_len
            I_mono[ray_idx] = S[i] * np.exp(-attenuation)
        
        # Apply Poisson noise per the source formula
        expected_counts = I_mono * n_phot
        noisy_counts = np.random.poisson(expected_counts).astype(np.float32) / n_phot
        
        # Accumulate energy-weighted noisy counts
        counts += energy * noisy_counts

    # 5. Convert to projection data
    # Guard against zero counts to avoid log(0)
    counts = np.maximum(counts, 1e-10)
    proj = -np.log(counts)
    
    return proj

def get_mass_attenuation(energy, compound):
    """Lookup mass attenuation coefficient from standard X-ray tables."""
    # [G] Standard implementation queries a precomputed table (e.g., NIST XCOM)
    # or calls a library like xraylib.CS_Total_CP. Returns array in cm^2/g.
    pass

def trace_volume_rays(vol, geom):
    """Trace rays through the volume and return path lengths per material."""
    # [G] Standard ray-marching or DDA traversal. Returns list of lists.
    pass
```

## Subtopics

### Forward Spectral Projection
A forward projection method that simulates spectral CT data, accounting for material attenuation and detector response [S1, unofficial]. It is chosen when generating synthetic projection data for a known volume and material composition, rather than reconstructing a volume from data [S1, unofficial]. It differs from general spectral simulation by being a specific API call (`model.forward_spectral`) that encapsulates the geometry and material mapping logic [S1, unofficial]. Typical parameters include `n_phot` (1e6) and a list of material dictionaries `mats` [S1, unofficial]. A key pitfall is that the output `counts` must be converted to projection data using the negative logarithm (`proj = -numpy.log(counts)`) [S1, unofficial].

### DRR Simulation
Simulates Digital Reconstructed Radiographs (DRRs) from CT volumes using the Beer-Lambert model for X-ray attenuation [S2, unofficial]. The intensity at pixel $(i, j)$ is $I(r) = I_0 \cdot \exp(-\int \mu(x) dx)$, where $\mu(x)$ is the linear attenuation coefficient approximated by summing normalised HU values along the projection direction [S2, unofficial]. Ray directions for pixel $(i, j)$ are $d_{ij} = \text{normalize}((i - W/2)/f, -(j - H/2)/f, -1)$ [S2, unofficial]. Standard parameters use a DRR resolution of 128 × 128 pixels, camera distance $d = 50.0$, and focal length $f = 1280$ pixels (≈ 10° FoV) [S2, unofficial]. Orthogonal camera transforms are defined as $C_{\text{cor}} = \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & 0 & -1 & d \\ 0 & 0 & 1 & d \\ 0 & 0 & 0 & 1 \end{bmatrix}$, $C_{\text{sag}} = \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & -1 & 0 & 0 \\ 0 & -1 & 0 & 0 \\ 0 & 0 & 0 & 1 \end{bmatrix}$, and $C_{\text{ax}} = \begin{bmatrix} 1 & 0 & 0 & 0 \\ 0 & 1 & 0 & 0 \\ 0 & 0 & 1 & d \\ 0 & 0 & 0 & 1 \end{bmatrix}$ [S2, unofficial]. This yields 49,152 training rays per patient (3 × 128 × 128) [S2, unofficial]. The simulation assumes perfect geometric calibration, exact orthogonality, noiseless scatter-free projections, and attenuation-model consistency [S2, unofficial]. Scatter in real acquisitions violates the Beer-Lambert forward model and cannot be compensated by regularisation alone [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and the number of energy bins. $O(N_{\text{rays}} \cdot N_{\text{energy}})$ [G].
*   **Memory Complexity:** Linear with respect to the number of rays and energy bins for storing intermediate attenuation and count arrays [G].
*   **Performance:** Dependent on the efficiency of the ray integration and the speed of the X-ray library calls (e.g., `xraylib`) [S1, unofficial].
*   **Strengths:** Accurately models beam hardening and spectral effects, which are absent in monochromatic simulations [S1, unofficial].
*   **Limitations:** Computationally expensive due to the integration over multiple energy bins [S1, unofficial]. Requires accurate knowledge of the material composition and density in the volume [S1, unofficial].
*   **When to use:** When simulating realistic CT data for algorithm development, especially for beam-hardening correction or material decomposition studies [S1, unofficial].

## Background
The method is implemented in the `flexTOMO` library, specifically within the `cicwi/flexTOMO` repository [S1, unofficial]. It builds on the principles of X-ray physics, utilizing the Beer-Lambert law and Poisson statistics for noise modeling [S1, unofficial]. The library uses `xraylib` for retrieving mass attenuation coefficients [S1, unofficial].

## Sources
[S1] cicwi/flexTOMO, https://github.com/cicwi/flexTOMO, fetched 2026-10-27 (unofficial)
[S2] Sparse-View Lung Nodule Volumetry from Digitally Reconstructed Radiographs via AReT: Anatomy-Regularized TensoRF, https://doi.org/10.48550/arxiv.2606.02639, fetched 2026-10-26 (unofficial)
