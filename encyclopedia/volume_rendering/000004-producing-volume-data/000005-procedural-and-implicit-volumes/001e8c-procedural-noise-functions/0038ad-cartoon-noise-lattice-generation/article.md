# Cartoon Noise Lattice Generation

Also called: Cartoon Noise Set Generation, Macro/Body/Edge/Flow Lattice Generation

## Overview
Cartoon Noise Lattice Generation produces independent, tileable noise lattices for stylized, cartoon-style cloud rendering by separating noise into four distinct functional groups: macro (outline), body (fill), edge (detail), and flow (disturbance) [S1, unofficial]. Unlike standard gradient noise which produces a single continuous field, this method generates multiple specialized lattices with specific frequency constraints to ensure stability and avoid fixed-direction ripple artifacts, building on [Gradient-based Noise (Perlin/Simplex)](3860-gradient-based-noise-perlin-simplex.md).

## How it works
**Inputs and Outputs:**
- **Inputs:** A `MaskParameters` object containing resolution (512, 1024, or 2048), global height bounds (`global_min_cm`, `global_max_cm`), weather type, seed, coverage percentage, cloud region count, and optional `CloudHeightRule` configurations [S1, unofficial].
- **Outputs:** A `_CartoonNoiseSet` dataclass containing four tuples of numpy arrays: `macro_lattices`, `body_lattices`, `edge_lattices`, and `flow_lattices` [S1, unofficial].

**Data Structures:**
1.  **`MaskParameters`**:
    -   `resolution`: `int` (512, 1024, 2048) [S1, unofficial].
    -   `global_min_cm`, `global_max_cm`: `int` defining the vertical volume bounds [S1, unofficial].
    -   `seed`: `int` for deterministic generation [S1, unofficial].
    -   `cloud_region_count`: `int` (1–500) [S1, unofficial].
    -   `cloud_height_rules`: `tuple[CloudHeightRule, ...]` [S1, unofficial].
2.  **`CloudHeightRule`**:
    -   `type_id`: `int` [S1, unofficial].
    -   `center_min_cm`, `center_max_cm`: `int` defining the target center height range [S1, unofficial].
    -   `height_delta_min_cm`, `height_delta_max_cm`: `int` defining the vertical thickness range [S1, unofficial].
3.  **`_CartoonNoiseSet`**:
    -   `macro_lattices`: `tuple[np.ndarray, ...]` [S1, unofficial].
    -   `body_lattices`: `tuple[np.ndarray, ...]` [S1, unofficial].
    -   `edge_lattices`: `tuple[np.ndarray, ...]` [S1, unofficial].
    -   `flow_lattices`: `tuple[np.ndarray, ...]` [S1, unofficial].

