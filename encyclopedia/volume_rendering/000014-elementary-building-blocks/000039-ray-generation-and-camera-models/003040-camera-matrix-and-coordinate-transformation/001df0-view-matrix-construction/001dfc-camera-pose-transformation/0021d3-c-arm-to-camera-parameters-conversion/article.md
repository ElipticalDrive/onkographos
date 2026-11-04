# C-Arm to Camera Parameters Conversion


## Overview
This method converts C-arm geometric parameters (Source-to-Image Distance, AP angle, lateral angle, center, table translation) into camera position, look-at point, and up vector in RAS coordinates [S1, unofficial]. It explicitly models the C-arm's mechanical degrees of freedom (AP and lateral tilts) relative to a patient-centered coordinate system to generate synthetic Digital Radiography (DRR) views [S1, unofficial].

## How it works
**Inputs:** `sid` (float, mm), `ap_angle` (float, degrees), `lat_angle` (float, degrees), `center_ras` (3-element array, mm), `table_si` (float, mm, default 0.0).
**Outputs:** `cam_pos` (3-element array, RAS), `look_at` (3-element array, RAS), `look_up` (3-element array, RAS).
**Data structures:** All vectors are 3-element arrays/tensors; rotation matrices are $3 \times 3$ matrices [G].
**Steps:**
1. Convert angles to radians: $\text{ap\_rad} = \text{deg2rad}(\text{ap\_angle})$, $\text{lat\_rad} = \text{deg2rad}(\text{lat\_angle})$ [S1, unofficial].
2. Adjust center for table translation along the Superior-Inferior axis: $\text{center} = \text{center\_ras} + [0, 0, \text{table\_si}]^T$ [S1, unofficial].
3. Define local source position at distance `sid` along the negative Y-axis: $\text{source\_local} = [0, -\text{sid}, 0]^T$ [S1, unofficial].
4. Construct lateral rotation matrix $R_{lat}$ around the Z-axis: $R_{lat} = \begin{bmatrix} \cos(\text{lat\_rad}) & -\sin(\text{lat\_rad}) & 0 \\ \sin(\text{lat\_rad}) & \cos(\text{lat\_rad}) & 0 \\ 0 & 0 & 1 \end{bmatrix}$ [S1, unofficial].
5. Construct AP rotation matrix $R_{ap}$ around the X-axis: $R_{ap} = \begin{bmatrix} 1 & 0 & 0 \\ 0 & \cos(\text{ap\_rad}) & -\sin(\text{ap\_rad}) \\ 0 & \sin(\text{ap\_rad}) & \cos(\text{ap\_rad}) \end{bmatrix}$ [S1, unofficial].
6. Combine rotations and apply to source: $R_{total} = R_{ap} R_{lat}$, $\text{source\_rotated} = R_{total} \text{source\_local}$ [S1, unofficial].
7. Calculate camera position: $\text{cam\_pos} = \text{source\_rotated} + \text{center}$ [S1, unofficial].
8. Set look-at point: $\text{look\_at} = \text{center}$ [S1, unofficial].
9. Define local up vector and rotate it: $\text{up\_local} = [0, 0, 1]^T$, $\text{look\_up} = R_{total} \text{up\_local}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def c_arm_to_camera_params(sid, ap_angle, lat_angle, center_ras, table_si=0.0):
    ap_rad = np.deg2rad(ap_angle)
    lat_rad = np.deg2rad(lat_angle)
    center = np.array(center_ras) + np.array([0.0, 0.0, table_si])
    source_local = np.array([0.0, -sid, 0.0])
    cos_lat, sin_lat = np.cos(lat_rad), np.sin(lat_rad)
    R_lat = np.array([[cos_lat, -sin_lat, 0.0],
                      [sin_lat,  cos_lat, 0.0],
                      [0.0,      0.0,     1.0]])
    cos_ap, sin_ap = np.cos(ap_rad), np.sin(ap_rad)
    R_ap = np.array([[1.0, 0.0,         0.0],
                     [0.0, cos_ap, -sin_ap],
                     [0.0, sin_ap,  cos_ap]])
    R_total = R_ap @ R_lat
    source_rotated = R_total @ source_local
    cam_pos = source_rotated + center
    look_at = center
    up_local = np.array([0.0, 0.0, 1.0])
    look_up = R_total @ up_local
    return cam_pos, look_at, look_up
```

**Parameters:**
*   `sid`: Source-to-Image Distance in mm. Typical values: Not given in the sources.
*   `ap_angle`: Anteroposterior angle in degrees. 0° = AP view. Positive = caudal (source tilts inferiorly). Negative = cranial (source tilts superiorly) [S1, unofficial].
*   `lat_angle`: Lateral angle in degrees. 0° = AP view. Positive = RAO (source moves to patient's right). Negative = LAO (source moves to patient's left) [S1, unofficial].
*   `center_ras`: Center point of the CT scan in RAS coordinates (mm). RAS = Right, Anterior, Superior [S1, unofficial].
*   `table_si`: Table translation in superior-inferior direction in mm (default: 0.0). Positive = table moves superior (head up). Negative = table moves inferior (head down) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Degenerate Input: If `sid` is 0, the source coincides with the center, leading to an undefined view direction. Guard against `sid <= 0` [G].

## Subtopics

### [Random C-Arm View Sampling](2596-random-c-arm-view-sampling.md)
A method to generate random camera-to-world view matrices for C-arm X-ray projections by uniformly sampling geometric parameters (Source-to-Image Distance, AP angle, lateral angle, table SI translation) and converting them to camera coordinates.

## Operating characteristics
*   Time and memory complexity are $O(1)$, involving a fixed number of arithmetic operations and matrix multiplications [G].
*   The method assumes ideal C-arm geometry with perfect rotations around fixed axes [G].
*   It is suitable for generating synthetic X-ray projections when the C-arm geometry is known or simulated [G].
*   It does not model X-ray physics effects such as scatter, beam hardening, or detector distortion [G].

## Background
This method originates from the `flashdrr` library, which provides differentiable volume rendering for X-ray imaging [S1, unofficial]. It implements the standard geometric model of C-arm X-ray systems, where the X-ray source rotates around the isocenter (center of the CT scan) [S1, unofficial].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
