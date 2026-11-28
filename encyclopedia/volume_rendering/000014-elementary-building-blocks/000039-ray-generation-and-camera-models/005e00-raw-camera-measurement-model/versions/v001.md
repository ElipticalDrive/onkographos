# Raw Camera Measurement Model
Also called: RawNeRF Camera Model, Sensor Noise Model

## Overview
The Raw Camera Measurement Model describes the physical process by which a camera sensor converts incoming photon flux into a high-bit-depth digital signal, serving as the input domain for methods like RawNeRF [S1]. It distinguishes itself from standard rendering pipelines by explicitly modeling the noise characteristics (shot and read noise) and the linear, high-dynamic-range nature of raw sensor data, rather than assuming a tone-mapped or gamma-corrected output [S1].

## How it works
**Inputs and Outputs**
*   **Input:** The expected number of photons arriving at a pixel during the shutter open time, denoted as $x_i$ (proportional to irradiance).
*   **Output:** A noisy digital measurement $y_i$, typically stored as a 10- to 14-bit integer [S1].

**Data Structures**
The raw image is represented as a 2D array of unsigned integers. The bit depth is typically 10, 12, or 14 bits [S1]. The values are offset by a "black level" to accommodate negative noise components [S1].

**Steps**
1.  **Photon Capture:** Photons hit the sensor pixels and are converted into an electrical charge proportional to the incident light intensity [S1].
2.  **Digitization:** The analog electrical charge is recorded as a high bit-depth digital signal [S1].
3.  **Black Level Offset:** The raw values are offset by a fixed "black level" value. This offset ensures that the digital representation can handle negative deviations caused by noise without clipping to zero immediately [S1].
4.  **Noise Addition:** Noise is added to the signal. The combined effect of photon arrival statistics (shot noise) and readout circuitry noise (read noise) is modeled [S1].
5.  **Measurement:** The final value $y_i$ is obtained. To recover the underlying signal, the black level is subtracted from $y_i$ [S1].

**Formulas**
The relationship between the noisy measurement $y_i$ and the expected photon count $x_i$ is defined by the noise distribution. The error $y_i - x_i$ is zero-mean [S1]. The noise distribution is modeled as Gaussian, where the variance $\sigma^2$ is an affine function of the mean $x_i$ [S1]:

$$
y_i \sim \mathcal{N}(x_i, \sigma^2(x_i))
$$

Where the variance is given by:

$$
\sigma^2(x_i) = \alpha x_i + \beta
$$

*   $\alpha$: Represents the gain related to shot noise (Poisson process variance) [S1].
*   $\beta$: Represents the variance of the read noise [S1].

**Parameters**
*   **Bit Depth:** Typical raw sensor data uses 10 to 14 bits [S1].
*   **Black Level:** A constant offset value specific to the sensor, required to allow for negative measurements due to noise [S1].

**Edge Cases and Pitfalls**
*   **Precision:** Raw data preserves dynamic range, but the noise variance increases with signal intensity (affine variance) [S1].
*   **Negative Values:** Because of the black level offset, raw values can be small positive integers that correspond to negative physical quantities after subtraction [S1].
*   **Non-Linearity:** While raw data is linear with respect to photon count, subsequent pipeline stages (white balance, tone mapping) are non-linear and corrupt the noise distribution [S1].

## Subtopics

### Camera Irradiance Model
A model demonstrating that pixel brightness is directly related to specific intensity inside a camera, assuming a properly focused imaging system [S2, unofficial]. It establishes the geometric link between scene radiance and the sensor signal.

## Operating characteristics
*   **Complexity:** The measurement model itself is $O(1)$ per pixel for noise simulation, involving simple arithmetic operations [S1].
*   **Memory:** Requires storage for high-bit-depth images (10-14 bits per channel) [S1].
*   **Strengths:** Preserves maximum dynamic range and allows for flexible post-processing (exposure, white balance) [S1]. Accurately models the statistical properties of real-world sensor data [S1].
*   **Limitations:** Requires access to raw sensor data, which is not always available. The noise model parameters ($\alpha, \beta$) must be calibrated for the specific sensor [S1].

## Background
The model originates from the work on "NeRF in the Dark," which introduced the use of noisy raw images for high dynamic range view synthesis [S1]. It builds upon the understanding that raw camera measurements are a noisy, linear representation of photon flux, distinct from processed images [S1]. The connection between specific intensity and pixel brightness is rooted in classical imaging theory, such as that described by Horn and Sjoberg [S2, unofficial].

## Sources
[S1] NeRF in the Dark: High Dynamic Range View Synthesis from Noisy Raw Images, https://doi.org/10.1109/cvpr52688.2022.01571, fetched 2026-10-26
[S2] Volume Rendering Mathematical Models and Algorithmic Aspects, https://core.ac.uk/works/16970577, fetched 2026-10-29 (unofficial)
