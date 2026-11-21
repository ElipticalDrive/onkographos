# 1D Transfer Function

Also called: FT unidimensional

## Overview
A 1D Transfer Function maps a single scalar input variable, typically voxel intensity or density, to optical properties to govern the transmission of emission and absorption through a volume [S1, unofficial][S2]. It is distinguished from multidimensional transfer functions by its reliance on a single attribute, which makes it computationally efficient but unable to distinguish spatially distinct objects that share the same scalar value interval [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar value $v$ representing the voxel intensity or density at position $\mathbf{x}$ [S1, unofficial][S2].
*   **Output:** A transmission value scaled between $0$ and $1$ that governs opacity and color [S2].

**Data Structures:**
The function is defined by boundaries, bins, and transmission values scaled between $0$ and $1$ [S2].
*   `x_bounds`: A tuple of floats $(x_{min}, x_{max})$ defining the valid input range. Values outside this range are discarded [S2].
*   `nbins`: An integer specifying the number of bins. Linear interpolation is used between bins [S2].
*   **Bin Array:** A 1D array of size `nbins + 1` storing the transmission value for each bin boundary [S2].

**Steps:**
1. Initialize the transfer function with `x_bounds` and `nbins` [S2].
2. Populate the bin array with transmission values scaled between $0$ and $1$ [S2].
3. For each sample point with scalar value $v$:
    a. Check if $v \in [x_{min}, x_{max}]$. If not, discard or assign a default value [S2].
    b. Map $v$ to a bin index and compute the fractional offset [S2].
    c. Evaluate the transfer function using linear interpolation between adjacent bins to obtain a transmission value [S2].
4. Use the evaluated transmission value to govern emission and absorption during volume integration [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class OneDTransferFunction:
    def __init__(self, x_bounds, nbins=256):
        self.x_min, self.x_max = x_bounds
        self.nbins = nbins
        self.bin_values = [0.0] * (nbins + 1)

    def set_bin_value(self, index, value):
        self.bin_values[index] = max(0.0, min(1.0, value))

    def evaluate(self, value):
        if value < self.x_min or value > self.x_max:
            return 0.0
        t = (value - self.x_min) / (self.x_max - self.x_min)
        idx = int(t * self.nbins)
        idx = max(0, min(self.nbins, idx))
        if idx >= self.nbins:
            return self.bin_values[self.nbins]
        frac = t * self.nbins - idx
        return self.bin_values[idx] * (1.0 - frac) + self.bin_values[idx + 1] * frac
```

**Parameters:**
*   `x_bounds`: Tuple of floats $(x_{min}, x_{max})$. Example values are $(-10.0, -5.0)$ [S2].
*   `nbins`: Integer. Default value is $256$ [S2].

**Edge Cases and Pitfalls:**
*   Using a single attribute can be detrimental because 1D TFs cannot precisely identify objects that are spatially distinct but "activated" by the same value interval [S1, unofficial].

## Subtopics

### Pre-classification Mapping
Pre-classification mapping applies the transfer function to scalar values at the grid points (voxels) before sampling [S3, unofficial]. This approach prevents color bleeding if opacity-weighted colors are interpolated, but it does not reproduce high frequencies, which generates noticeable artifacts in the final image [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** O(1) per sample point [G].
*   **Memory Complexity:** O(nbins) for the bin array storage [G].
*   **Strengths:** Simple to implement and edit; computationally efficient [G].
*   **Limitations:** Ambiguity in classification when different materials share similar scalar values [S1, unofficial].
*   **When to use:** When scalar intensity is the primary discriminant and computational simplicity is prioritized [G].
*   **When not to use:** When spatially distinct objects have overlapping scalar ranges [S1, unofficial].

## Background
1D transfer functions are the simplest form of transfer functions, defined by a single input variable, typically intensity [S1, unofficial]. They were widely adopted in early volume rendering due to their simplicity [S1, unofficial]. However, limitations in distinguishing objects with similar intensities led to the development of multidimensional transfer functions [S1, unofficial].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] ColorTransferFunction yt.visualization.volume_rendering.transfer_functions.ColorTransferFunction, https://yt-project.org/doc/reference/api/yt.visualization.volume_rendering.transfer_functions.html, fetched 2026-10-29
[S3] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
