# Recursive Octree Traversal with Face Processing

Also called: DMC Face Processing (faceProcX/Y/Z), Face Processing, faceProcX/Y/Z

## Overview
Recursive Octree Traversal with Face Processing is a method that recursively processes octree faces to resolve subdivisions [S1, unofficial]. It differs from standard ray-casting volume rendering by explicitly handling hierarchical subdivision boundaries during traversal rather than integrating pixel colors [S1, unofficial]. It extends [Recursive Octree Traversal](2632-recursive-octree-traversal.md) by replacing the leaf-node callback with a hierarchical face and edge resolution process [S1, unofficial].

## How it works
**Inputs:** Not given in the sources.
**Outputs:** Not given in the sources.
**Data Structures:** Not given in the sources.
**Algorithm Steps:**
1. Resolve nodes using a `subTable` mapping [S1, unofficial].
2. If `has_lesser` is false, return [S1, unofficial].
3. Recursively call `faceProcX` on pairs of sub-nodes defined by `faceTableX` [S1, unofficial].
4. Recursively call `edgeProcY` on quads of sub-nodes defined by `edgeTableY` [S1, unofficial].
5. `faceProcY` and `faceProcZ` follow the same pattern but use `faceTableY/Z` and `edgeTableZ/Y` respectively [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def face_proc_x(node_a, node_b, face_table, sub_table, has_lesser):
    if not has_lesser:
        return
    resolved_a = sub_table[node_a]
    resolved_b = sub_table[node_b]
    # face_table contains pairs of child indices for recursion
    for idx_a, idx_b in face_table:
        face_proc_x(resolved_a[idx_a], resolved_b[idx_b], face_table, sub_table, has_lesser)

def edge_proc_y(node_a, node_b, node_c, node_d, edge_table, sub_table, has_lesser):
    if not has_lesser:
        return
    resolved = [sub_table[n] for n in (node_a, node_b, node_c, node_d)]
    # edge_table contains quads of child indices for recursion
    for idx_a, idx_b, idx_c, idx_d in edge_table:
        edge_proc_y(resolved[idx_a], resolved[idx_b], resolved[idx_c], resolved[idx_d], edge_table, sub_table, has_lesser)
```
**Edge Cases and Pitfalls:** Not given in the sources.

## Subtopics

### DMC Edge Processing (edgeProcX/Y/Z)
Recursively processes edges of the octree to resolve subdivisions and trigger vertex processing [S1, unofficial]. It uses `edgeTableX/Y/Z` to map child indices for recursion [S1, unofficial]. If any node is subdivided, it uses the child specified by `subTable[i][1]` [S1, unofficial]. If `has_lesser` is false, it returns [S1, unofficial]. After resolving edges, it calls `vertProc` on the 8 resolved sub-nodes [S1, unofficial]. `edgeProcY` and `edgeProcZ` follow the same pattern but use `edgeTableY` and `edgeTableZ` respectively [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] Svenny/isomesh, https://github.com/Svenny/isomesh, fetched 2026-10-27 (unofficial)
