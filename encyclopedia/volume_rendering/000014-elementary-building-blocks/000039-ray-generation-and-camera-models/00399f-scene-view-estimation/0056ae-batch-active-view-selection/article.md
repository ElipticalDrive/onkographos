# Batch Active View Selection

Also called: Greedy Batch Selection

## Overview
Batch Active View Selection selects a batch of $B$ camera views to capture new images for volume reconstruction, avoiding the selection of highly similar views by using a greedy optimization algorithm based on Fisher Information [S1]. It differs from single-view selection methods by planning a trajectory for an agent to acquire multiple training views simultaneously, addressing the combinatorial explosion of directly maximizing expected information gain across a batch [S1].

## How it works
The method uses a greedy approximation to maximize the trace of the product of the new view's Hessian and the inverse of the accumulated Fisher Information matrix, providing a $1/e$-approximation of the optimal solution [S1].

**Inputs:**
*   $D_{pool}$: The pool of candidate acquisition views.
*   $H''[w^* | D_{train}]$: The Fisher Information matrix (Hessian) computed from the initial training dataset $D_{train}$ at the current optimal weights $w^*$ [S1].
*   $B$: The number of views to select in the batch [S1].

**Outputs:**
*   $S_B$: The set of $B$ selected views [S1].

**Data Structures:**
*   $S_b$: A set containing the indices of views selected in steps $0$ to $b$.
*   $H_b$: The accumulated Fisher Information matrix after step $b$.

**Algorithm Steps:**
1.  Initialize the selected set $S_0$ to the empty set $\emptyset$ [S1].
2.  Initialize the accumulated Fisher Information matrix $H_0$ to the Hessian of the training data: $H_0 \leftarrow H''[w^* | D_{train}]$ [S1].
3.  For each batch step $b$ from $1$ to $B$:
    a.  Identify the candidate view $i$ from the remaining pool $D_{pool} \setminus S_{b-1}$ that maximizes the trace of the product of its Hessian and the inverse of the current accumulated matrix:
        $$ i \leftarrow \arg \max_{x_{acq} \in D_{pool} \setminus S_{b-1}} \text{tr} \left( H''[y_{i_{acq}} | x_{acq_i}, w^*] H_{b-1}^{-1} \right) $$
        where $H''[y_{i_{acq}} | x_{acq_i}, w^*]$ is the Hessian (Fisher Information contribution) of the candidate view $i$ [S1].
    b.  Add the selected view $i$ to the set: $S_b \leftarrow S_{b-1} \cup \{i\}$ [S1].
    c.  Update the accumulated Fisher Information matrix by adding the selected view's Hessian:
        $$ H_b \leftarrow H_{b-1} + H''[y_{i_{acq}} | x_{acq_i}, w^*] $$
        [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def batch_active_view_selection(candidate_pool, H_train, B):
    """
    Selects B views from candidate_pool using greedy Fisher Information maximization.
    
    Args:
        candidate_pool: List of view indices.
        H_train: Initial Fisher Information matrix (2D list) from training data.
        B: Number of views to select.
        
    Returns:
        selected: List of selected view indices.
    """
    selected = []
    H_acc = [row[:] for row in H_train]  # Deep copy of H_train
    H_inv = inverse_matrix(H_acc)

    for b in range(1, B + 1):
        best_view = None
        max_trace = -float('inf')

        for view_idx in candidate_pool:
            if view_idx in selected:
                continue
            # [G] Standard operation to compute or retrieve H''[y_i_acq | x_acq_i, w*]
            H_view = compute_hessian(view_idx)
            
            product = matmul(H_view, H_inv)
            current_trace = trace(product)
            
            if current_trace > max_trace:
                max_trace = current_trace
                best_view = view_idx

        selected.append(best_view)
        
        H_view_best = compute_hessian(best_view)
        H_acc = mat_add(H_acc, H_view_best)
        H_inv = inverse_matrix(H_acc)

    return selected

def trace(M):
    return sum(M[i][i] for i in range(len(M)))

def matmul(A, B):
    n, m, p = len(A), len(B[0]), len(B)
    return [[sum(A[i][k] * B[k][j] for k in range(p)) for j in range(m)] for i in range(n)]

def mat_add(A, B):
    return [[A[i][j] + B[i][j] for j in range(len(A[0]))] for i in range(len(A))]

def inverse_matrix(M):
    # [G] Standard matrix inversion; in practice, Cholesky or LU decomposition is used for stability.
    pass  # Implementation omitted; use standard linear algebra routine.

def compute_hessian(view_idx):
    # [G] Standard operation to compute the Hessian of the candidate view at current weights w*.
    pass  # Implementation omitted; depends on scene representation and loss function.
```

**Parameters:**
*   $B$: Batch size. When $B=1$, the algorithm is equivalent to single-view active selection [S1].

**Edge Cases and Pitfalls:**
*   **Pool Exhaustion:** If the candidate pool $D_{pool}$ becomes empty before $B$ views are selected, the loop terminates early [G].
*   **Numerical Stability:** The trace operation sums eigenvalues; numerical instability may occur if $H_{b-1}$ is ill-conditioned. In practice, maintaining the inverse explicitly or using Cholesky decomposition updates is recommended for stability and speed [G].

## Subtopics

### [Greedy Representative View Selection](17038-greedy-representative-view-selection.md)
Greedy Representative View Selection is a method for selecting a small, diverse subset of camera views to represent a 3D scene, ensuring that selected views capture information not present in previously chosen ones. It solves the problem of redundant view selection by maximizing geometric diversity among a candidate set. It differs from global maximization approaches by offering a linear-time approximation that yields similar results with significantly lower computational cost. [Greedy Representative View Selection](17038-greedy-representative-view-selection.md)

## Operating characteristics
*   **Approximation Guarantee:** The greedy selection provides a $1/e$-approximation of the optimal Fisher Information objective [S1].
*   **Complexity:** Not given in the sources.
*   **Strengths:** Avoids the combinatorial explosion of directly maximizing expected information gain between training and acquisition samples [S1]. It is designed for real-world scenarios where an agent plans a trajectory to acquire multiple training views simultaneously [S1].
*   **Limitations:** Greedy optimization may yield suboptimal global solutions compared to exhaustive search, though the approximation bound mitigates this [S1].
*   **When to use:** When batch acquisition is required and computational cost of global optimization is prohibitive [S1].

## Background
The method was introduced in the context of active mapping with 3D Gaussian Splatting and Radiance Fields, specifically in the paper "FisherRF: Active View Selection and Mapping with Radiance Fields Using Fisher Information" [S1]. It addresses the problem of selecting informative views for reconstruction by leveraging Fisher Information to quantify the information gain of potential new views [S1]. The approach is designed to avoid the combinatorial explosion of directly maximizing expected information gain between training and acquisition samples [S1].

## Sources
[S1] FisherRF: Active View Selection and Mapping with Radiance Fields Using Fisher Information, https://doi.org/10.1007/978-3-031-72624-8_24, fetched 2026-10-31
