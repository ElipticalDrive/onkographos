# Probe Sampling

Also called: SampleProbes

## Overview
Probe Sampling is a method for evaluating pre-computed irradiance data stored in a sparse grid, typically using Spherical Harmonics (SH) coefficients, to determine lighting attributes at arbitrary world positions. It solves the problem of efficiently retrieving view-dependent lighting information for volume rendering or global illumination by interpolating between discrete probe locations. Unlike generic operators that compute local gradients or boundaries from scalar voxel data, Probe Sampling retrieves pre-baked radiometric values, applying visibility weighting to occlude probes that are blocked by intervening geometry [S1, unofficial].

## How it works
**Inputs:**
- `WorldPosition`: The 3D world coordinate where irradiance is requested.
- `N`: The surface normal vector at `WorldPosition` (used for SH evaluation).
- `u_ProbeBoxOrigin`: The minimum corner of the bounding box containing the probe grid.
- `u_ProbeBoxSize`: The dimensions of the bounding box.
- `u_ProbeGridResolution`: The resolution of the probe grid (e.g., $N_x \times N_y \times N_z$).
- `Nudge`: A boolean flag indicating whether to offset the sampling position along the normal to prevent self-intersection or sampling artifacts.

**Outputs:**
- `FinalIrradiance`: A 3D vector representing the final irradiance color.

**Data Structures:**
- **Probe Grid:** A 3D texture or array storing SH coefficients. Each texel contains a set of coefficients (typically 9 for SH3, but the implementation details depend on the SH order used).
- **TexelCoordinates:** Integer indices $(x, y, z)$ identifying specific probes in the grid.

**Parameters:**
- `NudgeScale`: Fixed constant $0.4$ used for normal offsetting [S1, unofficial].
- `Epsilon`: Fixed constant $0.000001$ used to prevent division by zero during normalization [S1, unofficial].

**Steps:**
1. **Position Nudging:** If `Nudge` is true, adjust the sampling position to avoid sampling from inside the surface or causing self-occlusion artifacts.
   $$ \text{WorldPosition} \leftarrow \text{WorldPosition} + \mathbf{N} \times 0.4 $$
2. **Coordinate Transformation:** Convert the world position into the probe grid's normalized sample space $[0, 1]$.
   $$ \text{SamplePoint} = \frac{\text{WorldPosition} - \text{u\_ProbeBoxOrigin}}{\text{u\_ProbeBoxSize}} $$
   $$ \text{SamplePoint} = \text{SamplePoint} \times 0.5 + 0.5 $$
   *Note: If any component of `SamplePoint` is outside $[0, 1]$, the function returns zero irradiance.*
3. **Volume Coordinates:** Map the normalized sample point to the discrete grid coordinates.
   $$ \text{VolumeCoords} = \text{SamplePoint} \times \text{u\_ProbeGridResolution} $$
4. **Box Determination:** Identify the 8 surrounding voxels in the grid.
   $$ \text{MinSampleBox} = \lfloor \text{VolumeCoords} \rfloor $$
   $$ \text{MaxSampleBox} = \lceil \text{VolumeCoords} \rceil $$
5. **Trilinear Weights:** Compute the trilinear interpolation weights for the 8 corners based on the fractional part of `VolumeCoords`. Let $w_i$ be the weight for corner $i$ ($i=0..7$).
6. **Texel Mapping:** Map the min/max combinations of x, y, z to the 8 texel coordinates `TexelCoordinates[0..7]`.
7. **Visibility-Weighted Interpolation:**
   Initialize `Alpha = 0.0` and `FinalSH` as an empty vector.
   Loop $i$ from 0 to 7:
   a. Fetch SH coefficients: $\text{sh}[i] = \text{GetSH}(\text{TexelCoordinates}[i])$.
   b. Compute visibility: $\text{ProbeVisibility} = \text{GetVisibility}(\text{TexelCoordinates}[i], \text{WorldPosition}, \mathbf{N})$. This step typically involves raycasting or depth testing to determine if the probe is occluded from the viewpoint or surface.
   c. Accumulate alpha: $\text{Alpha} \leftarrow \text{Alpha} + w_i \times (1.0 - \text{ProbeVisibility})$.
   d. Update weights: $w_i \leftarrow w_i \times \text{ProbeVisibility}$.
8. **Normalization:**
   $$ \text{WeightSum} = 1.0 - \text{Alpha} $$
   Loop $i$ from 0 to 7:
   $$ \text{sh}[i] \leftarrow \text{sh}[i] \times \frac{w_i}{\max(\text{WeightSum}, \text{Epsilon})} $$
   $$ \text{FinalSH} \leftarrow \text{FinalSH} + \text{sh}[i] $$
