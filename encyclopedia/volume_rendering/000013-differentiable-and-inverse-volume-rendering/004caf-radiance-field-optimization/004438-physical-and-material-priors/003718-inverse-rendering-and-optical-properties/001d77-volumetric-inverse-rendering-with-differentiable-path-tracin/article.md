# Volumetric Inverse Rendering with Differentiable Path Tracing

Also called: Differentiable Volumetric Inverse Rendering, Differentiable Volumetric Path Tracing Optimization, Mitsuba 3 Volumetric Inverse Rendering, Mitsuba 3 Volumetric Inverse Rendering Tutorial, Volumetric Inverse Rendering with Differentiable Path Replay Backpropagation, Volumetric Path Tracing with Adjoint Differentiation

## Overview
Volumetric Inverse Rendering with Differentiable Path Tracing is a technique to optimize a 3D volume density (extinction/scattering) stored on a regular grid to match a set of synthetic reference images [S2, S3, unofficial]. It accounts for direct and indirect illumination by using path replay backpropagation to compute derivatives of delta tracking and volumetric multiple scattering [S2, S3, unofficial]. It sets itself apart from its siblings by explicitly optimizing physical volume parameters (grid-based extinction) using a differentiable path tracer (specifically `prbvolpath`) rather than neural field representations [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of reference images $I_{ref}^{(i)}$ captured from $N$ viewpoints; a scene geometry (typically a container with a `null` BSDF); an initial volume density grid $\sigma_t$; and optimization hyperparameters (learning rate, iterations, resolution) [S2, S3, unofficial].
*   **Outputs:** An optimized volume density grid $\sigma_t$ that minimizes the difference between rendered images and reference images [S2, S3, unofficial].

**Data Structures:**
*   **Volume Grid:** A 3D tensor of shape $(V, V, V, 1)$ representing the extinction coefficient $\sigma_t$. In Mitsuba 3, this is stored as a `TensorXf` accessed via the key `'object.interior_medium.sigma_t.data'` [S2, S3, unofficial].
*   **Scene Parameters:** Traversed via `mi.traverse(scene)` to identify differentiable tensors [S2, S3, unofficial].

**Algorithm Steps:**

1.  **Sensor Setup:** Create $N$ sensors (typically $N=5$ for simple scenes, up to 50-100 for complex ones) placed on a half-circle around the origin [S2, S3, unofficial]. For each sensor $i \in [0, N-1]$:
    *   Calculate angle: $\theta_i = \frac{180.0}{N} \cdot i - 90.0$ [S2, S3, unofficial].
    *   Create rotation transform $R_y(\theta_i)$ around the Y-axis [S2, S3, unofficial].
    *   Create look-at transform from target $[0,0,0]$ to camera origin $[0,0,4]$ with up vector $[0,1,0]$ [S2, S3, unofficial].
    *   Combine transforms: $T_{to\_world} = R_y(\theta_i) \cdot T_{look\_at}$ [S2, S3, unofficial].
    *   Load sensor with type `'perspective'`, FOV 45, film resolution $64 \times 64$, and `'tent'` filter [S2, S3, unofficial].

2.  **Reference Rendering:** Render the reference scene (containing the ground-truth volume) using the `prbvolpath` integrator for each sensor with $S_{ref}$ samples per pixel (SPP) [S2, S3, unofficial]. Store results in $I_{ref}$.
    *   Typical $S_{ref} = 512$ [S2, S3, unofficial].

3.  **Optimization Scene Initialization:** Create a scene with a uniform volume grid of resolution $V_{init} \times V_{init} \times V_{init}$ (e.g., $16^3$) [S2, S3, unofficial].
    *   Initialize $\sigma_t$ to a low constant value (e.g., $0.002$) [S2, S3, unofficial]. This facilitates optimization by making it easier to increase density than to remove parts of a very dense volume [S2, S3, unofficial].
    *   Set the container's BSDF to `'null'` to make it invisible [S2, S3, unofficial].
    *   Set the volume `scale` parameter to $40.0$ [S2, S3, unofficial]. This scales the extinction range to $[10^{-6} \cdot \text{scale}, \text{scale}]$ [S2, S3, unofficial].

4.  **Optimizer Setup:**
    *   Traverse the scene to extract the volume data key: `key = 'object.interior_medium.sigma_t.data'` [S2, S3, unofficial].
    *   Initialize an Adam optimizer with learning rate $\eta = 0.02$ [S2, S3, unofficial].
    *   Assign the volume data to the optimizer: `opt[key] = params[key]` [S2, S3, unofficial].

5.  **Optimization Loop:** For each iteration $it$ from $0$ to $I_{count}-1$:
    *   Initialize `total_loss = 0.0`.
    *   For each sensor $j$ from $0$ to $N-1$:
        *   Render the scene using `mi.render(scene, params, sensor=sensors[j], spp=8, seed=it)` [S2, S3, unofficial].
        *   Compute L2 loss: $L_j = \text{mean}((I_{rendered} - I_{ref}^{(j)})^2)$ [S2, S3, unofficial].
        *   Backpropagate gradients: `dr.backward(L_j)` [S2, S3, unofficial].
        *   Take a gradient step: `opt.step()` [S2, S3, unofficial].
        *   Clamp density values: `opt[key] = dr.clip(opt[key], 1e-6, 1.0)` [S2, S3, unofficial].
        *   Update scene parameters: `params.update(opt)` [S2, S3, unofficial].
        *   Accumulate loss: `total_loss += L_j` [S2, S3, unofficial].

6.  **Volume Upsampling (Multi-resolution):**
    *   After initial optimization at low resolution, upsample the volume grid to a higher resolution (e.g., $64^3$) [S2, S3, unofficial].
    *   Use `dr.resample(opt[key], shape=(64, 64, 64, 1), filter='linear')` [S3, unofficial]. Note: `dr.upsample` is also used in some implementations [S2, unofficial], but `resample` with a linear filter is preferred to avoid negative values from cubic overshoot [S3, unofficial].
    *   Update parameters: `params.update(opt)` [S2, S3, unofficial]. The optimizer automatically resets internal state (e.g., momentum) when the tensor size changes [S2, S3, unofficial].
    *   Continue optimization with increased SPP (e.g., $S_{opt} = 8 \cdot S_{ref}$) [S2, S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import drjit as dr
import mitsuba as mi

mi.set_variant('cuda_ad_rgb')

def setup_sensors(sensor_count=5):
    from mitsuba import ScalarTransform4f as T
    sensors = []
    for i in range(sensor_count):
        angle = 180.0 / sensor_count * i - 90.0
        sensor_rotation = T().rotate([0, 1, 0], angle)
        sensor_to_world = T().look_at(target=[0, 0, 0], origin=[0, 0, 4], up=[0, 1, 0])
        sensors.append(mi.load_dict({
            'type': 'perspective',
            'fov': 45,
            'to_world': sensor_rotation @ sensor_to_world,
            'film': {
                'type': 'hdrfilm',
                'width': 64, 'height': 64,
                'filter': {'type': 'tent'}
            }
        }))
    return sensors

def render_reference_images(scene_ref, sensors, ref_spp=512):
    return [mi.render(scene_ref, sensor=sensors[i], spp=ref_spp) for i in range(len(sensors))]

def create_optimization_scene(v_res=16, scale=40.0):
    scene_dict = {
        'type': 'scene',
        'integrator': {'type': 'prbvolpath'},
        'object': {
            'type': 'cube',
            'bsdf': {'type': 'null'},
            'interior': {
                'type': 'heterogeneous',
                'sigma_t': {
                    'type': 'gridvolume',
                    'grid': mi.VolumeGrid(dr.full(mi.TensorXf, 0.002, (v_res, v_res, v_res, 1))),
                    'to_world': mi.ScalarTransform4f().translate(-1).scale(2.0)
                },
                'scale': scale
            }
        },
        'emitter': {'type': 'constant'}
    }
    return mi.load_dict(scene_dict)

def optimize_volume(scene, sensors, ref_images, iteration_count=20, spp=8):
    params = mi.traverse(scene)
    key = 'object.interior_medium.sigma_t.data'
    
    opt = mi.ad.Adam(lr=0.02)
    opt[key] = params[key]
    params.update(opt)
    
    for it in range(iteration_count):
        total_loss = 0.0
        for sensor_idx in range(len(sensors)):
            img = mi.render(scene, params, sensor=sensors[sensor_idx], spp=spp, seed=it)
            loss = dr.mean(dr.square(img - ref_images[sensor_idx]))
            dr.backward(loss)
            opt.step()
            opt[key] = dr.clip(opt[key], 1e-6, 1.0)
            params.update(opt)
            total_loss += loss
    return scene, params

def upsample_volume(opt, key, new_shape=(64, 64, 64, 1)):
    opt[key] = dr.resample(opt[key], shape=new_shape, filter='linear')
```

## Subtopics

### [Inverse Rendering with Differential Trackers](15034-inverse-rendering-with-differential-trackers.md)
An unbiased method for estimating gradients with respect to scattering volume parameters using differential ratio tracking with a majorant, avoiding separate medium control density hyperparameters.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Each iteration requires rendering $N$ views [G].
*   **Memory Complexity:** Not given in the sources. Proportional to the volume grid size ($V^3$) and the number of sensors [G].
*   **Performance:** Measured on standard hardware using Mitsuba 3. Initial low-resolution optimization ($16^3$) is fast, while high-resolution ($64^3$) with high SPP is computationally expensive [S2, S3, unofficial].
*   **Strengths:** Multi-resolution approach improves convexity and recovers both shape and detail [S2, S3, unofficial]. Path replay backpropagation handles indirect illumination and multiple scattering [S2, S3, unofficial].
*   **Limitations:** Requires multiple viewpoints to constrain the volume [S2, S3, unofficial]. Initialization to low density is critical [S2, S3, unofficial]. Upsampling requires careful filter choice to avoid negative densities [S3, unofficial].
*   **When to use:** When optimizing physical volume parameters (extinction/scattering) for novel view synthesis or reconstruction from multi-view images [S2, S3, unofficial].
*   **When not to use:** Not given in the sources. Single-view reconstruction or neural field representations are alternative approaches [G].

## Background
This method is implemented in Mitsuba 3 using the `prbvolpath` integrator, `heterogeneous` volume plugin, and `gridvolume` plugin [S2, S3, unofficial]. It is documented in the Mitsuba 3 tutorial for volumetric inverse rendering [S2, S3, unofficial]. The approach builds on differentiable path tracing techniques, specifically utilizing path replay backpropagation for efficient gradient computation in participating media [S2, S3, unofficial].

## Sources
[S1] facebookresearch/volumetric_primitives, https://github.com/facebookresearch/volumetric_primitives, fetched 2026-10-26 (unofficial)
[S2] Volumetric inverse rendering - Mitsuba 3, https://mitsuba.readthedocs.io/en/stable/src/inverse_rendering/volume_optimization.html, fetched 2026-10-29 (unofficial)
[S3] Volumetric inverse rendering, https://mitsuba.readthedocs.io/en/latest/src/inverse_rendering/volume_optimization.html, fetched 2026-10-29 (unofficial)
