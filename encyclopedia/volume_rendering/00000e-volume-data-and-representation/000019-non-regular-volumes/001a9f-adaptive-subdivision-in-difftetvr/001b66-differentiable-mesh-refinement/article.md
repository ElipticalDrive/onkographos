# Differentiable Mesh Refinement

Also called: Gradient-based mesh refinement

## Overview
Differentiable Mesh Refinement is a post-processing optimization step that improves the geometric fidelity of a triangular mesh by adjusting vertex positions to better fit an underlying implicit occupancy field [S1, unofficial]. It distinguishes itself from topological refinement methods (such as [Recursive Subdivision](1304-recursive-subdivision.md)) by operating directly on the existing mesh topology, using face subsampling to maintain computational efficiency during the optimization loop [S1, unofficial]. This method builds on [Adaptive Subdivision in DiffTetVR](807-adaptive-subdivision-in-difftetvr.md) by providing a final polishing step that decouples geometry adjustment from topological changes [G].

## How it works
**Inputs and Outputs:**
*   **Input:** Initial triangular mesh vertex positions $V_0$, a trained neural network implicit field model `model`, a target threshold $\tau$ (typically $0.5$ for sigmoid outputs), conditioning inputs $c$ (e.g., camera pose), maximum refinement steps $K$, and batch size $B$ [S1, unofficial].
*   **Output:** Optimized vertex positions $V^*$ that minimize the discrepancy between the mesh surface and the implicit field's thresholded level set [S1, unofficial].

