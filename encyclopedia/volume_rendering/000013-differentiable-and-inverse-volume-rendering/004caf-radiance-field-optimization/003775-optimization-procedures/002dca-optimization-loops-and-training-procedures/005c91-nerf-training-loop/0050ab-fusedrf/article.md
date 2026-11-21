# FusedRF

Also called: Fusing Multiple Radiance Fields

## Overview
FusedRF is a method to fuse multiple teacher Radiance Fields (RFs) into a single, compact student RF representation using distillation, allowing for composited scene rendering with the memory and time complexity of a single RF [S1, unofficial]. Unlike naive composition which requires parallel tracing into multiple RFs (increasing memory and render time linearly with scene count), FusedRF fuses them into one RF, maintaining constant memory and render time [S1, unofficial]. Unlike retraining from scratch using rendered RGB views, FusedRF uses 3D distillation from existing RFs, reducing training time by 3x [S1, unofficial].

## How it works
**Inputs:**
- $N$ teacher radiance fields $T_1, \dots, T_N$ (typically MLPs or voxel-based representations encoding density $\sigma$, color $c$, and potentially other attributes).
- A set of training rays $\mathcal{R}$ sampled from the camera poses associated with the teacher fields.
- Identification of a "background" teacher $T_{bg}$ (the dominant scene in the composition) [S1, unofficial].

**Outputs:**
- A single student radiance field $S$ capable of rendering the composited scene.

**Data Structures:**
- The student $S$ and teachers $T_i$ are typically implemented as neural networks (e.g., MLPs) or structured grids. The specific architecture is inherited from the source RFs. The method assumes access to query functions $T_i(\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, c)$ and $S(\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, c)$, where $\mathbf{x}$ is the 3D position and $\mathbf{d}$ is the viewing direction [S1, unofficial].
- Training points $\mathcal{P}$: A set of 3D coordinates $\mathbf{x}$ sampled along rays.

**Parameters:**
- $\delta$: Step size for ray marching.
- $\alpha_{threshold}$: Threshold for pruning low-density points (value not explicitly given in sources, described as "low density") [S1, unofficial].
- $\alpha_{keep}$: Threshold for keeping high-density points (value not explicitly given in sources, described as "high density") [S1, unofficial].
- Loss weights: Not explicitly specified; standard L2 losses are used [S1, unofficial].

**Steps:**

1. **Initialization:** Initialize the student RF $S$ with the weights of the background teacher $T_{bg}$ (the dominant teacher in the composition). This speeds up the distillation process [S1, unofficial].

2. **Ray Marching and Point Selection:**
   For each training ray $\mathbf{r} \in \mathcal{R}$:
   a. Sample points $\{\mathbf{x}_j\}$ along the ray for each teacher $T_i$.
   b. For every sampled point $\mathbf{x}_j$ in teacher $T_i$, calculate the activated volumetric density $\alpha_{i,j}$ using the formula:
      $$ \alpha_{i,j} = 1 - e^{-\sigma_{i,j} \delta} $$
      where $\sigma_{i,j}$ is the density output by $T_i$ at $\mathbf{x}_j$ and $\delta$ is the step size [S1, unofficial].
   c. Prune points with lower density values and select points with higher density values from the teachers. The union of these selected points forms the training set $\mathcal{P}$ for the student $S$. Specifically, for each location along the ray, the point with the higher $\alpha$ value is chosen for contribution [S1, unofficial].

3. **Distillation (Supervised Loss):**
   For each point $\mathbf{x} \in \mathcal{P}$:
   a. Query the density $\sigma$, alpha $\alpha$, and color $c$ for the student $S$ and all relevant teachers $T_i$ at $\mathbf{x}$.
   b. Compute the supervised L2 loss between the student's outputs and the corresponding teacher's outputs for both color and density:
      $$ \mathcal{L}_{distill} = || c_S(\mathbf{x}) - c_{T}(\mathbf{x}) ||_2^2 + || \sigma_S(\mathbf{x}) - \sigma_{T}(\mathbf{x}) ||_2^2 $$
      where $T$ is the teacher from which the point $\mathbf{x}$ was selected [S1, unofficial].
   c. Update $S$ to minimize $\mathcal{L}_{distill}$.

