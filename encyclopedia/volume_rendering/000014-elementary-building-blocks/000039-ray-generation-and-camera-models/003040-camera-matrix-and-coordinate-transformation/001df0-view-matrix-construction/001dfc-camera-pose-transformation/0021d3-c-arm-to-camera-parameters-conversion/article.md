# C-Arm to Camera Parameters Conversion

Also called: C-Arm View Matrix Construction, C-Arm Pose Generation

## Overview
This method converts C-arm X-ray geometric parameters (Source-to-Image Distance, AP angle, lateral angle, center point, and table translation) into a camera pose defined by position, look-at point, and up vector in RAS (Right-Anterior-Superior) coordinates. It solves the problem of generating physically plausible camera matrices for digital tomosynthesis or projection rendering from high-level interventional radiology acquisition settings. It differs from generic [Rigid Body Transformation (SE3)](11475-rigid-body-transformation-se3.md) by explicitly modeling the kinematic constraints of a C-arm gantry (rotation around specific axes relative to the patient table) rather than applying arbitrary SE3 transformations.

## How it works
**Inputs:**
*   `sid`: Source-to-Image Distance in mm.
*   `ap_angle`: Anteroposterior angle in degrees.
*   `lat_angle`: Lateral angle in degrees.
*   `center_ras`: 3D vector $(x, y, z)$ representing the center of the volume in RAS coordinates (mm).
*   `table_si`: Table translation in the Superior-Inferior direction in mm.

**Outputs:**
*   `cam_pos`: Camera (X-ray source) position in RAS coordinates.
*   `look_at`: Look-at point (isocenter) in RAS coordinates.
*   `look_up`: Up vector for the camera orientation in RAS coordinates.

**Data Structures:**
*   Vectors are 3-element arrays/tensors.
*   Rotation matrices are $3 \times 3$ matrices.

**Parameters and Conventions:**
*   Coordinate System: RAS (Right, Anterior, Superior).
*   `ap_angle`: $0^\circ$ is AP view. Positive values indicate caudal tilt (source tilts inferiorly). Negative values indicate cranial tilt (source tilts superiorly) [S1, unofficial].
*   `lat_angle`: $0^\circ$ is AP view. Positive values indicate RAO (Right Anterior Oblique, source moves to patient's right). Negative values indicate LAO (Left Anterior Oblique, source moves to patient's left) [S1, unofficial].
*   `table_si`: Positive values move the table superiorly (head up). Negative values move the table inferiorly (head down) [S1, unofficial].

**Algorithm Steps:**

1.  **Angle Conversion:** Convert angles from degrees to radians.
    $$ \text{ap\_rad} = \text{deg2rad}(\text{ap\_angle}) $$
    $$ \text{lat\_rad} = \text{deg2rad}(\text{lat\_angle}) $$

2.  **Center Calculation:** Determine the isocenter (look-at point) by applying the table translation to the volume center. The translation occurs along the Superior-Inferior axis (Z-axis in RAS).
    $$ \text{center} = \text{center\_ras} + [0, 0, \text{table\_si}] $$

3.  **Local Source Position:** Define the initial source position in a local coordinate system where the source is located at a distance `sid` along the negative Y-axis (assuming the C-arm starts in a standard AP position with source below or anterior depending on convention, here defined as negative Y).
    $$ \text{source\_local} = [0, -\text{sid}, 0] $$

4.  **Rotation Matrices:** Construct rotation matrices for the lateral and AP rotations.
    *   Lateral rotation ($R_{\text{lat}}$) around the Z-axis:
        $$ R_{\text{lat}} = \begin{bmatrix} \cos(\text{lat\_rad}) & -\sin(\text{lat\_rad}) & 0 \\ \sin(\text{lat\_rad}) & \cos(\text{lat\_rad}) & 0 \\ 0 & 0 & 1 \end{bmatrix} $$
    *   AP rotation ($R_{\text{ap}}$) around the X-axis:
        $$ R_{\text{ap}} = \begin{bmatrix} 1 & 0 & 0 \\ 0 & \cos(\text{ap\_rad}) & -\sin(\text{ap\_rad}) \\ 0 & \sin(\text{ap\_rad}) & \cos(\text{ap\_rad}) \end{bmatrix} $$