9. **SH Evaluation:** Evaluate the Spherical Harmonics function using the accumulated coefficients and the normal $\mathbf{N}$.
   $$ \text{FinalIrradiance} = \max(\text{SampleSH}(\text{FinalSH}, \mathbf{N}), 0.0) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ProbeSampling(WorldPosition, N, u_ProbeBoxOrigin, u_ProbeBoxSize, u_ProbeGridResolution, Nudge):
    # 1. Nudge position
    if Nudge:
        WorldPosition = WorldPosition + N * 0.4

    # 2. Transform to sample space
    SamplePoint = (WorldPosition - u_ProbeBoxOrigin) / u_ProbeBoxSize
    SamplePoint = SamplePoint * 0.5 + 0.5

    # Pitfall: Check bounds
    if any(c < 0.0 or c > 1.0 for c in SamplePoint):
        return vec3(0.0)

    # 3. Volume coordinates
    VolumeCoords = SamplePoint * u_ProbeGridResolution

    # 4. Box corners
    MinSampleBox = floor(VolumeCoords)
    MaxSampleBox = ceil(VolumeCoords)

    # 5. Trilinear weights
    # Trilinear function returns 8 weights based on Min/Max and VolumeCoords
    weights = TrilinearWeights(MinSampleBox, MaxSampleBox, VolumeCoords)

    # 6. Map to Texel Coordinates
    TexelCoordinates = GetTexelCoordinates(MinSampleBox, MaxSampleBox)

    # 7. Visibility-weighted accumulation
    Alpha = 0.0
    FinalSH = vec3(0.0) # Assuming SH evaluation results in a vec3 irradiance, 
                        # or FinalSH holds coefficients depending on implementation.
                        # Based on notes, FinalSH accumulates scaled SH data.
    
    # Note: The source implies FinalSH is accumulated from SH data, 
    # but the return value is SampleSH(FinalSH, N). 
    # This implies FinalSH holds the weighted SH coefficients.
    # However, the loop updates 'sh[i]' which are fetched SH data.
    # Let's assume SH coefficients are stored in a structure that can be summed.
    
    WeightedSHCoeffs = [] # List of SH coefficient vectors
    
    for i in range(8):
        sh_coeffs = GetSH(TexelCoordinates[i])
        visibility = GetVisibility(TexelCoordinates[i], WorldPosition, N)
        
        Alpha += weights[i] * (1.0 - visibility)
        weights[i] = weights[i] * visibility
        
        # Store weighted coefficients for later summation
        WeightedSHCoeffs.append(sh_coeffs)

    WeightSum = 1.0 - Alpha
    Epsilon = 0.000001
    
    # 8. Normalize and Sum
    # The source says: Scale sh[i] by Trilinear[i] / max(WeightSum, Epsilon) and add to FinalSH
    # This implies FinalSH is the sum of the normalized weighted SH coefficients.
    
    FinalSH = vec3(0.0) # Or appropriate SH coefficient vector size
    
    # Re-fetch or use stored? The source loop structure suggests doing it in one pass or two.
    # The source code snippet shows:
    # sh[i] = GetSH...
    # Alpha += ...
    # Trilinear[i] *= ...
    # Then later: Scale sh[i] by Trilinear[i] / WeightSum and add to FinalSH.
    # This requires storing sh[i] or re-fetching. We store it.
    
    # Let's rewrite the loop to be strictly Python-like and self-contained as per instructions
    # We need to store the fetched SH data.
    
    fetched_sh = [GetSH(TexelCoordinates[i]) for i in range(8)]
    
    # Re-calculate weights with visibility applied (as per source logic)
    # Note: The source updates weights[i] in the first loop.
    # We must re-run the visibility logic or store it. 
    # To be safe and self-contained, we compute visibility again or store it.
    # Given the constraint of no undefined helpers, we assume GetVisibility is available.
    
    for i in range(8):
        visibility = GetVisibility(TexelCoordinates[i], WorldPosition, N)
        # Recalculate the effective weight used for normalization
        # The source logic: Alpha += w_i * (1-v). w_i is the trilinear weight.
        # Then w_i *= v.
        # Then scale by w_i / WeightSum.
        
        # We need the original trilinear weight. 
        # Let's assume 'weights' array holds the original trilinear weights.
        
        effective_weight = weights[i] * visibility
        normalized_weight = effective_weight / max(WeightSum, Epsilon)
        
        FinalSH = FinalSH + fetched_sh[i] * normalized_weight

    # 9. Evaluate SH
    Irradiance = SampleSH(FinalSH, N)
    return max(Irradiance, 0.0)
