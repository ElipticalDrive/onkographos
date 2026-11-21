# Phase Range Optimization for Simultaneous Color

Also called: Differentiated Phase Range RGB Holography, RGB Phase Range Optimization, SLM Phase Range Optimization

## Overview
This method determines the optimal phase modulation range for each color channel (Red, Green, Blue) in a Simultaneous Color Holography system to overcome depth-wavelength ambiguity inherent in the Angular Spectrum Method (ASM) kernel [S1]. It ensures that the holographic patterns for each wavelength are sufficiently distinct to reconstruct unique color channels, while minimizing the phase range to preserve effective bit depth and refresh rate [S1]. It sets apart from sibling methods by focusing specifically on the *magnitude* of the phase excursion per channel rather than the loss function or focal stack geometry [S1].

## How it works
**Inputs:**
- Target RGB image $I_{target}$.
- Wavelengths for Red ($\lambda_R$), Green ($\lambda_G$), and Blue ($\lambda_B$).
- Propagation distance $z$.
- SLM pixel pitch $p$.
- SLM resolution $N \times N$.

**Outputs:**
- Optimal phase range values $\Phi_R, \Phi_G, \Phi_B$ for each channel.
- The optimized SLM phase pattern $\Phi_{SLM}(x,y)$ (composite of all channels).

**Data Structures:**
- Phase range vector $\mathbf{\Phi} = [\Phi_R, \Phi_G, \Phi_B]$.
- SLM phase map $\Phi_{SLM}$ of size $N \times N$, values in $[0, \Phi_{max}]$ or normalized $[0, 1]$.

**Parameters:**
- Red channel phase range $\Phi_R$: Fixed at $2\pi$ [S1].
- Blue channel phase range $\Phi_B$: Variable, searched in range $[2\pi, 8\pi]$ [S1].
- Green channel phase range $\Phi_G$: Set as the arithmetic mean of $\Phi_R$ and $\Phi_B$ [S1].
- Loss function: L2 norm between target and simulated hologram [S1].

**Algorithm:**
The method uses an experimental search procedure via gradient descent to find the minimal $\Phi_B$ that yields acceptable image quality.

1.  **Initialize:** Set $\Phi_R = 2\pi$. Set initial $\Phi_B = 2\pi$.
2.  **Calculate Green:** Compute $\Phi_G = (\Phi_R + \Phi_B) / 2$.
3.  **Forward Model:** For the current $\mathbf{\Phi}$, simulate the hologram reconstruction for each channel using the Angular Spectrum Method (ASM).
    -   Compute the complex field $U_R$ for Red using phase modulation limited to $\Phi_R$.
    -   Compute the complex field $U_G$ for Green using phase modulation limited to $\Phi_G$.
    -   Compute the complex field $U_B$ for Blue using phase modulation limited to $\Phi_B$.
    -   Combine fields to form the total intensity $I_{sim}$ at the target plane.
