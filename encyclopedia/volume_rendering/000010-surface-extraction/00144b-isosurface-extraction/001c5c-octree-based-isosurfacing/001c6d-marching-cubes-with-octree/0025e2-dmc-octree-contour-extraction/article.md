# DMC Octree Contour Extraction

Also called: Discrete Marching Cubes Octree

## Overview
DMC Octree Contour Extraction is an adaptive isosurface extraction method that traverses a pre-built octree to generate a triangle mesh via dual-vertex construction and contour processing [S1, unofficial]. It solves the problem of extracting high-quality surfaces from volumetric data with varying density by recursively subdividing cells based on field error metrics [S1, unofficial]. It sets itself apart from its sibling [Linear Hashed Marching Cubes](1294-linear-hashed-marching-cubes.md) by using a hierarchical octree structure with explicit contour propagation across shared faces and edges, rather than a flat hash-based lookup, and from [Adaptive Resolution Isosurface Rendering with Two-Pass Discontinuity Handling](1340-adaptive-resolution-isosurface-rendering-with-two-pass-disco.md) by relying on a single-pass recursive traversal with dual-vertex QEF solving rather than a two-pass discontinuity-aware approach [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A root node of a `DMC_Octree` representing the volumetric space, a `ScalarField` providing field values and gradients, and a `BuildArgs` configuration structure.
*   **Output:** A `Mesh` containing the extracted triangle vertices and indices, with global position and scale applied.

**Data Structures**
*   `DMC_OctreeNode`:
    *   `isSubdivided()`: Boolean method indicating if the node has children.
    *   `operator[]`: Accessor for child nodes (array of 8).
    *   `normal`: `glm::vec3`, stores the averaged normal for dual vertices.
    *   `dualVertex`: `glm::vec4`, stores the solved vertex position (x, y, z, w).
    *   `material`: Stores material information if `vertex.w <= 0`.
*   `BuildArgs`:
    *   `use_early_split_stop`: `bool`, enables early stopping based on field magnitude.
    *   `use_simple_split_policy`: `bool`, enables simple splitting based on root size or error.
    *   `use_random_sampling`: `bool`, enables random sampling for dual vertex generation.
    *   `epsilon`: `double`, error threshold for splitting and vertex solving.
    *   `solver`: `QefSolver4D`, accumulates planes for least-squares vertex solving.
    *   `field`: `ScalarField`, provides `getValue(pos)` and `getGradient(pos)`.
    *   `rng`: Random number generator state.

**Algorithm Steps**

1.  **Initialization**: Create an empty `Mesh` and `VertexMap`. Set the global position and scale on the mesh using the octree's transformation parameters. Call `cellProc` on the root node.
2.  **Cell Processing (`cellProc`)**:
    *   If the current node is not subdivided, return.
    *   Retrieve the 8 child nodes into an array `sub[8]`.
    *   **Face Processing**: Iterate `i` from 0 to 3. Call `faceProcX`, `faceProcY`, and `faceProcZ` with the two child nodes defining each face, determined by `faceTableX/Y/Z[i]`.
    *   **Edge Processing**: Iterate `i` from 0 to 1. Call `edgeProcX`, `edgeProcY`, and `edgeProcZ` with the four child nodes defining each edge, determined by `edgeTableX/Y/Z[i]`.
    *   **Vertex Processing**: Call `vertProc` with all 8 child nodes `sub[0]` through `sub[7]`.
3.  **Face Processing (`faceProcX/Y/Z`)**:
    *   Initialize `subTable` mapping 8 indices to parent nodes and their child index.
    *   Iterate `i` from 0 to 7: If the parent node is subdivided, `sub[i]` is its specific child; otherwise, `sub[i]` is the parent node itself.
    *   Set `has_lesser = true` if any node was subdivided.
    *   If `has_lesser` is false, return immediately (no geometry for this face).
    *   **Recursive Face Calls**: Iterate `i` from 0 to 3. Call the corresponding `faceProc` function recursively with the two sub-nodes defined by `faceTable[i]`.
    *   **Edge Calls**: Iterate `i` from 0 to 1. Call the corresponding `edgeProc` function with the four sub-nodes defined by `edgeTable[i]`.
    *   **Vertex Call**: Call `vertProc` with all 8 sub-nodes.
4.  **Edge Processing (`edgeProcX/Y/Z`)**:
    *   (Details omitted in source notes, but implied to handle edge contours using the 4 defining child nodes).
5.  **Vertex Processing (`vertProc`)**:
    *   (Details omitted in source notes, but implied to handle vertex contours using the 8 defining child nodes).
6.  **Node Building (`buildNode`)**:
    *   If `size` is 1, generate a dual vertex and return.
    *   If `use_early_split_stop` is true and `shouldStopSplitting` returns true, generate a dual vertex and return.
    *   If `use_simple_split_policy` is true:
        *   If `size` equals `m_rootSize` or `shouldSplit` returns true, subdivide the node. Calculate `child_size` as `size/2`. Recursively call `buildNode` for all 8 children with their respective `min_corners`.
        *   Otherwise, generate a dual vertex.
    *   If `use_simple_split_policy` is false:
        *   If `size * 8 < m_rootSize` and `generateDualVertex` returns true, return.
        *   Otherwise, subdivide the node, calculate `child_size` as `size/2`, and recursively call `buildNode` for all 8 children.
7.  **Dual Vertex Generation (`generateDualVertex`)**:
    *   Reset the QEF solver. Initialize `avg_normal` to zero.
    *   **Sampling**:
        *   If `use_random_sampling` is true: Calculate `points_cnt` as `int(6.0 * sqrt(size))`. Create a uniform distribution `odist(0, size)`. For `points_cnt` iterations, generate a random offset, compute local and global points, get the field value and gradient, and add a plane defined by `point_4d` (local position, local value) and `normal_4d` (gradient, -1.0) to the solver. Accumulate the gradient in `avg_normal`.
        *   If `use_random_sampling` is false: Define `min_offset` as `0.25 * size` and `max_offset` as `size - min_offset`. Define 8 sample offsets at these corners. For each of the 8 samples, compute local/global points, field value, gradient, and add a plane to the solver. Accumulate the gradient in `avg_normal`.
    *   Normalize `avg_normal` and store it in `node->normal`.
    *   Define `lower_bound` as `(min_corner, 0)` and `upper_bound` as `(min_corner + size, 0)`. Solve for the vertex using the solver with these bounds.
    *   Evaluate the error of the solved vertex. If `error > args.epsilon`, relax bounds to lowest/max float values and re-solve. Store the final vertex in `node->dualVertex`.
    *   If `vertex.w <= 0`, compute the global value and material, and store the material in `node->material`.
    *   Return true if `error <= args.epsilon`, false otherwise.
8.  **Splitting Decision (`shouldSplit`)**:
    *   Compute the 8 global corner positions of the cell. Compute the field values at these corners.
    *   Initialize `error` to 0.
    *   **Edge Midpoints**: Define `dim1_corners`, `dim1_id_offsets`, and `dim1_offsets` for edge midpoints. For each dimension (0-2) and each of the 4 edges in that dimension, compute the midpoint position, predict the value as the average of the two endpoint values, get the actual field value and gradient magnitude `k`, and add `abs(predict - actual) / k` to `error`. If `error > epsilon`, return true.
    *   **Face Midpoints**: Define `dim2_corners`, `dim2_id_offsets`, and `dim2_offsets` for face midpoints. For each dimension (0-2) and each of the 2 faces in that dimension, compute the face center position, predict the value as the average of the 4 corner values, get the actual field value and gradient magnitude `k`, and add `abs(predict - actual) / k` to `error`. If `error > epsilon`, return true.
    *   **Cell Center**: Compute the cell center position. Predict the value as the average of all 8 corner values. Get the actual field value and gradient magnitude `k`. Add `abs(predict - actual) / k` to `error`. If `error > epsilon`, return true.
    *   Return false if the accumulated error did not exceed `epsilon`.
9.  **Early Stop Decision (`shouldStopSplitting`)**:
    *   Compute the base global point and the cell side length. Compute the diagonal length as `side * sqrt(3)`.
    *   For each of the 8 corners, if the absolute value of the field is less than or equal to the diagonal length, return false (do not stop splitting).
    *   If all 8 corners have field values with absolute value greater than the diagonal, return true (stop splitting).

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

class DMCOctreeContour:
    def __init__(self, root_node, field, args):
        self.root = root_node
        self.field = field
        self.args = args
        self.mesh = Mesh()
        self.vertex_map = VertexMap()
        
    def extract(self):
        self.mesh.set_global_transform(self.root.global_transform)
        self.cell_proc(self.root)
        return self.mesh

    def cell_proc(self, node):
        if not node.is_subdivided():
            return
        
        sub = [node[i] for i in range(8)]
        
        # Process faces
        for i in range(4):
            self.face_proc_x(sub[face_table_x[i][0]], sub[face_table_x[i][1]])
            self.face_proc_y(sub[face_table_y[i][0]], sub[face_table_y[i][1]])
            self.face_proc_z(sub[face_table_z[i][0]], sub[face_table_z[i][1]])
            
        # Process edges
        for i in range(2):
            self.edge_proc_x(sub[edge_table_x[i][0]], sub[edge_table_x[i][1]], sub[edge_table_x[i][2]], sub[edge_table_x[i][3]])
            self.edge_proc_y(sub[edge_table_y[i][0]], sub[edge_table_y[i][1]], sub[edge_table_y[i][2]], sub[edge_table_y[i][3]])
            self.edge_proc_z(sub[edge_table_z[i][0]], sub[edge_table_z[i][1]], sub[edge_table_z[i][2]], sub[edge_table_z[i][3]])
            
        # Process vertices
        self.vert_proc(sub)

    def face_proc_x(self, node_a, node_b):
        sub_table = []
        has_lesser = False
        for i in range(8):
            parent = self.get_parent_for_face_x(i) # Implementation detail: maps face index to parent node
            if parent.is_subdivided():
                sub_table.append(parent[i])
                has_lesser = True
            else:
                sub_table.append(parent)
        
        if not has_lesser:
            return

        for i in range(4):
            self.face_proc_x(sub_table[face_table_x[i][0]], sub_table[face_table_x[i][1]])
        for i in range(2):
            self.edge_proc_y(sub_table[edge_table_y[i][0]], sub_table[edge_table_y[i][1]], sub_table[edge_table_y[i][2]], sub_table[edge_table_y[i][3]])
        for i in range(2):
            self.edge_proc_z(sub_table[edge_table_z[i][0]], sub_table[edge_table_z[i][1]], sub_table[edge_table_z[i][2]], sub_table[edge_table_z[i][3]])
        self.vert_proc(sub_table)

    def face_proc_y(self, node_a, node_b):
        sub_table = []
        has_lesser = False
        for i in range(8):
            parent = self.get_parent_for_face_y(i)
            if parent.is_subdivided():
                sub_table.append(parent[i])
                has_lesser = True
            else:
                sub_table.append(parent)
        
        if not has_lesser:
            return

        for i in range(4):
            self.face_proc_y(sub_table[face_table_y[i][0]], sub_table[face_table_y[i][1]])
        for i in range(2):
            self.edge_proc_x(sub_table[edge_table_x[i][0]], sub_table[edge_table_x[i][1]], sub_table[edge_table_x[i][2]], sub_table[edge_table_x[i][3]])
        for i in range(2):
            self.edge_proc_z(sub_table[edge_table_z[i][0]], sub_table[edge_table_z[i][1]], sub_table[edge_table_z[i][2]], sub_table[edge_table_z[i][3]])
        self.vert_proc(sub_table)

    def face_proc_z(self, node_a, node_b):
        sub_table = []
        has_lesser = False
        for i in range(8):
            parent = self.get_parent_for_face_z(i)
            if parent.is_subdivided():
                sub_table.append(parent[i])
                has_lesser = True
            else:
                sub_table.append(parent)
        
        if not has_lesser:
            return

        for i in range(4):
            self.face_proc_z(sub_table[face_table_z[i][0]], sub_table[face_table_z[i][1]])
        for i in range(2):
            self.edge_proc_x(sub_table[edge_table_x[i][0]], sub_table[edge_table_x[i][1]], sub_table[edge_table_x[i][2]], sub_table[edge_table_x[i][3]])
        for i in range(2):
            self.edge_proc_y(sub_table[edge_table_y[i][0]], sub_table[edge_table_y[i][1]], sub_table[edge_table_y[i][2]], sub_table[edge_table_y[i][3]])
        self.vert_proc(sub_table)

    def build_node(self, node, size, min_corner):
        if size == 1:
            self.generate_dual_vertex(node, min_corner, size)
            return

        if self.args.use_early_split_stop and self.should_stop_splitting(node, min_corner, size):
            self.generate_dual_vertex(node, min_corner, size)
            return

        if self.args.use_simple_split_policy:
            if size == self.m_root_size or self.should_split(node, min_corner, size):
                child_size = size / 2
                for i in range(8):
                    child_min = min_corner + self.get_child_offset(i, child_size)
                    self.build_node(node[i], child_size, child_min)
            else:
                self.generate_dual_vertex(node, min_corner, size)
        else:
            if size * 8 < self.m_root_size and self.generate_dual_vertex(node, min_corner, size):
                return
            child_size = size / 2
            for i in range(8):
                child_min = min_corner + self.get_child_offset(i, child_size)
                self.build_node(node[i], child_size, child_min)

    def generate_dual_vertex(self, node, min_corner, size):
        solver = QefSolver4D()
        avg_normal = Vec3(0, 0, 0)
        
        if self.args.use_random_sampling:
            points_cnt = int(6.0 * math.sqrt(size))
            for _ in range(points_cnt):
                offset = random.uniform(0, size)
                local_pos = min_corner + Vec3(offset, offset, offset) # Simplified random point generation
                global_pos = self.transform(local_pos)
                val = self.field.get_value(global_pos)
                grad = self.field.get_gradient(global_pos)
                k = grad.length()
                if k > 0:
                    normal_4d = Vec4(grad.x, grad.y, grad.z, -1.0)
                    point_4d = Vec4(local_pos.x, local_pos.y, local_pos.z, val)
                    solver.add_plane(normal_4d, point_4d)
                    avg_normal += grad
        else:
            min_offset = 0.25 * size
            max_offset = size - min_offset
            offsets = [
                Vec3(min_offset, min_offset, min_offset),
                Vec3(max_offset, min_offset, min_offset),
                Vec3(min_offset, max_offset, min_offset),
                Vec3(max_offset, max_offset, min_offset),
                Vec3(min_offset, min_offset, max_offset),
                Vec3(max_offset, min_offset, max_offset),
                Vec3(min_offset, max_offset, max_offset),
                Vec3(max_offset, max_offset, max_offset)
            ]
            for off in offsets:
                local_pos = min_corner + off
                global_pos = self.transform(local_pos)
                val = self.field.get_value(global_pos)
                grad = self.field.get_gradient(global_pos)
                k = grad.length()
                if k > 0:
                    normal_4d = Vec4(grad.x, grad.y, grad.z, -1.0)
                    point_4d = Vec4(local_pos.x, local_pos.y, local_pos.z, val)
                    solver.add_plane(normal_4d, point_4d)
                    avg_normal += grad
        
        avg_normal = avg_normal.normalize()
        node.normal = avg_normal
        
        lower_bound = Vec4(min_corner.x, min_corner.y, min_corner.z, 0)
        upper_bound = Vec4(min_corner.x + size, min_corner.y + size, min_corner.z + size, 0)
        
        vertex = solver.solve(lower_bound, upper_bound)
        error = self.evaluate_error(vertex, min_corner, size)
        
        if error > self.args.epsilon:
            lower_bound = Vec4(-float('inf'), -float('inf'), -float('inf'), 0)
            upper_bound = Vec4(float('inf'), float('inf'), float('inf'), 0)
            vertex = solver.solve(lower_bound, upper_bound)
            
        node.dual_vertex = vertex
        
        if vertex.w <= 0:
            global_val = self.field.get_value(self.transform(Vec3(vertex.x, vertex.y, vertex.z)))
            node.material = self.compute_material(global_val)
            
        return error <= self.args.epsilon

    def should_split(self, node, min_corner, size):
        corners = [min_corner + self.get_child_offset(i, size) for i in range(8)]
        corner_vals = [self.field.get_value(self.transform(c)) for c in corners]
        
        error = 0.0
        
        # Edge midpoints
        for dim in range(3):
            for edge in range(4):
                mid_idx = self.get_edge_midpoint_index(dim, edge)
                p1_idx, p2_idx = self.get_edge_endpoints(dim, edge)
                mid_pos = (corners[p1_idx] + corners[p2_idx]) / 2.0
                predicted_val = (corner_vals[p1_idx] + corner_vals[p2_idx]) / 2.0
                actual_val = self.field.get_value(self.transform(mid_pos))
                grad = self.field.get_gradient(self.transform(mid_pos))
                k = grad.length()
                if k > 0:
                    error += abs(predicted_val - actual_val) / k
                if error > self.args.epsilon:
                    return True
                    
        # Face midpoints
        for dim in range(3):
            for face in range(2):
                face_indices = self.get_face_indices(dim, face)
                mid_pos = sum([corners[i] for i in face_indices]) / 4.0
                predicted_val = sum([corner_vals[i] for i in face_indices]) / 4.0
                actual_val = self.field.get_value(self.transform(mid_pos))
                grad = self.field.get_gradient(self.transform(mid_pos))
                k = grad.length()
                if k > 0:
                    error += abs(predicted_val - actual_val) / k
                if error > self.args.epsilon:
                    return True
                    
        # Cell center
        center_pos = sum(corners) / 8.0
        predicted_val = sum(corner_vals) / 8.0
        actual_val = self.field.get_value(self.transform(center_pos))
        grad = self.field.get_gradient(self.transform(center_pos))
        k = grad.length()
        if k > 0:
            error += abs(predicted_val - actual_val) / k
            
        return error > self.args.epsilon

    def should_stop_splitting(self, node, min_corner, size):
        side = size
        diagonal = side * math.sqrt(3)
        corners = [min_corner + self.get_child_offset(i, size) for i in range(8)]
        
        for c in corners:
            global_pos = self.transform(c)
            val = self.field.get_value(global_pos)
            if abs(val) <= diagonal:
                return False
        return True
```

**Parameters**
*   `epsilon`: The error threshold for dual vertex generation and splitting decisions. Typical value is not specified in this code snippet but is accessed via `args.epsilon` [S1, unofficial].
*   `use_early_split_stop`: Boolean flag to enable early stopping of subdivision based on field value magnitude relative to cell diagonal [S1, unofficial].
*   `use_simple_split_policy`: Boolean flag to enable simple splitting policy based on root size or `shouldSplit` result [S1, unofficial].
*   `use_random_sampling`: Boolean flag to enable random sampling for dual vertex generation instead of fixed corner sampling [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision**: The QEF solver uses 4D planes. If the gradient magnitude `k` is zero, the plane addition is skipped to avoid division by zero in error calculation [S1, unofficial].
*   **Boundaries**: If the initial QEF solve fails (error > epsilon), bounds are relaxed to infinity to ensure a solution is found, potentially at the cost of accuracy [S1, unofficial].
*   **Degenerate Input**: If `size` is 1, a dual vertex is generated immediately without sampling, which may lead to artifacts if the field is not well-behaved at the smallest scale [S1, unofficial].

## Subtopics

### [MDC Octree Isosurface Extraction](3676-mdc-octree-isosurface-extraction.md)
MDC (Multi-Dimensional Cell) Octree Isosurface Extraction is an adaptive surface reconstruction algorithm that uses an octree data structure to recursively subdivide a 3D volume and generate a triangle mesh. It solves the problem of extracting isosurfaces from volumetric data with varying density, handling topology via vertex clustering and collapse logic to ensure manifold connectivity. It sets itself apart from its sibling [DMC Octree Contour Extraction](3661-dmc-octree-contour-extraction.md) by explicitly managing vertex clustering across shared edges and faces to resolve topological ambiguities in complex subdivisions, whereas DMC relies on a different contour extraction logic.

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Not given in the sources.
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Not given in the sources.
*   **Strengths**: Adaptive resolution allows for high detail where needed and coarser representation elsewhere. Dual-vertex construction via QEF can produce smoother surfaces than linear interpolation.
*   **Limitations**: Relies on a pre-built octree. The complexity of contour propagation across faces and edges is not fully detailed in the sources.
*   **When to use**: When adaptive resolution is required and the volume data supports efficient octree construction.
*   **When not to use**: When a flat, uniform resolution is sufficient, or when memory constraints prevent storing the octree structure.

## Background
*   **Origin**: The method is described in the unofficial source [S1, unofficial] by Svenny/isomesh.
*   **History**: Not given in the sources.
*   **Usage**: Used in the isomesh project for isosurface extraction.

## Sources
[S1] Svenny/isomesh, https://github.com/Svenny/isomesh, fetched 2026-10-27 (unofficial)
