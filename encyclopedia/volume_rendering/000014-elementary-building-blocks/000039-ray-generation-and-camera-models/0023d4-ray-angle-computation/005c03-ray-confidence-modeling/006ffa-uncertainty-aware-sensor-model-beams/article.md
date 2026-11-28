# Uncertainty-aware sensor model (beams)
Also called: Continuous occupancy measurement model, continuous sensor model, ours (beams)

## Overview
The uncertainty-aware sensor model (beams) is a volumetric measurement integration method that models sensor observations as soft beams with angular and range uncertainty, rather than infinitely thin rays [S1]. It solves the problem of aliasing artifacts and poor reconstruction of thin structures (e.g., branches, cables) that arise when ray density is low relative to voxel resolution or when discrete/TSDF models are used [S1]. It is distinguished from its sibling [Uncertainty-aware sensor model (rays)](17099-uncertainty-aware-sensor-model-rays.md) by explicitly incorporating angular uncertainty, which improves surface and thin-object fidelity at the cost of increased memory and runtime [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A hierarchical map representation (specifically using Haar wavelets for compression and multi-resolution access) [S1]. Sensor measurements consisting of range values and associated angular uncertainties derived from sensor calibration and pose estimation [S1].
- **Output:** Updated volumetric map with continuous occupancy values, reflecting sub-voxel accuracy and reduced entropy in free space [S1].

**Data Structures:**
The method relies on a Haar wavelet-based representation of the volume. This structure allows for efficient coarse-to-fine integration and compression [S1]. The specific bit-packing or field layout of the wavelet coefficients is not detailed in the sources, but the structure supports hierarchical access to skip unnecessary work during updates [S1].

**Algorithm Steps:**
1.  **Beam Definition:** For each sensor measurement, define a "soft beam" volume instead of a line. The beam's geometry is determined by the sensor's range measurement and its angular uncertainty cone [S1].
2.  **Uncertainty Incorporation:** Map the angular uncertainty (from calibration/pose) and range uncertainty into the beam's cross-sectional width or probability distribution along the ray path [S1].
3.  **Hierarchical Integration:** Perform a specialized measurement integration algorithm that exploits the hierarchical nature of the map [S1].
    -   Start at the coarsest resolution level [S1].
    -   Determine if the beam intersects the current voxel or region [S1].
    -   If the region is fully free or fully occupied at the current resolution, apply the update and skip children [S1].
    -   If the region is partially occupied or the beam passes through, recurse to finer levels [S1].
    -   This guarantees identical results to a naive integrator applying updates at the highest resolution throughout the field of view, but with significant speedup [S1].
4.  **Occupancy Update:** Update the wavelet coefficients (or underlying voxel values) based on the continuous occupancy model, accounting for the probability of occupancy within the soft beam volume [S1].

**Formulas and Parameters:**
The sources do not provide explicit mathematical formulas for the continuous occupancy measurement model or the beam geometry. They reference inspiration from work [16] (not provided in the source list) for the continuous model [S1].
-   **Angular Uncertainty:** Derived from sensor calibration and pose estimation [S1].
-   **Range Uncertainty:** Incorporated into the model [S1].

**Edge Cases and Pitfalls:**
-   **Low Ray Density:** The model specifically addresses aliasing artifacts that occur in regions where ray density is low compared to voxel resolution [S1].
-   **Thin Objects:** Discrete models fail here; this model succeeds by allowing sub-voxel accuracy [S1].
-   **High Entropy:** Traditional thin-ray models create high-entropy free space regions that are hard to compress; this model mitigates that [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def integrate_measurement(map, measurement):
    """
    Integrates a single sensor measurement into the wavelet-based map.
    measurement: object containing range, angular_uncertainty, and pose info.
    map: Hierarchical Haar wavelet map structure.
    """
    # Define the soft beam geometry based on measurement uncertainty
    # Note: Exact geometric formulation of the "soft beam" is not provided in sources [S1]
    beam = create_soft_beam(measurement.range, measurement.angular_uncertainty)
    
    # Start hierarchical integration from the root (coarsest level)
    integrate_recursive(map.root, beam)

def integrate_recursive(voxel, beam):
    """
    Recursively updates the map, skipping fully determined regions.
    """
    # Check if the beam intersects this voxel's spatial extent
    if not intersects(voxel.bounds, beam):
        return

    # If the voxel is at the finest resolution, apply the continuous occupancy update
    if voxel.is_leaf():
        # Update occupancy based on the beam's probability distribution
        # Note: Specific update rule for continuous occupancy is not detailed in sources [S1]
        voxel.update_occupancy(beam)
        return

    # If the voxel is fully free or fully occupied at this resolution, 
    # we can potentially skip or apply a coarse update.
    # The sources state the algorithm guarantees identical results to naive integration
    # while skipping unnecessary work [S1].
    
    # Recurse to children
    for child in voxel.children:
        integrate_recursive(child, beam)
```

## Operating characteristics
-   **Memory:** Produces larger maps than the ray-based variant, with an increase of 30% to 70% depending on resolution [S1]. This is due to the need to update more voxels and store more fine details/partial occupancy values [S1].
-   **Runtime:** Approximately 50% increase in runtime compared to the ray-based variant [S1]. The hierarchical integration algorithm speeds up the process, but the uncertainty-aware model is inherently more costly [S1].
-   **Quality:** Significantly improves the reconstruction of surfaces and thin objects (branches, cables, fences) compared to discrete occupancy, TSDFs, and the ray-based uncertainty model [S1]. Performs on par with TSDF-based methods (like Voxblox) for general surface reconstruction but superior for thin structures [S1].
-   **Compression:** The wavelet representation enables good compression, mitigating some of the memory overhead [S1].
-   **Scalability:** The wall time could be reduced significantly using multi-threading, as voxel updates are independent [S1].

## Background
The method is described in the context of efficient volumetric mapping of multi-scale environments using wavelet-based compression [S1]. It was developed to address limitations in existing multi-resolution mapping frameworks, such as Octomap (discrete, poor thin object reconstruction) and Voxblox (TSDF-based, struggles with thin objects) [S1]. The continuous occupancy measurement model is inspired by prior work [16] (not in source list) [S1]. The method was evaluated on the Newer College dataset (Cloister sequence) using FastLIO2 for odometry and point cloud generation [S1].

## Sources
[S1] Efficient volumetric mapping of multi-scale environments using wavelet-based compression, https://zenodo.org/records/7993757, fetched 2026-10-27