4. **Smoothing (Alpha-Composited Loss):**
   After the initial distillation phase, render RGB values for rays by accumulating individual color values weighted by activated volumetric density using the Volumetric Rendering Equation:
   $$ C(\mathbf{r}) = \sum_{j=1}^{M} T_j c_j, \quad \text{where } T_j = \left( \prod_{k=1}^{j-1} (1 - \alpha_k) \right) \alpha_j $$
   Apply an alpha-composited RGB pixel loss for a few iterations to smooth results around object boundaries:
   $$ \mathcal{L}_{render} = || C_S(\mathbf{r}) - C_{target}(\mathbf{r}) ||_2^2 $$
   where $C_{target}$ is the ground truth composed color (either from naive composition or rendered views) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fused_rf_distillation(teachers, background_teacher_idx, rays, student, steps_per_ray):
    """
    Fuses multiple teacher radiance fields into a single student radiance field.
    
    Args:
        teachers: List of teacher RF query functions [T1, T2, ...]
        background_teacher_idx: Index of the dominant teacher for initialization
        rays: List of rays, each defined by origin and direction
        student: Student RF query function (initialized with background_teacher weights)
        steps_per_ray: Number of sample points per ray
    
    Returns:
        student: The fused radiance field
    """
    
    # Step 1: Initialization
    # Initialize student weights with the background teacher's weights
    student.weights = teachers[background_teacher_idx].weights
    
    # Step 2 & 3: Distillation Loop
    for ray in rays:
        # Collect candidate points from all teachers
        candidate_points = []
        
        for t_idx, teacher in enumerate(teachers):
            # Sample points along the ray
            points = sample_points_along_ray(ray, steps_per_ray)
            
            for p in points:
                # Query teacher for density and color
                sigma, color = teacher(p.position, ray.direction)
                
                # Calculate activated volumetric density
                # alpha = 1 - exp(-sigma * delta)
                # Note: delta is assumed to be constant or derived from step size
                delta = ray.step_size
                alpha = 1.0 - math.exp(-sigma * delta)
                
                candidate_points.append({
                    'position': p.position,
                    'teacher_idx': t_idx,
                    'sigma': sigma,
                    'color': color,
                    'alpha': alpha
                })
        
        # Select points: Union of high-density points from teachers
        # Prune low-density points, keep high-density points
        # The point with the higher alpha value is chosen for contribution
        selected_points = []
        
        # Group points by spatial proximity or simply select max alpha per ray segment
        # The source describes: "The points with low density are pruned out while 
        # the ones with high density are utilized. The union of these selected points..."
        # And: "For every sampled point... the point with a higher alpha value is chosen"
        
        # Implementation detail: For each ray segment (or discrete sample index),
        # select the teacher point with the maximum alpha.
        for i in range(steps_per_ray):
            # Get points at index i from all teachers
            points_at_i = [cp for cp in candidate_points if cp['position'].index_in_ray == i]
            
            if not points_at_i:
                continue
                
            # Select point with highest alpha
            best_point = max(points_at_i, key=lambda p: p['alpha'])
            
            # Pruning: if alpha is too low, discard? 
            # Source says "prune points with lower density values". 
            # We assume a threshold or relative selection. 
            # For simplicity, we keep the max alpha point as the representative.
            selected_points.append(best_point)
        
        # Step 3: Compute Distillation Loss
        distill_loss = 0.0
        for pt in selected_points:
            # Query student at the selected point's position
            s_sigma, s_color = student(pt['position'], ray.direction)
            
            # Query the corresponding teacher
            t_sigma = pt['sigma']
            t_color = pt['color']
            
            # L2 loss on density and color
            loss_sigma = (s_sigma - t_sigma) ** 2
            loss_color = sum((s_c - t_c) ** 2 for s_c, t_c in zip(s_color, t_color))
            
            distill_loss += loss_sigma + loss_color
        
        # Update student weights to minimize distill_loss
        # (Optimization step handled by external optimizer)
        update_student_weights(student, distill_loss)

    # Step 4: Smoothing Phase
    for iteration in range(smoothing_iterations):
        render_loss = 0.0
        for ray in rays:
            # Render student output
            s_color = render_ray_volumetric(student, ray)
            
            # Get target color (from naive composition or ground truth)
            t_color = render_ray_naive_composition(teachers, ray)
            
            # Alpha-composited RGB L2 loss
            loss = sum((s_c - t_c) ** 2 for s_c, t_c in zip(s_color, t_color))
            render_loss += loss
        
        # Update student weights to minimize render_loss
        update_student_weights(student, render_loss)

    return student

