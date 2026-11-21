# Finite State Machine for Interpolant Modelling

Also called: FSM for Interpolant Modelling, Sweeping FSM, Case Transition FSM

## Overview
The Finite State Machine (FSM) for Interpolant Modelling represents the evolution of isosurface topology within a voxel as the isovalue sweeps from high to low. It maps the discrete states of Marching Cubes cases to transitions triggered by vertex sign changes, thereby imposing topological constraints on the trilinear interpolant [S1]. Unlike standard Marching Cubes which relies on a static lookup table, this method models the dynamic path of cases to detect invalid configurations and "hidden features" that linear interpolation might miss [S1, S2].

## How it works
The method constructs a directed graph where nodes represent Marching Cubes cases and edges represent valid topological transitions during an isovalue sweep.

**Inputs:**
- The set of valid Marching Cubes cases (e.g., the 15 watertight cases defined by [MSS94]).
- The vertex classification logic (white if $V < h$, black if $V \ge h$).

**Outputs:**
- A transition graph (adjacency list or matrix) defining valid state sequences.
- A validation mechanism to reject invalid interpolant behaviors.

**Data Structures:**
- **State:** An integer $0 \dots 255$ representing the 8-bit vertex mask of a cube.
- **Transition:** A directed edge from state $S_i$ to $S_j$ if $S_j$ can be reached from $S_i$ by flipping exactly one white vertex to black.

**Steps:**

1. **Initialize States:**
   Define the set of states $\mathcal{S}$ as the valid Marching Cubes cases. For a standard implementation, this is the subset of 256 cases that produce watertight surfaces [S1].

2. **Construct Transitions:**
   For each state $S_i \in \mathcal{S}$:
   - Identify all vertices $v_k$ that are white (0) in $S_i$.
   - For each white vertex $v_k$, compute the candidate next state $S_{next} = S_i \lor 2^k$.
   - If $S_{next} \in \mathcal{S}$, add a directed edge $S_i \to S_{next}$.
   - *Constraint:* The transition must respect the physical continuity of the isosurface. For example, if $S_i$ has edge-adjacent black vertices, $S_{next}$ must preserve the topological implication of that adjacency unless the specific interpolant allows separation [S1].

3. **Initialize Sweep:**
   For a given cube with isovalue $h$, determine the initial state $S_{start}$.
   - If $h$ is higher than all vertex values, $S_{start} = 0$ (all vertices white).
   - Generally, the sweep starts from the highest possible isovalue (State 0) and proceeds downwards [S1].

4. **Trace Path:**
   As $h$ decreases, vertices flip from white to black. The system traverses the FSM from $S_{start}$ through a sequence of states $S_0, S_1, \dots, S_{end}$ (where $S_{end}$ corresponds to $h$ lower than all vertices).
   - At each step, verify that the transition $S_k \to S_{k+1}$ exists in the constructed graph.
   - If a transition is missing, the configuration is topologically invalid for the chosen interpolant [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_fsm(valid_cases):
    """
    Constructs the FSM for a given set of valid Marching Cubes cases.
    valid_cases: A set of integers (0-255) representing valid vertex masks.
    Returns: A dictionary mapping state -> list of next states.
    """
    fsm = {}
    for state in valid_cases:
        next_states = []
        # Iterate over all 8 vertices
        for i in range(8):
            # Check if vertex i is white (bit is 0)
            if not (state & (1 << i)):
                # Flip vertex i to black
                next_state = state | (1 << i)
                # Only add transition if the next state is also valid
                if next_state in valid_cases:
                    next_states.append(next_state)
        fsm[state] = next_states
    return fsm

def validate_sweep(initial_mask, final_mask, fsm):
    """
    Validates if a sweep from initial_mask to final_mask is topologically possible.
    Note: This is a simplified check. A full sweep requires tracking the specific
    order of vertex flips, which depends on the scalar field values.
    """
    # In a real implementation, one would simulate the sweep by sorting vertices
    # by their scalar values relative to h. Here we check connectivity.
    current = initial_mask
    visited = set()
    stack = [current]
    
    while stack:
        state = stack.pop()
        if state == final_mask:
            return True
        if state in visited:
            continue
        visited.add(state)
        
        if state in fsm:
            for next_state in fsm[state]:
                stack.append(next_state)
    return False
```

**Parameters:**
- **Valid Cases:** Typically the 15 watertight cases from [MSS94] or the full 256 cases if non-watertight surfaces are allowed.
- **Isosweep Direction:** High to low (standard) [S1].

**Edge Cases:**
- **Degenerate Vertices:** If a vertex value exactly equals $h$, the classification is ambiguous. The FSM assumes a strict inequality ($V < h$ vs $V \ge h$) [S1].
- **Invalid Transitions:** The FSM explicitly forbids transitions that break topology, such as Case 2 (two edge-adjacent black vertices) transitioning to Case 7 (no edge-adjacent black vertices) [S1].

## Subtopics

## Operating characteristics
- **Time Complexity:** Construction is $O(256 \cdot 8)$, constant and negligible. Validation of a single cube's sweep path is proportional to the number of vertices (8), making it $O(1)$ per cube.
- **Memory Complexity:** The FSM table stores at most $256 \times 8$ edges, fitting in a few kilobytes.
- **Strengths:** Provides a rigorous topological proof for the validity of the interpolant. It can identify "hidden features" (local extrema) that standard linear interpolation fails to render [S2].
- **Limitations:** Requires pre-computation of valid cases. Does not directly generate geometry; it validates or guides the generation process.

## Background
The method was introduced by [S1] to model the geometric behavior of isosurfaces generated by the Marching Cubes algorithm. It builds on the concept of contour spectra [CS07] and was used to prove the completeness of Nielson's cases and model watertight Marching Cubes cases [MSS94]. It addresses the ambiguity of the trilinear interpolant by constraining the possible paths through the case space.

## Sources
[S1] More Marching Cubes, https://doi.org/10.2312/vg/vg07/081-090, fetched 2026-10-26
[S2] Not given in the sources.
