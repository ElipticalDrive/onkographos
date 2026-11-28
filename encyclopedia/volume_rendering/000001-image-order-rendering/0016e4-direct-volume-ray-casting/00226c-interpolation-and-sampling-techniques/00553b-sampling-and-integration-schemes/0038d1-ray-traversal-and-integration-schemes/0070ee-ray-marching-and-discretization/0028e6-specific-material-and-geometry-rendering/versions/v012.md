# Specific Material and Geometry Rendering

Also called: Material Parameter Setting, DVGO Density and Color Rendering, Procedural Volumetric Cloud Rendering, Mr-Voxel Volume Rendering, Ray-marched Volumetric Fog, Surface and DRR Rendering Modes, Material Percentage Volume Rendering, Volumetric Heat Haze, Volumetric Geometry Rendering

## Overview
Specific Material and Geometry Rendering is a family of techniques that define how volumetric data is mapped to optical properties (density, color, emission) and how specific geometric constraints or material compositions are handled during ray traversal. It solves the problem of translating abstract simulation data or procedural definitions into physically plausible or artistically controlled radiance contributions within the volume rendering integral. It sets itself apart from general sampling schemes by focusing on the *definition* of material properties [5335-material-parameter-setting.md], the handling of complex global geometries like planetary cloud shells [3480-procedural-volumetric-cloud-rendering.md], or the specific compositing of multi-material scalar fields [S2, unofficial].

## Shared principles
The family shares the fundamental volume rendering equation, where the final color $C(r)$ along a ray $r$ is the integral of emitted and in-scattered light attenuated by transmittance:
$$
C(r) = \int_{t_n}^{t_f} T(t) \sigma_t(t) c(t) dt
$$
where $T(t) = \exp(-\int_{t_n}^{t} \sigma_t(\tau) d\tau)$ is the transmittance, $\sigma_t$ is the extinction coefficient (density), and $c(t)$ is the color/emission term [5335-material-parameter-setting.md].

Key shared mechanisms include:
1.  **Material Definition:** Mapping raw data (scalar grids, noise fields) to $\sigma_t$ and $c(t)$ via transfer functions, procedural noise, or neural networks [5335-material-parameter-setting.md][12839-dvgo-density-and-color-rendering.md].
2.  **Geometric Constraints:** Defining the spatial bounds of the volume, whether as a local box, a spherical shell, or a procedural mask [3480-procedural-volumetric-cloud-rendering.md][4248-ray-marched-volumetric-fog.md].
3.  **Integration:** Discretizing the integral via ray marching, accumulating radiance and transmittance step-by-step [3874-mr-voxel-volume-rendering.md][4248-ray-marched-volumetric-fog.md].

## Subtopics

### [Procedural Volumetric Cloud Rendering](3480-procedural-volumetric-cloud-rendering.md)
A real-time technique for rendering planetary-scale cloud layers using ray marching through 3D noise fields. It treats clouds as a global spherical shell rather than a local box, solving the problem of rendering large, physically plausible cloudscapes that conform to a planet's curvature. It distinguishes itself from grid-based methods by using procedural noise for density rather than sampled voxel data.

### [Mr-Voxel Volume Rendering](3874-mr-voxel-volume-rendering.md)
A direct volume ray casting method that performs ray marching through a single volume region, accumulating emitted light and computing transmittance and scattering contributions from a single light source. It is distinguished by its specific implementation of Russian Roulette for early termination and its assumption of a single-volume, single-light scene structure.

### [Ray-marched Volumetric Fog](4248-ray-marched-volumetric-fog.md)
Simulates aerial perspective and light shafts by treating fog as a 3D volume of participating media traced via forward ray-marching from the camera to the scene. It solves the problem of rendering dynamic, light-interacting fog that absorbs and scatters light, distinguishing itself from backward ray-marching by allowing early-out optimizations when transmittance becomes negligible.

### [Material Parameter Setting](5335-material-parameter-setting.md)
Defines the optical and geometric properties of volumetric media by mapping raw data grids to physical attributes such as density, color, and emission. It differs from sibling methods by focusing on the *definition* of these properties via explicit channel bindings and parameter sliders (Standard Volume Material) or low-level API calls (VolPack), rather than the traversal or shading algorithms themselves.

