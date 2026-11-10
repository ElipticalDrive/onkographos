# 1D Transfer Function

Also called: FT unidimensional

## Overview
A 1D Transfer Function maps a single scalar input variable, typically voxel intensity or density, to optical properties (opacity and color) to govern the transmission of emission and absorption through a volume [S1, unofficial][S2]. It is distinguished from multidimensional transfer functions by its reliance on a single attribute, which makes it computationally efficient but unable to distinguish spatially distinct objects that share the same scalar value interval [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar value $v$ representing the voxel intensity or density at position $\mathbf{x}$ [S1, unofficial][S2].
*   **Output:** Opacity $\alpha \in [0, 1]$ and color $C$ (emission/reflection) [S2].

**Data Structures:**
The function is defined by boundaries, bins, and transmission values scaled between 0 and 1 [S2].
*   `x_bounds`: A tuple of floats $(x_{min}, x_{max})$ defining the valid input range. Values outside this range are discarded [S2].
*   `nbins`: An integer specifying the number of bins. Linear interpolation is used between bins [S2].
*   **Gaussian Layers:** Features are often added as Gaussian distributions defined by location $x_0$, width $w$, and height $h$ [S2]. The value is calculated as:
    $$f(x) = h \exp\left(-\frac{(x-x_0)^2}{w}\right)$$
    [S2]

**Steps:**
1.  Initialize the transfer function with `x_bounds` and `nbins` [S2].
2.  Define features using Gaussian distributions by specifying location, width, and height [S2].
3.  For each sample point with scalar value $v$:
    a. Check if $v \in [x_{min}, x_{max}]$. If not, discard or assign default opacity [S2].
    b. Calculate the combined transmission value by summing contributions from all defined bins/layers [S2].
    c. Map the resulting value to opacity $\alpha$ and color $C$ [S2].
4.  Use the calculated $\alpha$ and $C$ during ray casting integration [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OneDTransferFunction:
    def __init__(self, x_bounds, nbins=256):
        self.x_min, self.x_max = x_bounds
        self.nbins = nbins
        self.layers = []

    def add_gaussian(self, location, width, height):
        """Add a Gaussian distribution to the transfer function."""
        self.layers.append({
            'location': location,
            'width': width,
            'height': height
        })

    def evaluate(self, value):
        """Evaluate opacity and color for a given scalar value."""
        # Discard values outside bounds
        if value < self.x_min or value > self.x_max:
            return 0.0, (0.0, 0.0, 0.0)

        # Calculate transmission value by summing Gaussian contributions
        transmission = 0.0
        for layer in self.layers:
            x0 = layer['location']
            w = layer['width']
            h = layer['height']
            # Gaussian formula: f(x) = h * exp(-(x - x0)^2 / w)
            contribution = h * math.exp(-((value - x0) ** 2) / w)
            transmission += contribution

        # Map transmission to opacity (scaled between 0 and 1)
        # Note: Exact mapping formula not specified in sources; assume linear or direct scaling
        alpha = min(transmission, 1.0)
        
        # Color assignment not detailed in sources for generic 1D TF
        # [G] Typically mapped from a colormap based on value or transmission
        color = (0.0, 0.0, 0.0) 
        
        return alpha, color
```

**Parameters:**
*   `x_bounds`: Tuple of floats $(x_{min}, x_{max})$. Typical values depend on dataset range (e.g., $(-10.0, -5.0)$) [S2].
*   `nbins`: Integer. Typical value is 256 [S2].

**Edge Cases and Pitfalls:**
*   Using a single attribute can be detrimental because 1D TFs cannot precisely identify objects that are spatially distinct but "activated" by the same value interval [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample point, assuming a fixed number of layers/bins.
*   **Memory Complexity:** $O(N)$ where $N$ is the number of bins or layers.
*   **Strengths:** Simple to implement and edit; computationally efficient.
*   **Limitations:** Ambiguity in classification when different materials share similar scalar values [S1, unofficial].
*   **When to use:** When scalar intensity is a strong discriminator of material properties.
*   **When not to use:** When spatially distinct objects have overlapping scalar ranges [S1, unofficial].

## Background
1D transfer functions are the simplest form of transfer functions, defined by a single input variable, typically intensity [S1, unofficial]. They were widely adopted in early volume rendering due to their simplicity [S1, unofficial]. However, limitations in distinguishing objects with similar intensities led to the development of multidimensional transfer functions [S1, unofficial].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] ColorTransferFunction yt.visualization.volume_rendering.transfer_functions.ColorTransferFunction, https://yt-project.org/doc/reference/api/yt.visualization.volume_rendering.transfer_functions.html, fetched 2026-10-29
