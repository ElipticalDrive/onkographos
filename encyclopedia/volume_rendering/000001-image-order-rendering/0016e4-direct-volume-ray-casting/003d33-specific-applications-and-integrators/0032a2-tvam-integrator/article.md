# TVAM Integrator

Also called: common TVAM integrator base class

## Overview
The TVAM Integrator is a base class for volumetric integrators designed for Tomographic Volumetric Additive Manufacturing (TVAM) [S1, unofficial]. It solves the problem of simulating light transport from projector emitters through a photosensitive resin volume to calculate polymerization doses [S1, unofficial]. Unlike standard camera-based ray casting, it samples rays from projector emitters and incorporates manufacturing physics, such as polymerization thresholds and inhibitor dynamics, into the integrand [S1, unofficial]. It distinguishes itself from siblings like the Radon Integrator by providing the foundational sampling infrastructure and physical model evaluation, while specialized variants like the CornerIntegrator handle specific boundary conditions [S1, unofficial].

## How it works
The integrator manages the lifecycle of ray generation for projector-based volume rendering, handling both regular and stochastic sampling modes, and evaluates a physical model for polymerization based on accumulated light dose.

**Inputs and Outputs:**
*   **Inputs:** Scene description containing a target shape (by ID) and an interior medium; projector emitter configurations; parameters for physical modeling ($K, M, t_p, t_{op}, t_{inhibitor}$) and weighting ($weight\_object, weight\_void, weight\_limit, weight\_sparsity$) [S1, unofficial].
*   **Outputs:** Ray packets sampled from projectors; accumulated polymerization and inhibitor values; sparsity metrics; final integrand values for optimization or rendering [S1, unofficial].

**Data Structures:**
*   **Physical Model Parameters:**
    *   $K$: Exponent for threshold functions (default 2) [S1, unofficial].
    *   $M$: Exponent for sparsity calculation (default 4) [S1, unofficial].
    *   $t_p$: Polymerization threshold (default 0.1) [S1, unofficial].
    *   $t_{op}$: Over-polymerization limit (default 0.2) [S1, unofficial].
    *   $t_{inhibitor}$: Inhibitor threshold (default 0.2) [S1, unofficial].
    *   $weight\_object$: Weight for inside-region evaluation (default 1) [S1, unofficial].
    *   $weight\_void$: Weight for outside-region evaluation (default 1) [S1, unofficial].
    *   $weight\_limit$: Weight for over-polymerization limit (default 1) [S1, unofficial].
    *   $weight\_sparsity$: Weight for sparsity penalty (default 0) [S1, unofficial].
*   **Scene Parsing:** Identifies the target shape (ID 'target' by default) and ensures exactly one interior medium exists [S1, unofficial].

**Numbered Steps:**

1.  **Initialization (`__init__`):** Load physical model parameters ($K, M, t_p, t_{op}, t_{inhibitor}$) and weights from properties [S1, unofficial].
2.  **Preparation (`prepare`):**
    *   If `regular_sampling` is true, force samples per pixel (spp) to 1, as shooting from the pixel center makes spp > 1 nonsensical [S1, unofficial].
    *   Check if `wavefront_size` (calculated as `active_size * spp`) exceeds $2^{32}$. If so, raise an exception [S1, unofficial].
3.  **Ray Sampling (`sample_rays`):**
    *   Compute discrete sample position `idx` by repeating `active_pixels` spp times [S1, unofficial].
    *   Compute pixel coordinates:
        $$ pos.y = \lfloor pixel\_idx / w \rfloor $$
        $$ pos.x = pixel\_idx - row\_idx \times w $$
        where $w$ is the pixel width [S1, unofficial].
    *   Convert pixel position to floating point range $[0, 1]$:
        *   If `regular_sampling` is true, add 0.5 offset to both coordinates.
        *   Else, add `sampler.next_2d()` offset to both coordinates [S1, unofficial].
        *   Scale by $1/w$ and $1/h$ respectively [S1, unofficial].
    *   Compute time sample:
        $$ time = emitter\_idx / n\_patterns $$
        If `sample_time` is true, add `sampler.next_1d()` to the numerator before dividing [S1, unofficial].
    *   Sample ray from emitter using the computed position and time.
    *   Scale ray weight by `print_time` (which is $1 / pdf\_of\_time\_sample$) [S1, unofficial].
