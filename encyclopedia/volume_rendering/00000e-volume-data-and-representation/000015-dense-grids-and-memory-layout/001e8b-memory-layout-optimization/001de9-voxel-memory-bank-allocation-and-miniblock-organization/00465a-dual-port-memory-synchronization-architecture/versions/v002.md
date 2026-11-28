# Dual-Port Memory & Synchronization Architecture

Also called: SSC San Diego Volumetric Control Card, Volumetric #2 Card Memory Interface, Volumetric #2 Dual-Port Memory & Synchronization Architecture

## Overview
The Dual-Port Memory & Synchronization Architecture is a hardware interface method for 3-D volumetric displays that transforms 2D image points into 3D voxels by synchronizing memory readout with the physical position of a moving display surface (helix or piston) [S1, unofficial]. It differs from sibling methods like [Skewed Memory Organization](2351-skewed-memory-organization-for-conflict-free-ray-access.md) or [Cubic Addressing Scheme](7285-cubic-addressing-scheme.md) by relying on external mechanical synchronization signals rather than static memory layout tricks to resolve spatial mapping, allowing direct memory mapping for high-speed host writes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 2D image points (X, Y, Intensity) from a host PC; mechanical index signals (`*System_0_Index`, `*System_180_Index`) from the display surface shaft sensors [S1, unofficial].
*   **Output:** Sequential X and Y coordinate streams sent to the Newport Electro-Optics Systems (NEOS) computer interface controller board for each voxel [S1, unofficial].

**Data Structures:**
The core data structure is the **Voxel Word**, stored in Dual-Port RAM [S1, unofficial]. It is 32 bits wide with the following layout [S1, unofficial]:

| Bits | Field | Size | Description |
| :--- | :--- | :--- | :--- |
| 31–20 | X Coordinate | 12 bits | 4096 addressability [S1, unofficial]. |
| 19–8 | Y Coordinate | 12 bits | 4096 addressability [S1, unofficial]. |
| 7–0 | Intensity | 8 bits | 255 levels + blanking (all zeros) [S1, unofficial]. |
| 17 | X11 / Ext Video | 1 bit | Configurable via jumpers [S1, unofficial]. |
| 18 | Y11 / Full Width | 1 bit | Configurable via jumpers [S1, unofficial]. |

Bits 17 and 18 are configurable via jumpers to serve as X11 MSB or Extended Video, and Y11 MSB or Full Width Video, respectively [S1, unofficial].

**Memory Configuration:**
The Dual-Port RAM holds up to 65,536 × 32-bit words [S1, unofficial]. It is typically configured as sixteen 4K pages or eight 8K pages [S1, unofficial]. Port A is mapped to a 256-kB address block in the 1–16 MB host range for direct memory-mapped writes [S1, unofficial]. Port B is the read port, controlled by Counter B [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:** The host writes voxel data to Port A via direct memory mapping [S1, unofficial]. The initial address for Counter B is loaded from an initial register, defining the start of the current memory page [S1, unofficial].
2.  **Sequential Readout:** Port B reads sequentially starting at Counter B's address [S1, unofficial].
3.  **Counter Increment:** Counter B increments per voxel readout [S1, unofficial].
4.  **Synchronization Reload:** Upon receiving `*System_0_Index` or `*System_180_Index` from shaft sensors, Counter B reloads from the initial register [S1, unofficial].
5.  **Helix Mode:** Counter B always increments [S1, unofficial].
6.  **Piston Mode:** Selectable jumpers cause Counter B to start incrementing at 0 degrees, then reverse direction at 180 degrees to count down [S1, unofficial].
7.  **Timer Trigger:** Timer 0 of the 82C54-2 generates a periodic active low pulse to trigger the readout [S1, unofficial].

**Formulas:**
*   Readout Period: $T = N \times 100 \text{ ns}$, where $N > 1$ is an integer [S1, unofficial].
*   Max Voxels per Second: $V_{max} = \frac{1}{T}$ [S1, unofficial].
*   Example: If $N=13$, $T = 1.30 \text{ \mu s}$, allowing $38,461$ voxels per second [S1, unofficial].

**Parameters:**
*   **Clock:** 10 MHz input to the 82C54-2 timer [S1, unofficial].
*   **N (Divisor):** Integer $> 1$. Typical value $N=13$ [S1, unofficial].
*   **Refresh Rate:** Nominally 20 Hz, balancing voxel count per frame against flicker [S1, unofficial].
*   **Memory Size:** 65,536 words (Volumetric #2) vs 12,288 words (Volumetric #1) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Flicker:** Lower refresh rates allow more voxels per frame but cause excessive flicker [S1, unofficial]. Increasing brightness also exacerbates flicker unless the refresh rate is increased [S1, unofficial].
*   **Synchronization:** The system relies on accurate `*System_0_Index` and `*System_180_Index` signals from the shaft sensors to reset the address counter; failure to sync causes spatial distortion [S1, unofficial].
*   **Jumpers:** Incorrect jumper settings for X11/Y11 bits may result in incorrect coordinate mapping or video mode selection [S1, unofficial].

## Subtopics

### Memory Readout & Piston Mode Control
Configures the 3-D memory as sixteen 4K or eight 8K pages and controls readout direction/counting via selectable jumpers [S1, unofficial]. In piston mode, `*System_0_Index` reinitializes memory readout to the beginning of the image and starts counting up, while `*System_180_Index` causes address counter “B” to reverse direction and begin counting down [S1, unofficial]. The readout rate is controlled by Timer 0 of the 82C54-2 programmed to mode 2 (divide by N) with $N=13$, producing a 100-ns active low pulse every 1.30 µs [S1, unofficial]. This allows up to 38,461 voxels to be read and displayed in 1/20 second, supporting a nominal 20 Hz refresh rate [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Readout rate is controlled by the 82C54-2 programmable timer divisor $N$ [S1, unofficial].
*   **Memory Complexity:** Supports up to 65,536 32-bit words, mapped to a 256-kB address block [S1, unofficial].
*   **Measured Performance:** At $N=13$, readout rate is 1.30 µs per voxel, allowing up to 38,461 voxels per second [S1, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Direct memory mapping allows faster data transfers than I/O memory commands used in the Volumetric #1 card [S1, unofficial]. Synchronization with mechanical motion ensures accurate 3D positioning [S1, unofficial].
*   **Limitations:** Refresh rate is fixed by hardware timer settings; lower refresh rates cause flicker [S1, unofficial]. Limited to specific mechanical configurations (helix/piston) [S1, unofficial].
*   **When to use:** When using SSC San Diego's 3-D Volumetric Display System with NEOS 40K voxel scanners in green/red master-slave configurations [S1, unofficial].
*   **When not to use:** Not given in the sources.

## Background
The architecture was developed by SSC San Diego for their 3-D Volumetric Display System, which uses a rotating 36-inch-diameter double helix with an 18-inch depth [S1, unofficial]. It is the second-generation interface (Volumetric #2), succeeding the Volumetric #1 card which stored 12,288 × 24-bit words and used I/O memory commands [S1, unofficial]. The system supports viewing angles up to 180 degrees without polarizing glasses by reflecting a laser beam from a diffused moving surface [S1, unofficial].

## Sources
[S1] DTIC ADA358303: Improved Second-Generation 3-D Volumetric Display System. Revision 2, https://archive.org/details/DTIC_ADA358303, fetched 2026-10-26 (unofficial)
