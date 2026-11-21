# Depth of Field and Lens Effects

Also called: Camera Blur, Focal Depth, Focal Depth / Camera Blur

## Overview
This method family implements optical effects that deviate from the ideal pinhole camera model, primarily depth of field blur based on focal distance and geometric lens distortions [S1, book][S2, S3, unofficial]. It sets itself apart from sibling ray generation methods by introducing a finite aperture for ray origins, requiring sampling over a 2D area rather than a single point, and by modifying ray directions to simulate optical phenomena like magnification or shift [S1, book][S2, S3, unofficial].

## How it works
**Inputs and Outputs**
Inputs: camera-to-world transformation matrix $\mathbf{C}$, base ray direction $\mathbf{d}_{\text{base}}$, focal point $\mathbf{f}$, aperture radius $r$, noise buffer $\mathcal{N}$, and lens distance parameter $L$ [S2, unofficial]. Outputs: ray origins $\mathbf{o}$, ray directions $\mathbf{d}$, and intersection results [S2, unofficial].

**Data Structures**
- $\mathbf{C}$: $4 \times 4$ affine transformation matrix mapping local camera coordinates to world space [S2, unofficial].
- $\mathcal{N}$: 2D texture or buffer storing precomputed random values for aperture sampling [S2, unofficial].
- $\mathbf{f}$: 3D world-space vector defining the focal plane center [S1, book].
- $r$: scalar defining the aperture radius or disk size [S1, book].
- $L$: scalar controlling the magnitude of the noise-driven offset [S2, unofficial].

**Numbered Steps**
1. Retrieve the base ray direction $\mathbf{d}_{\text{base}}$ and camera matrix $\mathbf{C}$ for the current pixel [S2, unofficial].
2. Sample a 2D offset from the noise buffer $\mathcal{N}$ and scale it by the lens distance parameter $L$ to produce $\mathbf{u}_{\text{local}}$ [S2, unofficial].
3. Transform $\mathbf{u}_{\text{local}}$ to world space using $\mathbf{C}$ to obtain the aperture offset $\mathbf{u}_{\text{world}}$ [S2, unofficial].
4. Compute the displaced ray origin $\mathbf{o} = \mathbf{C} \cdot [0, 0, 0, 1]^T + \mathbf{u}_{\text{world}}$ [S2, unofficial].
5. Calculate the new ray direction $\mathbf{d} = \text{normalize}(\mathbf{f} - \mathbf{o})$ to pass through the focal point [S1, book][S2, unofficial].
6. Perform ray-scene intersection (e.g., AABB intersection) using $\mathbf{o}$ and $\mathbf{d}$ [S2, unofficial].
7. For post-processing glare, apply exponentially weighted line kernels to the accumulated image to simulate polygonal aperture diffraction [S3, unofficial].

**Formulas**
- Ray origin: $\mathbf{o} = \mathbf{C}_{\text{origin}} + \mathbf{u}_{\text{world}}$
- Ray direction: $\mathbf{d} = \frac{\mathbf{f} - \mathbf{o}}{\|\mathbf{f} - \mathbf{o}\|}$
- Offset generation: $\mathbf{u}_{\text{local}} = \mathcal{N}(x, y) \cdot L$

**Parameters**
- Focal point $\mathbf{f}$: typically placed at the primary subject distance [S1, book].
- Focal disk radius $r$: controls blur intensity; not given in the sources.
- Number of frames/samples: set explicitly for focal depth accumulation instead of subframe blending [S1, book].

**Edge Cases and Pitfalls**
- Precision loss occurs when $\mathbf{o}$ approaches $\mathbf{f}$, causing $\mathbf{d}$ to become unstable; clamp the minimum distance to focus [G].
- Noise buffer bounds must be clamped to $[0, 1]$ to prevent out-of-bounds sampling [G].
- Degenerate aperture radius $r=0$ collapses to a pinhole model and should be handled as a fallback [G].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_dof_rays(c2w, dir_base, noise_buffer, len_dis, focal_point):
    # 1. Sample aperture offset from noise buffer
    u_local = noise_buffer.sample() * len_dis
    
    # 2. Transform offset to world space
    u_world = c2w.transform_direction(u_local)
    
    # 3. Compute displaced ray origin
    cam_origin = c2w.transform_point([0.0, 0.0, 0.0])
    origin = cam_origin + u_world
    
    # 4. Calculate new ray direction targeting focal point
    direction = normalize(focal_point - origin)
    
    # 5. Perform intersection
    hit = aabb_intersect(origin, direction)
    return origin, direction, hit
```

## Subtopics

### [Lens Shift](1851-lens-shift.md)
A technique for reusing random samples across cameras with different aperture sizes. It scales the lens coordinate vector by the ratio of the target radius to the base radius. It is chosen when rendering multi-view volumes with varying optical parameters to avoid generating new random samples. It differs from thin lens DOF by focusing on geometric adaptation rather than blur integration.

### [Thin Lens Depth of Field](4922-thin-lens-depth-of-field.md)
Simulates optical blur by sampling a finite aperture disk and directing rays through a focal plane. Objects at the focal distance are sharp; others are blurred. It is chosen for realistic image synthesis. It differs from lens shift by simulating the physical effect of defocus rather than adapting samples between views.

### [Magnifier](6754-magnifier.md)
A volumetric lens effect that magnifies regions of interest by bending rays through a virtual focal point, using a transition region to compress surrounding context. It is chosen for interactive exploration of large volumes. It differs from DOF by preserving sharpness while altering scale, rather than altering sharpness based on distance.

### Depth of Field Ray Tracing
A procedural approach to generating DOF rays using noise buffers and distance-based offsets. It generates an offset using a noise buffer and lens distance, transforms it to world space, and calculates the new ray direction and origin based on the distance to focus. It performs AABB intersection on the resulting rays. This method is distinct from analytical thin lens sampling by relying on noise-driven perturbations rather than uniform disk sampling [S2, unofficial].

### Polygonal Aperture Glare/Star Burst
A post-processing effect that simulates polygonal aperture diffraction patterns (star burst) using exponentially weighted line kernels. It applies these kernels to the scene-linear image before tone mapping to produce N-blade bokeh highlights [S3, unofficial].

## Operating characteristics

## Background
Depth of field simulation in ray tracing was popularized by the thin lens model described in standard computer graphics literature [S1, book]. The procedural noise-driven approach is implemented in a Taichi-based renderer [S2, unofficial]. Polygonal aperture glare effects using exponentially weighted line kernels are integrated into a volumetric rendering engine's post-processing imager [S3, unofficial]. Lens shift techniques for multi-view rendering address sample reuse in volumetric contexts where aperture sizes vary [S1, book].

## Sources
[S1] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S2] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
[S3] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
