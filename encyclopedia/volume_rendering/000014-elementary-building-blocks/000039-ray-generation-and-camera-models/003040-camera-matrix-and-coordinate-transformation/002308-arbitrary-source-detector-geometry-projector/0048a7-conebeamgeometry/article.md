# ConeBeamGeometry

Also called: C-arm CBCT Geometry & Sampling Setup, CBCT Geometry, Cone Beam CT Acquisition

## Overview
ConeBeamGeometry defines the physical and geometric parameters for a cone-beam computed tomography (CBCT) acquisition, specifically for systems where the X-ray source and detector move along a trajectory (typically circular) relative to the object. It serves as the foundational data structure for the [Arbitrary Source-Detector Geometry Projector](2975-arbitrary-source-detector-geometry-projector.md), providing the fixed distances and detector dimensions required to map 3D world coordinates to 2D detector pixels. Unlike standard parallel-beam assumptions, this model accounts for the divergent nature of the X-ray beam, requiring precise magnification factors to relate voxel sizes to detector bin sizes.

## How it works
### Inputs and Outputs
*   **Inputs:** Physical scanner specifications (distances, pixel size), acquisition trajectory (angles), and detector array dimensions.
*   **Outputs:** A geometry object containing attributes used to compute projection rays and backprojector weights.

### Data Structures
The geometry is represented as a class or struct with the following attributes [S2, unofficial]:

| Attribute | Type | Description | Default Value |
| :--- | :--- | :--- | :--- |
| `DSO` | float | Source-to-Isocenter Distance (SID) | 500 (mm) |
| `DSD` | float | Source-to-Detector Distance (SDD) | 800 (mm) |
| `det_pixels` | int | Number of detector pixels per side (assumed square) | 256 |
| `pixel_size` | float | Physical size of a single detector pixel (mm) | 1.0 |
| `angles` | array | Array of rotation angles (radians) from 0 to $2\pi$ | 180 angles |

### Formulas and Parameters
The geometry relies on two primary magnification factors to relate the physical world to the digital sampling grid [S1]:

1.  **Projection Magnification ($\alpha$):** Relates the source-detector distance to the detector bin size.
    $$ \alpha = \frac{\text{DSD}}{\text{pixel\_size}} $$
2.  **Isosampling Magnification ($t_z$):** Relates the source-isocenter distance to the voxel size at the isocenter.
    $$ t_z = \frac{\text{DSO}}{t_z^{\text{voxel}}} $$
    Where $t_z^{\text{voxel}}$ is the physical voxel size at the center of rotation.

**Example Parameters (C-arm System) [S1]:**
*   Focal spot to detector distance (DSD): 1295 mm
*   Detector bin size: 0.2 mm
*   $\alpha = 1295 / 0.2 = 6475$
*   Focal spot to rotation center (DSO): 820 mm
*   Iso-sampling voxel size: 0.127 mm
*   $t_z = 820 / 0.127 = 6456$
*   Field of View (FOV): 65 mm (for a $512 \times 512$ detector)

### Pseudocode
The following Python-like pseudocode initializes the geometry object and computes the necessary magnification factors. It assumes standard vector operations are available.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class ConeBeamGeometry:
    def __init__(self, DSO=500.0, DSD=800.0, det_pixels=256, pixel_size=1.0, n_angles=180):
        """
        Initialize Cone Beam CT Geometry.
        
        Args:
            DSO: Source-to-Isocenter Distance in mm.
            DSD: Source-to-Detector Distance in mm.
            det_pixels: Number of pixels per side of the square detector.
            pixel_size: Physical size of one detector pixel in mm.
            n_angles: Number of projection angles.
        """
        self.DSO = DSO
        self.DSD = DSD
        self.det_pixels = det_pixels
        self.pixel_size = pixel_size
        self.n_angles = n_angles
        
        # Compute projection magnification factor
        # alpha relates detector bin size to source distance
        self.alpha = DSD / pixel_size
        
        # Generate angles from 0 to 2*pi
        # Note: Specific trajectory (e.g., circular orbit) determines exact angles
        self.angles = np.linspace(0, 2 * np.pi, n_angles, endpoint=False)

    def get_magnification_factor(self, voxel_size_at_isocenter):
        """
        Calculate the isosampling magnification factor tz.
        
        Args:
            voxel_size_at_isocenter: Physical voxel size at the center of rotation in mm.
            
        Returns:
            tz: Magnification factor relating DSO to voxel size.
        """
        if voxel_size_at_isocenter <= 0:
            raise ValueError("Voxel size must be positive.")
        return self.DSO / voxel_size_at_isocenter

    def compute_fov(self):
        """
        Compute the Field of View (FOV) at the isocenter.
        
        Returns:
            fov: Field of View in mm.
        """
        # FOV is determined by detector size and magnification
        # Detector physical width = det_pixels * pixel_size
        # However, FOV at isocenter is scaled by DSO/DSD
        detector_width_mm = self.det_pixels * self.pixel_size
        fov = detector_width_mm * (self.DSO / self.DSD)
        return fov
