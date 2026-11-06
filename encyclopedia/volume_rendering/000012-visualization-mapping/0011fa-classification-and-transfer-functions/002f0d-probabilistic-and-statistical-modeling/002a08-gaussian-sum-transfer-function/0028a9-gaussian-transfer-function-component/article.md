# Gaussian Transfer Function Component

Also called: Gaussian TF Component, Single Gaussian TF

## Overview
A Gaussian Transfer Function Component is a single Gaussian function used to define alpha (opacity) contributions in a 1D transfer function, parameterized by mean, sigma, and a height factor [S1, unofficial]. It serves as the fundamental building block for Gaussian Sum Transfer Functions, mapping a scalar field value to an opacity contribution [S1, unofficial]. It is distinguished from its siblings by its reliance on the standard Gaussian probability density function shape, whereas the Generalized Exponential Transfer Function uses exponential bases and the Laplacian of Gaussian Mixture Model uses the second derivative of Gaussians [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar data value $x \in [0, 1]$.
*   **Output:** Opacity contribution $\alpha \in [0, 1]$.

**Data Structures:**
The component is encapsulated in a `GaussianObject` struct with the following fields [S1, unofficial]:
*   `mean` (float): The center of the Gaussian.
*   `sigma` (float): The standard deviation (width) of the Gaussian.
*   `heightFactor` (float): A scaling factor for the peak height.
*   `alphaArray` (std::vector<float>): A precomputed lookup table of size `resolution` (default 1024) storing opacity values for discrete bins [S1, unofficial].

**Formulas:**
The value at any point $x$ is calculated as [S1, unofficial]:
$$
\text{value}(x) = \frac{\text{heightFactor}}{\sigma \sqrt{2\pi}} e^{-\frac{(x - \text{mean})^2}{2\sigma^2}}
$$

The peak height at $x = \text{mean}$ is [S1, unofficial]:
$$
\text{height}(\text{mean}) = \frac{\text{heightFactor}}{\sigma \sqrt{2\pi}}
$$

The `heightFactor` is related to a desired peak height $h$ by [S1, unofficial]:
$$
\text{heightFactor} = h \cdot \sigma \sqrt{2\pi}
$$

**Parameters:**
*   `mean`: Default 0.5f [S1, unofficial].
*   `sigma`: Default 1.0f [S1, unofficial].
*   `heightFactor`: Default 1.0f [S1, unofficial].
*   `resolution`: Default 1024 [S1, unofficial].

**Steps:**
1.  **Initialization:** Create a `GaussianObject` with default values (`mean=0.5f`, `sigma=1.0f`, `heightFactor=1.0f`) and an `alphaArray` of size 1024 [S1, unofficial]. Alternatively, initialize with specific parameters `mean`, `sigma`, `heightFactor`, and `resolution` [S1, unofficial].
2.  **Precomputation:** Update the `alphaArray` to store opacity values for the transfer function lookup [S1, unofficial].
    *   Calculate `invResolution = 1.0 / resolution`.
    *   Iterate over array indices $i$ from 0 to `resolution - 1`.
    *   Compute the midpoint of the bin: $x = (i + 0.5) \cdot \text{invResolution}$.
    *   Calculate the Gaussian value at $x$ using the formula above.
    *   Clamp the result to $[0, 1]$.
    *   Store the clamped value in `alphaArray[i]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class GaussianObject:
    def __init__(self, mean=0.5, sigma=1.0, heightFactor=1.0, resolution=1024):
        self.mean = mean
        self.sigma = sigma
        self.heightFactor = heightFactor
        self.resolution = resolution
        self.alphaArray = [0.0] * resolution
        self.update()

    def value(self, x):
        """Calculate the Gaussian value at x."""
        if self.sigma == 0:
            return 0.0
        exponent = -((x - self.mean) ** 2) / (2 * self.sigma ** 2)
        return (self.heightFactor / (self.sigma * math.sqrt(2 * math.pi))) * math.exp(exponent)

    def height(self):
        """Calculate the peak height at mean."""
        return self.heightFactor / (self.sigma * math.sqrt(2 * math.pi))

    def setHeight(self, h):
        """Set the desired peak height h and update heightFactor."""
        self.heightFactor = h * self.sigma * math.sqrt(2 * math.pi)
        self.update()

    def update(self):
        """Precompute the alpha array."""
        invResolution = 1.0 / self.resolution
        for i in range(self.resolution):
            # Midpoint of the bin
            x = (i + 0.5) * invResolution
            # Calculate value and clamp to [0, 1]
            val = self.value(x)
            if val < 0.0:
                val = 0.0
            elif val > 1.0:
                val = 1.0
            self.alphaArray[i] = val
```

**Edge Cases and Pitfalls:**
*   **Sigma Zero:** Division by zero occurs if `sigma` is 0.0. The implementation should handle this, e.g., by returning 0.0 or treating it as a Dirac delta (though the source code implies a default of 1.0f, preventing this in standard usage) [S1, unofficial].
*   **Continuity:** The Gaussian Sum Transfer Function, which uses these components, is noted as NOT C1 continuous [S1, unofficial]. This discontinuity arises from the summation of individual Gaussian components and their precomputed lookup tables, potentially causing artifacts at boundaries or when components are sharply defined.
*   **Clamping:** Values must be explicitly clamped to $[0, 1]$ to ensure valid opacity [S1, unofficial].

## Subtopics

### [Gaussian Transfer Function Composition](6243-gaussian-transfer-function-composition.md)
An interactive method to construct transfer functions by blending 2D Gaussian opacity functions with associated color definitions in the map space. It extends the 1D component to 2D parameter spaces for more complex classification.

## Operating characteristics
*   **Time Complexity:** Evaluation of a single component is $O(1)$ using the precomputed `alphaArray`. Initialization is $O(\text{resolution})$ due to the precomputation loop [S1, unofficial].
*   **Memory Complexity:** $O(\text{resolution})$ for the `alphaArray` [S1, unofficial].
*   **Strengths:** Simple to implement and understand; provides smooth, bell-shaped opacity transitions; efficient lookup via precomputed table [S1, unofficial].
*   **Limitations:** Single component has limited flexibility in shaping the transfer function; the overall Gaussian Sum TF is not C1 continuous, which may lead to visual artifacts [S1, unofficial].
*   **When to use:** When a simple, smooth opacity peak is needed for a specific data range.
*   **When not to use:** When C1 continuity is required or when complex, multi-modal opacity distributions are needed without summing multiple components.

## Background
The Gaussian Transfer Function Component is part of the TransferFunctionModule, an unofficial implementation by wilsonCernWq [S1, unofficial]. It is used in volume rendering pipelines to map scalar data to opacity, often as part of a larger Gaussian Sum Transfer Function. The module supports reading transfer functions from JSON formats, including ParaView's format, though some ParaView-specific features (like diverging color space interpolation) are not fully supported [S1, unofficial].

## Sources
[S1] wilsonCernWq/TransferFunctionModule, https://github.com/wilsonCernWq/TransferFunctionModule, fetched 2026-10-27 (unofficial)
