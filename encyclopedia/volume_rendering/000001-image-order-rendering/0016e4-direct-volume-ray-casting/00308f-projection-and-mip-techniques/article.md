# Projection and MIP Techniques
Also called: MIP (Maximum Intensity Projection)

## Overview
This family encompasses volume rendering techniques that project 3D scalar data onto a 2D image plane by aggregating values along viewing rays, rather than compositing optical properties via the emission-absorption model. Unlike standard Direct Volume Ray Casting [19-direct-volume-ray-casting.md] which preserves depth cues through opacity accumulation, these methods discard depth information to emphasize specific intensity ranges (high or low) or total density, solving the problem of visualizing high-contrast structures like vessels or bones against noisy backgrounds [S1, unofficial][S2][S3, unofficial].

## Shared principles
All methods in this family share the following core mechanism, building on [Direct volume ray casting](19-direct-volume-ray-casting.md):

1.  **Ray Definition**: For each pixel $(x, y)$, a ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined and intersected with the volume bounding box to determine $t_{near}$ and $t_{far}$ [Parent].
2.  **Sampling**: The ray segment is discretized into $N$ samples at positions $\mathbf{p}_i$. Scalar values $s_i$ are retrieved via trilinear interpolation [Parent].
3.  **Aggregation**: Instead of the front-to-back compositing equation, a specific aggregation operator $A$ is applied to the sequence of scalar values $\{s_1, \dots, s_N\}$:
    *   **Maximum**: $A = \max_i(s_i)$
    *   **Minimum**: $A = \min_i(s_i)$
    *   **Summation**: $A = \sum_i s_i \cdot \Delta t$ (Line Integral)
    *   **Approximation**: $A \approx \text{OrthographicProjection}(V)$
4.  **Classification**: The aggregated scalar $A$ is mapped to a color $\mathbf{c}$ using a transfer function $T(A) \rightarrow \mathbf{c}$ [Parent]. Opacity is typically set to $1.0$ or derived from the transfer function, but depth is not accumulated [Parent].

## Subtopics

### [Maximum Intensity Projection](902-maximum-intensity-projection.md)
Projects the maximum scalar value along each ray. It highlights high-intensity structures (e.g., contrast-enhanced vessels) while suppressing lower-intensity background. It is order-independent and discards depth cues.

### [Projected Slabs Approximation](1449-projected-slabs-approximation.md)
Accelerates perspective rendering by partitioning the view frustum into parallel slabs, approximating perspective projection with orthographic-like processing to leverage hardware acceleration.

### [Forward Projection (aTx)](1492-forward-projection-atx.md)
Computes the line integral of density along each ray, simulating X-ray attenuation. It aggregates all encountered densities, providing a cumulative measure of material thickness.

### [Minimum Intensity Projection](5299-minimum-intensity-projection.md)
Projects the minimum scalar value along each ray. It is used to visualize low-density structures (e.g., air-filled lungs) that are obscured by high-density tissues in MIP.

### [Projection](7205-projection.md)
Sums voxel values along the projection axis. It visualizes total accumulated intensity, decreasing noise but reducing contrast due to background accumulation.

### [Maximum Intensity Projection (MIP)](10702-maximum-intensity-projection-mip.md)
A specific implementation of MIP that projects the maximum scalar value, discarding depth information and losing the ability to differentiate front/back structures with identical max intensity.

### Perspective Approximation via Orthogonal Slices
An approximation for perspective volume rendering that uses view-orthogonal slices instead of concentric spherical shells, which are expensive to generate [S2]. The only difference from exact ray casting is that the distances of sample positions along a ray depend on the ray's orientation [S2]. Exact perspective ray casting produces associated sample positions forming concentric spherical shells, not planes [S2].

### ProjectionVolumeStyle
A non-composable render style. An example image shows Bronchial tubes rendered with maximum intensity projection (MIP) [S3, unofficial].

## Background
Originating from the need for fast visualization of volumetric data, particularly in medical imaging (CT, MRI, Angiography), these techniques predate or run parallel to full ray casting. MIP became standard in diagnostic radiology for viewing contrast-enhanced vasculature [S1, unofficial][S2]. The "Projected Slabs" approximation was introduced to enable interactive perspective rendering on early graphics hardware by avoiding expensive spherical shell generation [S2].

## Sources
[S1] GPU Raycast Volume Rendering with Transfer ... - Visualization Library, https://visualizationlibrary.org/documentation/pag_guide_raycast_volume.html, fetched 2026-10-26 (unofficial)
[S2] Interactive Volume Rendering Using 3D Texture-Mapping ..., https://web.cs.ucdavis.edu/~okreylos/PhDStudies/Winter2000/TextureMapping.html, fetched 2026-10-27
[S3] instantreality 1.0 - tutorial - Volume Rendering, https://doc.instantreality.org/tutorial/volume-rendering/, fetched 2026-10-28 (unofficial)
