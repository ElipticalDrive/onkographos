# Conservative Density Evaluation

Also called: Conservative Density

## Overview
Conservative Density Evaluation is a ray marching optimization technique that uses a user-specified, inexpensive-to-evaluate "conservative density" to gate the execution of expensive material graphs (such as Albedo, Emissive Color, and Extinction calculations) [S1]. It solves the performance bottleneck of evaluating full volumetric material logic for every sample along a ray, particularly in atmospheric cloud rendering, by skipping expensive computations when the conservative density indicates no participating medium exists [S1]. Unlike standard material evaluation which runs for every sample, this method uses a cheap scalar/vector check to gate the expensive evaluation, saving performance when cloud density is zero [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A `Vector3` conservative density value, typically computed via a separate, inexpensive graph (e.g., a top-down 2D texture lookup or simple geometric check) [S1]. The X component of this vector represents the participating medium conservative density [S1].
*   **Output:** The final material properties (Albedo, Emissive Color, Extinction) if density > 0; otherwise, the sample is skipped or treated as empty [S1].

**Data Structures:**
*   **Conservative Density Vector:** A 3-component floating-point vector (`Vector3`) [S1].
    *   `X`: The scalar conservative density value used for the branching decision [S1].
    *   `Y`, `Z`: Unused for the branching logic in the standard implementation described [S1].

**Numbered Steps:**
1.  **Evaluate Conservative Density:** Compute the conservative density input graph. This graph must be inexpensive to evaluate [S1]. The result is stored in a `Vector3` variable [S1].
2.  **Check Density Threshold:** Inspect the X component of the conservative density vector [S1].
3.  **Conditional Material Evaluation:**
    *   If `ConservativeDensity.X > 0`: Proceed to evaluate the expensive material graph plugged into the Main Material node. This graph computes Albedo, Emissive Color, and Extinction [S1].
    *   If `ConservativeDensity.X <= 0`: Skip the expensive material evaluation entirely and proceed to the next ray marching sample [S1].
4.  **Access Material Values:** If the expensive graph was evaluated, access its output values using the Volumetric Advanced Input node. This node retrieves the values computed in Step 3 without recomputing the conservative density [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_volumetric_sample(ray_sample_position):
    # Step 1: Evaluate the inexpensive conservative density graph
    # This graph might be a simple texture lookup or geometric check
    conservative_density_vector = evaluate_conservative_density_graph(ray_sample_position)
    
    # Step 2: Check the X component of the conservative density
    density_x = conservative_density_vector.x
    
    # Step 3: Conditional branching based on density
    if density_x > 0:
        # Step 3a: Evaluate the expensive material graph
        # This computes Albedo, Emissive Color, and Extinction
        material_output = evaluate_expensive_material_graph(ray_sample_position)
        
        # Step 4: Access values from the expensive evaluation
        # The Volumetric Advanced Input node retrieves these values
        albedo = material_output.albedo
        emissive_color = material_output.emissive_color
        extinction = material_output.extinction
    else:
        # Step 3b: Skip expensive evaluation
        # The sample is treated as empty/non-participating
        albedo = vec3(0.0)
        emissive_color = vec3(0.0)
        extinction = 0.0
        
    return albedo, emissive_color, extinction
```

**Parameters:**
*   **Conservative Density Threshold:** The threshold is implicitly `0.0`. If `X > 0`, the material is evaluated; otherwise, it is skipped [S1].
*   **Conservative Density Graph:** The specific implementation of the inexpensive graph is user-defined but must satisfy the condition that it is greater than 0 where cloud/participating medium exists [S1].

**Edge Cases and Pitfalls:**
*   **Alpha Holdout Interaction:** Cloud rendering will use the expensive path (i.e., skip the optimization) when `r.PostProcessing.PropagateAlpha` is enabled and when any features, such as Exponential Height Fog, Volumetric Cloud, and Sky Atmosphere, have alpha holdout enabled [S1].
*   **Dynamic Branching Limitation:** Unreal Engine material graphs do not support dynamic branching natively [S1]. The Conservative Density input is required to enable this skipping behavior; without it, the expensive material graph would be evaluated for every sample [S1].
*   **Ray Tracing Limitations:** The cloud system does not support Ray Tracing features for volumetric clouds; they are only taken into account as rendered into the Sky Light [S1].

## Operating characteristics
*   **Performance:** Significantly reduces cost when cloud density is zero by avoiding expensive material graph evaluations [S1]. The optimization is effective because the conservative density check is "inexpensively reasonable to evaluate" compared to the full material graph [S1].
*   **Limitations:** The optimization is bypassed when alpha holdout is enabled on relevant features or when `r.PostProcessing.PropagateAlpha` is active [S1]. It does not apply to ray-traced reflections, as volumetric clouds do not support ray tracing for reflections [S1].
*   **When to use:** Use when rendering atmospheric clouds or participating media where large portions of the ray march through empty space, and the engine/material system lacks native dynamic branching support [S1].
*   **When not to use:** Do not use if alpha holdout is required for the specific volumetric features, as the optimization will be disabled [S1].

## Background
Conservative Density Evaluation was introduced in Unreal Engine as an optimization for the Volumetric Cloud system [S1]. It addresses the limitation of Unreal Engine's material graph system, which does not support dynamic branching natively [S1]. The feature allows technical artists to manually implement dynamic branching logic by using the Conservative Density input to gate expensive material evaluations [S1]. It is part of the broader Volumetric Cloud Component in Unreal Engine [S1].

## Sources
[S1] Volumetric Cloud Component in Unreal Engine | Unreal Engine 5.8 Documentation | Epic Developer Community, https://dev.epicgames.com/documentation/en-us/unreal-engine/volumetric-cloud-component-in-unreal-engine, fetched 2026-10-30