**Data Structures:**
*   **Vertices:** A PyTorch `torch.nn.Parameter` tensor of shape $(N_v, 3)$, where $N_v$ is the number of vertices [S1, unofficial].
*   **Faces:** A tensor of shape $(N_f, 3)$ containing indices into the vertex tensor [S1, unofficial].
*   **Optimizer:** RMSprop optimizer instance configured for the vertex parameters [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Initialize the mesh vertices as a trainable parameter $V = \text{Parameter}(V_0)$ and configure an RMSprop optimizer with learning rate $\eta = 10^{-5}$ [S1, unofficial].
2.  **Iteration:** For each refinement step $k = 1 \dots K$:
    a. **Face Subsampling:** Select a batch of $B$ faces to manage GPU memory [S1, unofficial].
    b. **Point Sampling:** For each face, sample points on the triangle surface using barycentric coordinates drawn from a Dirichlet distribution with parameters $(0.5, 0.5, 0.5)$ [S1, unofficial].
    c. **Geometric Normal:** Compute the face normal from the current vertex positions and normalize it with a numerical stability constant [S1, unofficial].
    d. **Implicit Field Query:** Query the neural network for scalar field values at the sampled points, passing conditioning inputs $c$ [S1, unofficial]. Split large batches into chunks of $20,000$ points to prevent memory overflow [S1, unofficial].
    e. **Target Normal:** Compute the target normal via automatic differentiation of the scalar field with respect to the point positions, then normalize [S1, unofficial].
    f. **Loss Calculation:** Compute a combined loss measuring field value proximity to $\tau$ and alignment between geometric and target normals [S1, unofficial].
    g. **Optimization Step:** Backpropagate through the loss to compute gradients with respect to $V$, and update vertices using the optimizer [S1, unofficial].

**Formulas:**
*   Sampled point: $p = \sum_{i=0}^{2} \epsilon_i v_i$, where $\epsilon \sim \text{Dirichlet}((0.5, 0.5, 0.5))$ [S1, unofficial].
*   Geometric normal: $n_{geo} = \frac{(v_1 - v_0) \times (v_2 - v_1)}{\|(v_1 - v_0) \times (v_2 - v_1)\| + 10^{-10}}$ [S1, unofficial].
*   Target normal: $n_{target} = \frac{-\nabla_p S}{\|-\nabla_p S\| + 10^{-10}}$, where $S = \sigma(f_{\text{decode}}(p, c))$ [S1, unofficial].
*   Loss: $L = \underbrace{\frac{1}{B}\sum_{j=1}^{B} (S_j - \tau)^2}_{L_{val}} + 0.01 \underbrace{\frac{1}{B}\sum_{j=1}^{B} \|n_{geo,j} - n_{target,j}\|^2}_{L_{norm}}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
v = torch.nn.Parameter(v0.clone())
optimizer = optim.RMSprop([v], lr=1e-5)

for step in range(self.refinement_step):
    face_batch = faces.sample(self.refine_max_faces)
    face_vertex = v[face_batch]  # (B, 3, 3)
    
    eps = np.random.dirichlet((0.5, 0.5, 0.5), size=face_batch.shape[0])
    face_point = (face_vertex * eps[:, :, None]).sum(dim=1)  # (B, 3)
    
    face_v1 = face_vertex[:, 1, :] - face_vertex[:, 0, :]
    face_v2 = face_vertex[:, 2, :] - face_vertex[:, 1, :]
    face_normal = torch.cross(face_v1, face_v2)
    face_normal = face_normal / (face_normal.norm(dim=1, keepdim=True) + 1e-10)
    
    # Chunked field query
    face_value = []
    for p_chunk in split(face_point, 20000):
        logits = model.decode(p_chunk, c)
        face_value.append(torch.sigmoid(logits))
    face_value = torch.cat(face_value, dim=0)
    
    normal_target = -autograd.grad([face_value.sum()], [face_point], create_graph=True)[0]
    normal_target = normal_target / (normal_target.norm(dim=1, keepdim=True) + 1e-10)
    
    loss_target = (face_value - threshold).pow(2).mean()
    loss_normal = (face_normal - normal_target).pow(2).sum(dim=1).mean()
    loss = loss_target + 0.01 * loss_normal
    
    loss.backward()
    optimizer.step()
    optimizer.zero_grad()
```

**Parameters:**
*   Learning rate $\eta$: $10^{-5}$ [S1, unofficial].
*   Normal loss weight $\lambda$: $0.01$ [S1, unofficial].
*   Batch size (faces per step): Determined by `self.refine_max_faces` [S1, unofficial].
*   Chunk size for field query: $20,000$ points [S1, unofficial].
*   Numerical stability constant: $10^{-10}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Faces:** If a face has zero area, the geometric normal computation yields zero vectors. The addition of $10^{-10}$ prevents division by zero but does not correct the gradient direction [G]. Ensure input meshes do not contain degenerate triangles before optimization [G].
*   **Precision:** The use of `create_graph=True` in automatic differentiation allows gradients to flow through the field query, but deep networks may suffer from vanishing gradients [G]. Monitor gradient norms during training [G].
*   **Boundary Conditions:** The method assumes the implicit field is well-defined in the vicinity of the mesh. If the mesh moves far from the trained region of the field, the gradients may become meaningless [G]. Constrain vertex updates if necessary [G].

## Subtopics

### [Mesh Refinement via Vertex Optimization](989-mesh-refinement-via-vertex-optimization.md)
Improves geometric fidelity by adjusting vertex positions to fit an occupancy field. Distinguishes itself by operating directly on mesh topology with face subsampling. Choose when a final polishing step is needed after topological refinement.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of sampled points per batch, as each iteration requires one forward pass through the neural network and one automatic differentiation pass [S1, unofficial].
*   **Memory Complexity:** Proportional to the batch size of faces and the chunk size for field queries, with memory usage managed by subsampling faces and splitting large point batches [S1, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Decouples geometry optimization from topology changes; computationally efficient via subsampling; leverages differentiable rendering pipelines for end-to-end training [S1, unofficial].
*   **Limitations:** Requires a pre-trained implicit field; sensitive to initial mesh quality; may converge to local minima if the initial mesh is far from the target surface [G].
*   **When to use:** After topological refinement (e.g., [Recursive Subdivision](1304-recursive-subdivision.md)) to smooth and align the mesh with the implicit representation [G].
*   **When not to use:** When the mesh topology itself is insufficient (e.g., missing details in empty regions), as this method only adjusts positions, not connectivity [G].

## Background
The method is implemented within the Differentiable Volumetric Rendering framework [S1, unofficial]. It addresses the limitation of fixed-resolution meshes in capturing fine geometric details without excessive computational cost by optimizing vertex positions against a differentiable implicit field [S1, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
