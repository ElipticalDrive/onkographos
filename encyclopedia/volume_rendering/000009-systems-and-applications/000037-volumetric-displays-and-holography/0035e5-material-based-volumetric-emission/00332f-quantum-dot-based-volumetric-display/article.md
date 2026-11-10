# Quantum-Dot-Based Volumetric Display

Also called: QD Volumetric Display

## Overview
A volumetric display system that eliminates occlusion caused by electrical wiring by embedding quantum dots (QDs) in a polymer substrate, where external light irradiation excites the QDs to emit visible light spontaneously [S1]. Unlike conventional volumetric displays that use LED arrays or scattering strings, this method defines voxels via the physical placement of active material rather than internal power connections, resolving image distortion from scattering centers [S1]. It sets itself apart from siblings like [Inkjet Printing-Based Volumetric Display](6810-inkjet-printing-based-volumetric-display.md) by using pre-fabricated solid QD blocks rather than liquid inks, and from [Downing Three-Color Display](7247-downing-three-color-display.md) by relying on external excitation rather than internal phosphor conversion layers [S1].

## How it works
**Inputs:**
- A 3D grid of target voxels $V_{ijk}$ where $i, j, k \in \{1, \dots, N\}$.
- A set of QD types $\{S_1, S_2, \dots, S_M\}$, where each type $S_m$ has a specific emission spectrum.
- An external excitation source (e.g., UV laser or projector) with wavelength $\lambda_{exc}$.

**Outputs:**
- A physical 3D display object composed of stacked polymer layers containing QD voxels.
- Visible 3D images formed by the spontaneous emission of light from excited QDs when irradiated.

**Data Structures:**
The display is represented as a 3D array $V$ of size $N \times N \times N$.
- $V_{ijk}$: The voxel at coordinates $(i, j, k)$.
- Value of $V_{ijk}$:
  - $0$: Empty voxel (no QDs).
  - $m \in \{1, \dots, M\}$: Voxel contains QDs of type $S_m$.
- In the prototype, $N=8$ and $M=2$ (Red and Green) [S1].

**Fabrication Steps:**
1.  **Block Preparation:** QDs are encapsulated in clear polydimethylsiloxane (PDMS) polymer to form solid blocks. Two block types are created: 'red block' (emits red) and 'green block' (emits green) [S1].
2.  **Voxel Cutting:** Each solid block is cut into cubes of size $2.5 \text{ mm} \times 2.5 \text{ mm} \times 2.5 \text{ mm}$, referred to as 'QD voxels' [S1].
3.  **Layer Arrangement:** QD voxels are arranged into a mould according to the design for a specific layer (e.g., $z=k$). Eight layers are prepared separately [S1].
4.  **Polymer Encapsulation:** A defoamed mixture of PDMS base and curing agent is poured into the mould containing the arranged voxels [S1].
5.  **Solidification:** The polymer is solidified by heating at $130^\circ\text{C}$ for approximately 3 hours. This results in one solidified layer containing the QD voxels [S1].
6.  **Stacking:** The eight solidified layers are stacked to obtain the final $8 \times 8 \times 8$ voxel volumetric display [S1].

**Excitation and Emission:**
- The display is irradiated with external light (e.g., UV at $325 \text{ nm}$) [S1].
- QDs absorb energy and transition to a higher energy state, emitting visible light spontaneously [S1].
- The observed color from a specific viewpoint is the sum of the light spectra from all QDs along the line of sight. This summation is spectral, not arithmetic, and is perceived as a single color by the human eye (e.g., Red + Green = Yellow) [S1].

**Pseudocode for Design Specification:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Define display parameters
N = 8  # Grid dimension (8x8x8)
M = 2  # Number of QD types
QD_TYPES = {
    1: {"name": "Red", "peak_emission_nm": 629, "shift_nm": 14},
    2: {"name": "Green", "peak_emission_nm": 541, "shift_nm": 11}
}

# Define the 3D voxel grid
# V[i][j][k] stores the QD type index (0 for empty)
V = [[[0 for k in range(N)] for j in range(N)] for i in range(N)]

# Example: Set a voxel to Red QD
V[0][0][0] = 1

# Example: Set a voxel to Green QD
V[0][0][1] = 2

# Fabrication Logic
def fabricate_layer(layer_z, layer_design):
    """
    layer_design: list of (x, y, qd_type) tuples for voxels in this layer
    """
    mould = create_mould()
    for (x, y, qd_type) in layer_design:
        voxel = cut_qd_voxel(qd_type) # Cut 2.5mm cube from solid block
        mould.place(voxel, x, y, layer_z)
    
    pour_polymer(mould)
    solidify(mould, temp=130, duration_hours=3)
    return mould.get_layer()

def assemble_display():
    layers = []
    for z in range(N):
        # layer_design would be derived from V[:,:,z]
        layer_design = get_layer_design(V, z)
        layers.append(fabricate_layer(z, layer_design))
    
    display = stack_layers(layers)
    return display