```

### Edge Cases and Pitfalls
*   **Sampling Condition Violation:** In real-data conditions, the ideal sampling condition (often referred to as Equation 6 in literature [S1]) is not rigorously satisfied. This occurs when the voxel size does not perfectly align with the system's magnification, requiring fallback to interpolation handling (e.g., B-spline interpolation) during backprojection [S1].
*   **Voxel Size Mismatch:** Using a larger voxel size (e.g., 0.508 mm vs 0.127 mm) to increase FOV degrades resolution. The degradation is induced by the interpolation required to map the large voxels to the detector geometry, not by the physical system itself [S1].
*   **Precision:** Ensure floating-point precision is maintained when calculating magnification factors, especially when $\alpha$ and $t_z$ differ significantly (e.g., 6475 vs 6456 in the C-arm example [S1]).

## Subtopics

### [t-beam projectors](3067-t-beam-projectors.md)
Uses a fan-beam approximation; suitable for 2D slices or when computational cost of full cone-beam is prohibitive.

### [TIGRE Geometry Setup](5605-tigre-geometry-setup.md)
A specific implementation of cone-beam geometry for the TIGRE toolbox; choose when using TIGRE's optimized FDK or iterative solvers.

### [Projection Geometry to Vector Conversion](13361-projection-geometry-to-vector-conversion.md)
Converts geometric parameters into vectorized ray paths; choose when implementing custom GPU kernels that require flat arrays of ray origins and directions.

## Operating characteristics
*   **Time Complexity:** The geometry setup itself is $O(1)$ or $O(N_{angles})$ for angle generation. The computational cost lies in the subsequent projection/backprojection steps, which are $O(N_{voxels} \cdot N_{angles})$ for direct methods.
*   **Memory Complexity:** $O(N_{angles} \cdot N_{det\_pixels}^2)$ to store projection data. The geometry object itself is $O(1)$.
*   **Strengths:** Accurately models divergent beam geometry, essential for CBCT reconstruction. Supports variable FOV by adjusting voxel size relative to magnification factors.
*   **Limitations:** Requires careful interpolation when voxel sizes do not match the ideal sampling condition [S1]. Computationally expensive for large volumes and high-resolution detectors.
*   **When to use:** When reconstructing data from a CBCT scanner (e.g., C-arm systems) where the source and detector move in a circular or helical trajectory.
*   **When not to use:** For parallel-beam CT or when using simplified fan-beam approximations where cone-beam artifacts are negligible.

## Background
Cone-beam CT geometry originated with the development of C-arm X-ray systems for interventional radiology and dental imaging. The specific parameters cited (1295 mm SDD, 820 mm SID) are derived from experimental setups on GE Healthcare C-arm systems [S1]. The geometry is a critical component in algorithms like the Feldkamp-Davis-Kress (FDK) algorithm, which approximates the exact cone-beam reconstruction. The distinction between projection magnification ($\alpha$) and isosampling magnification ($t_z$) highlights the challenge of balancing field of view and resolution in digital radiography [S1].

## Sources
[S1] Magnification-driven B-spline Interpolation for Cone Beam Projection and Backprojection, https://doi.org/10.1002/mp.15179, fetched 2026-10-26
[S2] ayush-chauhan-1202/physics-aware-ct-reconstruction, https://github.com/ayush-chauhan-1202/physics-aware-ct-reconstruction, fetched 2026-10-27 (unofficial)