**Algorithm Steps:**
1.  **Validate Parameters:** Ensure `resolution` is in {512, 1024, 2048}, `global_max_cm > global_min_cm`, and `0 <= coverage_percent <= 100` [S1, unofficial].
2.  **Generate Height Rules:** For each cloud type, determine vertical bounds. If a `CloudHeightRule` is provided, use its `center_min_cm` and `center_max_cm` sampled via `rng.integers`. Otherwise, derive bounds from legacy constants: for `type_id == 10`, use span multipliers `(0.62, 0.92)`; for others, use `(0.22, 0.58)` [S1, unofficial]. Calculate `delta_min` and `delta_max` using `VOLUME_BOX_HEIGHT_MULTIPLIER` (value not given) applied to the span [S1, unofficial].
3.  **Generate Lattices:** Call `_make_noise_lattices` for each of the four groups (macro, body, edge, flow). Use specific frequencies for each group to ensure they are small and stable, preventing fixed-direction ripple artifacts [S1, unofficial].
4.  **Construct Set:** Pack the resulting tuples into the `_CartoonNoiseSet` dataclass [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from dataclasses import dataclass

@dataclass(frozen=True)
class CloudHeightRule:
    type_id: int
    center_min_cm: int
    center_max_cm: int
    height_delta_min_cm: int
    height_delta_max_cm: int

@dataclass(frozen=True)
class MaskParameters:
    resolution: int
    global_min_cm: int
    global_max_cm: int
    seed: int
    coverage_percent: int
    cloud_region_count: int
    cloud_height_rules: tuple[CloudHeightRule, ...] = ()

@dataclass(frozen=True)
class _CartoonNoiseSet:
    macro_lattices: tuple[np.ndarray, ...]
    body_lattices: tuple[np.ndarray, ...]
    edge_lattices: tuple[np.ndarray, ...]
    flow_lattices: tuple[np.ndarray, ...]

def _make_noise_lattices(group_type: str, params: MaskParameters, rng: np.random.Generator) -> tuple[np.ndarray, ...]:
    """
    Generates lattices for a specific group (macro, body, edge, flow).
    Uses specific frequencies to ensure stability and avoid artifacts.
    Implementation details of the noise generation kernel are not provided in sources.
    """
    # Placeholder for the actual noise lattice generation logic
    # which uses the group-specific frequency constraints.
    pass

def generate_cartoon_noise_lattices(params: MaskParameters) -> _CartoonNoiseSet:
    if params.resolution not in (512, 1024, 2048):
        raise ValueError("Resolution must be 512, 1024, or 2048.")
    if params.global_max_cm <= params.global_min_cm:
        raise ValueError("Max height must be greater than min height.")
    if not (0 <= params.coverage_percent <= 100):
        raise ValueError("Coverage percent must be between 0 and 100.")
    
    rng = np.random.default_rng(params.seed)
    
    # Generate lattices for each group
    macro_lattices = _make_noise_lattices("macro", params, rng)
    body_lattices = _make_noise_lattices("body", params, rng)
    edge_lattices = _make_noise_lattices("edge", params, rng)
    flow_lattices = _make_noise_lattices("flow", params, rng)
    
    return _CartoonNoiseSet(
        macro_lattices=macro_lattices,
        body_lattices=body_lattices,
        edge_lattices=edge_lattices,
        flow_lattices=flow_lattices
    )
```

**Edge Cases and Pitfalls:**
-   **Resolution Constraints:** Only 512, 1024, and 2048 are supported [S1, unofficial].
-   **Height Bounds:** `global_max_cm` must be strictly greater than `global_min_cm` [S1, unofficial].
-   **Artifact Avoidance:** Frequencies must be carefully selected per group to prevent fixed-direction ripple artifacts [S1, unofficial].

## Subtopics

### Cartoon Flow Noise Generation
A variant that computes a coherent flow field by combining the generated lattices. It clips the weighted sum of flow, macro, and body noise channels: `coherent = np.clip(0.58 * flow + 0.27 * macro + 0.15 * body, 0.0, 1.0)` [S1, unofficial]. It then modulates this coherent flow around the center value 0.5 by a strength parameter: `return np.clip(0.5 + (coherent - 0.5) * strength, 0.0, 1.0)` [S1, unofficial]. This is used to drive disturbance in the cloud volume.

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** Not given in the sources.
-   **Strengths:** Produces stable, tileable noise suitable for cartoon-style rendering; separates concerns (outline, fill, detail, flow) for artistic control.
-   **Limitations:** Requires specific resolution support; relies on hardcoded legacy constants for height rules if not explicitly configured.
-   **When to use:** When generating stylized, non-photorealistic cloud volumes.
-   **When not to use:** When photorealistic, single-field noise is sufficient.

## Background
Originated in the open-source project `Noise-Cloud` by mingtu149411 [S1, unofficial]. The method is designed to address the visual artifacts of standard noise in cloud rendering by decomposing the noise field into specialized components.

## Sources
[S1] mingtu149411/Noise-Cloud, https://github.com/mingtu149411/Noise-Cloud, fetched 2026-10-27 (unofficial)