### [DVGO Density and Color Rendering](12839-dvgo-density-and-color-rendering.md)
DVGO (Direct Volume Gaussian Optimization) renders 3D scenes by combining a sparse voxel grid with neural networks to represent density and view-dependent color. It solves the problem of efficiently synthesizing novel views by decoupling geometry (density) from appearance (color). It is distinct from standard NeRF by using a voxel grid for density and a shallow MLP for color, allowing for faster training and inference while maintaining high fidelity.

### Surface and DRR Rendering Modes
*   **Definition:** Surface rendering mode computes color $C(s)$ according to the Phong equation using the volume gradient. Digitally Reconstructed Radiographs (DRRs) mode sets $C(s) = (0,0,0)$, attenuating only the backlight by accumulated transmittance $t$. Maximum Intensity Projections (MIPs) output the opacity $\alpha$ of the maximum-opacity interval along the ray [S1, unofficial].
*   **Parameter:** A uniform variable tells the fragment shader which rendering mode to use [S1, unofficial].

### Material Percentage Volume Rendering
*   **Definition:** A technique for rendering volumes with mixtures of materials (e.g., bone, muscle) by creating material percentage volumes and compositing them [S2, unofficial].
*   **Steps:** Assume the scalar field was sampled above Nyquist frequency or low-pass filtered. Differentiate materials by scalar value or additional composition info. Create material percentage volumes, where each is a scalar field representing one material. Associate color and opacity with each material. Obtain composite color and opacity by linearly combining the color and opacity for each material percentage volume. Use a matte volume (scalar field 0-1) to slice the volume or perform spatial set operations. Transform the volume so one axis is perpendicular to the image plane. Project plane by plane in a back-to-front manner and composite to form the final image [S2, unofficial].

### Volumetric Heat Haze
*   **Definition:** Produces the same result as Volumetric Geometry but adds a heat haze refraction effect. Requires V-Ray. Uses Surface channel gradient to change ray direction [S3].
*   **Parameter:** Heat Haze (`hhfactor`): Multiplier for refraction based on Surface channel gradient. 1 approximates normal air heat haze from temperature. 0 produces no effect. Larger values needed if using smoke/other channels as source. Only available in Volumetric Heat Haze mode [S3].
*   **Pitfall:** Heat Haze only affects camera view, not shadows cast through volume. If Heat Haze shader intersects with VRayMtl refraction, Max depth of VRayMtl may need increase [S3].

### Volumetric Geometry Rendering
*   **Definition:** Produces the same visual result as Volumetric mode but uses procedural geometry made of multiple transparent layers. Requires V-Ray. Used for exporting deep images and render elements (normals, velocity, multi matte) not available in standard Volumetric mode [S3].
*   **Pitfall:** Approximate and Approximate+Shadows options for Scattering in Smoke color window are not supported in this mode. Cutter Geom does not work when Mode is set to Volumetric Geometry [S3].

## Background
The concept of rendering specific materials within volumes dates back to early volume visualization work. Drebin, Carpenter, and Hanrahan developed techniques for rendering volumes containing mixtures of materials, such as CT data containing bone, muscle, and flesh, by creating material percentage volumes [S2, unofficial]. Modern implementations extend this to procedural generation for clouds [3480-procedural-volumetric-cloud-rendering.md] and neural representations like DVGO [12839-dvgo-density-and-color-rendering.md]. Specific rendering modes like DRRs and MIPs are standard in medical and industrial visualization contexts [S1, unofficial], while heat haze and volumetric geometry modes are specialized features in production renderers like V-Ray for 3ds Max [S3].

## Sources
[S1] CISC/CMPE 454, GPU Volume Rendering, https://watkins.cs.queensu.ca/~jstewart/454/notes/44/44-gpu-volume-rendering.html, fetched 2026-10-30 (unofficial)
[S2] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S3] FireSmoke Rendering - Phoenix for 3ds Max, https://documentation.chaos.com/space/PHX4MAX/124396558, fetched 2026-10-27
