# Neural volume rendering

Also called: Neural Radiance Fields (NeRF)

## Overview
Neural volume rendering synthesizes novel views of a scene by optimizing a continuous 5D function (spatial location and viewing direction) to predict volume density and view-dependent color [S1]. It solves the view synthesis problem by replacing explicit geometric representations (like meshes or voxel grids) with a neural network that acts as the volume representation [S1]. Unlike traditional direct volume ray casting which samples a discrete 3D texture, this method queries a coordinate-based Multi-Layer Perceptron (MLP) or a discrete voxel grid of features along camera rays [S1][S3, unofficial]. It sets itself apart from siblings like [Direct Volume Rendering](1184-direct-volume-rendering.md) by learning the appearance and geometry implicitly from images, enabling high-fidelity reconstruction from sparse input views [S1].

## Shared principles
The method relies on the volume rendering integral, discretized into a summation over sample points along a ray [S3, S5, unofficial]. The ray $r$ is parameterized by $r(t) = O + tD$, entering the volume at $t_{start}$ and exiting at $t_{end}$ [Volume Raycasting Discretization](831-volume-raycasting-discretization.md). The data model is sampled using $s(t)$ [Volume Raycasting Discretization](831-volume-raycasting-discretization.md).

The compositing is performed front-to-back [S2, S3, S8, S9, S11, S14, S16, S20, S23, S26, unofficial]. Let $C'_i$ be the accumulated color and $\alpha'_i$ be the accumulated opacity after sample $i$, with initial conditions $C'_0 = 0$ and $\alpha'_0 = 0$ [S9, unofficial]. For each sample $i$ along the ray:
1.  Sample the volume at the current ray position to get a scalar value $d_i$ [S2, S5, unofficial].
2.  Apply a transfer function to map the scalar value to color $C_i$ and opacity $\alpha_i$ [S2, S5, unofficial].
3.  Update accumulated color:
    $$ C'_i = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i $$
4.  Update accumulated opacity:
    $$ \alpha'_i = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i $$
5.  If $\alpha' \ge 1$ or a threshold (typically 0.99) [S2, S18, unofficial], terminate the ray (early ray termination) [S2, S3, S4, unofficial].

Rays are cast from the eyepoint, one for each pixel [S1][S4, unofficial]. The volume is typically represented as a 3D texture [S4, S3, unofficial]. Gradients for shading are calculated using a central difference operator [S6, unofficial]. To avoid normal discontinuities that cause visible rings on smooth surfaces, the gradient is computed using a two-voxel offset rather than a one-voxel offset [S18, unofficial]. Sampling is performed using trilinear interpolation [S1][S5, unofficial]. A fixed random offset is applied to each ray's start position to prevent sample layers from appearing as bands [S18, unofficial]. The final accumulated color is typically corrected with gamma 2.2 [S9, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random
import math

def vec3(x, y, z):
    return [x, y, z]

def pow_vec3(v, exponent):
    return [v[0]**exponent, v[1]**exponent, v[2]**exponent]

def compute_ray_exit(origin, direction, bounds):
    # Slab intersection for axis-aligned box [G]
    t0 = max((bounds[0][i] - origin[i]) / direction[i] if direction[i] != 0 else float('inf') for i in range(3))
    t1 = min((bounds[1][i] - origin[i]) / direction[i] if direction[i] != 0 else float('inf') for i in range(3))
    return max(t0, t1)

def trilinear_interpolate(volume, pos, dims):
    # Standard trilinear interpolation [G]
    vx = pos[0] * (dims[0] - 1)
    vy = pos[1] * (dims[1] - 1)
    vz = pos[2] * (dims[2] - 1)
    x0, y0, z0 = int(math.floor(vx)), int(math.floor(vy)), int(math.floor(vz))
    x1, y1, z1 = x0 + 1, y0 + 1, z0 + 1
    x0 = max(0, min(x0, dims[0]-1))
    x1 = max(0, min(x1, dims[0]-1))
    y0 = max(0, min(y0, dims[1]-1))
    y1 = max(0, min(y1, dims[1]-1))
    z0 = max(0, min(z0, dims[2]-1))
    z1 = max(0, min(z1, dims[2]-1))
    wx = vx - x0
    wy = vy - y0
    wz = vz - z0
    c00 = volume[x0][y0][z0] * (1-wx) + volume[x1][y0][z0] * wx
    c01 = volume[x0][y0][z1] * (1-wx) + volume[x1][y0][z1] * wx
    c10 = volume[x0][y1][z0] * (1-wx) + volume[x1][y1][z0] * wx
    c11 = volume[x0][y1][z1] * (1-wx) + volume[x1][y1][z1] * wx
    r0 = c00 * (1-wy) + c10 * wy
    r1 = c01 * (1-wy) + c11 * wy
    return r0 * (1-wz) + r1 * wz

