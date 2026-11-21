# Extended Phase Range SLM Modulation

Also called: Depth Replica Removal via Extended Phase Range, Extended Phase Range, Extended Phase Range SLM for Depth Replica Reduction, Replica reduction, Wavelength-Dependent Phase SLM

## Overview
Extended Phase Range SLM Modulation is a technique used in simultaneous color holography to mitigate depth replicas (ghost images) by exploiting the wavelength-dependent phase response of Liquid Crystal Spatial Light Modulators (SLSMs) [S1]. Unlike standard 0-2$\pi$ SLMs which produce perfect intensity copies of the target image at specific replica planes, this method uses SLMs with extended phase ranges (greater than 2$\pi$) that vary significantly across RGB wavelengths to break the wavelength-depth ambiguity [S1]. This approach sets it apart from [Computational Phase Modulation (CGH Generation)](7284-computational-phase-modulation-cgh-generation.md) by specifically addressing the multi-wavelength crosstalk inherent in single-SLM color holography, and differs from [Phase-Only Hologram Approximations](13999-phase-only-hologram-approximations.md) by relying on physical device dispersion rather than algorithmic approximation to suppress artifacts.

## How it works
**Inputs and Outputs:**
*   **Input:** Target RGB image data; SLM physical parameters (thickness $d$, refractive index range $n_{max}, n_{min}$, dispersion $-dn/d\lambda$).
*   **Output:** Phase-only hologram pattern $\phi(x,y)$ optimized for the specific SLM's wavelength-dependent phase response.

**Data Structures:**
*   **SLM Model:** A mapping function $R(\lambda)$ returning the maximum phase shift $\phi_{max}(\lambda)$ for a given wavelength $\lambda$.
    *   Fields: `wavelength` (float, nm), `phase_range` (float, multiples of $\pi$).
    *   Typical values for Holoeye Pluto-2.1-Vis-016 [S1]:
        *   Red ($\lambda \approx 630$ nm): $2.4\pi$
        *   Green ($\lambda \approx 532$ nm): $5.9\pi$
        *   Blue ($\lambda \approx 450$ nm): $7.4\pi$

**Method Steps:**
1.  **Characterize SLM Dispersion:** Determine the phase response $\phi_\lambda$ for each primary color channel. The phase shift introduced by the LC layer is defined as:
    $$ \phi_\lambda = \frac{2\pi d \Delta n(\lambda)}{\lambda} $$
    where $d$ is the LC layer thickness and $\Delta n(\lambda) = n(\lambda) - n_{min}$ is the change in refractive index relative to the minimum state [S1]. Note that $n$ depends on $\lambda$ due to material dispersion [S1].

2.  **Analyze Wavelength Dependence:** Calculate the derivative of phase with respect to wavelength to quantify replica suppression potential:
    $$ \frac{d\phi}{d\lambda} $$
    A larger magnitude of this derivative indicates stronger wavelength dependence, which diminishes the coherence of replica planes across different colors [S1]. This derivative increases with larger $\Delta n = n_{max} - n_{min}$ and higher dispersion $-dn/d\lambda$ [S1].

3.  **Generate Hologram:** Compute the phase pattern $\phi(x,y)$ using a standard CGH algorithm (e.g., Gerchberg-Saxton or direct calculation) but constrain the phase values to the specific extended range $[0, \phi_{max}(\lambda)]$ for each channel during the optimization or quantization step. Unlike standard methods that wrap phase to $2\pi$, this method allows the phase to exceed $2\pi$ according to the device's physical limits [S1].