5.  **Combined Rotation:** Multiply the rotation matrices. The order specified is AP applied after Lateral (or Lateral applied first in local space, then AP).
    $$ R_{\text{total}} = R_{\text{ap}} \times R_{\text{lat}} $$

6.  **Rotated Source:** Apply the combined rotation to the local source position.
    $$ \text{source\_rotated} = R_{\text{total}} \times \text{source\_local} $$

7.  **Camera Position:** Translate the rotated source vector by the center point.
    $$ \text{cam\_pos} = \text{source\_rotated} + \text{center} $$

8.  **Look-At Point:** The look-at point is the calculated center.
    $$ \text{look\_at} = \text{center} $$

9.  **Up Vector:** Define a local up vector (typically Z-axis in RAS) and rotate it by the same combined rotation matrix to maintain consistent camera orientation.
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
    # Table moves in SI direction (Z-axis in RAS)
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

**Edge Cases and Pitfalls:**
*   **Precision:** Ensure floating-point precision is sufficient for large SID values (e.g., >1000 mm) to avoid jitter in `cam_pos`.
*   **Coordinate System Consistency:** The method assumes RAS coordinates for `center_ras` and outputs. If the rendering engine uses a different coordinate system (e.g., LPS or NIFTI IJK), `center_ras` must be converted to that system before input, or the output vectors must be transformed.
*   **Degenerate Input:** If `sid` is 0, the source coincides with the center, leading to undefined viewing direction. If `ap_angle` and `lat_angle` are both 0, the source is at $[0, -sid, 0]$ relative to center, which is a valid standard AP view.

## Subtopics

### [Random C-Arm View Sampling](2596-random-c-arm-view-sampling.md)
Random C-Arm View Sampling generates a set of random camera-to-world view matrices for C-arm X-ray projections by uniformly sampling geometric parameters (Source-to-Image Distance, AP angle, lateral angle, and table SI translation) and converting them to camera coordinates. It differs from deterministic acquisition strategies like [Low Discrepancy View Sampling](8328-low-discrepancy-view-sampling.md) by producing independent, non-ordered views, which is suitable for data augmentation or stochastic rendering rather than structured tomographic reconstruction.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per view. The calculation involves a fixed number of trigonometric operations, matrix multiplications, and vector additions.
*   **Memory Complexity:** $O(1)$. Only a few temporary vectors and matrices are allocated.
*   **Strengths:** Physically interpretable parameters; direct mapping from clinical acquisition settings; computationally inexpensive.
*   **Limitations:** Assumes a specific C-arm kinematic model (rotations around fixed axes relative to the patient table); does not account for detector tilt or non-rigid deformations.
*   **When to use:** When generating synthetic projections from CT volumes for training or validation, and the desired view is specified by clinical angles (AP/Lateral) rather than arbitrary camera poses.
*   **When not to use:** When the camera pose is already known in world coordinates, or when the C-arm geometry deviates significantly from the assumed kinematic model (e.g., complex multi-axis rotations not decomposable into simple AP/Lat angles).

## Background
This method is derived from the `flashdrr` library, which provides differentiable digital radiography rendering [S1, unofficial]. It is commonly used in medical imaging research for generating synthetic X-ray projections (DRRs) from CT volumes to simulate C-arm fluoroscopy or to train deep learning models for registration and segmentation. The RAS coordinate system is standard in medical imaging (NIfTI, DICOM) for representing patient anatomy.

## Sources
[S1] pcarnah/flashdrr, https://github.com/pcarnah/flashdrr, fetched 2026-10-26 (unofficial)
