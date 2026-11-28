# Distance-Based Thresholding

Also called: sphere test thresholding

## Overview
Distance-based thresholding is a pruning technique used in wavelet-compressed signed-distance volumes to reduce coding size by removing coefficients that do not contribute to the reconstructed surface [S1, S2]. It prevents geometric errors by ensuring that only coefficients whose support intersects the zero-set (the surface) are retained [S1, S2]. It differs from generic magnitude-based thresholding by using spatial distance information rather than coefficient magnitude to determine significance [S1, S2].

## How it works
**Inputs and Outputs:**
- Input: A wavelet-transformed signed-distance volume (multiresolution decomposition), the original or reconstructed distance field values $d(\cdot)$, and wavelet filter parameters.
- Output: A pruned set of wavelet coefficients where insignificant values are set to zero.

**Data Structures:**
- Wavelet subbands: Each subband contains coefficients corresponding to specific frequency bands and orientations.
- Distance field: A 3D grid of signed distance values $d(x, y, z)$.

**Parameters:**
- $r$: Radius of the bounding sphere for the wavelet support. This depends only on the lengths of the wavelet filters and the number of filter applications [S1].
- $s$: The sample location in the reconstructed volume corresponding to the center of the wavelet basis function [S1, S2].

**Steps:**
1. Iterate through each wavelet subband.
2. For the current subband, determine the spatial support of the wavelets with respect to the reconstructed samples [S1, S2].
3. Compute the radius $r$ of a bounding sphere that contains the wavelet support [S1, S2].
4. For each wavelet coefficient in the subband:
    a. Compute the sample location $s$ in the reconstructed volume corresponding to the center of the wavelet basis function [S1, S2].
    b. Sample the distance value $d(s)$ at location $s$ [S1, S2].
    c. Perform the **Sphere Test**: If $d(s) > r$, the surface does not intersect the bounding sphere. Set the wavelet coefficient to zero [S1, S2].
    d. If $d(s) \le r$, the surface may intersect the sphere. Perform the **Support Sign Test**: Examine the distance values at all sample points within the wavelet support [S2]. If all distance values in the support have the same sign (all positive or all negative), the surface does not pass through the support. Set the coefficient to zero [S2].
    e. Otherwise, retain the coefficient [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distance_based_thresholding(wavelet_subbands, distance_field, wavelet_support_radii):
    """
    Prunes wavelet coefficients in a signed-distance volume based on spatial distance.
    
    Args:
        wavelet_subbands: List of subbands, each containing a list of coefficients.
                          Each coefficient has attributes: center_location, support_samples.
        distance_field: Function or grid lookup d(x, y, z) returning signed distance.
        wavelet_support_radii: Dictionary mapping subband index to radius r.
        
    Returns:
        Pruned wavelet subbands with insignificant coefficients set to 0.
    """
    for subband_index, subband in enumerate(wavelet_subbands):
        r = wavelet_support_radii[subband_index]
        
        for coefficient in subband:
            # Step 4a: Get center location s
            s = coefficient.center_location
            
            # Step 4b: Sample distance at center
            d_s = distance_field(s.x, s.y, s.z)
            
            # Step 4c: Sphere Test
            if d_s > r:
                coefficient.value = 0
                continue
            
            # Step 4d: Support Sign Test (only if sphere test fails)
            # Check if surface intersects the support
            all_same_sign = True
            first_sign = None
            
            for sample_point in coefficient.support_samples:
                d_val = distance_field(sample_point.x, sample_point.y, sample_point.z)
                
                # Determine sign (0 is treated as crossing or boundary)
                # If any value is 0, the surface intersects
                if d_val == 0:
                    all_same_sign = False
                    break
                
                if first_sign is None:
                    first_sign = (d_val > 0)
                elif (d_val > 0) != first_sign:
                    all_same_sign = False
                    break
            
            if all_same_sign:
                coefficient.value = 0
            # Else: retain coefficient
```

**Edge Cases and Pitfalls:**
- **Spurious Components:** Aggressive thresholding can result in spurious surface components (new handles or islands) appearing in the distance field [S1, S2].
- **Lack of Formal Proof:** There is no formalism provided to prove that new components or handles are not added under this method [S1, S2].
- **Verification:** For complicated surfaces, a verification step is recommended that compares the original distance volume against the reconstructed distance volume after thresholding to warn of irregularities [S1, S2].
- **More Accurate Test:** A more accurate test explicitly checking distance samples in the wavelet support for zero-set membership was noted as potentially more expensive but was not implemented in the source [S1].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of wavelet coefficients. The sphere test is $O(1)$ per coefficient. The support sign test requires sampling all points in the wavelet support, which is constant but depends on filter length [S1, S2].
- **Memory Complexity:** No additional memory required beyond the distance field and wavelet coefficients [S1, S2].
- **Strengths:** Reduces coding size efficiently by removing geometrically irrelevant coefficients [S1, S2]. Simple to implement [S1].
- **Limitations:** Risk of introducing topological errors (spurious components) [S1, S2]. No formal guarantee of topological preservation [S1, S2].
- **When to use:** When efficient compression of signed-distance volumes is required and some risk of topological error is acceptable or can be mitigated by verification [S1, S2].
- **When not to use:** When strict topological preservation is required without post-hoc verification [S1, S2].

## Background
Distance-based thresholding was introduced in the context of multiresolution distance volumes for progressive surface compression [S1, S2]. It is part of a system that stores surfaces as wavelet-compressed signed-distance volumes, enabling representation of surfaces with complex topology [Parent]. The method relies on the property that wavelets decay in magnitude and have local support, allowing spatial tests to determine significance [S1, S2].

## Sources
[S1] Multiresolution Distance Volumes for Progressive Surface Compression, https://digital.library.unt.edu/ark:/67531/metadc1409821/?q=volume data compression rendering, fetched 2026-10-28
[S2] Multiresolution Distance Volumes for Progressive Surface Compression, https://digital.library.unt.edu/ark:/67531/metadc1409942/?q=volume data compression rendering, fetched 2026-10-28
