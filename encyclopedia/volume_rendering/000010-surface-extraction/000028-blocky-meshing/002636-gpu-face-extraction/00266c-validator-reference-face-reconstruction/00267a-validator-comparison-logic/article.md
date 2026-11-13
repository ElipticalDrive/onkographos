# Validator Comparison Logic
Also called: validate

## Overview
This method provides the verification logic for GPU-based greedy meshing by comparing generated face vectors against reference meshes and other implementations. It ensures bitwise reproducibility of the mesh extraction process by detecting mismatches in face attributes and enforcing canonical ordering for digest-based validation. It differs from the parent [Validator Reference Face Reconstruction](3794-validator-reference-face-reconstruction.md) by focusing on the comparison and error reporting phase rather than the generation of the reference data itself.

## How it works
The method consists of two primary functions: `require_equal` for element-wise comparison and `validate` for orchestrating the full test suite.

### Data Structures
Faces are represented as 64-bit unsigned integers (`uint64_t`). The bit layout is defined as follows [S1, unofficial]:
- **Axis**: bits 18–20 (3 bits, mask `0x7`, shift 18).
- **Positive**: bit 20 (1 bit, mask `0x1`, shift 20). *Note: The source lists bit 20 for both Axis and Positive, implying a potential overlap or specific encoding where Positive is a sub-field or distinct flag at that position. Based on standard bit packing, we assume distinct fields. If Axis is 18-20, Positive at 20 overlaps. We follow the source's explicit mask/shift instructions.*
- **Depth**: bits 12–17 (6 bits, mask `0x3F`, shift 12).
- **U**: bits 0–5 (6 bits, mask `0x3F`, shift 0).
- **V**: bits 6–11 (6 bits, mask `0x3F`, shift 6).
- **Material**: bits 21–31 (11 bits, mask `0x1FF`, shift 21).

### Step 1: Element-wise Comparison (`require_equal`)
1.  Compare `actual` and `expected` vectors element by element.
2.  If a mismatch is found at index `mismatch`, extract fields from both `actual[mismatch]` and `expected[mismatch]` using the bit masks and shifts defined above.
3.  Throw a `std::runtime_error` containing the details of the mismatch (index, axis, positive, depth, u, v, material for both values).

### Step 2: Validation Orchestration (`validate`)
1.  Generate `expected` faces from voxel data.
2.  Generate `reference` faces from a known-correct reference mesh data structure.
3.  Generate `optimized` faces from the SBUG (Streaming Binary Greedy Meshing) mesh data.
4.  Call `require_equal(reference, expected)`.
5.  Call `require_equal(optimized, expected)`.
6.  Call `require_equal(optimized, reference)`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_fields(value: uint64) -> dict:
    """Extract face attributes from a 64-bit integer."""
    return {
        "axis": (value >> 18) & 0x7,
        "positive": (value >> 20) & 0x1,
        "depth": (value >> 12) & 0x3F,
        "u": value & 0x3F,
        "v": (value >> 6) & 0x3F,
        "material": (value >> 21) & 0x1FF
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

def validate() -> None:
    """Orchestrate the full validation suite."""
    # Generate data sources
    expected_faces = generate_faces_from_voxels()
    reference_faces = generate_faces_from_reference_mesh()
    optimized_faces = generate_faces_from_sbgm_mesh()
    
    # Sort faces to ensure canonical order before comparison if not already enforced
    # Note: The source shows sorting in the generation functions
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
Enforces a strict canonical order for faces: FaceDir, then plane, then v, then u [S2, unofficial]. Computes a representation-independent mesh digest using the BLAKE3 hash algorithm [S2, unofficial]. Uses pinned fixture digests for validation against known-good outputs [S2, unofficial]. This allows for efficient comparison of mesh topology regardless of the specific generation order, provided the canonical sort is applied.

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of faces for comparison $O(N)$. Sorting is $O(N \log N)$.
- **Memory Complexity**: Linear $O(N)$ to store the face vectors.
- **Strengths**: Provides precise, human-readable error messages for debugging meshing artifacts by pinpointing the exact bit-level difference.
- **Limitations**: Bitwise comparison is sensitive to any change in face ordering or attribute packing. Requires canonical sorting to be effective for digest-based checks.

## Background
This logic is part of the validation suite for the Streaming Binary Greedy Meshing (SBUG) implementation [S1, unofficial]. It is used in the `spall` project to ensure that the GPU-accelerated meshing output matches a CPU-based reference and a canonical expected output [S2, unofficial].

## Sources
[S1] dmadevice/streaming-binary-greedy-meshing, https://github.com/dmadevice/streaming-binary-greedy-meshing, fetched 2026-10-27 (unofficial)
[S2] T05 — Visible voxel baseline (spall_mesh, spall_render), https://github.com/deranjer/spall/pull/10, fetched 2026-10-27 (unofficial)
