# Quantum-Dot-Based Volumetric Display

Also called: QD Volumetric Display

## Overview
A volumetric display system that eliminates occlusion caused by internal electrical wiring by embedding quantum dots (QDs) in a transparent polymer substrate [S1]. Unlike conventional displays that use LED arrays or scattering strings, this method uses external light irradiation to excite QDs, causing them to emit visible light spontaneously and form 3D voxels without internal power connections [S1]. It is distinguished from [Inkjet Printing-Based Volumetric Display](6810-inkjet-printing-based-volumetric-display.md) by its use of solid-state QD blocks rather than liquid dyes, and from [Downing Three-Color Display](7247-downing-three-color-display.md) by its reliance on external excitation rather than internal phosphor layers driven by electron beams or specific optical addressing.

## How it works
**Inputs:**
*   External excitation light source (UV or visible, typically 302–325 nm) [S1].
*   A 3D grid of voxels $V_{ijk}$ defined by the spatial arrangement of QD types.

**Outputs:**
*   Visible 3D image formed by spontaneous emission from excited QDs, viewable from multiple angles simultaneously [S1].

**Data Structures:**
*   **Voxel ($V_{ijk}$):** A cubic element of size $2.5 \text{ mm} \times 2.5 \text{ mm} \times 2.5 \text{ mm}$ [S1].
*   **Voxel Composition:** Each voxel is cut from a solid block of QDs encapsulated in clear polydimethylsiloxane (PDMS) polymer [S1].
*   **QD Types ($S_m$):** A set $\{S_1, S_2, \dots, S_M\}$ where $M$ is the number of distinct emission colors. In the prototype, $M=2$ (Red and Green) [S1].
*   **Voxel Value:** $V_{ijk} = 0$ if no QDs are present; otherwise, $V_{ijk} \in \{S_1, \dots, S_M\}$ [S1].
*   **Grid Dimensions:** $N \times N \times N$. In the prototype, $N=8$, resulting in an $8 \times 8 \times 8$ grid [S1].

**Fabrication Steps:**
1.  **Block Preparation:** Prepare solid blocks of QDs encapsulated in PDMS. Two types are created: 'Red Block' and 'Green Block' [S1].
2.  **Voxel Cutting:** Cut each solid block into cubes of size $2.5 \text{ mm} \times 2.5 \text{ mm} \times 2.5 \text{ mm}$, termed 'QD voxels' [S1].
3.  **Layer Arrangement:** Arrange the QD voxels into a mold according to a specific 2D layer design. Eight separate layers are prepared [S1].
4.  **Polymer Encapsulation:** Pour defoamed polymer (a mixture of base and curing agent) into the mold containing the arranged voxels [S1].
5.  **Solidification:** Heat the mold at $130^\circ\text{C}$ for approximately 3 hours to solidify the polymer, creating one solidified layer containing the QD voxels [S1].
6.  **Stacking:** Stack the eight solidified layers to form the final $8 \times 8 \times 8$ volumetric display [S1].

**Excitation and Emission:**
1.  **Irradiation:** The display is exposed to external light (e.g., UV at 302 nm or 325 nm) [S1].
2.  **Absorption:** QDs absorb high-energy photons. PDMS is transparent to visible light and has minimal effect on QD absorption spectra [S1].
3.  **Emission:** QDs relax by emitting lower-energy visible photons.
    *   Red Block Peak: $629 \text{ nm}$ (measured with 325 nm excitation) [S1].
    *   Green Block Peak: $541 \text{ nm}$ (measured with 325 nm excitation) [S1].

**Color Perception Model:**
The image observed from an axis is the spectral sum of voxels along that line of sight. For example, the image from the x-axis $A_{ij}$ is:
$$ A_{ij} = \sum_{k=1}^{N} V_{ijk} $$
This summation represents the addition of light spectra, recognized as a single color by the human eye (e.g., Red + Green = Yellow) [S1].

**Parameters:**
*   **Voxel Size:** $2.5 \text{ mm}$ cube [S1].
*   **Grid Size:** $8 \times 8 \times 8$ (prototype) [S1].
*   **Excitation Wavelengths:** 302 nm (transilluminator) or 325 nm (laser) [S1].
*   **Curing Temperature:** $130^\circ\text{C}$ [S1].
*   **Curing Time:** ~3 hours [S1].

**Edge Cases and Pitfalls:**
*   **Red Shift:** Encapsulation in PDMS causes peak emissions to shift to longer wavelengths due to subtle energy flow to the polymer. Red block shifts by $14 \text{ nm}$; Green block shifts by $11 \text{ nm}$ [S1].
*   **Order Dependence:** The sequence of voxels relative to the excitation source affects the emitted color due to absorption and transmission coefficients. Arranging Red then Green (RG) yields a different spectrum than Green then Red (GR) [S1].
*   **Spectral Shift Magnitude:** The peak photoluminescence from the green block shifts by approximately $29\%$ when comparing RG vs. GR combinations [S1].
*   **Chromaticity Separation:** The distance between RG and GR emission points on the CIE chromaticity diagram is $3.56 \times 10^{-2}$. This is distinguishable by the human eye, as the differentiation threshold is approximately $0.5 \times 10^{-2}$ [S1].
*   **Non-Uniformity:** Variance in pixel RGB values may arise from fabrication-induced non-uniformity in voxel sizes [S1].

## Operating characteristics
*   **Occlusion:** Eliminates occlusion caused by electrical wiring, a significant degradation factor in conventional LED/string-based volumetric displays [S1].
*   **Definition:** Offers ultrahigh definition due to the discrete nature of QD voxels and lack of scattering centers from wires [S1].
*   **Color Range:** Supports a wide range of colors through the combination of different QD types (e.g., Red, Green, and potentially Blue) [S1].
*   **Viewing Angles:** Images can be viewed simultaneously from multiple orthogonal viewpoints (x, y, z axes) [S1].
*   **Limitations:**
    *   Requires external excitation source; the display emits no visible light under white light irradiation [S1].
    *   Color perception is dependent on the viewing direction relative to the excitation source due to order-dependent absorption/transmission [S1].
    *   Fabrication complexity involves precise cutting, molding, and stacking of layers [S1].
*   **Performance:** The prototype demonstrated chequered, striped, and uniform patterns visible from all three orthogonal directions [S1].

## Background
*   **Origin:** Proposed and experimentally demonstrated by researchers at The University of Tokyo [S1].
*   **Publication:** "Design, Implementation and Characterization of a Quantum-Dot-Based Volumetric Display," *Scientific Reports*, 2015 [S1].
*   **Development:** The study addresses the historical issue of occlusion in volumetric displays caused by electrical wiring or scattering centers in LED arrays and string-based projectors [S1]. The prototype used an $8 \times 8 \times 8$ grid with two QD types (Red and Green) to demonstrate multi-view 3D imaging [S1].

## Sources
[S1] Design, Implementation and Characterization of a Quantum-Dot-Based Volumetric Display, https://doi.org/10.1038/srep08472, fetched 2026-10-26
