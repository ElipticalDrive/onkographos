# Optical Addressing of Photochromic Mixtures

Also called: Colour-selective PM colouration, Geometrical optical intersection, Optical Addressing of Multi-Colour Photochromic Material Mixture, Optical Addressing of Photochromic Materials, Position- and colour-selective colouration, Position-selective PM colouration, Two-channel optical addressing

## Overview
This method realizes a dynamic, full-colour volumetric display by exploiting the dual-light response of photochromic molecules (PMs) within a solid mixture. It solves the problem of 3D spatial addressing and colour selection in a single homogeneous medium by using the geometric intersection of two control light channels: ultraviolet (UV) light for colouration and visible (Vis) light for decolouration. Unlike sibling methods that rely on layered plates or complex 3D nanomaterial alignment, this approach uniformly mixes PM types, allowing high-density voxel arrangements determined by the molecular scale (~1 nm) and simple orthogonal optical geometry [S1].

## How it works
### Inputs and Outputs
**Inputs:**
1.  **PM Volume:** A transparent solid block containing a mixture of photochromic molecules (e.g., diarylethenes) dissolved in a polymer matrix (PDMS).
2.  **UV Light Source:** Monochromatic light (e.g., 302 nm) for inducing colouration (open-ring to closed-ring transition).
3.  **Vis Light Source:** Spatially modulated visible light (e.g., from a projector or SLM) for inducing decolouration (closed-ring to open-ring transition) and selecting colour via spectrum management.
4.  **Control Signals:** Synchronization signals for two Spatial Light Modulators (SLMs) or projectors.

**Outputs:**
*   A 3D image rendered within the volume, where voxels are coloured based on the local balance of UV and Vis irradiation.

### Data Structures and Layout
**PM Volume Construction:**
*   **Material:** Diarylethene PMs dissolved in toluene, mixed with polydimethylsiloxane (PDMS) polymer.
*   **Dimensions:** Approximately $1 \text{ cm} \times 1 \text{ cm} \times 3 \text{ cm}$ (single-colour) or $1 \text{ cm} \times 1 \text{ cm} \times 2 \text{ cm}$ (multi-colour) [S1].
*   **Preparation:**
    *   Single-colour: 2 mg PM per 1 mL toluene.
    *   Multi-colour: 1 mg yellow PM and 1 mg blue PM per 2 mL toluene.
    *   Mixing ratio: 4.4 mL polymer to 2 mL PM solution.
    *   Solidification: Heated at 100 °C for ~3 hours [S1].
*   **Optical Properties:** The PM is transparent (open-ring) initially. UV irradiation turns it coloured (closed-ring). Vis irradiation returns it to transparent. Different PM types absorb different Vis wavelengths for decolouration [S1].

**System Architecture:**
*   **SLM1:** Controls UV irradiation pattern (depth selection).
*   **SLM2:** Controls Vis irradiation pattern (2D spatial pattern and colour selection).
*   **Geometry:** UV and Vis sources are arranged to irradiate the PM volume from different angles, typically orthogonal [S1].

### Algorithm Steps
1.  **Initialization:** Prepare the PM volume as described above. Ensure UV and Vis sources are aligned orthogonally to the volume.
2.  **Synchronization Setup:** Configure SLM1 (UV) and SLM2 (Vis) to operate synchronously. Define the arrival time $T$ and duration $\Delta T$ for both channels. Strict synchronization is required to prevent unintended coloration [S1].
3.  **Depth Scanning Loop:** For each depth layer $z$ from $1$ to $N$ (where $N$ is the depth resolution):
    a.  **UV Modulation:** SLM1 projects a pattern that irradiates only the target depth layer $z$ with UV light.
    b.  **Vis Modulation:** Simultaneously, SLM2 projects a 2D pattern corresponding to the desired image for layer $z$. The spectrum of this Vis light determines the colour of the voxels in this layer.
    c.  **Irradiation:** Maintain irradiation for a duration $\Delta t$.
    d.  **Repeat:** Increment $z$ and repeat steps 3a–3c until all layers are scanned.

### Formulas and Parameters
**Timing Constraints:**
Let $T_{UV}$ and $T_{Vis}$ be the arrival times of UV and Vis light, and $\Delta T_{UV}$ and $\Delta T_{Vis}$ be their durations.
*   If $T_{UV} < T_{Vis}$, unintended coloration occurs during the interval $[T_{UV}, T_{Vis}]$ [S1].
*   If $T_{UV} = T_{Vis}$ and $\Delta T_{UV} > \Delta T_{Vis}$, unintended coloration occurs during $[T_{Vis} + \Delta T_{Vis}, T_{UV} + \Delta T_{UV}]$ [S1].
*   **Requirement:** $T_{UV} \ge T_{Vis}$ and $\Delta T_{UV} \le \Delta T_{Vis}$ (or strict equality with precise hardware sync) to avoid artifacts [S1].

