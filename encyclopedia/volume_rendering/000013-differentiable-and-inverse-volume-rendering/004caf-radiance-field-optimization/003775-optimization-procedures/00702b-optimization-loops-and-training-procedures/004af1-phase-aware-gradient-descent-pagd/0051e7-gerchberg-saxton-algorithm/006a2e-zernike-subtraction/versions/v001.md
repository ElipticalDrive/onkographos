# Zernike Subtraction


## Overview
Zernike subtraction is an iterative optimization method that directly optimizes spot quality at each farfield target point by iteratively subtracting measured aberrations, applicable to non-imaging-based indirect feedback metrics [S1, unofficial]. It sets itself apart from its sibling, the [Gerchberg-Saxton Algorithm](14575-gerchberg-saxton-algorithm.md), by replacing global phase-only constraints with a local, aberration-space correction derived from sweeping global Zernike modes to determine their specific contribution to local spot quality [S1, unofficial].

## How it works
**Inputs:**
- Desired target points in the farfield.
- A baseline hologram targeting these points.
- A figure of merit for spot quality (e.g., fluorescence intensity, trap depth) [S1, unofficial].
- Maximum Zernike order $D$ [S1, unofficial].

**Outputs:**
- An aberration-corrected hologram that negates local aberration to within measurement accuracy [S1, unofficial].

**Data Structures:**
- **Aberration Map:** A $D$-dimensional vector representing the optimal Zernike coefficients for a specific 3D point.
- **Calibration Grid:** A set of measured aberration vectors for points in $\hat{y}$-$\hat{z}$ planes, used for interpolation [S1, unofficial].

**Steps:**
1. **Generate Baseline:** Create a baseline hologram targeting the desired farfield points using standard holographic methods [S1, unofficial].
2. **Sweep Zernike Orders:** For each target spot, superimpose and sweep a global Zernike order $Z_d$ (where $d \in [1, D]$). Measure the change in the spot quality figure of merit to determine the contribution of each Zernike mode to that specific spot [S1, unofficial].
3. **Subtract Correction:** Using aberration-space holography, subtract the measured anisoplanatic correction from each spot. This involves calculating the residual aberration required to maximize the figure of merit [S1, unofficial].
4. **Iterate:** Repeat steps 2 and 3 with global Zernike orders up to $D$ until the aberration-space spot array converges, negating local aberration to within measurement accuracy [S1, unofficial].
5. **Interpolation (for Volumetric Calibration):** For any 3D point within the calibration volume, fit and interpolate the resulting calibration point aberration maps to determine the $D$-dimensional aberration vector that optimizes focus [S1, unofficial].

**Parameters:**
- $D$: The maximum Zernike order used for subtraction. Typical value: $D=27$ [S1, unofficial].
- $k_x$: The constant of proportionality between nearfield and farfield, defined as $k_x \approx x/f$, where $f$ is the effective focal length [S1, unofficial].

**Pitfalls:**
- The method relies on indirect feedback metrics (e.g., atomic trap depth, two-photon fluorescence intensity) rather than direct wavefront sensing, which requires careful calibration of the figure of merit [S1, unofficial].
- Precision is limited by measurement accuracy; convergence is achieved only to within this limit [S1, unofficial].

## Subtopics

### Volumetric Aberration Calibration
Calibrates aberration throughout a 3D medium using $D=27$th-order Zernike subtraction to maximize spot brightness in parallel using a camera [S1, unofficial]. It images and calibrates arrays of spots in $\hat{y}$-$\hat{z}$ planes, then fits and interpolates the resulting calibration point aberration maps to determine the $D$-dimensional aberration vector for any 3D point within the volume [S1, unofficial]. This enables dynamic projection of arbitrary 3D patterns with optimum brightness [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- The method achieves parallelization that reduces measurement time by a factor of $O(100\times)$ compared to sequential methods [S1, unofficial].
- Memory complexity is not explicitly given, but requires storing $D$-dimensional aberration vectors for interpolation [S1, unofficial].

**Measured Performance:**
- Achieved a volumetric display with $O(10^8)$ addressable voxels [S1, unofficial].
- Spot size: $\sim 3 \times 3 \times 34 \, \mu m^3$ [S1, unofficial].
- Display volume: $\sim 3.75 \times 3.75 \times 10 \, mm^3$ [S1, unofficial].
- Lateral bounds determined by the Nyquist limit; depth bounds by cuvette dimensions [S1, unofficial].

**Scaling:**
- Larger volumes or tighter spots are possible by increasing or decreasing the effective focal length $f$, which scales the constant $k_x \approx x/f$ [S1, unofficial].
- Brightness scales logarithmically with enclosed display volume for both isoplanatic and aberration-space corrections [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Directly optimizes spot quality; applicable to non-imaging-based indirect feedback; enables video-rate aberration-corrected volumetric display [S1, unofficial].
- **Limitations:** Requires calibration of aberration maps; performance limited by measurement accuracy and anisoplanatism [S1, unofficial].

**When to Use:**
- Use when direct wavefront sensing is not available and indirect metrics (fluorescence, trap depth) can be measured [S1, unofficial].
- Use for volumetric displays requiring high addressable voxel counts with aberration correction [S1, unofficial].

**When Not to Use:**
- Not suitable when direct wavefront sensing is available and preferred for higher precision [S1, unofficial].
- Not suitable for systems where video-rate performance is not required and simpler isoplanatic approximations suffice [S1, unofficial].

## Background
Originated in the paper "Full-volume aberration-space holography" (2025) [S1, unofficial]. The method was developed to enable high-performance computer-generated holography by correcting for anisoplanatism in volumetric displays, specifically for applications like optical tweezers and two-photon microscopy [S1, unofficial]. It builds on the Gerchberg-Saxton Algorithm by adding a layer of local aberration correction derived from Zernike mode sweeping [S1, unofficial].

## Sources
[S1] Full-volume aberration-space holography, https://arxiv.org/abs/2505.08777v1, fetched 2026-10-26 (unofficial)