4.  **Quantization:** Map the continuous phase values to the SLM's bit depth. Simulation indicates that PSNR and SSIM remain stable for 6 bits and above, but lower bit depths exacerbate quantization errors as the phase range increases [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_extended_phase_hologram(target_rgb, slm_params, wavelength):
    """
    Generates a hologram pattern accounting for extended phase range.
    
    Args:
        target_rgb: 3D array (H, W, 3) representing the target image intensity for R, G, B.
        slm_params: Object containing 'd' (thickness), 'n_min', 'n_max_lambda' (dict mapping lambda to n_max).
        wavelength: Array of wavelengths [lambda_r, lambda_g, lambda_b] in nm.
    
    Returns:
        phase_pattern: 2D array (H, W) of phase values in radians.
    """
    # 1. Calculate max phase shift for each channel based on SLM physics
    # phi_max = 2 * pi * d * (n_max(lambda) - n_min) / lambda
    phase_ranges = []
    for lam in wavelength:
        n_max = slm_params.n_max_lambda[lam]
        delta_n = n_max - slm_params.n_min
        phi_max = (2 * math.pi * slm_params.d * delta_n) / lam
        phase_ranges.append(phi_max)
    
    # 2. Initialize phase pattern (e.g., using Gerchberg-Saxton or direct calculation)
    # This step depends on the parent method [Computational Phase Modulation]
    # We assume a standard iterative Fourier transform algorithm is used here.
    phase_pattern = initialize_phase_pattern(target_rgb) 
    
    # 3. Apply extended phase constraints during iteration
    # In standard 2pi SLMs, phase is wrapped: phase = phase % (2*pi)
    # Here, we do not wrap to 2pi, but allow it to scale with the channel's specific range.
    # However, since a single SLM pattern is displayed for all colors simultaneously,
    # the physical phase shift is determined by the LC orientation 's'.
    # The effective phase for color c is phi_c = phi_pattern * (phi_max_c / 2pi)
    # The optimization must ensure the resulting intensities match target_rgb.
    
    for iteration in range(max_iterations):
        # Forward propagation to sensor plane for each color
        for c, lam in enumerate(wavelength):
            # Propagate phase_pattern modulated by the specific wavelength's efficiency
            # The physical phase shift is phi_phys = phase_pattern * (phase_ranges[c] / (2*pi))
            # Note: phase_pattern is normalized to [0, 2pi] for the SLM input, 
            # but the optical effect scales by phase_ranges[c].
            field_c = propagate(phase_pattern * (phase_ranges[c] / (2 * math.pi)), lam)
            
            # Replace amplitude with target amplitude
            target_amp = np.sqrt(target_rgb[:, :, c])
            field_c = field_c / (np.abs(field_c) + 1e-8) * target_amp
            
            # Backward propagation to SLM plane
            phase_pattern_c = propagate_back(field_c, lam)
            
            # Update phase pattern (e.g., average or weighted update)
            # [G] Standard GS update rule: phase_pattern = angle(phase_pattern_c)
            # This is a simplification; actual implementation may use weighted averaging 
            # or specific optimization loss functions [S1].
            phase_pattern = np.angle(phase_pattern_c)
            
        # 4. Quantize to SLM bit depth
        # [G] Assuming 8-bit SLM for implementation, though 6-bit is sufficient per sources.
        phase_pattern = np.round(phase_pattern / (2 * math.pi) * 255) / 255 * (2 * math.pi)
        
    return phase_pattern
```

**Edge Cases and Pitfalls:**
*   **Standard SLM Behavior:** If the SLM has a uniform 0-2$\pi$ range across all colors, perfect copies (replicas) appear at specific depth planes, degrading image quality [S1].
*   **Quantization Errors:** As the phase range increases, the limited bit depth of the SLM becomes more noticeable, potentially increasing quantization noise [S1].
*   **Dispersion Control:** The user has minimal control over the absolute wavelength $\lambda$ or the material dispersion $-dn/d\lambda$; these are fixed by the hardware [S1].

## Subtopics

### CGH-based parallel focal point generation
Generates multiple focal points simultaneously. It does not explicitly address wavelength-dependent replica reduction but may benefit from extended phase range to reduce crosstalk between focal planes. [Link](8277-cgh-based-parallel-focal-point-generation.md)

### Phase-Only Hologram Approximations
Uses algorithms to approximate complex fields with phase-only masks. Extended phase range is a hardware-specific modulation technique that can be combined with these approximations to improve quality. [Link](13999-phase-only-hologram-approximations.md)

## Operating characteristics
*   **Time Complexity:** Determined by the underlying CGH generation algorithm (e.g., Gerchberg-Saxton iterations). The extended phase range adds negligible computational overhead, primarily affecting the phase wrapping/constraint step.
*   **Memory Complexity:** Standard for CGH generation.
*   **Measured Performance:** Simulation using Holoeye Pluto-2.1-Vis-016 (Red: $2.4\pi$, Green: $5.9\pi$, Blue: $7.4\pi$) showed substantial reduction in depth replicas compared to a uniform $2\pi$ SLM [S1]. This improves target plane hologram quality and creates defocused content at other depths, reducing high-frequency out-of-focus light at the sensor plane [S1].
*   **Strengths:** Effectively breaks wavelength-depth ambiguity; reduces ghost images in simultaneous color holography; improves accommodation cues.
*   **Limitations:** Requires specific SLM hardware with high dispersion and extended phase range; quantization errors increase with phase range if bit depth is insufficient (though stable above 6 bits) [S1].
*   **When to use:** Simultaneous color holography on single SLMs where depth replicas are a significant artifact.
*   **When not to use:** Monochromatic holography (where wavelength dependence is irrelevant) or when using SLMs with uniform 0-2$\pi$ response.

## Background
The technique was proposed to address the limitation of placing content arbitrarily in 3D volume and the issue of in-focus images appearing at out-of-focus depths, which hinders accommodation [S1]. It builds on the understanding that standard 0-2$\pi$ SLMs produce perfect copies at replica planes [S1]. The method leverages the physical properties of Liquid Crystal materials, specifically the refractive index difference $\Delta n$ and dispersion $-dn/d\lambda$, to create wavelength-dependent phase shifts [S1]. The specific implementation details and simulation results were published by Kim et al. in 2023 [S1].

## Sources
[S1] Simultaneous Color Computer Generated Holography, https://doi.org/10.48550/arxiv.2303.11287, fetched 2026-10-30
