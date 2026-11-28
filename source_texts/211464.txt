# Neural Radiance Fields (NeRF): Replication & Ablation Study

This repository contains a PyTorch implementation of a lightweight Neural Radiance Field (NeRF), built from scratch to synthesize novel views of continuous 3D scenes from a sparse set of 2D images. 

This project was developed for the COMP-6909 course at Memorial University of Newfoundland.

## Project Overview

Traditional 3D computer graphics rely on discrete geometry (polygon meshes). This project explores the paradigm shift of using coordinate-based deep neural networks (Multi-Layer Perceptrons) to optimize a continuous volumetric scene function.

The project is split into two phases:
1. **Phase 1 (Replication):** Building the core pipeline (ray casting, stratified sampling, positional encoding, and differentiable volume rendering) to successfully synthesize novel camera angles of a 3D target geometry.
2. **Phase 2 (Ablation Study):** An architectural extension that systematically removes Positional Encoding to empirically prove the impact of "spectral bias" in standard neural networks.

## Phase 1 Results: Novel View Synthesis

The model was trained on a synthetic dataset (Lego bulldozer). To accommodate local hardware constraints, the rendering equation was modified with a dynamic white-background composite, allowing for successful volumetric convergence within a strict 50-iteration budget.

*[assets/Screenshot 2026-03-24 203914.png](https://github.com/nkrumahdarlington7-dotcom/tiny-nerf-replication/blob/main/assets/Screenshot%202026-03-24%20203914.png)*

## Phase 2 Results: Positional Encoding Ablation

Standard MLPs inherently act as low-pass filters, struggling to learn high-frequency geometric details from low-dimensional $(x,y,z)$ spatial inputs. To prove this, the positional encoding (high-frequency sine/cosine mapping) was removed from the pipeline. 

As shown below, the ablated model fails to capture structural fidelity, rendering a smooth, undefined spatial average. This empirically proves that mapping spatial coordinates to a higher-dimensional space is a strict requirement for high-fidelity neural rendering.

*https://github.com/nkrumahdarlington7-dotcom/tiny-nerf-replication/blob/main/assets/Screenshot%202026-03-24%20210224.png*

## Technical Implementation

* **Language/Framework:** Python, PyTorch
* **Architecture:** 8-Layer Multi-Layer Perceptron (MLP) with a skip connection.
* **Rendering:** Differentiable Volume Rendering via accumulated transmittance. 
* **Input Mapping:** High-frequency Positional Encoding ($L=10$ for spatial coordinates, $L=4$ for viewing directions).

## How to Run

1. Clone this repository: `git clone https://github.com/YourUsername/tiny-nerf-replication.git`
2. Open `NeRF_Project.ipynb` in a Jupyter Notebook environment.
3. The notebook will automatically download the required `tiny_nerf_data.npz` dataset upon running the first cell.
4. Run the Phase 1 cell to train the full model, followed by the Phase 2 cell to train the ablated model.

## Documentation
The complete formal write-up, formatted as an IEEE conference paper detailing the mathematics, implementation, and conclusions, is available in the [`report/`](report/) directory.
