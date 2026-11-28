# Viewpoint-surrounding Hemisphere System

Also called: Holographic tabletop system, Tabletop display using space-division, Time-division tabletop display

## Overview
The Viewpoint-surrounding Hemisphere System is a tabletop volumetric display method where viewpoints are created above a view volume in which virtual objects are formed [S1, unofficial]. It is based on multi-view technology composed of multiple projection optics and can be implemented using space-division or time-division techniques [S1, unofficial]. It differs from its siblings by specifically targeting the tabletop form factor with a hemispherical or conical screen geometry that directs light upward to intersect above the table surface, enabling 360-degree viewing without crosstalk [S1, unofficial].

## How it works
The method generates a 3D view volume above a tabletop by projecting elemental images onto a rotating or static screen geometry. There are four primary implementation variants described in the literature: Space-Division (fVisiOn), Time-Division (Takaki and Uchida), Time-Division (Kim et al.), and Holographic (Lim et al.).

### Inputs and Outputs
*   **Inputs:** 3D scene data or elemental image arrays; rotation speed (for time-division); projector parameters.
*   **Outputs:** A 3D volumetric image visible from 360 degrees above the tabletop.

### Data Structures
The source does not specify a universal data structure. Implementation depends on the variant:
*   **Space-Division:** An array of elemental images, one for each projector.
*   **Time-Division:** A sequence of elemental images synchronized with rotation.
*   **Holographic:** Holographic data (phase/amplitude) for the SLM.

### Steps

#### Variant 1: Space-Division (fVisiOn)
1.  **Screen Setup:** Position an anisotropic diffusing screen in the shape of a cone directly beneath the tabletop [S1, unofficial].
2.  **Projector Placement:** Arrange $N$ projectors evenly spaced along the circumference beneath the screen. The source specifies $N=288$ with a spacing interval of $1.25^\circ$ [S1, unofficial].
3.  **Projection:** Project 2D elemental images onto the conic screen from each projector [S1, unofficial].
4.  **Optical Convergence:** The conical screen provides a wide diffusing angle vertically and a narrow horizontal diffusing angle (approximately $0.4^\circ$) to enable 360-degree viewpoints without crosstalk [S1, unofficial].
5.  **View Volume Formation:** The optical axes of each projector intersect above the tabletop, forming a view volume observable from 360 degrees [S1, unofficial].

#### Variant 2: Time-Division (Takaki and Uchida)
1.  **Setup:** Position a high-speed projector along the table’s central axis [S1, unofficial].
2.  **Optics:** Use a rotating decentered Fresnel lens [S1, unofficial].
3.  **Light Redirection:** The decentered lens redirects light from each elemental image off-axis to converge at a point on the circumference above the tabletop [S1, unofficial].
4.  **Synchronization:** The decentered lens rotates around the central axis. The high-speed projector synchronizes with the rotation [S1, unofficial].
5.  **Viewpoint Generation:** This synchronization generates 800 viewpoints [S1, unofficial].

#### Variant 3: Time-Division (Kim et al.)
1.  **Setup:** Use an inclined and off-axis anisotropic diffusing screen [S1, unofficial].
2.  **Projection:** Project elemental images onto the rotating screen in the normal direction using folded high-speed projector optics [S1, unofficial].
3.  **Overlap:** Lights emanating from each elemental image on the screen overlap each other above the tabletop [S1, unofficial].
4.  **View Volume:** The overlap creates a view volume in the region above the table [S1, unofficial].

#### Variant 4: Holographic (Lim et al.)
1.  **SLM:** Use a high-speed spatial light modulator (SLM) to reconstruct holographic images [S1, unofficial].
2.  **Scanning:** Employ rotation scanning optics and parabolic mirrors to create a 360-degree perspective [S1, unofficial].
3.  **Focusing:** Use aspheric lenses or freeform mirror-based reflective optics to focus light waves [S1, unofficial].
4.  **Effect:** This provides natural 3D images with accommodation effects at each viewpoint [S1, unofficial].

### Parameters
*   **Space-Division:** Number of projectors: 288. Spacing interval: $1.25^\circ$. Horizontal diffusing angle: $\approx 0.4^\circ$ [S1, unofficial].
*   **Time-Division (Takaki/Uchida):** Number of viewpoints: 800 [S1, unofficial].

### Pitfalls
*   **Space-Division:** Requires numerous projection optics, leading to alignment challenges [S1, unofficial].
*   **Time-Division:** Suffers from mechanical vibrations, lack of optical brightness, and flickering problems [S1, unofficial].

## Operating characteristics
*   **Space-Division:** High hardware complexity (288 projectors in the cited example). Alignment is difficult. No flickering or mechanical vibration issues.
*   **Time-Division:** Lower hardware count (single projector in cited examples). Issues include mechanical vibration, reduced brightness, and flickering.
*   **Holographic:** Offers accommodation effects. Complexity lies in high-speed SLM and scanning optics.

## Background
The system is described in the context of tabletop volumetric displays suitable for applications where viewpoints are created above the view volume [S1, unofficial]. Representative systems include "fVisiOn" by Yoshida et al. [S1, unofficial], time-division displays by Takaki and Uchida [S1, unofficial] and Kim et al. [S1, unofficial], and holographic systems by Lim et al. [S1, unofficial] and Heo et al. [S1, unofficial].

## Sources
[S1] Volumetric 3D Display: Features and Classification, https://www.coppjournal.org/journal/view.html?uid=1568, fetched 2026-10-28 (unofficial)
