# Differential Infrared Thermography (DIT)

Also called: DIT thermography

## Overview
Differential Infrared Thermography (DIT) is a technique used to estimate the transmittance ($e^{-\delta_{max}}$) of a transmissive target from the Radiative Transfer Equation (RTE) to avoid the direct inversion of the ill-posed integral source term [S1, unofficial]. It simplifies the 3-D spray structure as a thin slab layer to measure the local attenuation coefficient by comparing measurements with two different radiating backgrounds, assuming the target's temperature/radiance remains constant between measurements [S1, unofficial]. It sets itself apart from full volumetric tomography by providing an effective apparent temperature and total optical depth (TOD) without resolving the internal line-of-sight distribution, serving as a preprocessing step to recover the local extinction coefficient for subsequent RTE solving [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   Two raw IR images of the spray field: $I_{meas,1}$ and $I_{meas,2}$ [S1, unofficial].
    *   Two corresponding background radiance images (chamber radiation without spray): $I_{bg,1}$ and $I_{bg,2}$ [S1, unofficial].
    *   Assumption: The spray field remains statistically consistent (local attenuation coefficient and source term are identical) between the two measurements [S1, unofficial].
*   **Outputs:**
    *   Integrated slab emissivity $\varepsilon_{DIT}(x)$ [S1, unofficial].
    *   Total Optical Depth (TOD) $\delta_{max}(x)$ [S1, unofficial].
    *   Local extinction coefficient profile $\alpha(r)$ (via Abel inversion) [S1, unofficial].
    *   Effective DIT radiance $I_{DIT}$ and associated temperature [S1, unofficial].

**Data Structures**
*   **Images:** 2D arrays of radiance values.
*   **Discretized Spray Cross-Section:** Represented as concentric rings indexed by $j$.
*   **Geometric Kernel Matrix $K$:** A matrix where element $K_{ij}$ corresponds to the path length of the $i$-th ray passing through the $j$-th concentric ring [S1, unofficial].
*   **Vectors:**
    *   $\Delta$: Vector of known Total Optical Depths (TOD) derived from $\delta_{max}(x)$ [S1, unofficial].
    *   $A$: Unknown vector of radial extinction coefficients $\alpha(r)$ [S1, unofficial].

**Formulas**
The Radiative Transfer Equation (RTE) relating measured radiance to local properties is [S1, unofficial]:

$$
I_{meas} = I_{bg} e^{-\delta_{max}} + \int_{0}^{S_{max}} j(s) e^{-\delta(s)} ds
$$

Where:
*   $I_{meas}$ is observed radiance.
*   $I_{bg}$ is background radiance.
*   $j(s)$ is the local emission source term.
*   $s$ is the coordinate along the ray path of total length $S_{max}$ (background at $s=0$, detector at $s=S_{max}$).
*   $\delta(s)$ is the local optical depth (LOD), i.e., cumulative attenuation from position $s$ to the detector.
*   $\delta_{max}$ is the total optical depth (TOD) of the medium along the entire line of sight.

Under the thin slab approximation, the integral term reduces, and the measured images are expressed as [S1, unofficial]:

$$
I_{meas,1}(x) = I_{bg,1} (1 - \varepsilon_{DIT}(x)) + \varepsilon_{DIT}(x) I_{DIT}(x) \quad (13a)
$$
$$
I_{meas,2}(x) = I_{bg,2} (1 - \varepsilon_{DIT}(x)) + \varepsilon_{DIT}(x) I_{DIT}(x) \quad (13b)
$$

The integrated spray emissivity $\varepsilon_{DIT}$ is calculated from the four measured images [S1, unofficial]:

$$
\varepsilon_{DIT}(x) = \frac{I_{meas,2} - I_{meas,1}}{I_{bg,2} - I_{bg,1}} \quad (14)
$$

The maximum optical depth $\delta_{max}(x)$ is determined from the emissivity using the Beer-Lambert law approximation [S1, unofficial]:

$$
\delta_{max}(x) = -\ln(1 - \varepsilon_{DIT}(x)) \quad (12b)
$$

Under the assumption of axisymmetry, the total optical depth $\delta_{max}(x)$ is the Abel transform of the local extinction-coefficient profile $\alpha(r)$ [S1, unofficial]:

$$
\delta_{max}(x) = 2 \int_{x}^{r_{max}} \frac{\alpha(r) r}{\sqrt{r^2 - x^2}} dr \quad (16)
$$

The continuous integral is approximated by a finite sum in linear algebraic form [S1, unofficial]:

$$
\Delta = K A \quad (17)
$$

Where $\Delta$ is the vector of TOD, $A$ is the unknown vector of radial extinction coefficient $\alpha(r)$, and $K$ is the geometric kernel matrix.

