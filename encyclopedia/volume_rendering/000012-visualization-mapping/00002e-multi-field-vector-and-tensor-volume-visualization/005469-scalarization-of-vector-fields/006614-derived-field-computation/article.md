# Derived Field Computation

Also called: Derived Fields, Field Composition via Functions, Recursive Field Generators

## Overview
Derived Field Computation generates new data fields by applying mathematical functions to existing base fields or other derived fields [S1]. It differs from direct visualization methods by decoupling the data definition from the rendering pipeline, allowing for recursive composition and arbitrary mathematical operations that require data from neighboring grid patches [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** One or more base fields (e.g., velocity components $v_x, v_y, v_z$) or previously computed derived fields, accessed via a data container [S1].
*   **Output:** A new field represented as a NumPy array containing the computed values at each cell or sample point [S1].

**Data Structures:**
*   **Field Data:** The underlying data is stored in a multi-dimensional array structure corresponding to the spatial domain [S1]. When higher-order operations are required, the data structure includes buffer cells [S1].
*   **Buffer Cells:** These are additional cells surrounding the primary computational domain, populated with data from neighboring grids (covering grids) to support stencils that extend beyond a single grid patch [S1].

**Steps:**
1.  **Define the Function:** Create a function that accepts the `field` metadata and a `data` dictionary containing the input fields [S1].
2.  **Access Input Data:** Retrieve the required base fields (e.g., `data["x-velocity"]`) from the `data` dictionary [S1].
3.  **Apply Buffering (if needed):** If the computation requires neighbors, ensure the data container requests the appropriate number of buffer cells [S1]. The container automatically handles fetching data from covering grids [S1].
4.  **Compute Values:** Perform element-wise operations on the input arrays using standard mathematical functions [S1].
5.  **Return Result:** Return the resulting array, which can be used for further derived field generation or visualization [S1].

**Formulas:**
The magnitude of a velocity vector $\mathbf{v}$ is computed as [S1]:
$$ V = \sqrt{v_x^2 + v_y^2 + v_z^2} $$
where $v_x, v_y, v_z$ are the velocity components in the x, y, and z directions, respectively [S1].

**Parameters:**
*   **Buffer Cells:** The number of buffer cells specified when requesting 3D data [S1].

**Pitfalls:**
*   **Boundary Conditions:** Finite difference stencils may extend beyond the computational domain of a single grid patch [S1]. This is handled by utilizing covering grids to provide buffer cells from neighboring grids [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def VelocityMagnitude(field, data):
    """
    Computes the magnitude of the velocity vector from base velocity fields.
    
    Args:
        field: Metadata object for the derived field.
        data: Dictionary containing the base fields.
              Expected keys: "x-velocity", "y-velocity", "z-velocity".
    
    Returns:
        numpy.ndarray: The magnitude of the velocity at each cell.
    """
    # Access base fields as NumPy arrays
    vx = data["x-velocity"]
    vy = data["y-velocity"]
    vz = data["z-velocity"]
    
    # Compute magnitude using element-wise operations
    # V = sqrt(vx^2 + vy^2 + vz^2)
    magnitude = (vx**2.0 + vy**2.0 + vz**2.0)**0.5
    
    return magnitude
```

## Subtopics

### [Vector Field Generation from Image](19568-vector-field-generation-from-image.md)
This method synthesizes a 2D vector field from a scalar input image to enable visualization techniques such as Line Integral Convolution (LIC). It solves the problem of creating flow-like textures from static imagery by deriving directionality from image intensity gradients. It differs from direct gradient visualization by rotating the gradient vectors by 90 degrees, ensuring the resulting field is tangent to the intensity contours rather than perpendicular to them.

### [Scalar Field Functions](23563-scalar-field-functions.md)
Scalar Field Functions are analytical mathematical functions used to generate synthetic scalar fields for testing and validating gradient reconstruction accuracy in volume rendering [G]. They provide a known ground truth gradient, allowing for precise error measurement against reconstructed gradients [G]. They are distinguished from data-driven methods by their deterministic, closed-form nature, which allows for controlled variation of spatial frequency and complexity [G].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of cells in the domain, $O(N)$, as operations are element-wise [G].
*   **Performance:** Generating derived fields is described as "straightforward and fast" when using NumPy array operations [S1].
*   **Strengths:** High flexibility; allows recursive field generation; supports complex mathematical operations including those requiring neighbor data (via buffer cells) [S1].
*   **Limitations:** Requires careful management of buffer cells for higher-order methods [S1].
*   **When to use:** When visual attributes cannot be directly extracted from base fields and require mathematical composition or spatial derivatives [S1].

## Background
Derived Field Computation is a core feature of the `yt` analysis toolkit, designed for astrophysical simulation data [S1]. It was introduced to allow users to define custom field quantities that are functions of base fields or other derived fields, enabling automatic recursive field generation [S1]. The method leverages NumPy for efficient array operations and utilizes a covering grid system to handle data access across distributed or hierarchical grid patches [S1].

## Sources
[S1] YT: A Multi-Code Analysis Toolkit for Astrophysical Simulation Data, https://digital.library.unt.edu/ark:/67531/metadc834104/?q=fluid simulation volume rendering, fetched 2026-10-28