```

*Correction on Pseudocode Logic based on Source:*
The source notes describe a specific loop structure.
1. `sh[i] = GetSH(...)`
2. `Alpha += Trilinear[i] * (1.0 - ProbeVisibility)`
3. `Trilinear[i] *= ProbeVisibility`
4. Later: `Scale sh[i] by Trilinear[i] / max(WeightSum, 0.000001f) and add to FinalSH`

This implies `FinalSH` is the sum of the *normalized* SH coefficients.
The `SampleSH` function then evaluates this sum.

Refined Pseudocode:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ProbeSampling(WorldPosition, N, u_ProbeBoxOrigin, u_ProbeBoxSize, u_ProbeGridResolution, Nudge):
    if Nudge:
        WorldPosition = WorldPosition + N * 0.4

    SamplePoint = (WorldPosition - u_ProbeBoxOrigin) / u_ProbeBoxSize
    SamplePoint = SamplePoint * 0.5 + 0.5

    if any(c < 0.0 or c > 1.0 for c in SamplePoint):
        return vec3(0.0)

    VolumeCoords = SamplePoint * u_ProbeGridResolution
    MinSampleBox = floor(VolumeCoords)
    MaxSampleBox = ceil(VolumeCoords)
    
    # TrilinearWeights returns 8 weights corresponding to the 8 corners
    weights = TrilinearWeights(MinSampleBox, MaxSampleBox, VolumeCoords)
    TexelCoordinates = GetTexelCoordinates(MinSampleBox, MaxSampleBox)

    Alpha = 0.0
    fetched_sh = []
    
    for i in range(8):
        sh_coeffs = GetSH(TexelCoordinates[i])
        fetched_sh.append(sh_coeffs)
        
        visibility = GetVisibility(TexelCoordinates[i], WorldPosition, N)
        
        Alpha += weights[i] * (1.0 - visibility)
        weights[i] = weights[i] * visibility

    WeightSum = 1.0 - Alpha
    Epsilon = 0.000001
    
    FinalSH = vec3(0.0) # Assuming SH evaluation yields a vec3, or FinalSH is a vector of coeffs.
                        # The source says "Return max(SampleSH(FinalSH, N), 0.0f)".
                        # This implies FinalSH is the input to SampleSH.
                        # If SampleSH takes coefficients, FinalSH must be a vector of coefficients.
                        # If SampleSH takes irradiance, FinalSH must be irradiance.
                        # Given "Scale sh[i] ... and add to FinalSH", and sh[i] are SH coeffs,
                        # FinalSH is likely a vector of SH coefficients.
                        
    # However, standard SH evaluation sums coefficients weighted by basis functions.
    # Here, the weights are applied to the coefficients directly.
    # This suggests FinalSH accumulates the weighted coefficients.
    
    # Let's assume SH coefficients are vectors of size K (e.g., 9 for SH3).
    # FinalSH should be initialized as a zero vector of size K.
    
    # Since the return is SampleSH(FinalSH, N), FinalSH is the coefficient vector.
    
    # Re-initialize FinalSH as zero vector of appropriate size
    FinalSH = zero_vector(size_of_sh_coeffs)

    for i in range(8):
        normalized_weight = weights[i] / max(WeightSum, Epsilon)
        FinalSH = FinalSH + fetched_sh[i] * normalized_weight

    Irradiance = SampleSH(FinalSH, N)
    return max(Irradiance, 0.0)
```

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample, as it involves a fixed number of operations (8 texel fetches, visibility checks, and arithmetic). The cost depends on the complexity of `GetVisibility` (raycasting cost) and `SampleSH` (SH evaluation cost).
- **Memory Complexity:** Depends on the probe grid size. Each probe stores SH coefficients.
- **Strengths:** Fast evaluation compared to real-time ray tracing for indirect lighting. Smooth interpolation between probes reduces noise.
- **Limitations:** Visibility calculation can be expensive if raycasting is used. Artifacts may occur if the probe grid is too sparse or if visibility weights are inaccurate. Self-intersection artifacts are mitigated by nudging.
- **When to use:** When pre-computed lighting is available and real-time evaluation is required. Suitable for static or slowly changing scenes.
- **When not to use:** For dynamic scenes where lighting changes rapidly, or when high-fidelity dynamic shadows are required without pre-computation.

## Background
Probe Sampling is commonly used in real-time rendering engines for global illumination, particularly in techniques like Irradiance Volumes or Probe-Based Global Illumination. It allows for the efficient storage and retrieval of indirect lighting information. The method leverages Spherical Harmonics to represent the angular distribution of light, enabling smooth interpolation and view-dependent shading.

## Sources
[S1] swr06/VolumetricDisplay, https://github.com/swr06/VolumetricDisplay, fetched 2026-10-26 (unofficial)
