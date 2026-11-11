# Volumetric and Atmospheric Rendering

Also called: Standard 3ds Max atmospheric

## Overview
Volumetric and Atmospheric Rendering visualizes grid content as a standard 3ds Max atmospheric volume, primarily used for fire and smoke [S1]. It solves the problem of rendering participating media without generating explicit mesh surfaces, relying instead on ray marching through a density field to compute color and opacity [S1]. Unlike surface-based rendering modes which generate geometry and allow standard materials, this method handles shading exclusively through Volumetric Render Settings using color gradients and graphs [S1].

## Shared principles
The method operates on the principle that a Signed Distance Function $f(\mathbf{p})$ provides a lower bound on the distance to the nearest surface. For any point $\mathbf{p}$ in space, $f(\mathbf{p})$ returns the shortest Euclidean distance to the implicit surface, with a sign indicating position: $f(\mathbf{p}) < 0$ inside, $f(\mathbf{p}) = 0$ on the surface, and $f(\mathbf{p}) > 0$ outside [S2, S5, 9643].

The core algorithm, known as sphere tracing, proceeds as follows for each pixel:
1.  **Ray Definition**: A ray $r(t) = \mathbf{o} + t\mathbf{d}$ is defined, where $\mathbf{o}$ is the camera origin and $\mathbf{d}$ is the normalized direction vector [G].
2.  **Initialization**: The accumulated distance $t$ is initialized to 0 (or a near-plane value) [S1, S3].
3.  **Marching Loop**:
    *   Evaluate the scene SDF at the current ray position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ to get the distance $d = f(\mathbf{p})$ [S1, S34, unofficial].
    *   **Termination Check**: If $d < \epsilon$ (a small convergence threshold, e.g., $0.001$), the ray has hit the surface [S1, S39, unofficial]. If $t > t_{max}$ (maximum ray distance), the ray has escaped [S2, S8, unofficial].
    *   **Step**: Advance the ray by the distance value: $t \leftarrow t + d$ [S1, S34, unofficial]. This step size is guaranteed not to penetrate the surface because the SDF satisfies the Lipschitz condition with constant 1 [S27][S51, unofficial].
4.  **Shading**: Upon hit, compute surface properties (normal, color) and lighting [S1].

The scene SDF is typically composed of primitives using Boolean operations: union is `min(d1, d2)`, intersection is `max(d1, d2)`, and subtraction is `max(-d1, d2)` [S39, unofficial]. Transformations are applied by mapping the query point $\mathbf{p}$ to the local space of the primitive via $M^{-1}\mathbf{p}$ before evaluation [S3850].

## Subtopics

### [Ray Marching with Single Scattering](2057-ray-marching-with-single-scattering.md)
Focuses on general volumetric single scattering without specific planetary geometry or atmospheric density models. Choose when rendering generic fog or clouds rather than a planet's sky.

### [SDF Cloud Rendering](2064-sdf-cloud-rendering.md)
Uses Signed Distance Fields for cloud geometry rather than exponential density falloff on a sphere. Choose when detailed cloud shapes are required over atmospheric scattering effects.

### [Volume Ray Marching with Temporal Accumulation](4804-volume-ray-marching-with-temporal-accumulation.md)
Adds temporal stability to the ray marching process. Choose when flickering artifacts in dynamic scenes are a concern.

### [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md)
Models planetary atmospheres using spherical geometry and Rayleigh scattering. Choose when rendering global sky effects or planetary atmospheres.

### Wax Volume Raymarching
Raymarches the interior volume of a glass container to render wax [S2, unofficial]. The surface normal is computed using the gradient of the density field, and fragment depth is set explicitly using `gl_FragDepth` in the opaque pass [S2, unofficial]. The shading model includes wrap lighting, soft Fresnel, fake subsurface scattering based on traversed thickness, and emission driven by temperature and proximity to a light bulb [S2, unofficial]. Artifacts are negligible, with 0.00% of surface voxels causing a round-trip to the threshold, even at 60 Hz with ±30% jitter [S2, unofficial].

### X-Ray Visualization
A visualization method that computes average density values along a ray cast through the data, visualizing the result as a grey scale image similar to a medical X-ray [S3]. The view of the data set can be rotated in any of the 3 directions (X, Y, Z axes) in degrees (-360 ... 0 ... 360) [S3].

## Background
The method is part of the Phoenix for 3ds Max suite, specifically the "Volumetric" mode, which visualizes content as a standard 3ds Max atmospheric volume [S1]. It is used mostly for fire and smoke simulations [S1]. Unlike surface render modes that generate a mesh based on a specified channel, volumetric modes do not generate surfaces, meaning 3ds Max and V-Ray materials cannot be used for shading; instead, shading is described in the Volumetric Render Settings window using simple color gradients and graph diagrams [S1].

## Sources
[S1] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
[S2] Étape 4 : surface de cire raymarchée et liquide volumique, https://github.com/nathanddrl/lava-lamp/pull/4, fetched 2026-10-27 (unofficial)
[S3] VOLUME RENDERING Application by Thomas Lidy, https://www.cg.tuwien.ac.at/courses/Visualisierung/2001-2002/Ergebnisse/Beispiel1/LidyT/description.html, fetched 2026-10-29
