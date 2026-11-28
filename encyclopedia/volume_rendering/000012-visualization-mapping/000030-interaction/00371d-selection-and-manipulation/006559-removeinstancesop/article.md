# RemoveInstancesOp

Also called: Instance Deletion with Undo, Ordered Instance Removal

## Overview
`RemoveInstancesOp` is an interaction primitive that removes selected Gaussian instances from the live rendering list while preserving their underlying data to enable strict Last-In-First-Out (LIFO) undo operations [S1, unofficial]. It differs from standard undo mechanisms by enforcing a rigid edit history where older operations cannot be undone until the current operation is reversed, ensuring that selection ranges remain meaningful for subsequent actions [S1, unofficial].

## How it works
The method operates on a list of Gaussian splats, managing their active state and history.

**Inputs and Outputs:**
*   **Input:** A list of instances with associated state flags (specifically `State.selected` and `State.locked`), and a selection range definition.
*   **Output:** The modified instance list (with selected instances removed from the active set) and a history stack entry containing the removed instances for undo.

**Data Structures:**
*   `instances`: A dynamic list or array of Gaussian splats.
*   `flags`: An array or bitmask parallel to `instances`, where `flags[i]` indicates the state of instance `i`.
*   `removed`: A temporary storage buffer holding the instances removed in the current operation.
*   `ranges`: A data structure defining the contiguous or scattered indices of selected instances.

**Steps:**
1.  **Compute Selection Ranges:** Iterate through the instance list to identify indices where the selection predicate is true.
    *   Predicate: `flags[i] === State.selected` [S1, unofficial].
    *   Constraint: Exclude instances where `flags[i] === State.locked` [S1, unofficial]. Locked instances are immune to removal.
2.  **Execute Removal:**
    *   Call the removal routine on the instance container: `this.splat.instances.remove(this.ranges)` [S1, unofficial].
    *   Store the removed instances in the undo buffer: `this.removed` [S1, unofficial].
    *   Update the rendering state: `updateState()` [S1, unofficial].
3.  **Undo Operation:**
    *   Restore instances: Call `this.splat.instances.insert(this.removed)` [S1, unofficial].
    *   Clear the undo buffer: `this.removed` is cleared [S1, unofficial].
    *   Update the rendering state: `updateState()` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class RemoveInstancesOp:
    def __init__(self, splat_system):
        self.splat = splat_system
        self.removed = []

    def execute(self):
        # 1. Compute selection ranges
        # Identify indices where selected is true AND locked is false
        selected_indices = []
        for i in range(len(self.splat.instances)):
            is_selected = (self.splat.flags[i] == State.selected)
            is_locked = (self.splat.flags[i] == State.locked)
            
            # Pitfall: Locked instances survive deletion
            if is_selected and not is_locked:
                selected_indices.append(i)
        
        # Convert indices to ranges if the underlying structure requires it
        ranges = self._indices_to_ranges(selected_indices)
        
        # 2. Execute removal
        # Remove instances from the live list
        self.removed = self.splat.instances.remove(ranges)
        
        # Update rendering state (e.g., rebuild acceleration structures)
        self.splat.update_state()

    def undo(self):
        # 3. Undo operation
        if not self.removed:
            return

        # Insert previously removed instances back into the list
        self.splat.instances.insert(self.removed)
        
        # Clear the undo buffer
        self.removed = []
        
        # Update rendering state
        self.splat.update_state()

    def _indices_to_ranges(self, indices):
        # Helper to convert a list of indices into contiguous ranges
        # [G]: Standard algorithm to group consecutive integers
        if not indices:
            return []
        
        ranges = []
        start = indices[0]
        end = indices[0]
        
        for i in range(1, len(indices)):
            if indices[i] == end + 1:
                end = indices[i]
            else:
                ranges.append((start, end))
                start = indices[i]
                end = indices[i]
        
        ranges.append((start, end))
        return ranges
```

**Parameters:**
*   `State.selected`: Boolean flag indicating user selection.
*   `State.locked`: Boolean flag preventing modification.
*   Typical values: Not specified in sources.

**Edge Cases and Pitfalls:**
*   **Locked Instances:** Instances marked as locked must be excluded from the removal range even if selected [S1, unofficial].
*   **LIFO Enforcement:** The method strictly enforces LIFO order. Older operations remain blocked from undoing until the current operation is undone [S1, unofficial].

## Subtopics

### Object Removal without Bounding Boxes
In scenarios where instances do not have explicit bounding boxes, object removal is accomplished by reducing the density of points associated with the target instance to zero [S2]. This approach avoids the need for geometric bounding volume calculations, relying instead on attribute manipulation to effectively hide or remove the instance from the rendered scene [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of selected instances for range computation. Removal and insertion complexity depends on the underlying data structure of `splat.instances` (e.g., $O(N)$ for array-based lists, $O(1)$ for linked lists with pointers).
*   **Memory Complexity:** $O(K)$ where $K$ is the number of removed instances, stored in `this.removed` for undo.
*   **Strengths:** Strict LIFO undo history ensures predictable interaction behavior; locked instance protection prevents accidental data loss.
*   **Limitations:** Relies on a specific state flag system; undo history is strictly linear and cannot branch.
*   **When to use:** Interactive editing of Gaussian splat scenes where undo/redo fidelity is critical.
*   **When not to use:** When non-LIFO undo is required or when instances lack a selectable state flag.

## Background
The method originates from the `supersplat` project, an interactive Gaussian Splatting editor [S1, unofficial]. It addresses the need for robust scene editing in neural rendering workflows, where users frequently need to remove specific instances (e.g., artifacts, unwanted objects) while maintaining the ability to reverse mistakes [S1, unofficial]. The strict LIFO undo mechanism was designed to prevent the ambiguity that can arise when multiple overlapping selection operations are undone out of order [S1, unofficial].

## Sources
[S1] playcanvas/supersplat, https://github.com/playcanvas/supersplat, fetched 2026-10-27 (unofficial)
[S2] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
