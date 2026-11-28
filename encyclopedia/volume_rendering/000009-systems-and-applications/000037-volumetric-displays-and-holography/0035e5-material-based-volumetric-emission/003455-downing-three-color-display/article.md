# Downing Three-Color Display

Also called: Two-photon absorption volumetric display, Volumetric Display (Two-Photon Upconversion)

## Overview
The Downing Three-Color Display is a volumetric imaging technique that generates true 3D colored pictures by addressing voxels within a rare-earth ion doped glass medium using two intersecting infrared laser beams [S1, unofficial]. Unlike 2D representations with depth cues, this method allows the eye to focus at different depths (ocular accommodation) and be viewed from any angle [S1, unofficial]. It sets itself apart from sibling methods like [Inkjet Printing-Based Volumetric Display](6810-inkjet-printing-based-volumetric-display.md) by using optical addressing of a transparent solid medium rather than mechanical deposition, and from [Quantum-Dot-Based Volumetric Display](6943-quantum-dot-based-volumetric-display.md) by relying on two-photon upconversion in doped glass rather than fluorescence in suspended particles [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Two infrared laser beams of appropriate wavelengths intersect within the display medium [S1, unofficial]. The medium consists of stacked layers of rare-earth doped glass [S1, unofficial].
*   **Output:** A visible 3D image formed by fluorescent voxels at the intersection of the laser beams, viewable from any direction with natural ocular accommodation [S1, unofficial].

**Data Structures**
The display medium is not a single monolithic block but a stack of thin glass layers [S1, unofficial].
*   **Layer Structure:** Alternating layers of glass doped with specific rare-earth ions are stacked to prevent optical cross-talk between voxels in different depth planes [S1, unofficial].
*   **Dopant Assignment:** Each layer (or set of layers) is doped with a specific ion to produce a primary color: Praseodymium (Pr) for red emission, Erbium (Er) for green emission, and Thulium (Tm) for blue emission [S1, unofficial].
*   **Laser Arrays:** Instead of single scanning beams, linear arrays of infrared diode lasers are used to increase refresh rate and manage power density [S1, unofficial].

**Step-by-Step Method**
1.  **Beam Generation:** Two infrared laser beams are generated at excitation wavelengths of 800 nm and 1,550 nm [S2, unofficial].
2.  **Scanning:** The beams are scanned throughout the display volume using linear laser arrays rather than a single pair to allow faster scanning and lower power per voxel [S1, unofficial].
3.  **Intersection:** The two beams intersect at a specific 3D coordinate $(x, y, z)$ within the glass medium [S1, unofficial].
4.  **Two-Photon Absorption (Upconversion):** An active ion in the glass, initially at energy level $E_1$, absorbs a photon from the first infrared beam (wavelength $\lambda_{12}$) and is excited to energy level $E_2$ [S1, unofficial]. The ion remains at $E_2$ for a lifetime $t_2$ [S1, unofficial]. If the second infrared beam (wavelength $\lambda_{23}$) illuminates the ion during this lifetime $t_2$, the ion absorbs a second photon and is further excited to energy level $E_3$ [S1, unofficial].
5.  **Emission:** The ion remains at $E_3$ for a lifetime $t_3$, then relaxes back to level $E_1$, emitting a photon of visible light [S1, unofficial].
6.  **Voxel Definition:** Visible light is generated *only* where the two beams intersect [S1, unofficial]. Scanning both beams throughout the display volume draws the 3D pattern [S1, unofficial].

**Formulas**
The energy level transitions are defined as:
$$ E_1 + h\nu_{12} \rightarrow E_2 $$
$$ E_2 + h\nu_{23} \rightarrow E_3 $$
$$ E_3 \rightarrow E_1 + h\nu_{visible} $$
Where:
*   $E_1, E_2, E_3$ are energy levels of the active ion [S1, unofficial].
*   $h$ is Planck's constant [G].
*   $\nu_{12}$ is the frequency of the first infrared beam ($\lambda_{12}$) [S1, unofficial].
*   $\nu_{23}$ is the frequency of the second infrared beam ($\lambda_{23}$) [S1, unofficial].
*   $\nu_{visible}$ is the frequency of the emitted visible light [S1, unofficial].
*   The condition for emission is that the second photon arrives within the lifetime $t_2$ of the intermediate state $E_2$ [S1, unofficial].

**Parameters**
*   **Excitation Wavelengths:** 800 nm and 1,550 nm [S2, unofficial].
*   **Dopants:** Praseodymium (Red), Erbium (Green), Thulium (Blue) [S1, unofficial].
*   **Medium:** Rare-earth ion doped glass, transparent to visible light [S1, unofficial].

**Pitfalls and Edge Cases**
*   **Glass Damage:** High power is required for single-pair scanning to achieve video rates [S1, unofficial]. The power density needed would exceed the damage threshold of most glasses [S1, unofficial].
    *   *Mitigation:* Use many lasers (linear arrays) to refresh voxels more frequently with lower individual power [S1, unofficial].
*   **Cross-talk:** If the entire volume were fully doped with the same ions, light emitted from one voxel could excite neighbors or scatter, causing cross-talk [S1, unofficial].
    *   *Mitigation:* Stack thin layers of alternately doped glass [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def downing_display_render(scene_voxels, glass_stack):
    """
    Renders a 3D scene by scanning laser beams through a doped glass medium.
    scene_voxels: List of (x, y, z, color) tuples representing desired voxels.
    glass_stack: List of dicts with 'z' (depth) and 'dopant' ('Pr', 'Er', 'Tm').
    """
    color_to_dopant = {"red": "Pr", "green": "Er", "blue": "Tm"}
    lambda_1 = 800e-9  # 800 nm
    lambda_2 = 1550e-9 # 1550 nm

    for x, y, z, color in scene_voxels:
        # Find layers at the target depth
        target_layers = [layer for layer in glass_stack if layer['z'] == z]
        required_dopant = color_to_dopant.get(color)
        valid_layers = [l for l in target_layers if l['dopant'] == required_dopant]

        if not valid_layers:
            continue  # Skip if no appropriate dopant layer exists at this depth

        # Direct beams to intersect at (x, y, z)
        # Hardware-specific beam steering logic is not detailed in the sources [G]
        # steer_beams(x, y, z, lambda_1, lambda_2)

        # The physical upconversion process occurs automatically upon intersection:
        # 1. Ion at E1 absorbs photon from lambda_1 -> moves to E2
        # 2. Ion at E2 absorbs photon from lambda_2 (within lifetime t2) -> moves to E3
        # 3. Ion at E3 relaxes to E1, emitting visible light
```

## Operating characteristics
*   **Time Complexity:** Not explicitly defined in sources, but the use of linear laser arrays instead of single beams suggests an attempt to improve refresh rates for video-capable performance [S1, unofficial].
*   **Memory Complexity:** The display medium is static (doped glass); no dynamic memory is required for the medium itself [S1, unofficial].
*   **Measured Performance:** A prototype system developed at Stanford created 3D color pictures in a one-inch cube [S1, unofficial].
*   **Scaling:** Estimated cost for a ten-inch cube was $80,000, indicating significant cost barriers to scaling [S1, unofficial].
*   **Strengths:** True 3D with ocular accommodation (eye can focus at different depths) [S1, unofficial]. Viewable from any angle [S1, unofficial]. Dynamic scene rendering capability [S1, unofficial].
*   **Limitations:** High cost [S1, unofficial]. Transparent medium lacks opacity [S1, unofficial]. Limited by the availability of suitable rare-earth dopants and laser sources [S1, unofficial].
*   **When to use:** Applications requiring true 3D visualization with natural depth perception, such as medical imaging (MRI, CAT), aircraft tracking, and CAD [S1, unofficial].
*   **When not to use:** Cost-sensitive applications [S1, unofficial].

## Background
*   **Origin:** The technique was proposed by Downing et al. in 1996 [S1, unofficial]. Research groups had addressed the possibility as early as the early 1970s but lacked the required excitation sources or display materials [S1, unofficial].
*   **Development:** The solution came from the fiber optic industry in the form of high-quality rare-earth ion doped glasses [S1, unofficial]. A prototype system was developed at Stanford University [S1, unofficial].
*   **History:** The method is based on two-photon absorption (upconversion) processes in rare-earth ions [S1, unofficial]. It builds on the shared principles of [Material-Based Volumetric Emission](7614-material-based-volumetric-emission.md) by using a solid glass medium and infrared excitation to generate visible fluorescence at the intersection of beams [S1, unofficial].
*   **Usage:** Proposed for medical imaging, tracking, and CAD [S1, unofficial].

## Sources
[S1] "Volumetric Display", https://web.archive.org/web/20050909205829/http://www.stanford.edu/~matteoja/volume.html, fetched 2026-10-26 (unofficial)
[S2] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
