# Implicit Surface Taxonomy

Also called: Lipschitz Scalar Fields, SDF Classification

## Overview
This taxonomy categorizes implicit surface models into four distinct types based on their mathematical properties: Distance Field, Pseudo Signed Distance Field, Conservative Distance Field, and Lipschitz Scalar Field [S1]. It distinguishes these types to determine which implicit operators (such as Boolean combinations or inflation) can be applied without introducing geometric artifacts or discontinuities [S1]. Unlike [SDF Primitive Definitions](1578-sdf-primitive-definitions.md) which focuses on specific shapes, this article defines the validity conditions for general functions within a construction tree.

## How it works
The taxonomy classifies a scalar field function $f(\mathbf{p})$ based on its relationship to the true Euclidean signed distance $d(\mathbf{p})$ and the magnitude of its gradient $\|\nabla f\|$ [S1].

### Data Structures and Properties
The classification is defined by the following constraints:

1.  **Distance Field**:
    *   Condition: $f(\mathbf{p}) = d(\mathbf{p})$ [S1].
    *   Properties: Exact Euclidean distance. Satisfies the eikonal equation $\|\nabla f\| = 1$ everywhere except at singularities (medial axes) [S1].

2.  **Pseudo Signed Distance Field**:
    *   Condition: Piecewise differentiable function satisfying the eikonal property $\|\nabla f\| = 1$ wherever the gradient is well-defined [S1].
    *   Properties: Often arises from geometric editing or Boolean operations on exact SDFs. May lack continuity or exact distance values [S1].

3.  **Conservative Distance Field**:
    *   Condition: $|f(\mathbf{p})| \le |d(\mathbf{p})|$ [S1].
    *   Properties: The field value is a lower bound on the true distance. It does not need to be continuous and may exhibit unexpected high gradients in certain regions [S1].

4.  **Lipschitz Scalar Field**:
    *   Condition: Satisfies the Lipschitz property with constant 1, implying $\|\nabla f\| \le 1$ [S1].
    *   Properties: Continuous and has a bounded gradient. These properties are stronger than those of conservative distance fields [S1].

### Operator Compatibility and Pitfalls
The choice of field type dictates which implicit operators can be safely applied in a construction tree [S1].

*   **Sphere Tracing**: The original algorithm is applicable to Lipschitz scalar fields. It is also applicable to conservative distance fields, provided the conservative property ($|f| \le |d|$) ensures the step size remains a safe lower bound [S1].
*   **Smooth Boolean Operators**: These operators require the input functions to be Lipschitz or at least continuous [S1]. Applying smooth Boolean operators to discontinuous conservative distance fields may produce fields that are not continuous and fail to consistently define an implicit surface [S1].
*   **General Implicit Operators**: Most implicit operators require a true signed distance field to function correctly. Feeding them with a pseudo signed distance or a conservative field may introduce artifacts [S1].
*   **Inflation Operator**: The inflate operator, defined as $I(\epsilon) = f(\mathbf{p} + \epsilon \mathbf{n}) - k$ (or conceptually shifting the isosurface), is impacted by the degradation of surface quality in non-exact fields [S1]. Applying this operator to conservative or pseudo fields results in visible artifacts compared to exact distance fields [S1].

### Pseudocode for Classification Check
The following pseudocode demonstrates how to verify if a field satisfies the Lipschitz condition, a prerequisite for many smooth operators. Note that exact verification is generally impossible for arbitrary functions; this checks the gradient bound where differentiable.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def is_lipschitz_field(f, p, epsilon=1e-6):
    """
    Checks if the field f satisfies the Lipschitz condition ||grad f|| <= 1
    at point p by numerical differentiation.
    
    Args:
        f: Function evaluating the scalar field.
        p: 3D point (vector).
        epsilon: Step size for numerical gradient.
        
    Returns:
        True if ||grad f|| <= 1 + tolerance, False otherwise.
    """
    # Numerical gradient calculation
    grad_x = (f(p + [epsilon, 0, 0]) - f(p - [epsilon, 0, 0])) / (2 * epsilon)
    grad_y = (f(p + [0, epsilon, 0]) - f(p - [0, epsilon, 0])) / (2 * epsilon)
    grad_z = (f(p + [0, 0, epsilon]) - f(p - [0, 0, epsilon])) / (2 * epsilon)
    
    gradient_magnitude = sqrt(grad_x**2 + grad_y**2 + grad_z**2)
    
    # Tolerance for numerical error
    return gradient_magnitude <= 1.0 + 1e-5

def classify_field(f, p, true_distance_func=None):
    """
    Classifies the field f at point p based on available ground truth.
    
    Args:
        f: The implicit function.
        p: 3D point.
        true_distance_func: Optional function for exact distance d(p).
        
    Returns:
        String: 'Distance', 'Pseudo', 'Conservative', or 'Lipschitz'
    """
    val = f(p)
    
    # Check for Exact Distance Field
    if true_distance_func is not None:
        d = true_distance_func(p)
        if abs(val - d) < 1e-5:
            return 'Distance'
        elif val <= d and val >= -d: # |f| <= |d|
            # Further check for Lipschitz vs Conservative
            # Conservative fields may not be continuous or have bounded gradients
            # Lipschitz fields have ||grad f|| <= 1
            if is_lipschitz_field(f, p):
                return 'Lipschitz'
            else:
                return 'Conservative'
    
    # If we cannot compare to true distance, we rely on gradient properties
    # Pseudo SDFs satisfy ||grad f|| = 1
    # Lipschitz fields satisfy ||grad f|| <= 1
    
    # Note: Without true distance, distinguishing Pseudo from Lipschitz 
    # requires checking if ||grad f|| == 1 strictly.
    # This is a heuristic classification.
    
    grad_mag = compute_gradient_magnitude(f, p)
    
    if abs(grad_mag - 1.0) < 1e-5:
        return 'Pseudo' # Or Lipschitz if <= 1
    elif grad_mag <= 1.0:
        return 'Lipschitz'
    else:
        return 'Unknown' # Violates Lipschitz
```

## Operating characteristics
*   **Modeling Complexity**: Using exact Distance Fields is computationally expensive to compute for complex shapes but guarantees artifact-free results for all operators [S1].
*   **Artifact Risk**: Using Conservative or Pseudo fields reduces computational cost or simplifies construction but risks artifacts when using smooth Boolean operators or inflation [S1].
*   **Rendering Compatibility**: Sphere tracing works with Conservative and Lipschitz fields, but smooth blending requires Lipschitz continuity [S1].

## Background
The taxonomy was introduced in [MSLJ23] and completed with the inclusion of Lipschitz scalar fields, referencing foundational work by [KB89] and [Har96] [S1]. The classification highlights the trade-offs in implicit modeling trees, where internal nodes combine values from sub-trees [S1].

## Sources
[S1] Accelerating Signed Distance Functions, https://doi.org/10.1111/cgf.70258, fetched 2026-10-26
