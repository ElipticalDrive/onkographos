# Higher Order Algebraic Signed Distance Fields

Also called: HOASDFs

## Overview
Higher Order Algebraic Signed Distance Fields (HOASDFs) generalize traditional signed distance fields by approximating the distance function over a larger area using local Taylor approximations or least squares fit polynomials [S1, abstract only, unofficial]. This builds on [Basic Primitive Definitions](5346-basic-primitive-definitions.md) and [SDF Primitives & Operations](1075-sdf-primitives-operations.md).

## How it works
**Inputs:** A set of sample points with associated distance values and, optionally, partial derivatives (for Hermite interpolation) [S1, abstract only, unofficial].
**Outputs:** A query function $D(\mathbf{x})$ that returns the signed distance and potentially geometric properties (e.g., normal) at any point $\mathbf{x}$ [S1, abstract only, unofficial].

**Data Structures:**
The field stores coefficients for polynomial approximations [S1, abstract only, unofficial].
*   **Taylor Fields:** Store the value and partial derivatives up to order $n$ at each sample point [S1, abstract only, unofficial].
*   **Least Squares Fields:** Store coefficients of a polynomial fitted to a local neighborhood of samples [S1, abstract only, unofficial].

**Steps:**
1. **Formulation:** Formulate the sampling and reconstruction process as a function approximation problem combining local Taylor approximations [S1, abstract only, unofficial].
2. **Accuracy Characterization:** Derive the error bound for arbitrary sampling topologies and interpolation schemes by noting that the error bound of a combination of Taylor approximations is closed under convex barycentric combination [S1, abstract only, unofficial].
3. **Extension to Least Squares:** Extend the framework to least squares fit polynomial fields [S1, abstract only, unofficial].
4. **Implementation:** For regular grids, utilize GPU filtering to implement highly efficient queries on both Taylor and least squares fields [S1, abstract only, unofficial]. Store partial derivatives directly in the field data and use Hermite interpolation for queries, trading query time for higher accuracy [S1, abstract only, unofficial].
5. **Conversion:** Convert traditional zero order signed distance fields to higher order representations using a general framework [S1, abstract only, unofficial].
6. **Optimization:** Decrease filtering footprint size via simplicial interpolation schemes to balance performance and storage [S1, abstract only, unofficial].
7. **Geometric Inference:** Infer higher order geometric properties, such as surface normals, from the derivative data [S1, abstract only, unofficial].

**Formulas:**
The Taylor expansion of the distance function $D$ around a sample point $\mathbf{s}$ up to order $n$ is:
$$D(\mathbf{x}) \approx D(\mathbf{s}) + \nabla D(\mathbf{s}) \cdot (\mathbf{x} - \mathbf{s}) + \frac{1}{2} (\mathbf{x} - \mathbf{s})^T \mathbf{H}(\mathbf{s}) (\mathbf{x} - \mathbf{s}) + \dots$$
where $\nabla D(\mathbf{s})$ is the gradient and $\mathbf{H}(\mathbf{s})$ is the Hessian matrix at $\mathbf{s}$ [S1, abstract only, unofficial]. The final query result is a convex barycentric combination of these local expansions [S1, abstract only, unofficial].

**Parameters:**
*   **Order ($n$):** The degree of the Taylor approximations or polynomial fields used to represent the distance function [S1, abstract only, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Higher order expansions amplify floating-point errors near the surface; clamping or fallback to zero-order queries may be required [G].
*   **Boundaries:** Sampling irregularities at domain edges break barycentric weight computation; padding or boundary-aware interpolation is needed [G].
*   **Degenerate Input:** Collinear or coplanar sample points produce singular barycentric systems; fallback to inverse-distance weighting or grid-aligned queries is required [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_taylor_field(x, samples, order):
    # samples: list of objects with fields .position, .value, .gradient, .hessian, .third_order
    weights = compute_barycentric_weights(x, samples)
    result = 0.0
    for i, s in enumerate(samples):
        delta = x - s.position
        poly_val = s.value
        if order >= 1:
            poly_val += sum(s.gradient[j] * delta[j] for j in range(3))
        if order >= 2:
            poly_val += 0.5 * sum(s.hessian[j][k] * delta[j] * delta[k] for j in range(3) for k in range(3))
        if order >= 3:
            poly_val += sum(s.third_order[j][k][l] * delta[j] * delta[k] * delta[l] for j in range(3) for k in range(3) for l in range(3)) / 6.0
        result += weights[i] * poly_val
    return result

def compute_barycentric_weights(x, samples):
    # Standard convex barycentric weight calculation for arbitrary topology
    dists = [sum((x[j] - s.position[j])**2 for j in range(3)) for s in samples]
    inv_dists = [1.0 / (d + 1e-8) for d in dists]
    total = sum(inv_dists)
    return [w / total for w in inv_dists]
```

## Subtopics

### Taylor Approximation Fields
Use local Taylor expansions. Sets apart by leveraging derivative data for higher accuracy. Choose when query accuracy is prioritized over raw speed. [Link](#how-it-works) [S1, abstract only, unofficial]

### Least Squares Polynomial Fields
Fit polynomials to local neighborhoods. Sets apart by smoothing noise across samples. Choose when robustness to sampling irregularities is needed. [Link](#how-it-works) [S1, abstract only, unofficial]

### Hermite Interpolation
Stores partial derivatives directly. Sets apart by enabling exact gradient reconstruction at sample points. Choose when surface normals or precise geometric properties are required. [Link](#how-it-works) [S1, abstract only, unofficial]

## Operating characteristics
*   **Time Complexity:** Query time is higher than zero-order SDFs, especially for Hermite interpolation [S1, abstract only, unofficial].
*   **Memory Complexity:** Depends on the order and number of coefficients stored per sample. Higher order requires more storage per sample but may allow for coarser sampling grids [S1, abstract only, unofficial].
*   **Performance:** GPU filtering can be utilized for efficient queries on regular grids [S1, abstract only, unofficial].
*   **Accuracy:** Higher order fields provide better accuracy and can infer geometric properties like normals [S1, abstract only, unofficial].
*   **Storage:** Techniques decrease filtering footprint size to balance performance and storage, improving quality without increasing footprint [S1, abstract only, unofficial].
*   **When to use:** When higher accuracy is needed without increasing sampling density, or when geometric properties like normals are required [S1, abstract only, unofficial].
*   **When not to use:** When query performance is critical and lower accuracy is acceptable [S1, abstract only, unofficial].

## Background
Proposed by Gábor Valasek and Róbert Bán in January 2022 [S1, abstract only, unofficial]. It addresses limitations of classical sphere tracing and zero-order SDFs by leveraging higher-order derivative information [S1, abstract only, unofficial].

## Sources
[S1] Enhanced Sphere Tracing | Request PDF - ResearchGate, https://www.researchgate.net/publication/265020476_Enhanced_Sphere_Tracing, fetched 2026-11-03 (abstract only, unofficial)
