# NeuralVDB Activation Function Selection

Also called: Activation Function Selection for NeuralVDB, NeuralVDB Activation Heuristics

## Overview
This method selects the activation function for the hierarchical neural networks in NeuralVDB to balance reconstruction quality and convergence speed. It distinguishes itself by mapping specific geometric properties of the input volume (smoothness vs. structure) to specific activation types, rather than using a fixed default. It operates within the Density Activation stage of the NeuralVDB pipeline.

## How it works
**Inputs:**
- The geometric characteristics of the input volume (classified as "smooth/unstructured" or "flat/structured/artificial").
- The presence of Fourier Feature Mapping (FFM).

**Outputs:**
- The selected activation function type (`sin`, `tanh`, or `ReLU`).
- If `sin` is selected, the frequency parameter $\omega$.

**Data Structures:**
- No persistent data structures. This is a configuration step performed during network initialization or hyperparameter tuning.

**Steps:**
1. **Analyze Input Geometry:** Determine if the input volume contains primarily smooth, unstructured surfaces or flat, structured, artificial surfaces.
2. **Select Activation Type:**
   - If the volume is **flat, structured, or artificial**: Select **ReLU**.
   - If the volume is **smooth or unstructured**: Select **sin** or **tanh**.
3. **Configure sin Frequency (if applicable):**
   - If **sin** is selected, set the frequency $\omega$.
   - Range: $1.5 \le \omega \le 3.0$.
   - Default starting point: $\omega = 1.5$.
4. **Consider Convergence Needs:**
   - If faster convergence is prioritized for smooth/unstructured models, prefer **sin** over **tanh**. Note that **sin** is redundant with Fourier Feature Mapping but accelerates convergence [S1, unofficial].

**Formulas:**
- None. The selection is heuristic.

**Parameters:**
- `sin_frequency` ($\omega$): Float. Range $[1.5, 3.0]$. Default $1.5$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **High-Frequency Noise:** Using **ReLU** on smooth geometry can introduce high-frequency noise [S1, unofficial].
- **Edge Smoothing:** Using **sin** on geometry with sharp edges (from coarse input meshes) may smooth out those sharp edges [S1, unofficial].
- **Redundancy:** Using **sin** with Fourier Feature Mapping is redundant but beneficial for convergence speed [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_activation(volume_type: str, need_fast_convergence: bool = False) -> dict:
    """
    Select activation function for NeuralVDB based on volume characteristics.
    
    Args:
        volume_type: 'flat_structured' or 'smooth_unstructured'
        need_fast_convergence: If True, prefer sin for smooth types
        
    Returns:
        dict with keys 'activation' and optionally 'frequency'
    """
    if volume_type == 'flat_structured':
        return {'activation': 'relu'}
    
    # volume_type == 'smooth_unstructured'
    if need_fast_convergence:
        # sin accelerates convergence compared to ReLU/tanh [S1, unofficial]
        return {'activation': 'sin', 'frequency': 1.5}
    else:
        # tanh is a valid alternative for smooth models [S1, unofficial]
        # sin is also valid but may smooth sharp edges [S1, unofficial]
        # Defaulting to sin as it is noted to accelerate convergence
        return {'activation': 'sin', 'frequency': 1.5}
```

## Subtopics

## Operating characteristics
- **Time Complexity:** Selection is $O(1)$. The choice affects training time: **sin** accelerates convergence [S1, unofficial].
- **Memory Complexity:** No additional memory overhead for selection.
- **Performance:** **ReLU** is best for flat/structured models. **sin/tanh** are better for smooth/unstructured models [S1, unofficial].
- **Strengths:** Adapts to input geometry; **sin** speeds up convergence.
- **Limitations:** **ReLU** causes noise on smooth geometry; **sin** blurs sharp edges.

## Background
This heuristic was developed for the NeuralVDB system, which uses hierarchical neural networks to encode sparse 3D volumes. The method addresses the observation that different activation functions significantly impact reconstruction quality and convergence rates in this specific hybrid data structure context [S1, unofficial].

## Sources
[S1] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks, https://www.youtube.com/watch?v=IfgeduMv_ns, fetched 2026-11-04 (unofficial)
