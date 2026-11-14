# Voxel-Space Slice Rendering (Affine Rewriting Workaround)

Also called: isSliceMM restoration, obliqueRAS identity injection, voxel grid slice alignment

## Overview
This method restores 2D slice rendering on the native voxel grid for volumes with oblique acquisitions by modifying the volume's affine transformation matrix, solving the problem where oblique data appears rotated in world space [S1, unofficial]. Unlike [Volume Rotation via Affine Transform](11626-volume-rotation-via-affine-transform.md) which rotates the volume data or view, this approach rewrites the affine to its nearest axis-aligned equivalent, effectively dropping the shear component while preserving the voxel grid structure [S1, unofficial]. It differs from `ui.isPositionInMM` (v1 successor name) because the latter only affects crosshair coordinate conventions and leaves slice geometry bit-identical to world space [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volume object containing `originalAffine` (immutable, stored at load) and `hdr.affine` (mutable header affine).
- Output: A modified `hdr.affine` that is axis-aligned, resulting in 2D slices being drawn on the native voxel grid.

**Data Structures:**
- `volume.originalAffine`: 4x4 matrix (float32), immutable.
- `volume.hdr.affine`: 4x4 matrix (float32), mutable.
- `volume.obliqueRAS`: Derived 4x4 matrix, computed from affine.
- `volume.frac2mm`: Derived 3x3 matrix, computed from affine.

**Parameters:**
- `worldSpace`: Boolean toggle for rendering mode. Default `false` (voxel grid). [S1, unofficial]
- Keyboard shortcut: `W` to toggle `worldSpace`. Rebindable in VS Code. [S1, unofficial]

**Steps:**
1. **Check Axis-Alignment:** Determine if the volume is already axis-aligned. If `volume.hdr.affine` is already axis-aligned (diagonal or permutation matrix with no shear), skip modification [S1, unofficial].
2. **Derive Axis-Aligned Affine:** Using the public affine API, replace `volume.hdr.affine` with its nearest axis-aligned equivalent derived from `volume.originalAffine` [S1, unofficial].
   - Preserve each voxel axis's dominant direction, sign, and length to prevent data permutation or mirroring [S1, unofficial].
   - Ensure the world origin remains on the same voxel to prevent crosshair jumps during toggling [S1, unofficial].
   - The core system then derives an identity `obliqueRAS` from the new affine, laying slices back on the voxel grid [S1, unofficial].
3. **Apply to Canvas:** The correction is derived from the background volume's `originalAffine` and applied to all volumes on the canvas to maintain registration [S1, unofficial].
4. **Handle Overlays:** If overlays are added while in voxel-space mode, re-emit `nvArray` via the overlay message handler to ensure they use the corrected affine and stay registered [S1, unofficial].
5. **Reset:** To restore world-space rendering, call `resetVolumeAffine` to restore the original `hdr.affine` exactly [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def apply_slice_space(volume, world_space: bool):
    """
    Modifies volume.hdr.affine to enforce voxel-space rendering if world_space is False.
    Restores original affine if world_space is True.
    """
    if world_space:
        # Restore world-space rendering exactly
        volume.hdr.affine = copy_matrix(volume.originalAffine)
        return

    # Check if already axis-aligned
    if is_axis_aligned(volume.hdr.affine):
        return

    # Derive nearest axis-aligned equivalent from originalAffine
    # This preserves dominant direction, sign, and length of each voxel axis
    # and keeps the world origin on the same voxel
    new_affine = derive_axis_aligned_affine(volume.originalAffine)
    
    # Apply the new affine
    volume.hdr.affine = new_affine
    
    # The core system will recompute obliqueRAS, frac2mm, etc.
    # Ensure overlays are re-emitted if added during this mode
    # This is handled by the overlay message handler in the UI layer
```

**Edge Cases and Pitfalls:**
- **Meshes:** Meshes are not moved onto the voxel grid when the affine is rewritten [S1, unofficial].
- **MM Readout:** The mm readout switches to voxel-grid mm instead of scanner mm [S1, unofficial].
- **Persistence:** Modifying `hdr.affine` makes the change visible in the header dialog and persists in `.nvd` saves if active [S1, unofficial].
- **Precision:** Registration accuracy between background and overlay world-to-display maps matches to within $2 \times 10^{-14}$ in voxel space [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per volume for affine derivation and application, as it involves matrix operations on fixed-size 4x4 matrices.
- **Memory Complexity:** $O(1)$ additional memory, as no new data structures are created; the affine matrix is modified in place.
- **Measured Performance:** Verification on a ~19° oblique FLAIR.nii.gz showed that `obliqueRAS` max off-diagonal is 0 in voxel space vs 0.309 in world space [S1, unofficial].
- **Strengths:** Restores intuitive 2D slice viewing on the native voxel grid for oblique acquisitions without resampling or rotating the volume data.
- **Limitations:** Meshes do not align with the voxel grid; mm readouts are in voxel units, not scanner units; affine changes persist in saved files.
- **When to Use:** When viewing oblique acquisitions in 2D slice mode and the user expects the slices to align with the acquisition grid rather than world space.
- **When Not to Use:** When precise world-space alignment is required for meshes or when scanner-space mm coordinates are needed for readouts.

## Background
This method was implemented in `packages/niivue-react/src/sliceSpace.ts` for the niivue VS Code extension and PWA [S1, unofficial]. It addresses a regression in niivue v1.0, where the `opts.isSliceMM` option from 0.x was dropped, causing oblique acquisitions to always render in world space [S1, unofficial]. The workaround uses the public affine API to rewrite the volume's affine matrix, leveraging existing internal calculations (`frac2mmOrtho`, etc.) that were previously unused by the view [S1, unofficial]. The author plans to open an issue on the niivue core repo for a first-class slice-space option, after which this module may be deleted [S1, unofficial].

## Sources
[S1] feat(viewer): draw slices on the voxel grid by default, add a World Space toggle (#266), https://github.com/niivue/niivue-vscode/pull/268, fetched 2026-10-27 (unofficial)