**Performance Metrics:**
*   **Total Scan Time:** $T_{scan} = N \Delta t$ [S1].
*   **Max Rendering Time per Layer:** To suppress flicker at a frame rate of $F_{frame} \ge 60 \text{ Hz}$, the time to render one layer must satisfy:
    $$ \Delta t \le \frac{1}{F_{frame} \cdot N} $$
    For $F_{frame} = 60 \text{ Hz}$, $\Delta t \le \frac{1}{60 N}$ seconds [S1].

**Typical Parameters:**
*   **UV Wavelength:** 302 nm [S1].
*   **UV Intensity:** 0.84 mW/cm² [S1].
*   **Vis Intensity:** 15.04 mW/cm² [S1].
*   **PM Purity:** >99% HPLC purity [S1].
*   **Projector Specs (Example):** 500 ANSI Lumens, 1280 × 800 resolution, 30,000:1 contrast [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optical_addressing_pm_display(N_depth, frame_rate, uv_intensity, vis_intensity):
    """
    Simulates the control logic for optical addressing of photochromic mixtures.
    
    Args:
        N_depth (int): Number of depth layers (Z resolution).
        frame_rate (float): Target frame rate in Hz (e.g., 60).
        uv_intensity (float): UV light intensity in mW/cm^2.
        vis_intensity (float): Vis light intensity in mW/cm^2.
    """
    # Calculate maximum allowed time per layer to maintain frame rate
    max_dt = 1.0 / (frame_rate * N_depth)
    
    # System initialization
    # SLM1: UV modulator (controls depth Z)
    # SLM2: Vis modulator (controls XY pattern and colour spectrum)
    # PM Volume: Solidified mixture of diarylethenes in PDMS
    
    # Timing synchronization variables
    T_UV = 0.0  # Arrival time of UV
    T_Vis = 0.0 # Arrival time of Vis
    Delta_T_UV = 0.0 # Duration of UV
    Delta_T_Vis = 0.0 # Duration of Vis
    
    # Ensure strict synchronization to prevent unintended coloration
    # Condition: T_UV >= T_Vis and Delta_T_UV <= Delta_T_Vis
    # In practice, hardware triggers ensure T_UV == T_Vis and Delta_T_UV == Delta_T_Vis
    
    for z in range(1, N_depth + 1):
        # 1. Determine 2D pattern and colour for this depth layer
        # pattern_2d: Binary or grayscale mask for XY plane
        # spectrum_vis: RGB intensity ratios for colour selection
        pattern_2d, spectrum_vis = get_layer_data(z)
        
        # 2. Configure SLM1 for UV (Depth Selection)
        # SLM1 projects UV light only at depth z
        # Note: In physical implementation, this might involve focusing or 
        # scanning, but the principle is geometric intersection.
        configure_slm1_uv(pattern_2d, depth_z=z)
        
        # 3. Configure SLM2 for Vis (Pattern and Colour)
        # SLM2 projects Vis light with specific spectrum and 2D pattern
        configure_slm2_vis(pattern_2d, spectrum=spectrum_vis)
        
        # 4. Synchronized Irradiation
        # Start both sources simultaneously
        start_time = get_current_time()
        
        # Apply UV and Vis light
        apply_light(uv_intensity, vis_intensity)
        
        # Maintain for duration Delta_t
        # Delta_t must be <= max_dt
        Delta_t = min(max_dt, 0.001) # Example constraint, actual Delta_t depends on SLM speed
        sleep(Delta_t)
        
        # Stop light
        stop_light()
        
        # 5. Timing Check (Guard)
        # Verify that UV did not arrive before Vis or last longer than Vis
        # If T_UV < T_Vis or Delta_T_UV > Delta_T_Vis, artifacts occur
        if (T_UV < T_Vis) or (Delta_T_UV > Delta_T_Vis):
            log_warning("Timing mismatch detected: Unintended coloration risk")

def get_layer_data(z):
    """
    Retrieves the 2D spatial pattern and colour spectrum for a specific depth layer.
    Returns:
        tuple: (pattern_2d, spectrum_vis)
    """
    # Implementation depends on the 3D image data structure
    # pattern_2d: 2D array of pixel values (0 for transparent, 1 for coloured)
    # spectrum_vis: Dict or array of RGB intensities
    pass

def configure_slm1_uv(pattern, depth_z):
    """
    Configures the UV Spatial Light Modulator to irradiate the specified depth.
    """
    # In a real system, this might involve adjusting focus or using a scanning mirror
    # For the purpose of this algorithm, it represents the logical selection of depth z
    pass

def configure_slm2_vis(pattern, spectrum):
    """
    Configures the Vis Spatial Light Modulator (or projector) with the 2D pattern and colour spectrum.
    """
    # Sets the pixel values and colour channels (R, G, B) on the Vis SLM
    pass

def apply_light(uv_int, vis_int):
    """
    Applies the light sources with specified intensities.
    """
    # Hardware control to turn on UV and Vis sources
    pass

def stop_light():
    """
    Turns off all light sources.
    """
    pass

def get_current_time():
    """
    Returns the current timestamp.
    """
    pass

def sleep(duration):
    """
    Pauses execution for the specified duration.
    """
    pass

def log_warning(message):
    """
    Logs a warning message.
    """
    pass
```

### Edge Cases and Pitfalls
1.  **Timing Mismatch:** If UV light arrives before Vis light ($T_{UV} < T_{Vis}$) or if UV duration exceeds Vis duration ($\Delta T_{UV} > \Delta T_{Vis}$), unintended coloration occurs in the overlap region [S1]. Strict hardware synchronization is mandatory.
2.  **Boundary Blurring:** Boundaries between coloured and decoloured areas are blurred due to the spread of Vis light, causing extra decolouration at edges. This blurring is on a scale much larger than the wavelength of light [S1].
3.  **Colour Artifacts at Boundaries:** In multi-PM mixtures, if Vis light spreads, it may decolour one PM type more easily than another (e.g., yellow PM decolours more easily than blue PM), causing the boundary to appear blue instead of the intended colour [S1].
4.  **Solution for Blurring:** Use a highly directional light source, such as a laser-based projector, to minimize light spread [S1].
5.  **Minimum Voxel Size:** Theoretically limited by the size of a molecule (~1 nm), but practically limited by the diffraction limit of light and the resolution of the SLMs/projectors [S1].

## Operating characteristics
*   **Time Complexity:** The total scan time is $N \Delta t$, where $N$ is the depth resolution and $\Delta t$ is the rendering time per layer. To achieve a flicker-free frame rate of 60 Hz, $\Delta t \le 1/(60 N)$ [S1].
*   **Memory Complexity:** Not applicable in the traditional sense, as the "memory" is the physical PM volume. The data structure for the 3D image is stored in the host system.
*   **Resolution:**
    *   **Depth (Z):** Dominated by the resolution of SLM1 (UV) and the optical focusing/scanning mechanism.
    *   **Lateral (X, Y):** Dominated by the resolution of SLM2 (Vis) and the diffraction limit of light.
    *   **Theoretical Limit:** ~1 nm (molecular size) [S1].
*   **Strengths:**
    *   Simple hardware structure (uniform mixture, no 3D alignment of voxels) [S1].
    *   High-density voxel arrangement possible [S1].
    *   Full-colour capability via spectrum management [S1].
    *   Dynamic rewriting capability [S1].
*   **Limitations:**
    *   Blurring at boundaries due to light spread [S1].
    *   Frame rate limited by SLM speed, not molecular response time (picoseconds) [S1].
    *   Requires precise synchronization of UV and Vis sources [S1].
*   **When to Use:** Suitable for applications requiring dynamic, full-colour volumetric displays with simple hardware, such as information display and human-computer interaction [S1].
*   **When Not to Use:** When high-resolution boundaries are critical without post-processing or specialized optics, or when extremely high frame rates are required beyond SLM capabilities [S1].

## Background
*   **Origin:** Proposed and experimentally demonstrated by Hirayama R. et al. in 2016 [S1].
*   **Development:** The method builds on the properties of P-type photochromic materials (diarylethenes), which are thermally irreversible and exhibit colour states dependent on both UV and Vis irradiation. Previous T-type PM displays required layered plates and suffered from voxel overlap restrictions [S1]. This P-type mixture approach allows uniform mixing, enabling simpler hardware and higher density [S1].
*   **Usage:** Fields of information display and human–computer interaction [S1].

## Sources
[S1] Optical Addressing of Multi-Colour Photochromic Material Mixture for Volumetric Display., https://doi.org/10.1038/srep31543, fetched 2026-10-26
