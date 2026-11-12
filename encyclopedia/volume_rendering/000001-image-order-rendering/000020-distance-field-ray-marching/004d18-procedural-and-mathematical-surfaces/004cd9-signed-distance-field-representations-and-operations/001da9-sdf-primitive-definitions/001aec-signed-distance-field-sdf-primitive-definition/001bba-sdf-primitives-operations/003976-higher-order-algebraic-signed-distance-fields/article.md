# Higher Order Algebraic Signed Distance Fields

Also called: HOASDFs

## Overview
Higher Order Algebraic Signed Distance Fields (HOASDFs) generalize traditional signed distance fields by approximating the distance function over a larger area using local Taylor approximations or least squares fit polynomials [S1, abstract only, unofficial]. This approach allows for higher accuracy and potentially smaller filtering footprints compared to zero-order SDFs, building on [Basic Primitive Definitions](5346-basic-primitive-definitions.md) and [SDF Primitives & Operations](1075-sdf-primitives-operations.md).

## How it works
**Inputs:** A set of sample points with associated distance values and, optionally, partial derivatives (for Hermite interpolation).
**Outputs:** A query function $D(\mathbf{x})$ that returns the signed distance and potentially geometric properties (e.g., normal) at any point $\mathbf{x}$.

**Data Structures:**
The field stores coefficients for polynomial approximations. For a field of order $n$ in 3D, the number of coefficients per sample point is $\binom{n+3}{3}$.
*   **Taylor Fields:** Store the value and partial derivatives up to order $n$ at each sample point.
*   **Least Squares Fields:** Store coefficients of a polynomial fitted to a local neighborhood of samples.

**Steps:**
1.  **Formulation:** Formulate the sampling and reconstruction process as a function approximation problem using local Taylor approximations [S1, abstract only, unofficial].
2.  **Accuracy Characterization:** Derive the error bound for arbitrary sampling topologies. The error bound of a combination of Taylor approximations is closed under convex barycentric combination [S1, abstract only, unofficial].
3.  **Extension to Least Squares:** Extend the framework to least squares fit polynomial fields [S1, abstract only, unofficial].
4.  **Implementation:**
    *   For regular grids, utilize GPU filtering to implement efficient queries [S1, abstract only, unofficial].
    *   For Hermite interpolation, store partial derivatives directly in the field data [S1, abstract only, unofficial].
5.  **Conversion:** Convert traditional zero-order SDFs to higher-order representations using a general framework [S1, abstract only, unofficial].
6.  **Optimization:** Decrease filtering footprint size via simplicial interpolation schemes to balance performance and storage [S1, abstract only, unofficial].
7.  **Geometric Inference:** Infer higher-order geometric properties, such as surface normals, from the derivative data [S1, abstract only, unofficial].

**Parameters:**
*   **Order ($n$):** The degree of the Taylor approximations or polynomial fields. Typical values evaluated include 1, 2, and 3 [S1, abstract only, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_taylor_field(x, samples, order):
    """
    Query a Taylor-based HOASDF.
    x: Query point (vec3)
    samples: List of sample points with stored derivatives
    order: Degree of Taylor approximation
    """
    # Find nearest sample or use barycentric combination of surrounding samples
    # This step depends on the specific sampling topology and interpolation scheme
    # Not given in the sources. [G]
    
    # Compute barycentric coordinates for the query point relative to the samples
    # weights = compute_barycentric_weights(x, samples)
    
    # Evaluate the Taylor polynomial at x for each sample
    # Taylor expansion of distance function D around sample point s:
    # D(x) ≈ D(s) + ∇D(s)·(x-s) + 1/2 (x-s)^T H(s) (x-s) + ...
    
    result = 0
    for i, s in enumerate(samples):
        # Get stored derivatives up to order 'order' at s
        # derivatives = s.derivatives
        # delta = x - s.position
        
        # Evaluate polynomial using derivatives and delta
        # poly_val = evaluate_taylor_polynomial(delta, derivatives, order)
        
        # Combine using barycentric weights
        # result += weights[i] * poly_val
        
        pass # Placeholder for specific interpolation logic not detailed in source
    
    return result

def query_hermite_field(x, samples):
    """
    Query a Hermite-based HOASDF.
    x: Query point (vec3)
    samples: List of sample points with stored values and gradients
    """
    # Find surrounding samples for interpolation
    # Not given in the sources. [G]
    
    # Use Hermite interpolation to reconstruct the function and its gradient
    # This involves solving for coefficients that match value and gradient at sample points
    # Not given in the sources. [G]
    
    return distance, normal
```

## Subtopics

### Taylor Approximation Fields
Use local Taylor expansions. More expensive query times but higher accuracy. [Link](#how-it-works)

### Least Squares Polynomial Fields
Fit polynomials to local neighborhoods. Balances accuracy and storage. [Link](#how-it-works)

### Hermite Interpolation
Stores partial derivatives directly. Trades query time for accuracy. [Link](#how-it-works)

## Operating characteristics
*   **Time Complexity:** Query time is higher than zero-order SDFs, especially for Hermite interpolation [S1, abstract only, unofficial].
*   **Memory Complexity:** Depends on the order and the number of coefficients stored per sample. Higher order requires more storage per sample but may allow for coarser sampling grids.
*   **Performance:** GPU filtering can be utilized for efficient queries on regular grids [S1, abstract only, unofficial].
*   **Accuracy:** Higher order fields provide better accuracy and can infer geometric properties like normals [S1, abstract only, unofficial].
*   **Storage:** Can be more memory efficient than classical ray marching with zero-order SDFs due to larger effective filtering footprint [S1, abstract only, unofficial].
*   **When to use:** When higher accuracy is needed without increasing the sampling density, or when geometric properties like normals are required.
*   **When not to use:** When query performance is critical and lower accuracy is acceptable.

## Background
The method was proposed by Gábor Valasek and Róbert Bán in a preprint from January 2022 [S1, abstract only, unofficial]. It addresses the limitations of classical sphere tracing and zero-order SDFs by leveraging higher-order derivative information.

## Sources
[S1] Enhanced Sphere Tracing | Request PDF - ResearchGate, https://www.researchgate.net/publication/265020476_Enhanced_Sphere_Tracing, fetched 2026-11-03 (abstract only, unofficial)
