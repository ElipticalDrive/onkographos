# C-Arm to Camera Parameters Conversion


## Overview
This method converts C-arm geometric parameters (Source-to-Image Distance, AP angle, lateral angle, center point, and table translation) into a camera pose defined by position, look-at point, and up vector in RAS coordinates [S1, unofficial].

## How it works
**Inputs:**
*   `sid`: Source-to-Image Distance in mm [S1, unofficial].
*   `ap_angle`: Anteroposterior angle in degrees [S1, unofficial].
*   `lat_angle`: Lateral angle in degrees [S1, unofficial].
*   `center_ras`: 3D vector $(x, y, z)$ representing the center of the volume in RAS coordinates (mm) [S1, unofficial].
*   `table_si`: Table translation in the Superior-Inferior direction in mm [S1, unofficial].

**Outputs:**
*   `cam_pos`: Camera (X-ray source) position in RAS coordinates [S1, unofficial].
*   `look_at`: Look-at point (isocenter) in RAS coordinates [S1, unofficial].
*   `look_up`: Up vector for the camera orientation in RAS coordinates [S1, unofficial].

**Data Structures:**
*   Vectors are 3-element arrays or tensors [G].
*   Rotation matrices are $3 \times 3$ matrices [G].

**Parameters and Conventions:**
*   Coordinate System: RAS (Right, Anterior, Superior) [G].
*   `ap_angle`: $0^\circ$ is AP view. Positive values indicate caudal tilt (source tilts inferiorly). Negative values indicate cranial tilt (source tilts superiorly) [S1, unofficial].
*   `lat_angle`: $0^\circ$ is AP view. Positive values indicate RAO (source moves to patient's right). Negative values indicate LAO (source moves to patient's left) [S1, unofficial].
*   `table_si`: Positive values move the table superiorly (head up). Negative values move the table inferiorly (head down) [S1, unofficial].

**Algorithm Steps:**

1.  **Angle Conversion:** Convert angles from degrees to radians [S1, unofficial].
    $$ \text{ap\_rad} = \text{deg2rad}(\text{ap\_angle}) $$
    $$ \text{lat\_rad} = \text{deg2rad}(\text{lat\_angle}) $$

2.  **Center Calculation:** Determine the isocenter by applying the table translation to the volume center along the Superior-Inferior axis (Z-axis in RAS) [S1, unofficial].
    $$ \text{center} = \text{center\_ras} + [0, 0, \text{table\_si}] $$

3.  **Local Source Position:** Define the initial source position at a distance `sid` along the negative Y-axis [S1, unofficial].
    $$ \text{source\_local} = [0, -\text{sid}, 0] $$

4.  **Rotation Matrices:** Construct rotation matrices for the lateral and AP rotations [S1, unofficial].
    *   Lateral rotation ($R_{\text{lat}}$) around the Z-axis:
        $$ R_{\text{lat}} = \begin{bmatrix} \cos(\text{lat\_rad}) & -\sin(\text{lat\_rad}) & 0 \\ \sin(\text{lat\_rad}) & \cos(\text{lat\_rad}) & 0 \\ 0 & 0 & 1 \end{bmatrix} $$
    *   AP rotation ($R_{\text{ap}}$) around the X-axis:
        $$ R_{\text{ap}} = \begin{bmatrix} 1 & 0 & 0 \\ 0 & \cos(\text{ap\_rad}) & -\sin(\text{ap\_rad}) \\ 0 & \sin(\text{ap\_rad}) & \cos(\text{ap\_rad}) \end{bmatrix} $$

5.  **Combined Rotation:** Multiply the rotation matrices (AP applied after Lateral) [S1, unofficial].
    $$ R_{\text{total}} = R_{\text{ap}} \times R_{\text{lat}} $$

6.  **Rotated Source:** Apply the combined rotation to the local source position [S1, unofficial].
    $$ \text{source\_rotated} = R_{\text{total}} \times \text{source\_local} $$

7.  **Camera Position:** Translate the rotated source vector by the center point [S1, unofficial].
    $$ \text{cam\_pos} = \text{source\_rotated} + \text{center} $$

8.  **Look-At Point:** Set the look-at point to the calculated center [S1, unofficial].
    $$ \text{look\_at} = \text{center} $$

9.  **Up Vector:** Define a local up vector along the Z-axis and rotate it by the combined rotation matrix [S1, unofficial].
    $$ \text{up\_local} = [0, 0, 1] $$
    $$ \text{look\_up} = R_{\text{total}} \times \text{up\_local} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def c_arm_to_camera_params(sid, ap_angle, lat_angle, center_ras, table_si=0.0):
    """
    Convert C-arm geometric parameters to camera pose in RAS coordinates.
    
    Args:
        sid (float): Source-to-Image Distance in mm.
        ap_angle (float): Anteroposterior angle in degrees.
        lat_angle (float): Lateral angle in degrees.
        center_ras (array-like): Center point of the CT scan in RAS coordinates (mm).
        table_si (float): Table translation in mm (default 0.0).
        
    Returns:
        tuple: (cam_pos, look_at, look_up) as numpy arrays.
    """
    # 1. Convert angles to radians
    ap_rad = np.deg2rad(ap_angle)
    lat_rad = np.deg2rad(lat_angle)
    
    # 2. Calculate center point with table translation
    center = np.array(center_ras) + np.array([0.0, 0.0, table_si])
    
    # 3. Define initial source position at SID distance along negative Y axis
    source_local = np.array([0.0, -sid, 0.0])
    
    # 4. Construct lateral rotation matrix R_lat around Z-axis
    cos_lat = np.cos(lat_rad)
    sin_lat = np.sin(lat_rad)
    R_lat = np.array([
        [cos_lat, -sin_lat, 0.0],
        [sin_lat,  cos_lat, 0.0],
        [0.0,      0.0,     1.0]
    ])
    
    # 5. Construct AP rotation matrix R_ap around X-axis
    cos_ap = np.cos(ap_rad)
    sin_ap = np.sin(ap_rad)
    R_ap = np.array([
        [1.0, 0.0,         0.0],
        [0.0,  cos_ap, -sin_ap],
        [0.0,  sin_ap,  cos_ap]
    ])
    
    # 6. Combine rotations: R_total = R_ap @ R_lat
    R_total = R_ap @ R_lat
    
    # 7. Rotate source
    source_rotated = R_total @ source_local
    
    # 8. Calculate camera position
    cam_pos = source_rotated + center
    
    # 9. Set look-at point to center
    look_at = center
    
    # 10. Define local up vector and rotate it
    up_local = np.array([0.0, 0.0, 1.0])
    look_up = R_total @ up_local
    
    return cam_pos, look_at, look_up
```

## Subtopics

### [Random C-Arm View Sampling](2596-random-c-arm-view-sampling.md)
Random C-Arm View Sampling generates random camera-to-world view matrices for C-arm X-ray projections by uniformly sampling geometric parameters (SID, AP angle, lateral angle, table SI translation) and converting them to camera coordinates.

## Operating characteristics
Time and memory complexity: Not given in the sources [S1, unofficial].
Measured performance, scaling, strengths, limitations, and usage conditions: Not given in the sources [S1, unofficial].

## Background
This method is implemented in the `flashdrr` library for differentiable digital radiography rendering [S1, unofficial]. It is used in medical imaging research to generate synthetic X-ray projections (DRRs) from CT volumes [S1, unofficial]. The RAS coordinate system is standard in medical imaging formats such as NIfTI and DICOM [G].

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
