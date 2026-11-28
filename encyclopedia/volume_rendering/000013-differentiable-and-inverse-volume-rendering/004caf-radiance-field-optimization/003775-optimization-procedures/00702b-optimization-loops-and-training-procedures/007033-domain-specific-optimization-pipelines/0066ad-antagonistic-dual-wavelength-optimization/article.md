# Antagonistic Dual-Wavelength Optimization

Also called: fixed activation optimization

## Overview
Antagonistic Dual-Wavelength Optimization is a domain-specific training procedure for volumetric additive manufacturing that synthesizes inhibition projector patterns against fixed activation projector patterns to achieve a target volumetric polymerization state [S1, unofficial]. It differs from general radiance-field optimization by explicitly modeling the antagonistic interaction of two distinct wavelengths (activation and inhibition) to define an effective volume, rather than optimizing a single radiance field [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Target mesh geometry, vial geometry, fixed activation projector patterns (pre-computed or provided), and configuration parameters for both activation and inhibition scenes.
- **Outputs:** Optimized inhibition projector patterns, final activation and inhibition volumes, and the combined clamped effective volume.

**Data Structures:**
The method relies on two separate scene representations: one for the activation (polymerization) process and one for the inhibition process.
- **Scene Components:** Each scene contains a projector, sensor, integrator, and target definition.
- **Parameters:** The key trainable parameter is the inhibition pattern data (`params_inhib[patterns_key_inhib]`). The activation pattern data is fixed (`params_poly['projector.active_data']`).
- **Active Pixels Mask:** A boolean or integer mask identifying valid projector pixels, derived from a Radon transform or corner filter.

**Numbered Steps:**
1. **Initialize Scene Components:** Load the target mesh and vial geometry for both activation and inhibition projectors [S1, unofficial].
2. **Compute Target Volume:**
   - If the sensor is surface-aware, compute the volume directly from the sensor data [S1, unofficial].
   - Otherwise, discretize the scene and save the target as `.exr` and `.npy` files [S1, unofficial].
3. **Filter Active Pixels (Optional):**
   - Apply a Radon transform or corner filter integrator to initialize the projector active pixels mask [S1, unofficial].
   - Zero out the pattern data for inactive pixels [S1, unofficial].
   - **Pitfall:** If no active pixels are found, raise a `ValueError` [S1, unofficial].
4. **Setup Optimizer:** Initialize the Adam optimizer for the inhibition patterns only. The activation patterns are held fixed [S1, unofficial].
5. **Optimization Loop:** Iterate for `n_steps` (default 40):
   - **Progressive Rendering:** If progressive rendering is enabled, increase the integrator `max_depth` from the initial low value to the full `max_depth` at step 5 [S1, unofficial].
   - **Update Parameters:** Update inhibition parameters using the optimizer [S1, unofficial].
   - **Render Volumes:**
     - Render the polymerization volume ($vol_{polymerize}$) using the activation scene with random seed $i$ [S1, unofficial].
     - Render the inhibition volume ($vol_{inhibit}$) using the inhibition scene with random seed $i + 1,000,000$ [S1, unofficial].
   - **Compute Effective Volume:** Calculate the effective volume as the positive part of the difference:
     $$ vol_{effective} = \max(0, vol_{polymerize} - vol_{inhibit}) $$
     Alternatively, using dose inputs:
     $$ \text{Loss Input} = \max(\alpha D_{405} - D_{365}, 0) $$
     where $D_{405}$ is the activation dose, $D_{365}$ is the inhibition dose, and $\alpha$ is the activation scale factor [S1, unofficial].
   - **Calculate Loss:** Compare $vol_{effective}$ against the target volume using the specified loss function (e.g., threshold loss) [S1, unofficial].
   - **Backward Pass:** Perform the backward pass and check for convergence. If loss is zero, break the loop [S1, unofficial].
   - **Optimizer Step:** Update the optimizer step [S1, unofficial].
   - **Clamp Patterns:** Clamp inhibition pattern values to be non-negative:
     $$ pattern = \max(\text{detach}(pattern), 0) $$
     [S1, unofficial].
6. **Final Rendering:** After convergence, render final activation and inhibition states using reference settings (higher `spp`, `max_depth_ref`) [S1, unofficial].
7. **Save Results:** Save the resulting volumes and patterns [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_antagonistic(config_poly, config_inhib, output):
    # 1. Initialize scenes
    scene_poly = load_scene(config_poly)
    scene_inhib = load_scene(config_inhib)
    params_poly = mi.traverse(scene_poly)
    params_inhib = mi.traverse(scene_inhib)

    # 2. Setup components
    components_poly = setup_scene_components(config_poly, scene_poly, params_poly, os.path.join(output, "activation"), None)
    components_inhib = setup_scene_components(config_inhib, scene_inhib, params_inhib, os.path.join(output, "inhibition"), None)

    # 3. Validate domains
    validate_domains(config_poly, config_inhib)

    # 4. Filter active pixels (if needed)
    if 'filter_radon' in config_poly:
        radon_integrator = mi.load_dict({'type': 'radon'} | config_poly['filter_radon'])
        radon = mi.render(scene_poly, integrator=radon_integrator, spp=1)
        active_pixels = dr.compress(radon.array > 0.) + dr.opaque(mi.UInt32, 0)
        dr.eval(active_pixels)
        if len(active_pixels) == 0:
            raise ValueError("No active pixels found in the Radon transform.")
        params_poly['projector.active_pixels'] = active_pixels
        params_poly['projector.active_data'] = dr.zeros(mi.Float, dr.width(active_pixels))
        params_poly.update()
        del radon, radon_integrator
        dr.flush_malloc_cache()
        dr.sync_thread()
    elif 'filter_corner' in config_poly:
        corner_integrator = mi.load_dict({'type': 'corner', 'regular_sampling': True} | config_poly['filter_corner'])
        corner = mi.render(scene_poly, integrator=corner_integrator, spp=1)
        active_pixels = dr.compress(corner.array > 0.) + dr.opaque(mi.UInt32, 0)
        dr.eval(active_pixels)
        if len(active_pixels) == 0:
            raise ValueError("No active pixels found in the corner filter.")
        params_poly['projector.active_pixels'] = active_pixels
        params_poly['projector.active_data'] = dr.zeros(mi.Float, dr.width(active_pixels))
        params_poly.update()
        del corner, corner_integrator
        dr.flush_malloc_cache()
        dr.sync_thread()

    # 5. Setup loss
    if "loss" not in config_poly:
        config_poly['loss'] = {'type': 'threshold'}
    loss_type = config_poly['loss'].pop('type')
    loss_fn = losses[loss_type](config_poly['loss'])

    # 6. Setup optimizer for inhibition only
    patterns_key_inhib = components_inhib['patterns_key']
    optimizer_config = config_inhib.get("optimizer", {"type": "adam", "lr": 0.02})
    opt_inhib = mi.ad.Adam(**optimizer_config)
    opt_inhib[patterns_key_inhib] = params_inhib[patterns_key_inhib]

    n_steps = config_inhib.get('n_steps', 40)
    poly_scale = config_poly.get('activation_scale', 1.0) # alpha
    params_poly['projector.active_data'] = poly_scale * components_poly['patterns_fwd'].flatten()
    params_poly.update()

    target = components_poly['target']

    # 7. Optimization loop
    for i in trange(n_steps):
        if components_poly['progressive'] and i == 5:
            components_poly['integrator'].max_depth = components_poly['max_depth']
        if components_inhib['progressive'] and i == 5:
            components_inhib['integrator'].max_depth = components_inhib['max_depth']

        with dr.scoped_set_flag(dr.JitFlag.KernelHistory, True):
            params_inhib.update(opt_inhib)

            vol_polymerize = mi.render(
                scene_poly, params_poly,
                integrator=components_poly['integrator'],
                sensor=components_poly['sensor'],
                spp=components_poly['spp'],
                spp_grad=components_poly['spp_grad'],
                seed=i
            )
            dr.schedule(vol_polymerize)

            vol_inhibit = mi.render(
                scene_inhib, params_inhib,
                integrator=components_inhib['integrator'],
                sensor=components_inhib['sensor'],
                spp=components_inhib['spp'],
                spp_grad=components_inhib['spp_grad'],
                seed=i + 1000000
            )
            dr.schedule(vol_inhibit)

            dr.sync_thread()

            # Compute effective volume
            vol_effective = dr.maximum(vol_polymerize - vol_inhibit, 0)

            # Calculate loss
            loss = loss_fn(vol_effective, target)

            if loss == 0:
                break

            loss.backward()
            opt_inhib.step()

            # Clamp inhibition patterns
            params_inhib[patterns_key_inhib] = dr.maximum(dr.detach(params_inhib[patterns_key_inhib]), 0)
            params_inhib.update()

    # 8. Final rendering
    integrator_final = mi.load_dict({
        'type': 'volumeintegrator',
        'max_depth': config_inhib.get('max_depth_ref', 16),
        'spp': config_inhib.get('spp_ref', 16)
    })
    
    vol_final_poly = mi.render(scene_poly, params_poly, integrator=integrator_final, sensor=components_poly['sensor'])
    vol_final_inhib = mi.render(scene_inhib, params_inhib, integrator=integrator_final, sensor=components_inhib['sensor'])

    vol_combined = dr.maximum(vol_final_poly - vol_final_inhib, 0)

    # Save results
    save_vol(vol_final_poly, os.path.join(output, "vol_final_poly.exr"))
    save_vol(vol_final_inhib, os.path.join(output, "vol_final_inhib.exr"))
    save_vol(vol_combined, os.path.join(output, "vol_combined_clamped.exr"))

    return vol_final_poly, vol_final_inhib, vol_combined
```

**Parameters:**
- `activation_scale` ($\alpha$): Scaling factor for the activation dose. Set via command-line argument `--activation-scale` [S1, unofficial].
- `spp`: Samples per pixel for gradient rendering. Default: 4 [S1, unofficial].
- `spp_ref`: Samples per pixel for final reference rendering. Default: 16 [S1, unofficial].
- `n_steps`: Number of optimization steps. Default: 40 [S1, unofficial].
- `max_depth`: Maximum ray depth for gradient rendering. Default: 6 [S1, unofficial].
- `rr_depth`: Russian Roulette depth. Default: 6 [S1, unofficial].
- `progressive`: Boolean flag for progressive rendering. Default: False [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Progressive Rendering:** If enabled, the integrator `max_depth` is increased from the initial low value to the full `max_depth` at step 5 [S1, unofficial].
- **Surface-Aware Sensor:** The final sensor must not be surface-aware; if it is, a `ValueError` is raised [S1, unofficial].
- **No Active Pixels:** If no active pixels are found in the Radon transform or corner filter, a `ValueError` is raised [S1, unofficial].
- **Domain Mismatch:** Activation and inhibition reconstruction grids (resx, resy, resz) and scales must match, or a `ValueError` is raised [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of optimization steps (`n_steps`) and the cost of rendering two volumes per step.
- **Memory Complexity:** Proportional to the scene size and the number of projector patterns. Two separate scenes are maintained in memory.
- **Strengths:** Explicitly handles the antagonistic nature of dual-wavelength polymerization, allowing for precise control over the effective volume by balancing activation and inhibition.
- **Limitations:** Requires pre-computed or fixed activation patterns. The optimization is sensitive to the choice of loss function and the scaling factor $\alpha$.
- **When to Use:** When the manufacturing process involves two distinct wavelengths with antagonistic effects (e.g., polymerization and inhibition) and the activation pattern is known or fixed.
- **When Not to Use:** When a single-wavelength process is sufficient or when both activation and inhibition patterns need to be optimized simultaneously.

## Background
Antagonistic Dual-Wavelength Optimization was introduced by the EPFL Laboratory for Photonic and Integrated Systems (LAPD) in the context of tomographic volumetric additive manufacturing [S1, unofficial]. It addresses the challenge of creating complex 3D structures by leveraging the interaction between two light wavelengths, one promoting polymerization and the other inhibiting it. This method allows for the creation of structures with high resolution and complex geometries that are difficult to achieve with single-wavelength approaches.

## Sources
[S1] EPFL-LAPD/Antagonistic-Dual-Wavelength-Tomographic-Volumetric-Additive-Manufacturing, https://github.com/EPFL-LAPD/Antagonistic-Dual-Wavelength-Tomographic-Volumetric-Additive-Manufacturing, fetched 2026-10-26 (unofficial)