4.  **Compute Loss:** Calculate the L2 norm loss $L = || I_{target} - I_{sim} ||_2^2$.
5.  **Optimize:** Use gradient descent to adjust the SLM phase pattern $\Phi_{SLM}$ to minimize $L$, keeping $\mathbf{\Phi}$ fixed for this inner loop.
6.  **Evaluate Quality:** If the loss $L$ is below a predefined threshold (indicating sufficient quality), record the current $\Phi_B$ as a candidate.
7.  **Increment:** Increase $\Phi_B$ by a small step (e.g., $0.1\pi$).
8.  **Loop:** Repeat steps 2-7 until $\Phi_B$ reaches $8\pi$ or a maximum iteration count.
9.  **Select Optimal:** Choose the smallest $\Phi_B$ that achieved the target quality threshold.
    -   For **unnatural images** (high color contrast/unique channels), the minimum sufficient $\Phi_B$ is typically $5\pi$ [S1].
    -   For **natural images** (similar color channels), a phase difference of $4\pi$ (i.e., $\Phi_B = 6\pi$ if $\Phi_R=2\pi$) is adequate [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_phase_range(target_image, wavelengths, distance, slm_pitch, slm_res):
    """
    Determines optimal phase ranges for R, G, B channels.
    Returns (Phi_R, Phi_G, Phi_B) and the optimized SLM pattern.
    """
    Phi_R = 2 * math.pi
    Phi_B_min = 2 * math.pi
    Phi_B_max = 8 * math.pi
    step = 0.1 * math.pi
    
    # Thresholds derived from experimental findings [S1]
    # These represent the minimum quality required to stop searching
    # In practice, this is determined by the L2 loss value corresponding 
    # to "adequate quality" in the reference study.
    threshold_unnatural = 0.05  # Placeholder for specific L2 value from S1 Fig S9
    threshold_natural = 0.05    # Placeholder for specific L2 value from S1 Fig S9
    
    best_phi_b = Phi_B_max
    best_slm_pattern = None
    
    is_unnatural = detect_image_type(target_image) # [G] Helper to classify image type
    
    for Phi_B in range(Phi_B_min, Phi_B_max + step, step):
        Phi_G = (Phi_R + Phi_B) / 2
        
        # Inner optimization: Find SLM pattern for these fixed phase ranges
        slm_pattern = optimize_slm_pattern(target_image, wavelengths, distance, slm_pitch, slm_res, Phi_R, Phi_G, Phi_B)
        
        # Simulate reconstruction to get loss
        simulated_image = simulate_reconstruction(slm_pattern, wavelengths, distance, slm_pitch, slm_res, Phi_R, Phi_G, Phi_B)
        loss = l2_norm(target_image, simulated_image)
        
        # Check if quality is sufficient
        # Note: The source states "minimum phase range... is sufficient", implying we want the smallest Phi_B
        # that meets the quality criteria.
        if loss < threshold_unnatural if is_unnatural else threshold_natural:
            best_phi_b = Phi_B
            best_slm_pattern = slm_pattern
            break # Found the minimal sufficient range
            
    Phi_G_final = (Phi_R + best_phi_b) / 2
    
    return Phi_R, Phi_G_final, best_phi_b, best_slm_pattern

def optimize_slm_pattern(target, wavelengths, z, pitch, res, Phi_R, Phi_G, Phi_B):
    """
    Gradient descent to find SLM phase map minimizing L2 loss.
    """
    # Initialize SLM phase map randomly or with zero phase
    slm_phase = np.zeros((res, res))
    
    learning_rate = 0.01
    epochs = 1000 # [G] Typical value for such optimization
    
    for epoch in range(epochs):
        # Forward pass: Simulate ASM for each channel with constrained phase
        # Channel R: Phase modulated in [0, Phi_R]
        # Channel G: Phase modulated in [0, Phi_G]
        # Channel B: Phase modulated in [0, Phi_B]
        
        # Compute complex fields U_R, U_G, U_B using ASM propagation
        U_R = angular_spectrum_propagation(slm_phase, wavelengths[0], z, pitch) * mask_phase(slm_phase, Phi_R)
        U_G = angular_spectrum_propagation(slm_phase, wavelengths[1], z, pitch) * mask_phase(slm_phase, Phi_G)
        U_B = angular_spectrum_propagation(slm_phase, wavelengths[2], z, pitch) * mask_phase(slm_phase, Phi_B)
        
        # Combine intensities (incoherent sum for color)
        I_sim = np.abs(U_R)**2 + np.abs(U_G)**2 + np.abs(U_B)**2
        
        # Loss
        loss = np.sum((target - I_sim)**2)
        
        # Backpropagation to update slm_phase
        # [G] Standard gradient descent step on the phase variable
        grad = compute_gradient(loss, slm_phase, target, wavelengths, z, pitch, Phi_R, Phi_G, Phi_B)
        slm_phase -= learning_rate * grad
        
    return slm_phase

def mask_phase(phase_map, max_phase):
    """
    Constrains phase values to the specified range.
    """
    return np.clip(phase_map, 0, max_phase)
```

## Operating characteristics
- **Time Complexity:** The optimization is iterative (gradient descent). The search over $\Phi_B$ adds a linear factor relative to the number of steps in the range $[2\pi, 8\pi]$. Each step requires a full hologram optimization [S1].
- **Memory Complexity:** Requires storing the SLM phase map and intermediate complex fields for three wavelengths. Complexity is $O(N^2)$ where $N$ is the SLM resolution [S1].
- **Performance:** Larger phase ranges degrade image quality by reducing effective bit depth and tend to have slower refresh rates on physical SLMs [S1].
- **Strengths:** Overcomes depth-wavelength ambiguity; provides a systematic way to select SLMs or configure phase ranges for simultaneous color [S1].
- **Limitations:** Requires empirical tuning or simulation to determine the "minimal" range; unnatural images require significantly larger phase ranges ($5\pi$) than natural images ($4\pi$ difference) [S1].
- **When to use:** When using an SLM with extended phase range capability for simultaneous RGB holography, or when selecting an SLM for such a system [S1].

## Background
This method originates from research into Simultaneous Color Computer Generated Holography (CGH) [S1]. It addresses the specific challenge that the Angular Spectrum Method (ASM) kernel introduces depth-wavelength ambiguity, meaning different wavelengths can produce similar diffraction patterns for different depths. By ensuring the phase ranges are sufficiently different, unique holograms are produced for each color channel [S1]. The experimental procedure was detailed in studies involving Holoeye Pluto SLMs and standard simulated SLMs, comparing performance against depth division and bit division methods [S1].

## Sources
[S1] Simultaneous Color Computer Generated Holography, https://doi.org/10.48550/arxiv.2303.11287, fetched 2026-10-30
