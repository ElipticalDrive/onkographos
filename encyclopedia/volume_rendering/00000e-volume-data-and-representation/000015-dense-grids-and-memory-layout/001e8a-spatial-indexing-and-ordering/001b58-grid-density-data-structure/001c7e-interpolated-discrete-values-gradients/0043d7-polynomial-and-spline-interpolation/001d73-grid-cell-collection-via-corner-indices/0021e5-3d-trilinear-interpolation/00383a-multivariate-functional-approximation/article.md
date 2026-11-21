# Multivariate Functional Approximation

Also called: Functional approximation

## Overview
Multivariate Functional Approximation (MFA) is a continuous volume representation method that models 3D scalar fields using polynomial functions (specifically B-splines) defined over micro-blocks of the data grid [S1, unofficial]. It solves the problem of accurate off-grid sample evaluation and smooth shading in volume rendering by providing higher-order continuity than trilinear interpolation without requiring ghost voxels or boundary padding [S1, unofficial]. Unlike its sibling [3D Trilinear Interpolation](2613-3d-trilinear-interpolation.md), which is limited to C0 continuity and linear accuracy, MFA approximates C1 continuity across partition boundaries, reducing Mach ringing artifacts and polygon counts in isosurface extraction while maintaining a compact data footprint [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field dataset of resolution $n^3$, partitioned into micro-blocks.
*   **Output:** A set of "micro-models," each containing control points and knot vectors defining a trivariate polynomial function for its respective micro-block.

**Data Structures:**
The core data structure is the **Micro-model**. It stores the parameters required to evaluate the B-spline function.
*   **Fields:**
    *   `degree` (integer): The polynomial order of the fitted B-spline.
    *   `NCP` (integer): The Number of Control Points.
    *   `ControlPoints` (array): A list of 3D coordinates for the control points.
    *   `Knots` (array): The knot vector defining the B-spline basis functions.
*   **Storage Size:** The file size of a single micro-model in bytes is calculated as [S1, unofficial]:
    $$Size(Micro\text{-}model) = 1 + ((NCP + degree) \times 3 + NCP^3) \times 4$$
    *Note: The sources provide this formula directly. The term $NCP^3$ suggests a dense storage of control point data or a specific indexing scheme, though standard B-splines typically store $NCP$ points. The formula is cited exactly as provided in [S1, unofficial].*

**Parameters:**
*   **NCP (Number of Control Points):** The primary determinant of encoded model size and fitting accuracy. Valid range for a micro-block of resolution $n^3$ is $[degree+1, n]$ [S1, unofficial]. A larger NCP yields higher fitting accuracy but increases storage and query time. The optimal NCP for a given error bound is data-dependent and may require exhaustive search [S1, unofficial].
*   **Degree:** The polynomial order of the fitted B-spline. Higher degrees increase continuity and accuracy but also computational cost [S1, unofficial].

**Algorithm Steps:**
1.  **Partitioning:** Divide the input 3D dataset into micro-blocks. For multi-resolution frameworks, this may involve $m$ Levels of Detail (LODs), where the number of micro-blocks increases twofold along each axis ($x, y, z$) from LOD $i$ to $i+1$ [S1, unofficial].
2.  **Fitting:** For each micro-block, fit a multivariate polynomial (B-spline) to the scalar values within the block.
    *   Select `degree` and `NCP` based on desired accuracy and size constraints.
    *   Solve for control points and knots to minimize global error. This step has a time complexity of $O(n^4)$ for a 3D dataset of resolution $n^3$ across all possible NCP values [S1, unofficial].
3.  **Encoding:** Store the resulting `degree`, `NCP`, control points, and knots in the Micro-model structure.
4.  **Querying (Rendering):** To evaluate a sample point $P$ within a micro-block:
    *   Identify the micro-block containing $P$.
    *   Evaluate the trivariate B-spline function using the stored control points, knots, and degree at $P$.
    *   Return the interpolated value. This allows random access to both value and derivative with high-order accuracy [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_mfa_micro_model(point, micro_model):
    """
    Evaluates the scalar value at a given 3D point using a pre-fitted MFA micro-model.
    
    Args:
        point (tuple): (x, y, z) coordinates within the micro-block's local space.
        micro_model (dict): Contains 'degree', 'NCP', 'control_points', 'knots'.
        
    Returns:
        float: The interpolated scalar value.
    """
    # Extract parameters
    degree = micro_model['degree']
    ncp = micro_model['NCP']
    control_points = micro_model['control_points']
    knots = micro_model['knots']
    
    # Evaluate trivariate B-spline
    # Note: The exact B-spline evaluation algorithm (e.g., Cox-de Boor recursion)
    # is not detailed in the sources. This step assumes a standard B-spline evaluator.
    value = b_spline_evaluate_3d(point, degree, ncp, control_points, knots)
    
    return value

def encode_micro_block(data_block, target_ncc, target_degree):
    """
    Encodes a 3D data block into an MFA micro-model.
    
    Args:
        data_block (array): 3D array of scalar values.
        target_ncp (int): Desired Number of Control Points.
        target_degree (int): Desired polynomial degree.
        
    Returns:
        dict: The micro-model dictionary.
    """
    # Fit B-spline to data_block
    # This involves solving a least-squares problem or similar optimization
    # to find control points and knots that best approximate data_block.
    # Complexity is O(n^4) for the full dataset encoding [S1, unofficial].
    control_points, knots = fit_bspline_to_data(data_block, target_ncp, target_degree)
    
    micro_model = {
        'degree': target_degree,
        'NCP': target_ncp,
        'control_points': control_points,
        'knots': knots
    }
    return micro_model
```

**Edge Cases and Pitfalls:**
*   **Encoding Time:** For large-scale datasets ($n$ large) with multiple LODs ($m$ large), the total encoding time $O(8^m \cdot n^4)$ can become impractical [S1, unofficial]. Adaptive encoding methods are required to mitigate this [S1, unofficial].
*   **Query Bottleneck:** Query time is the main performance bottleneck for visualizing large-scale datasets using MFA [S1, unofficial].
*   **Boundary Continuity:** While MFA approximates C1 continuity, it does not guarantee it strictly without careful knot placement and control point selection. The sources state it "approximates C1 continuity" [S1, unofficial].

## Subtopics

### [Trivariate Cubic Polynomial Approximation for Scalar Fields](14573-trivariate-cubic-polynomial-approximation-for-scalar-fields.md)
This variant models 3D scalar field data (e.g., CT/MRI) by approximating subcubes of a cuberille volume with trivariate cubic polynomial functions that achieve C1 continuity on shared faces. It provides a compact analytic representation for isosurface generation, distinguishing itself from lower-order approximations by avoiding polygon shading artifacts like Mach ringing.

### C3 Data Approximation
A method for approximating volume data using C3 data, yielding polygonal isosurfaces. It offers two approximation types: (1,1,1) and (3,3,3) [S2, unofficial].
*   **(1,1,1) Approximation:** Yields a C0 smooth approximation. It is faster to calculate but produces more polygons (e.g., 121,486 polygons for a sphere test data) [S2, unofficial].
*   **(3,3,3) Approximation:** Yields at least C1 smooth approximation. It takes much longer to calculate but produces fewer polygons (e.g., 78,741 polygons for the same sphere test data) [S2, unofficial].
*   **Output Structure:** Each polygon has three to six vertices with XYZ coordinate values and a normal associated with each vertex [S2, unofficial].

## Operating characteristics
*   **Time Complexity:**
    *   Encoding: $O(n^4)$ for a 3D dataset of resolution $n^3$ across all possible NCP values [S1, unofficial].
    *   Multi-resolution Encoding: Total time for $m$ LODs is $O(8^m \cdot n^4)$ [S1, unofficial].
    *   Querying: Not explicitly given, but noted as the main performance bottleneck [S1, unofficial].
*   **Memory Complexity:** Determined by the Micro-model size: $1 + ((NCP + degree) \times 3 + NCP^3) \times 4$ bytes per micro-block [S1, unofficial].
*   **Measured Performance:**
    *   In isosurface extraction, (3,3,3) C3 approximation reduced polygon count from 121,486 to 78,741 for a sphere test data compared to (1,1,1) [S2, unofficial].
    *   (3,3,3) calculation takes much longer than (1,1,1) for the same image size [S2, unofficial].
*   **Strengths:**
    *   Higher accuracy for off-grid sample evaluation than trilinear interpolation [S1, unofficial].
    *   Approximates C1 continuity without ghost voxels, reducing data size overhead [S1, unofficial].
    *   Reduces Mach ringing artifacts in rendering [S1, unofficial].
    *   Compact analytic representation [S1, unofficial].
*   **Limitations:**
    *   High encoding time for large datasets [S1, unofficial].
    *   Query time is a bottleneck [S1, unofficial].
    *   Optimal NCP is data-dependent and may require exhaustive search [S1, unofficial].
*   **When to Use:** Suitable for interactive visualization of large-scale scientific datasets where rendering quality (smoothness, accuracy) is critical and encoding can be done offline or adaptively [S1, unofficial].
*   **When Not to Use:** When real-time encoding is required for very large datasets, or when memory bandwidth is severely constrained and the overhead of storing control points is prohibitive.

## Background
MFA was introduced as a method for in situ modeling of scientific data, providing random access to value and derivative with high-order accuracy [S1, unofficial]. It addresses the limitations of traditional local filters like trilinear interpolation in distributed visualization pipelines [S1, unofficial]. The method is particularly relevant for structured or unstructured grids with non-linear spatial correlations [S1, unofficial]. The C3 data approximation variant was explored in the context of scalar field modeling and visualization on parallel architectures like the Intel Delta [S2, unofficial].

## Sources
[S1] Adaptive Multi-Resolution Encoding for Interactive Large-Scale Volume ..., https://www.researchgate.net/publication/383700795_Adaptive_Multi-Resolution_Encoding_for_Interactive_Large-Scale_Volume_Visualization_through_Functional_Approximation, fetched 2026-10-30 (unofficial)
[S2] Scalar Field Modeling & Visualization on the Intel Delta', https://docs.lib.purdue.edu/cgi/viewcontent.cgi?article=2138&context=cstech, fetched 2026-10-26 (unofficial)
