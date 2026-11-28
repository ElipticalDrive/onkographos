# Validator Comparison Logic
Also called: validate

## Overview
This method provides the verification logic for GPU-based greedy meshing by comparing generated face vectors against reference meshes and other implementations [S1, unofficial]. It ensures bitwise reproducibility of the mesh extraction process by detecting mismatches in face attributes and enforcing canonical ordering for digest-based validation [S1, S2, unofficial]. It differs from the parent [Validator Reference Face Reconstruction](3794-validator-reference-face-reconstruction.md) by focusing on the comparison and error reporting phase rather than the generation of the reference data itself [S1, unofficial].

## How it works
The method consists of two primary functions: `require_equal` for element-wise comparison and `validate` for orchestrating the full test suite [S1, unofficial].

### Data Structures
Faces are represented as 64-bit unsigned integers (`uint64_t`) [S1, unofficial]. The bit layout is defined as follows [S1, unofficial]:
- **Axis**: bits 18–20 (mask 3, shift 18).
- **Positive**: bit 20 (mask 1, shift 20).
- **Depth**: bits 12–17 (mask 63, shift 12).
- **U**: bits 0–5 (mask 63, shift 0).
- **V**: bits 6–11 (mask 63, shift 6).
- **Material**: bits 21–31 (mask 255, shift 21).

### Step 1: Element-wise Comparison (`require_equal`)
1. Compare `actual` and `expected` vectors element by element [S1, unofficial].
2. If they differ, find the first mismatch index `mismatch` [S1, unofficial].
3. Extract fields from `actual_value` and `expected_value` at `mismatch` using shifts and masks [S1, unofficial].
4. Throw `std::runtime_error` with details [S1, unofficial].

### Step 2: Validation Orchestration (`validate`)
1. Generate `expected` faces from voxels [S1, unofficial].
2. Generate `reference` faces from reference mesh data [S1, unofficial].
3. Generate `optimized` faces from SBUG mesh data [S1, unofficial].
4. Call `require_equal` for `(reference, expected)`, `(optimized, expected)`, and `(optimized, reference)` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_fields(value: uint64) -> dict:
    """Extract face attributes from a 64-bit integer."""
    return {
        "axis": (value >> 18) & 3,
        "positive": (value >> 20) & 1,
        "depth": (value >> 12) & 63,
        "u": value & 63,
        "v": (value >> 6) & 63,
        "material": (value >> 21) & 255
    }

def require_equal(actual: list[uint64], expected: list[uint64]) -> None:
    """Compare two face vectors element-wise."""
    if len(actual) != len(expected):
        raise RuntimeError(f"Length mismatch: {len(actual)} vs {len(expected)}")
    for i in range(len(actual)):
        if actual[i] != expected[i]:
            act_fields = extract_fields(actual[i])
            exp_fields = extract_fields(expected[i])
            raise RuntimeError(
                f"Mismatch at index {i}. "
                f"Actual: {act_fields}, Expected: {exp_fields}"
            )

def generate_faces_from_voxels() -> list[uint64]:
    # Generation logic is handled by the meshing pipeline; returns sorted face list
    return []

def generate_faces_from_reference_mesh() -> list[uint64]:
    # Generation logic is handled by the meshing pipeline; returns sorted face list
    return []

def generate_faces_from_sbgm_mesh() -> list[uint64]:
    # Generation logic is handled by the meshing pipeline; returns sorted face list
    return []

def validate() -> None:
    """Orchestrate the full validation suite."""
    expected_faces = generate_faces_from_voxels()
    reference_faces = generate_faces_from_reference_mesh()
    optimized_faces = generate_faces_from_sbgm_mesh()

    # Sort faces to ensure canonical order before comparison
    expected_faces.sort()
    reference_faces.sort()
    optimized_faces.sort()

    # Perform comparisons
    require_equal(reference_faces, expected_faces)
    require_equal(optimized_faces, expected_faces)
    require_equal(optimized_faces, reference_faces)
```

## Subtopics

### Canonical Face Ordering and Digest
Enforces a strict canonical order for faces: FaceDir, then plane, then v, then u [S2, unofficial]. Computes a representation-independent mesh digest using the BLAKE3 hash algorithm [S2, unofficial]. Uses pinned fixture digests for validation against known-good outputs [S2, unofficial]. This allows for efficient comparison of mesh topology regardless of the specific generation order, provided the canonical sort is applied [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This logic is part of the validation suite for the Streaming Binary Greedy Meshing (SBUG) implementation [S1, unofficial]. It is used in the `spall` project for validation [S2, unofficial].

## Sources
[S1] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
[S2] T05 — Visible voxel baseline (spall_mesh, spall_render), https://github.com/deranjer/spall/pull/10, fetched 2026-10-27 (unofficial)