4.  **Scene Parsing (`parse_scene`):** Iterate over scene shapes to find the target shape (matching `target_id`) and the interior medium. Raise an error if more than one medium is found [S1, unofficial].
5.  **Evaluation (`eval`):**
    *   Determine if the sample point is inside the target using a binary mask `target > 0` [S1, unofficial].
    *   If inside, evaluate `eval_in` using polymerization ($x_{polymerization}$) and inhibitor ($x_{inhibitor}$) values [S1, unofficial].
    *   If outside, evaluate `eval_out` using the same values [S1, unofficial].
    *   Calculate sparsity penalty based on patterns [S1, unofficial].
    *   Return the selected physical model value and sparsity metric [S1, unofficial].

**Formulas:**

The physical model evaluation uses the ReLU function $\text{relu}(x) = \max(0, x)$.

Inside the target ($target > 0$):
$$ \text{eval\_in} = weight\_object \cdot \text{relu}(t_p - x_{polymerization} + x_{inhibitor})^K + weight\_limit \cdot \text{relu}(x_{polymerization} - t_{op})^K $$

Outside the target ($target \le 0$):
$$ \text{eval\_out} = weight\_void \cdot \text{relu}(t_{inhibitor} - x_{inhibitor} + x_{polymerization})^K $$

Sparsity:
$$ \text{eval\_sparsity} = |patterns|^M \cdot weight\_sparsity $$

