# 2D Full-Color LED Array Rotational Display

Also called: LED Array Volumetric Display, Multivox Volumetric Display System, Rotational Volumetric Display

## Overview
A 2D full-color LED array rotational display renders volumetric images by rotating a 2D LED panel array around a central axis and synchronously updating the panel content to match the rotation angle [S2, unofficial]. It differs from laser-based swept-screen systems by generating voxels directly on an emissive surface without requiring external synchronization with electron beams or lasers [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel buffer (volume data) and a target rotation speed [S1, unofficial].
*   **Output:** A 3D volumetric image perceived by the human eye via persistence of vision [G].

**Data Structures:**
*   **Voxel Buffer:** A 3D array stored in shared memory, written by a client process and read by a driver process [S1, unofficial].
*   **LED Panel State:** The current 2D image frame to be displayed on the HUB75 panels [G].

**Parameters:**
*   **Images per rotation:** 512 frames per full revolution [S2, unofficial].
*   **Rotation Speed:** Several hundred RPM [S1, unofficial].
*   **Panel Type:** HUB75 LED panels [S1, unofficial].

**Steps:**
1.  **Initialization:** Configure the GPIO memory-mapped access base address (`BCM_BASE`) for the specific hardware (e.g., Raspberry Pi 4) [S1, unofficial]. Define GPIO mappings and panel layout in the driver configuration file [S1, unofficial].
2.  **Synchronization Setup:** Establish a sync signal using a single GPIO pin connected to a photodiode or similar sensor [S1, unofficial]. The signal is high for 180° of rotation and low for the other 180° [S1, unofficial].
3.  **Frame Calculation:** For each rotation, calculate the 512 required 2D slices [S2, unofficial]. Extract the 2D cross-section of the voxel buffer corresponding to the current rotation angle [G].
4.  **Buffer Update:** Write the calculated 2D frame data into the shared memory voxel buffer [S1, unofficial].
5.  **Scanout:** The driver reads the voxel buffer and drives the HUB75 LED panels in sync with the rotation angle detected by the sync signal [S1, unofficial].
6.  **Persistence:** The rapid rotation and frame updates blend the discrete 2D slices into a continuous 3D volume [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Configuration
BCM_BASE = None  # Base address for memory-mapped GPIO access [S1, unofficial]
FRAMES_PER_ROTATION = 512  # [S2, unofficial]
SYNC_PIN = None  # GPIO pin for photodiode sync [S1, unofficial]

def init_gpio():
    # Memory-map GPIO access using BCM_BASE [S1, unofficial]
    # Configure SYNC_PIN as input
    pass

def read_sync():
    # Read digital state of SYNC_PIN
    # Returns True for high (180°), False for low (180°) [S1, unofficial]
    return True  # [G]

def extract_slice(buffer, k):
    # Compute angle theta = (k / FRAMES_PER_ROTATION) * 2 * pi
    # Extract 2D cross-section of buffer at theta
    # Return 2D pixel array for HUB75 panels [G]
    return []

def drive_panels(image):
    # Write image data to HUB75 panels
    # Supports ABCDE addressing or ABC shift register addressing [S1, unofficial]
    pass

def main():
    init_gpio()
    k = 0
    while True:
        # Driver scans voxel buffer in sync with rotation [S1, unofficial]
        slice_data = extract_slice(voxel_buffer, k)
        drive_panels(slice_data)
        k = (k + 1) % FRAMES_PER_ROTATION
        # Timing is managed by hardware sync signal and rotation speed [G]
```

**Edge Cases and Pitfalls:**
*   **Hardware Compatibility:** The GPIO memory-mapped access assumes a Raspberry Pi 4. Using different Pi models requires changing `BCM_BASE` and is untested [S1, unofficial].
*   **Synchronization:** Precise timing between the screen's angular position and the image update is critical [G]. The photodiode sync signal must be correctly wired to distinguish the 180° high and low phases [S1, unofficial].

## Operating characteristics
*   **Complexity:** Time complexity is linear with respect to the number of voxels and frames per rotation [G]. Memory complexity is proportional to the voxel buffer size [G].
*   **Performance:** Requires high-speed GPIO updates and precise timing. The system runs on a Raspberry Pi 4 embedded in the spinning hardware [S1, unofficial].
*   **Strengths:** Self-contained system; no external synchronization devices needed; full-color output [S2, unofficial].
*   **Limitations:** Limited to the resolution of the LED panels; rotation speed limits the maximum volume size and frame rate; hardware-specific GPIO code [S1, unofficial].

## Background
The concept of using rotating 2D full-color LED arrays for volumetric display was proposed by Xie et al. in 2009 [S2, unofficial]. This method was later categorized as a "free-space" volumetric display by Sumalley et al. in 2018 [S2, unofficial]. A hardware implementation uses two HUB75 LED panels controlled by a Raspberry Pi 4 [S1, unofficial].

## Sources
[S1] Multivox: Volumetric Display, https://github.com/AncientJames/multivox, fetched 2026-10-26 (unofficial)
[S2] ボリュメトリックディスプレイ, https://www.jstage.jst.go.jp/article/jvrsj/29/3/29_27/_pdf/-char/ja, fetched 2026-10-30 (unofficial)
