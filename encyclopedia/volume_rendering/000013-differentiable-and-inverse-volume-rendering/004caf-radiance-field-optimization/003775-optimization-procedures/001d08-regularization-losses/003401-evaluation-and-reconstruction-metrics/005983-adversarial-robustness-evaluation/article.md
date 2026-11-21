# Adversarial Robustness Evaluation

Also called: Adversarial Attack Evaluation, BREN Ratio Analysis

## Overview
Adversarial Robustness Evaluation assesses the stability of reconstruction networks (such as those used in tomography or radiance field optimization) when subjected to small, targeted perturbations in the input data. It quantifies sensitivity by computing the **BREN ratio**, which measures the relative change in the reconstructed output versus the magnitude of the input perturbation. Unlike standard metrics that measure accuracy against ground truth, this method evaluates whether small input errors lead to disproportionately large, artifact-ridden outputs, indicating a lack of robustness [S1].

## How it works
The method evaluates a reconstruction network $\mathcal{R}$ by comparing its output on clean data against its output on perturbed data.

**Inputs and Outputs:**
- Input: Clean measurement data $\mathbf{y}$, perturbed measurement data $\mathbf{y}' = \mathbf{y} + \mathbf{e}$, where $\mathbf{e}$ is the perturbation vector.
- Output: The BREN ratio value.

**Data Structures:**
- $\mathbf{y}$: Vector of raw measurements (e.g., sinogram entries or Fourier samples).
- $\mathbf{e}$: Perturbation vector, typically generated via gradient ascent to maximize reconstruction error.
- $\hat{\mathbf{x}} = \mathcal{R}(\mathbf{y})$: Clean reconstruction.
- $\hat{\mathbf{x}}' = \mathcal{R}(\mathbf{y}')$: Perturbed reconstruction.

**Steps:**
1. **Generate Perturbation:** Compute a perturbation $\mathbf{e}$ with bounded norm $\|\mathbf{e}\| \leq \epsilon$. In iterative frameworks like ACID, this is done via gradient ascent on the reconstruction error [S1].
2. **Reconstruct:** Compute the clean reconstruction $\hat{\mathbf{x}}$ and the perturbed reconstruction $\hat{\mathbf{x}}'$.
3. **Compute BREN Ratio:** Calculate the ratio of the output change to the input change:
   $$
   \text{BREN} = \frac{\|\hat{\mathbf{x}}' - \hat{\mathbf{x}}\|_2}{\|\mathbf{y}' - \mathbf{y}\|_2} = \frac{\|\mathcal{R}(\mathbf{y} + \mathbf{e}) - \mathcal{R}(\mathbf{y})\|_2}{\|\mathbf{e}\|_2}
   $$
   [S1]

**Parameters:**
- $\epsilon$: Maximum perturbation strength. Typical values depend on the noise floor of the specific imaging modality (CT/MRI) [S1].

**Edge Cases and Pitfalls:**
- **Artifact Generation:** Small perturbations may corrupt the sparsity of the reconstructed image, leading to artifacts that are not present in the clean reconstruction [S1].
- **Inconsistency:** Perturbed results may become inconsistent with original measurements, violating the constraints used in traditional iterative reconstruction [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_bren_ratio(reconstruction_network, clean_data, perturbation_strength):
    """
    Computes the BREN ratio for a given reconstruction network and perturbation.
    """
    # Step 1: Generate perturbation
    # In practice, this often involves gradient ascent to find worst-case e
    # For a fixed epsilon, we assume e is provided or generated externally
    perturbation = generate_perturbation(clean_data, perturbation_strength)
    
    # Step 2: Create perturbed data
    perturbed_data = clean_data + perturbation
    
    # Step 3: Reconstruct both
    clean_reconstruction = reconstruction_network(clean_data)
    perturbed_reconstruction = reconstruction_network(perturbed_data)
    
    # Step 4: Compute L2 norms
    output_diff_norm = l2_norm(perturbed_reconstruction - clean_reconstruction)
    input_diff_norm = l2_norm(perturbation)
    
    # Step 5: Compute ratio
    if input_diff_norm == 0:
        return 0.0 # No perturbation applied
    
    bren_ratio = output_diff_norm / input_diff_norm
    return bren_ratio
```

## Subtopics

### [Adversarial Attack on ACID Framework](16431-adversarial-attack-on-acid-framework.md)
An iterative process to find optimized solutions in the intersection of data-driven priors, sparse images, and measurement solutions, attacked by computing tiny perturbations via gradient ascent.

## Operating characteristics
- **Sensitivity Indicator:** A BREN ratio substantially less than 1 indicates that the reconstruction network is robust, as small input perturbations cause relatively small output changes [S1].
- **Comparative Sensitivity:** Networks like AUTOMAP exhibit higher sensitivity (higher BREN ratios) compared to Med-50, Deep MRI, and MRI-VN, meaning small perturbations cause larger changes in the L2-norm of the output [S1].
- **Limitations:** The method relies on the assumption that sparsity is central to the reconstruction; if the network does not enforce sparsity, the interpretation of artifacts may differ [S1].

## Background
The BREN ratio was introduced in the context of stabilizing deep tomographic reconstruction. It was used to analyze the convergence and adversarial robustness of various deep learning-based reconstruction networks (such as AUTOMAP, Med-50, and MRI-VN) in CT and MRI experiments [S1]. The analysis highlighted that while these networks satisfy the BREN property, they remain sensitive to perturbations that corrupt image sparsity [S1].

## Sources
[S1] Stabilizing deep tomographic reconstruction: Part B. Convergence analysis and adversarial attacks., https://doi.org/10.1016/j.patter.2022.100475, fetched 2026-10-26