def render_ray_volumetric(rf, ray):
    """
    Renders a ray using the standard volumetric rendering equation.
    """
    points = sample_points_along_ray(ray, steps_per_ray)
    accumulated_color = [0.0, 0.0, 0.0]
    accumulated_alpha = 0.0
    transmittance = 1.0
    
    for p in points:
        sigma, color = rf(p.position, ray.direction)
        delta = ray.step_size
        alpha = 1.0 - math.exp(-sigma * delta)
        
        weight = transmittance * alpha
        accumulated_color = [c + weight * col for c, col in zip(accumulated_color, color)]
        accumulated_alpha += weight
        transmittance *= (1.0 - alpha)
        
    return accumulated_color

def sample_points_along_ray(ray, num_samples):
    """
    Samples points along a ray.
    """
    points = []
    for i in range(num_samples):
        t = ray.origin + ray.direction * (i * ray.step_size)
        points.append({'position': t, 'index_in_ray': i})
    return points
```

## Operating characteristics
- **Memory Complexity:** Constant with respect to the number of composed scenes. FusedRF maintains the memory footprint of a single RF. For example, on an RTX 3060 Ti (8GB), FusedRF maintains 25MB memory for 8 composed scenes, whereas DVGO goes Out-Of-Memory (OOM) and TensoRF requires 200MB [S1, unofficial].
- **Time Complexity:** Render time is constant with respect to the number of composed scenes. FusedRF takes ~2.04s for 8 scenes, while naive composition scales linearly (e.g., 160s for 8 scenes in the comparison table, though the table lists "Time: 160s" for 8 scenes in a column that seems to represent naive or another method, the text states "render times... with the number of components" for naive composition) [S1, unofficial]. Training time is reduced by 3x compared to retraining from scratch [S1, unofficial].
- **Performance:**
  - RTX 3060 Ti (8GB):
    - 1 Scene: 25MB / 2.04s
    - 2 Scenes: 25MB / 2.05s
    - 4 Scenes: 25MB / 2.04s
    - 8 Scenes: 25MB / 2.04s
  - Comparison (8 Scenes):
    - DVGO: OOM
    - TensoRF: 200MB / 17.2s
    - FusedRF: 25MB / 2.04s [S1, unofficial]
- **Quality:**
  - PSNR (Figure 1): Ours (w/o RGB) 36.71, Ours (Full) 37.20 [S1, unofficial].
  - PSNR (Figure 3b): Ours (w/o RGB) 35.18, Ours (Full) 39.89 [S1, unofficial].
  - PSNR (Figure 3d): Ours (w/o RGB) 40.32, Ours (Full) 38.78 [S1, unofficial].
- **Strengths:** Low memory footprint, fast rendering, fast convergence (3x faster than retraining), supports addition/deletion of scenes in the fused representation [S1, unofficial].
- **Limitations:** Requires existing trained teacher RFs; assumes rigid transformation (no rigid transformation applied in the described method) [S1, unofficial].

## Background
FusedRF was proposed to address the high memory and rendering time requirements of composing multiple Radiance Fields for XR and AR applications. Prior works generated new views by tracing each scene in parallel, which did not scale well. FusedRF distills information from multiple teacher RFs into a single student RF, facilitating further manipulations like addition and deletion [S1, unofficial].

## Sources
[S1] FusedRF: Fusing Multiple Radiance Fields, https://doi.org/10.48550/arXiv.2306.04180, fetched 2026-10-29 (unofficial)