# Emission Logic
def get_emission_spectrum(qd_type_index, order_from_source):
    """
    Returns the effective emission spectrum considering order-dependent shifts.
    order_from_source: list of QD types encountered from source to viewer.
    """
    # Note: The source indicates that order affects color due to absorption/transmission.
    # Specifically, RG vs GR combinations show significant spectral shifts.
    # This function would need to model the absorption of the preceding QDs.
    # For a single QD, it returns the base spectrum shifted by polymer inclusion.
    base_qd = QD_TYPES[qd_type_index]
    shifted_peak = base_qd["peak_emission_nm"] + base_qd["shift_nm"]
    return generate_spectrum(shifted_peak)

def observe_viewpoint(viewpoint_axis, viewpoint_index):
    """
    Simulates the color observed from a specific viewpoint.
    viewpoint_axis: 'x', 'y', or 'z'
    viewpoint_index: coordinate along the axis (1 to N)
    """
    accumulated_spectrum = empty_spectrum()
    
    # Determine the line of voxels along the viewing direction
    voxels_along_line = []
    if viewpoint_axis == 'x':
        for i in range(1, N+1):
            voxels_along_line.append(V[i-1][viewpoint_index-1][viewpoint_index-1]) # Simplified indexing
    elif viewpoint_axis == 'y':
        for j in range(1, N+1):
            voxels_along_line.append(V[viewpoint_index-1][j-1][viewpoint_index-1])
    elif viewpoint_axis == 'z':
        for k in range(1, N+1):
            voxels_along_line.append(V[viewpoint_index-1][viewpoint_index-1][k-1])
            
    # Sum spectra from all non-empty voxels along the line
    # Note: The order in voxels_along_line matters for absorption effects
    for qd_type in voxels_along_line:
        if qd_type != 0:
            spectrum = get_emission_spectrum(qd_type, voxels_along_line)
            accumulated_spectrum = add_spectra(accumulated_spectrum, spectrum)
            
    return perceived_color(accumulated_spectrum)
```

**Parameters:**
- **Voxel Size:** $2.5 \text{ mm} \times 2.5 \text{ mm} \times 2.5 \text{ mm}$ [S1].
- **Grid Size:** $8 \times 8 \times 8$ in the prototype ($N=8$) [S1].
- **QD Types:** $M=2$ (Red, Green) in the prototype [S1].
- **Excitation Wavelength:** $325 \text{ nm}$ (UV laser) used for characterization; UV irradiation used for operation [S1].
- **Polymer:** PDMS (polydimethylsiloxane), transparent at visible wavelengths [S1].
- **Curing Temperature:** $130^\circ\text{C}$ [S1].
- **Curing Time:** $\approx 3$ hours [S1].

**Edge Cases and Pitfalls:**
- **Red Shift:** Encapsulation in PDMS causes peak emissions to shift to longer wavelengths. Red block shifts by $14 \text{ nm}$; Green block shifts by $11 \text{ nm}$ due to subtle energy flow to the polymer [S1].
- **Order Dependence:** The order of voxels relative to the excitation source affects the emitted color due to absorption and transmission coefficients. Arranging Red then Green (RG) yields a different spectrum than Green then Red (GR) [S1].
- **Spectral Shift Magnitude:** The peak photoluminescence from the green block ($541 \text{ nm}$) shifts by approximately $29\%$ when comparing RG vs GR combinations [S1].
- **Chromaticity Separation:** The distance between RG and GR emission points on the CIE chromaticity diagram is $3.56 \times 10^{-2}$. This is distinguishable by the human eye, as the threshold for differentiation is approximately $0.5 \times 10^{-2}$ [S1].
- **Fabrication Nonuniformity:** Variance in pixel RGB values may depend on nonuniformity in voxel sizes arising from the fabrication process [S1].

## Operating characteristics
- **Occlusion:** Eliminates occlusion caused by electrical wiring, which is a significant degradation factor in conventional volumetric displays using LED arrays or scattering strings [S1].
- **Definition:** Offers ultrahigh definition due to the discrete nature of the QD voxels and lack of scattering centers [S1].
- **Color Range:** Provides a wide range of colours through the combination of different QD types (e.g., Red + Green = Yellow) [S1].
- **Viewing Angles:** Images can be viewed simultaneously from multiple orthogonal viewpoints (x, y, z axes) [S1].
- **Complexity:** The fabrication process is multi-step and requires precise alignment of layers and curing, potentially limiting scalability compared to simpler projection methods [S1].
- **Color Accuracy:** Subject to order-dependent color shifts, which must be accounted for in design to ensure consistent color perception from different angles [S1].

## Background
- **Origin:** Proposed and experimentally demonstrated by researchers at The University of Tokyo (2-11-16 Yayoi, Bunkyo-ku, Tokyo 113-8656, Japan) [S1].
- **Paper:** "Design, Implementation and Characterization of a Quantum-Dot-Based Volumetric Display" published in *Scientific Reports* (DOI: 10.1038/srep08472) [S1].
- **Motivation:** Previous volumetric displays suffered from occlusion due to electrical wiring or scattering centers, leading to significant degradation in 3D image quality [S1]. This method addresses that by using external light excitation of embedded QDs, removing the need for internal power connections [S1].

## Sources
[S1] Design, Implementation and Characterization of a Quantum-Dot-Based Volumetric Display, https://doi.org/10.1038/srep08472, fetched 2026-10-26
