# Conservative Density Evaluation

Also called: Conservative Density

## Overview
Conservative Density Evaluation is a ray marching optimization technique used in volumetric rendering to avoid the computational cost of evaluating expensive material graphs (such as Albedo, Emissive Color, and Extinction) for samples where no participating medium exists. It functions by introducing a cheap, user-specified "conservative density" check; if this value is zero or negative, the expensive material evaluation is skipped entirely for that sample. Unlike standard material graph evaluation which runs for every sample, this method uses a scalar/vector gate to prune empty space, significantly improving performance in atmospheric cloud rendering.

## How it works
The method operates as a two-stage sampling process within the ray marching loop. It relies on a specific input vector to determine whether the full material computation is necessary.

### Inputs and Outputs
*   **Inputs:**
    *   Ray marching parameters (origin, direction, step size) inherited from the parent traversal method.
    *   **Conservative Density Input:** A `Vector3` value computed via a lightweight graph or texture lookup. The X component is the critical trigger.
    *   **Main Material Graph:** The expensive material definition containing Albedo, Emissive Color, and Extinction.
*   **Outputs:**
    *   If Conservative Density X > 0: The result of the Main Material Graph evaluation.
    *   If Conservative Density X ≤ 0: A null/transparent result, skipping the Main Material Graph.

### Data Structures
The implementation requires a specific node structure in the material graph (specifically in Unreal Engine's material editor context):
1.  **Volumetric Advanced Output Node:** Accepts the `Vector3` Conservative Density input.
2.  **Volumetric Advanced Input Node:** Used to retrieve the results of the Main Material Graph evaluation.
3.  **Main Material Node:** Contains the expensive material logic.

### Steps
1.  **Evaluate Conservative Density:** Compute the conservative density value (typically a lightweight 2D texture lookup or simple scalar function) for the current ray marching position. This value is passed to the `Vector3` input of the Volumetric Advanced Output node.
2.  **Check Trigger Condition:** Inspect the X component of the Conservative Density vector.
    *   If $Density_X > 0$: Proceed to Step 3.
    *   If $Density_X \le 0$: Skip the expensive material evaluation and proceed to the next ray marching sample (Step 5).
3.  **Evaluate Main Material:** If the condition in Step 2 is met, evaluate the expensive material graph plugged into the Main Material node. This computes Albedo, Emissive Color, and Extinction.
4.  **Retrieve Results:** Access the computed values from the Main Material evaluation using the Volumetric Advanced Input node. This avoids recomputing the conservative density or material values.
5.  **Advance Ray:** Proceed to the next sample along the ray.

### Pseudocode
The following pseudocode illustrates the logic flow within a single ray marching step. It assumes the existence of a `EvaluateConservativeDensity` function and an `EvaluateExpensiveMaterial` function.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_step_with_conservative_density(ray_position):
    # Step 1: Evaluate the cheap conservative density
    # Returns a Vector3-like structure
    conservative_density = EvaluateConservativeDensity(ray_position)
    
    # Step 2: Check the X component trigger
    if conservative_density.x > 0:
        # Step 3: Evaluate the expensive material graph
        # Computes Albedo, Emissive Color, Extinction
        material_result = EvaluateExpensiveMaterial(ray_position)
        
        # Step 4: Access values (conceptually handled by the engine's 
        # Volumetric Advanced Input node to avoid recomputation)
        albedo = material_result.albedo
        emissive = material_result.emissive
        extinction = material_result.extinction
        
        return {
            'albedo': albedo,
            'emissive': emissive,
            'extinction': extinction,
            'skipped': False
        }
    else:
        # Step 5: Skip expensive evaluation
        # Return transparent/null values to proceed to next sample
        return {
            'albedo': 0.0,
            'emissive': 0.0,
            'extinction': 0.0,
            'skipped': True
        }
```

### Parameters
*   **Conservative Density Threshold:** The threshold is fixed at 0. The X component of the `Vector3` input must be strictly greater than 0 to trigger evaluation.
*   **Conservative Density Source:** Typically a top-down two-dimensional texture describing cloud density. The only rule is that it must be $> 0$ where cloud exists.

### Edge Cases and Pitfalls
*   **Dynamic Branching Limitation:** In environments like Unreal Engine's material graph, native dynamic branching is not supported. The Conservative Density input is *required* to enable this skipping behavior; without it, the expensive material graph would be evaluated for every sample regardless of density [S1].
*   **Alpha Holdout Interaction:** Cloud rendering will force the use of the expensive path (bypassing the optimization) when `r.PostProcessing.PropagateAlpha` is enabled and any features (Exponential Height Fog, Volumetric Cloud, Sky Atmosphere) have alpha holdout enabled [S1].
*   **Ray Tracing Limitations:** The cloud system does not support Ray Tracing features for volumetric clouds (e.g., reflections); it only accounts for clouds rendered into the Sky Light. Tracing volumetric clouds for reflections is significantly more costly and is not supported by this optimization path [S1].

## Operating characteristics
*   **Performance:** Significantly reduces frame time for atmospheric rendering by skipping expensive material evaluations in empty space. For example, in Unreal Engine, this optimization allows cloud rendering to remain within tight frame budgets (e.g., 0.6 ms) where it might otherwise exceed limits [S1].
*   **Memory:** Adds minimal overhead, requiring only the storage and evaluation of the conservative density texture/graph.
*   **Strengths:** Effective for large volumes with significant empty space (e.g., sky clouds). It decouples the density check from the complex material logic.
*   **Limitations:** Requires a "conservative" density approximation. If the conservative density is $> 0$ but the actual material is empty (false positive), the expensive path is still taken, wasting performance. If it is $0$ where material exists (false negative), visual artifacts occur. It is specific to engines that lack native dynamic branching in material graphs.

## Background
Conservative Density Evaluation was introduced as an optimization for ray marching atmospheric clouds in Unreal Engine. It addresses the high cost of evaluating full material graphs (Albedo, Emissive Color, Extinction) for every sample in volumetric cloud rendering. The method leverages the fact that cloud density is often sparse, allowing a cheap check to gate the expensive computation. It is particularly relevant in Unreal Engine's material system, which historically lacked native dynamic branching support for material graphs, necessitating input-based gating mechanisms [S1].

## Sources
[S1] Volumetric Cloud Component in Unreal Engine | Unreal Engine 5.8 Documentation | Epic Developer Community, https://dev.epicgames.com/documentation/en-us/unreal-engine/volumetric-cloud-component-in-unreal-engine, fetched 2026-10-30