def direct_volume_raycast(volume_data, camera, transfer_function, volume_bounds, volume_dims, step_size=0.001, threshold=0.99):
    """
    Renders a volume using fixed-step raycasting with front-to-back compositing.
    """
    h, w = camera.screen_height, camera.screen_width
    image = [[vec3(0.0, 0.0, 0.0) for _ in range(w)] for _ in range(h)]
    
    for y in range(h):
        for x in range(w):
            ray_origin, ray_direction = camera.cast_ray(x, y)
            accumulated_color = vec3(0.0, 0.0, 0.0)
            accumulated_opacity = 0.0
            
            # Ray initialization: fixed random offset to prevent banding [S18, unofficial]
            t = random.uniform(0.0, step_size)
            
            # Compute ray exit distance from volume bounds [G]
            t_end = compute_ray_exit(ray_origin, ray_direction, volume_bounds)
            
            while accumulated_opacity < threshold and t < t_end:
                # Compute current position in texture space [0,1]
                pos = [(ray_origin[i] + t * ray_direction[i]) for i in range(3)]
                # Sample volume at current position using trilinear interpolation [S1][S5, unofficial]
                scalar_value = trilinear_interpolate(volume_data, pos, volume_dims)
                
                # Apply transfer function to get color and opacity [S2, S5, unofficial]
                color, opacity = transfer_function(scalar_value)
                
                # Front-to-back compositing [S2, S5, unofficial]
                accumulated_color[0] += (1.0 - accumulated_opacity) * opacity * color[0]
                accumulated_color[1] += (1.0 - accumulated_opacity) * opacity * color[1]
                accumulated_color[2] += (1.0 - accumulated_opacity) * opacity * color[2]
                accumulated_opacity += (1.0 - accumulated_opacity) * opacity
                
                t += step_size
                
            # Gamma correction [S9, unofficial]
            image[y][x] = pow_vec3(accumulated_color, 1.0/2.2)
            
    return image
```

## Subtopics

### [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md)
Synthesizes novel views by querying a neural radiance field (a coordinate MLP) along camera rays and accumulating outputs using differentiable volume rendering. It solves the view synthesis problem by optimizing a continuous 5D function (spatial location and viewing direction) to predict volume density and view-dependent color. Unlike single-stage uniform sampling, this method employs a two-stage hierarchical sampling strategy: a coarse network estimates density to guide finer sampling in high-density regions, reducing aliasing and improving geometric detail. It operates on a flat batch of rays rather than a spatial grid, enabling efficient training on randomly sampled rays.

### [Iterative Ray Marching with Neural Volume Inference](1494-iterative-ray-marching-with-neural-volume-inference.md)
A CUDA-based rendering method that splits the volume integration of a neural radiance field into multiple kernel launches to handle rays that do not fully converge (reach alpha $\ge 1$) in a single pass. It distinguishes itself from in-shader neural volume ray marching by decoupling the neural network inference from the shading logic, using a counter-based compaction mechanism to track and process only active rays across iterations.

### [In-Shader Neural Volume Ray Marching](1495-in-shader-neural-volume-ray-marching.md)
A direct volume rendering technique where the neural network (MLP) is evaluated directly inside the CUDA kernel for each sample point, rather than pre-decoding the volume into a 3D texture or using sample-streaming. It supports multiple shading modes including gradient shading and single-shade heuristics. Unlike [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md) which often relies on hierarchical sampling strategies, this method performs a uniform or jittered march through the volume bounding box, evaluating the neural field on-the-fly to determine density and color at each step.

### [DVGO Volume Rendering Forward Pass](2363-dvgo-volume-rendering-forward-pass.md)
Renders a 3D scene represented by a direct voxel grid of density and color features into 2D images by casting rays and accumulating transmittance-weighted colors. It differs from NeRF by replacing the continuous MLP density query with a discrete, hardware-friendly voxel grid lookup, enabling faster training and inference while maintaining view-dependent color modeling.

### [MLP with Positional Encoding](2416-mlp-with-positional-encoding.md)
A neural rendering module that predicts RGB color from input features, view directions, and spatial positions using a Multi-Layer Perceptron (MLP) with positional encoding applied to both position and view direction. It differs from sibling methods like [NeRF Volumetric Rendering](979-nerf-volumetric-rendering-coarse-fine-sampling.md) by explicitly handling view-dependent appearance through concatenated positional encodings of the view vector, whereas simpler variants may assume view-independent color or use different encoding schemes.

### [Volumetric Primitive Radiance Field Integrator](2536-volumetric-primitive-radiance-field-integrator.md)
A path-tracing method for rendering scattering and emissive media represented by mixtures of kernel-based volumetric primitives (e.g., Gaussians, Epanechnikov) rather than voxel grids or point clouds. It modifies standard volumetric path tracing by replacing medium interaction routines with primitive-specific transmittance evaluation, free-flight distance sampling, and albedo evaluation. It distinguishes itself from its sibling, the [Volumetric Primitive Tomography Integrator](2542-volumetric-primitive-tomography-integrator.md), by supporting scattering and emissive media via multiple importance sampling with PDF balancing, whereas the tomography integrator integrates density with no scattering. It builds on [Neural volume rendering](1576-neural-volume-rendering.md) by using explicit geometric primitives instead of implicit neural fields for density and emission.

### [Neural Volume Raycasting Debug Visualization](2646-neural-volume-raycasting-debug-visualization.md)
A debugging visualization pipeline that renders rays through a neural volume model, applies activation functions to raw outputs, and visualizes sample points as instanced spheres colored by predicted radiance and filtered by an opacity threshold.

### [Neural Network Volume Classification](5271-neural-network-volume-classification.md)
A method for classifying volume data using an artificial neural network trained on user-painted samples to map voxel properties to an uncertainty value, which is then mapped to opacity for hardware-accelerated rendering.

### [Neural Network Volume Rendering via Fragment Program](6331-neural-network-volume-rendering-via-fragment-program.md)

## Background
Originating from the work on Neural Radiance Fields (NeRF) [S1], this family of methods represents a shift from explicit to implicit scene representations. The core idea is to learn a continuous function that maps 3D coordinates and viewing directions to color and density. This allows for high-quality novel view synthesis from a set of input images without requiring explicit mesh reconstruction or voxelization [S1]. The approach has been extended to various domains, including dynamic scenes, large-scale environments, and differentiable rendering pipelines [S3, unofficial].

## Sources
