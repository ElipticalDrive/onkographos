# Spot Size Reduction Methods
Also called: Addressable Area Reduction
## Overview
This method defines four optical and mechanical strategies to reduce the laser spot diameter and the corresponding addressable area on a rotating helical volumetric display [S1, unofficial]. The original system produced a 4 mm spot diameter, which these methods aim to reduce to approximately 1.7–2.0 mm [S1, unofficial].
## How it works
### Inputs and Outputs
- **Input**: The existing scanner-helix optical train, characterized by a throw distance, output lens focal lengths, and scanner resolution (256 × 256 deflectors) [S1, unofficial].
- **Output**: A reduced spot diameter (target ~1.7–1.8 mm) and a remapped addressable area (e.g., 13 × 13 or 24 × 24) within the 4095 × 4095 digital address space [S1, unofficial].

### Data Structures
The hardware state is defined by the following parameters:
- `throw_distance`: Distance from scanner to helix surface (mm) [S1, unofficial].
- `lens_focal_length`: Focal length of projection/output lenses (mm) [S1, unofficial].
- `scanner_resolution`: Number of addressable points per axis (integer) [S1, unofficial].
- `beam_size`: Diameter of the laser beam within the deflectors (mm) [S1, unofficial].
- `spot_diameter`: Resulting spot size at the helix surface (mm) [S1, unofficial].
- `addressable_area`: The physical or logical grid size occupied by the beam (e.g., 13 × 13) [S1, unofficial].

### Steps
The implementation involves selecting one of four mutually exclusive or combinable hardware modifications [S1, unofficial].

1. **Method 1: Reduce Throw Distance**
   - Reduce the physical distance from the scanner to the helix by approximately half [S1, unofficial].
   - Target throw distance: 1700 mm [S1, unofficial].
   - Result: Spot diameter of 1.7 mm; addressable area reduces to 13+ × 13+ [S1, unofficial].

2. **Method 2: Insert Projection Lens**
   - Insert a single projection lens in front of the scanner [S1, unofficial].
   - Select lens focal length from {500, 1000, 1500} mm [S1, unofficial].
   - Result: Best spot size is 1.8 mm [S1, unofficial]. Requires an 8-inch lens [S1, unofficial]. Introduces optical aberrations [S1, unofficial].

3. **Method 3: Change Output Lenses**
   - Modify the output optics to decrease angular magnification [S1, unofficial].
   - Replace the last output lens with a focal length of approximately 110 mm (original was 60 mm) [S1, unofficial].
   - Remap voxel images to fill the 4095 × 4095 address space [S1, unofficial].
   - Result: Spot size of 1.7 mm [S1, unofficial].

4. **Method 4: Increase Scanner Resolution**
   - Increase the beam size in the deflectors to 6.4 mm [S1, unofficial].
   - Increase scanner resolution from 256 × 256 to 512 × 512 [S1, unofficial].
   - Result: Spot size of 1.8 mm at 24 × 24 area [S1, unofficial]. Sub-millimeter spots possible if optimized to 13 × 13 area [S1, unofficial].
   - Penalty: Total voxel count reduces from 40,000 to 20,000 [S1, unofficial].

### Formulas
No explicit mathematical formulas are provided in the sources [S1, unofficial].

### Parameters
- **Throw Distance**: ~1700 mm [S1, unofficial].
- **Projection Lens Focal Lengths**: 500, 1000, or 1500 mm [S1, unofficial].
- **Output Lens Focal Length**: 60 mm (original) replaced by ~110 mm [S1, unofficial].
- **Deflector Beam Size**: 6.4 mm [S1, unofficial].

### Edge Cases and Pitfalls
- **Aberrations**: Method 2 introduces optical aberrations that are difficult to remove [S1, unofficial].
- **Voxel Count Reduction**: Method 4 halves the total number of voxels (40,000 to 20,000), reducing image density [S1, unofficial].
- **Misalignment**: Slight misalignment of scanner optics (e.g., shifting the 80 mm collimating lens by ±1 mm) has negligible effect on spot size if the output lens is adjusted to achieve the minimum spot size [S1, unofficial].
- **Lens Size**: Method 2 requires an 8-inch lens, which is physically large [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_spot_reduction_methods():
    """
    Evaluates four methods for reducing spot size on a helical volumetric display.
    Returns the recommended method based on source evaluation.
    """
    results = []

    # Method 1: Reduce throw distance
    method_1 = {
        "name": "Reduce Throw Distance",
        "throw_distance_mm": 1700,
        "spot_diameter_mm": 1.7,
        "addressable_area": "13+ x 13+",
        "pros": ["Direct", "Good spot size"],
        "cons": []
    }
    results.append(method_1)

    # Method 2: Insert projection lens
    method_2 = {
        "name": "Insert Projection Lens",
        "lens_focal_lengths_mm": [500, 1000, 1500],
        "spot_diameter_mm": 1.8,
        "lens_size_inches": 8,
        "pros": [],
        "cons": ["Aberrations", "Large lens required", "Worse spot size than Method 1"]
    }
    results.append(method_2)

    # Method 3: Change output lenses
    method_3 = {
        "name": "Change Output Lenses",
        "old_output_lens_mm": 60,
        "new_output_lens_mm": 110,
        "spot_diameter_mm": 1.7,
        "address_space": "4095 x 4095",
        "pros": ["Cost-effective", "Straightforward", "Good spot size"],
        "cons": []
    }
    results.append(method_3)

    # Method 4: Increase scanner resolution
    method_4 = {
        "name": "Increase Scanner Resolution",
        "old_resolution": "256 x 256",
        "new_resolution": "512 x 512",
        "beam_size_mm": 6.4,
        "spot_diameter_mm": 1.8,
        "addressable_area": "24 x 24",
        "total_voxels_old": 40000,
        "total_voxels_new": 20000,
        "pros": ["Higher resolution"],
        "cons": ["Halved voxel count", "Larger area"]
    }
    results.append(method_4)

    # Selection logic based on source evaluation
    # Method 3 is selected as most cost-effective and straightforward
    # Method 2 rejected due to aberrations and lens size
    # Method 4 penalized by halved voxel count
    
    recommended_method = method_3
    return recommended_method
```
## Subtopics

### Optical Alignment
Adjusting lens positions to compensate for misalignment without degrading spot size [S1, unofficial].

### Gaussian Beam Analysis
Using Gaussian beam traces for additional optical analysis [S1, unofficial].

## Operating characteristics
- **Performance**: Method 3 achieves a 1.7 mm spot size, comparable to Method 1, but is preferred for its cost-effectiveness and simplicity [S1, unofficial].
- **Trade-offs**: Method 1 requires physical modification of the throw distance [S1, unofficial]. Method 2 introduces aberrations and requires large optics [S1, unofficial]. Method 4 reduces the total voxel count by 50% [S1, unofficial].
- **Robustness**: The system is tolerant to slight misalignments (±1 mm shift in collimating lens) if the output lens is re-focused [S1, unofficial].

## Background
This method was developed for the Improved Second-Generation 3-D Volumetric Display System [S1, unofficial]. It addresses limitations in the initial design where the spot size was too large (4 mm) and the addressable area was insufficiently utilized [S1, unofficial]. The goal was to reduce the spot size to approximately 2 mm or less while maintaining a high voxel count [S1, unofficial].

## Sources
[S1] DTIC ADA358303: Improved Second-Generation 3-D Volumetric Display System. Revision 2, https://archive.org/details/DTIC_ADA358303, fetched 2026-10-26 (unofficial)