**Parameters:**
*   `sample_time`: bool, default False. If true, adds a stochastic offset to the time sample [S1, unofficial].
*   `print_time`: float, default 1.0. Used to scale the ray weight [S1, unofficial].
*   `target_id`: str, default 'target'. The ID of the target shape [S1, unofficial].
*   `transmission_only`: bool, default True. If true, only samples the transmission component of the BSDF at interfaces [S1, unofficial].
*   `regular_sampling`: bool, default False. If true, samples from the center of the pixel (0.5 offset) instead of using a random offset [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import drjit as dr
import mitsuba as mi

class TVAMIntegrator(mi.Integrator):
    def __init__(self, props):
        super().__init__(props)
        # Physical model parameters
        self.K = props.get('K', 2)
        self.M = props.get('M', 4)
        self.tp = props.get('tp', 0.1)
        self.top = props.get('top', 0.2)
        self.tinhibitor = props.get('tinhibitor', 0.2)
        
        # Weights
        self.weight_object = props.get('weight_object', 1)
        self.weight_void = props.get('weight_void', 1)
        self.weight_limit = props.get('weight_limit', 1)
        self.weight_sparsity = props.get('weight_sparsity', 0)
        
        # Sampling parameters
        self.sample_time = props.get('sample_time', False)
        self.print_time = props.get('print_time', 1.0)
        self.target_id = props.get('target_id', 'target')
        self.transmission_only = props.get('transmission_only', True)
        self.regular_sampling = props.get('regular_sampling', False)

    def prepare(self, scene):
        # Force spp to 1 if regular sampling is enabled
        if self.regular_sampling:
            self.spp = 1
        
        # Check wavefront size limit
        active_size = scene.rasterizer().active_size()
        if active_size * self.spp > 2**32:
            raise Exception("Wavefront size exceeds 2^32")

    def sample_rays(self, scene, sampler, spp):
        # Compute discrete sample positions
        active_pixels = scene.rasterizer().active_pixels()
        # Repeat active_pixels spp times to get all sample indices
        idx = dr.repeat(active_pixels, self.spp)
        
        w = scene.rasterizer().width()
        h = scene.rasterizer().height()
        
        pos = dr.zeros(mi.Point2f)
        
        for i in range(len(idx)):
            pixel_idx = idx[i]
            row_idx = pixel_idx // w
            pos.y[i] = row_idx
            pos.x[i] = pixel_idx - row_idx * w
            
            # Convert to [0, 1]
            if self.regular_sampling:
                pos.x[i] += 0.5
                pos.y[i] += 0.5
            else:
                pos.x[i] += sampler.next_2d().x
                pos.y[i] += sampler.next_2d().y
            
            pos.x[i] /= w
            pos.y[i] /= h
            
        # Compute time sample
        n_patterns = scene.projector().n_patterns() # Assumed method
        emitter_idx = ... # Derived from idx or sampler
        time = emitter_idx / n_patterns
        
        if self.sample_time:
            time += sampler.next_1d()
            time /= n_patterns # Adjust division if next_1d is [0,1)
        else:
            time /= n_patterns
            
        # Sample ray from emitter
        ray = scene.projector().sample_ray(pos, time)
        
        # Scale weight
        # pdf of time sample is 1/n_patterns if uniform, or derived from sampler
        # print_time scales by 1/pdf
        ray.weight *= self.print_time 
        
        return ray

    def parse_scene(self, scene):
        target_found = False
        medium_count = 0
        target_shape = None
        interior_medium = None
        
        for shape in scene.shapes():
            if shape.id() == self.target_id:
                target_found = True
                target_shape = shape
            if shape.medium() is not None:
                medium_count += 1
                interior_medium = shape.medium()
                
        if not target_found:
            raise Exception(f"Target shape '{self.target_id}' not found")
        if medium_count != 1:
            raise Exception("More than one interior medium found")
            
        return target_shape, interior_medium

    def eval_in(self, x_polymerization, x_inhibitor):
        term1 = self.weight_object * dr.relu(self.tp - x_polymerization + x_inhibitor) ** self.K
        term2 = self.weight_limit * dr.relu(x_polymerization - self.top) ** self.K
        return term1 + term2

    def eval_out(self, x_polymerization, x_inhibitor):
        return self.weight_void * dr.relu(self.tinhibitor - x_inhibitor + x_polymerization) ** self.K

    def eval_sparsity(self, patterns):
        return dr.abs(patterns) ** self.M * self.weight_sparsity

    def eval(self, x_polymerization, target, patterns, x_inhibitor):
        # target is a binary inside/outside mask
        val_in = self.eval_in(x_polymerization, x_inhibitor)
        val_out = self.eval_out(x_polymerization, x_inhibitor)
        
        # Select based on target mask
        physical_val = dr.select(target > 0, val_in, val_out)
        sparsity_val = self.eval_sparsity(patterns)
        
        return physical_val, sparsity_val
```

## Subtopics

### [CornerIntegrator](7445-cornerintegrator.md)
Specialized integrator that filters rays intersecting corners of the build volume to solve discretization artifacts from DMD pixels.

### [Radon Integrator](7470-radon-integrator.md)
Computes the Radon transform (projection) of a volume by tracing rays through a purely absorptive medium, calculating absorbed dose or optical density.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of active pixels and samples per pixel (spp), assuming constant cost per ray intersection and physical model evaluation [S1, unofficial].
*   **Memory Complexity:** Linear with respect to the number of active pixels and spp for storing ray packets and intermediate accumulation buffers [S1, unofficial].
*   **Performance:** Performance is heavily dependent on the complexity of the scene's BSDF and the cost of evaluating the physical model functions (`eval_in`, `eval_out`) [S1, unofficial].
*   **Strengths:** Provides a unified framework for TVAM simulation, supporting both regular and stochastic sampling, and integrating complex physical models for polymerization [S1, unofficial].
*   **Limitations:** The base class does not handle specific boundary conditions (like corner artifacts) or pure projection calculations; these require specialized subclasses [S1, unofficial].
*   **When to use:** When simulating the light dose distribution in TVAM processes for inverse design or optimization [S1, unofficial].
*   **When not to use:** When only geometric projections (Radon transform) are needed without physical model evaluation, or when corner-specific discretization artifacts are the primary concern [S1, unofficial].

## Background
The TVAM Integrator originates from the Dr. TVAM project, which focuses on inverse rendering for Tomographic Volumetric Additive Manufacturing [S1, unofficial]. It was developed to enable the simulation of light transport in volumetric 3D printing, where projectors illuminate a rotating vial of photosensitive resin [S1, unofficial]. The method integrates concepts from volume rendering with manufacturing-specific physics, such as polymerization thresholds and inhibitor dynamics, to predict the resulting part geometry [S1, unofficial].

## Sources
[S1] Dr. TVAM – Inverse Rendering for Tomographic Volumetric Additive Manufacturing, https://github.com/rgl-epfl/drtvam, fetched 2026-10-26 (unofficial)