**Steps**
1.  Acquire two raw IR images of the spray, $I_{meas,1}$ and $I_{meas,2}$, corresponding to two different background conditions $I_{bg,1}$ and $I_{bg,2}$, while ensuring the spray field remains statistically consistent [S1, unofficial].
2.  Acquire background images $I_{bg,1}$ and $I_{bg,2}$ (chamber radiation without spray) [S1, unofficial].
3.  Calculate the integrated slab emissivity $\varepsilon_{DIT}(x)$ for each pixel/position $x$ using Equation (14) [S1, unofficial].
4.  Calculate the Total Optical Depth (TOD) $\delta_{max}(x)$ using Equation (12b) [S1, unofficial].
5.  Discretize the spray cross-section into concentric rings. Construct the geometric kernel matrix $K$ where $K_{ij}$ is the path length of the $i$-th ray through the $j$-th ring [S1, unofficial].
6.  Solve the linear system $\Delta = K A$ for the unknown vector $A$ (radial extinction coefficient $\alpha(r)$) [S1, unofficial].
7.  (Optional/Contextual) Calculate the DIT radiance $I_{DIT}$ from Eq. (13a) or (13b) and convert to temperature [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from scipy.special import erf # Placeholder for Abel inversion solver if not using linear algebra directly

def compute_dit_emissivity(I_meas_1, I_meas_2, I_bg_1, I_bg_2):
    """
    Computes the integrated slab emissivity epsilon_DIT from four images.
    """
    # Equation 14
    epsilon_DIT = (I_meas_2 - I_meas_1) / (I_bg_2 - I_bg_1)
    return epsilon_DIT

def compute_total_optical_depth(epsilon_DIT):
    """
    Computes the maximum optical depth delta_max from emissivity.
    """
    # Equation 12b
    # Guard against log(0) or log(negative) due to noise
    epsilon_clamped = np.clip(epsilon_DIT, 0, 1 - 1e-6)
    delta_max = -np.log(1 - epsilon_clamped)
    return delta_max

def abel_inversion_linear(delta_max_vector, geometry):
    """
    Solves for radial extinction coefficient alpha(r) using the linear algebraic form of the Abel transform.
    
    Parameters:
    delta_max_vector: 1D array of TOD values for each ray position x.
    geometry: Object or matrix K containing path lengths.
    
    Returns:
    alpha_vector: 1D array of radial extinction coefficients alpha(r).
    """
    # Construct geometric kernel matrix K
    # K_ij corresponds to the path length of the i-th ray passing through the j-th concentric ring.
    # This requires knowledge of the discretization of the spray cross-section into concentric rings.
    K = geometry.construct_kernel_matrix()
    
    # Equation 17: Delta = K * A
    # Solve for A (alpha vector)
    # Note: This system may be ill-conditioned; regularization might be required in practice.
    # Using least squares or pseudo-inverse if K is not square or invertible.
    alpha_vector = np.linalg.lstsq(K, delta_max_vector, rcond=None)[0]
    
    return alpha_vector

def run_dit(I_meas_1, I_meas_2, I_bg_1, I_bg_2, geometry):
    """
    Full DIT pipeline.
    """
    # Step 1 & 2: Inputs are already acquired
    
    # Step 3: Calculate emissivity
    epsilon_DIT = compute_dit_emissivity(I_meas_1, I_meas_2, I_bg_1, I_bg_2)
    
    # Step 4: Calculate TOD
    delta_max = compute_total_optical_depth(epsilon_DIT)
    
    # Step 5 & 6: Invert Abel transform to get extinction coefficient
    alpha_r = abel_inversion_linear(delta_max, geometry)
    
    return epsilon_DIT, delta_max, alpha_r
```

**Parameters**
*   **Background Variation:** The two background conditions must be sufficiently different to ensure numerical stability in the division $(I_{bg,2} - I_{bg,1})$ [S1, unofficial].
*   **Statistical Consistency:** The spray structure must not change significantly between the two measurements [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Effective Temperature:** DIT temperature does not represent local spray temperature because the 3-D structure is simplified as a thin slab. It represents an effective apparent temperature associated with the region where the Local Optical Depth (LOD) is of the order of unity, based on the Eddington–Barbier approximation [S1, unofficial].
*   **Optically Thick Cold Vapor:** For optically thick cold vapor sprays in a warm environment, the DIT temperature is an effective temperature associated mainly with the outer spray region where the line-of-sight depth (LOD) is of the order of unity [S1, unofficial].
*   **Uncertainty from Drift:** If the spray structure or chamber radiation changes significantly between the two DIT measurements, the estimated emissivity and subsequent reconstruction contain additional uncertainty [S1, unofficial].
*   **Scattering:** The method neglects scattering effects produced by dispersed sprays [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The core DIT calculation (emissivity and TOD) is $O(N)$ where $N$ is the number of pixels. The Abel inversion via linear algebra is $O(M^3)$ or $O(M^2)$ depending on the solver, where $M$ is the number of radial rings.
*   **Memory Complexity:** $O(N)$ for images, $O(M^2)$ for the kernel matrix $K$.
*   **Strengths:** Avoids direct inversion of the ill-posed integral source term of the RTE. Provides a robust estimate of the extinction coefficient profile which is a prerequisite for solving the full RTE [S1, unofficial].
*   **Limitations:** Cannot recover internal radiance or temperature distribution of optically thick sprays on its own. The "temperature" is an effective value, not a local physical temperature. Sensitive to temporal variations in the spray or background between the two acquisitions [S1, unofficial].
*   **When to use:** When estimating the extinction coefficient profile for axisymmetric sprays to enable subsequent RTE solving, and when the spray is statistically stable over the acquisition time of the two frames [S1, unofficial].
*   **When not to use:** When local temperature distribution is required directly, or when the spray is highly dynamic/non-axisymmetric [S1, unofficial].

## Background
Differential Infrared Thermography (DIT) was originally used in spray studies to avoid direct inversion of the ill-posed integral term of the RTE [S1, unofficial]. Key references include Lamanna et al. (2013) and Golzke et al. (2016) [S1, unofficial]. The technique assumes that attenuation is caused exclusively by particles (originally soot, but applied to sprays with micrometer-scale droplets) and relies on the Eddington–Barbier approximation for interpreting the resulting temperature [S1, unofficial]. It is often used as a component of a larger tomographic reconstruction pipeline where the extinction coefficient derived from DIT is used to solve for the local source term [S1, unofficial].

## Sources
[S1] Tomographic Reconstruction of Infrared Radiance from Axisymmetric Sprays in a Hot Environment Using Multi-Parameter Regularization, https://doi.org/10.21203/rs.3.rs-10907544/v1, fetched 2026-10-26 (unofficial)
