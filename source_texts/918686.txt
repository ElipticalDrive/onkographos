# Dynamic 3DGS Mini-Movie Pipeline

## 📖 Overview
This project implements an end-to-end computer vision and graphics pipeline to reconstruct real-world environments using **3D Gaussian Splatting (3DGS)** and animate them dynamically within a custom rendering framework. The project bridges local design environments (macOS) with cloud-based hardware acceleration (Google Colab T4 GPU) to overcome hardware constraints.

By manipulating camera extrinsics, quaternion orientations, and high-dimensional tensors directly in PyTorch, the pipeline constructs a 10-second cinematic sequence featuring multi-object composition, procedural physics-like animations (wobbling, breathing scale effects), and dynamic camera tracking.

---

## 🛠️ Key Technical Features

* **Cross-Platform 3DGS Workflows:** Optimized training on Apple Silicon local hardware using `opensplat` (bypassing standard CUDA requirements) and transitioned seamlessly to cloud-based rendering via `gsplat` on Google Colab.
* **Virtual Viewfinder (`set_camera.py`):** Developed an interactive Open3D tool allowing real-time navigation (WASD controls) through a reconstructed scene's point cloud to frame shots and programmatically export pinhole camera extrinsic and intrinsic parameters.
* **Spatial Object Composition (`object_positions.py`):** Implemented a custom centroid-normalization utility (`center_it=True`) and spatial transformation tool to interactively place, rotate, and scale multiple isolated 3D models within a static background scene, exporting continuous 3D coordinate matrices.
* **Time-Based State Machine Trajectory:** Programmed complex, synchronized procedural behaviors (sinusoidal rotational wobble, organic respiratory scaling, and random positional noise/trembling reactions) using a centralized motion script controller.
* **Low-Level Tensor Manipulation:** Engineered a PyTorch-backed rasterization step that mathematically concatenates scene and object Gaussian attributes (`torch.cat`) in real-time, resolving cross-platform coordinate disparities and quaternion component ordering mismatch (SciPy's `xyzw` vs. gsplat's `wxyz`).

---

## 🏗️ Architecture & Pipeline Flow

```text
   [Real-World Video] 
           │
           ▼ (ffmpeg @ 5fps)
     [Raw Frames]
           │
           ▼ (ns-process-data & opensplat training)
   [Cleaned .ply Models]
           │
 ┌─────────┴────────────────────────┐
 │ LOCALLY (macOS + Open3D)         │
 │ 1. set_camera.py       ──► view.json
 │ 2. object_positions.py ──► Spatial Coordinates
 └─────────┬────────────────────────┘
           │
           ▼
 ┌──────────────────────────────────┐
 │ CLOUD RENDERING (Colab + PyTorch)│
 │ 3. generate_trajectory_and_render.py ──► motion_script.json
 │ 4. Real-time torch.cat + gsplat rasterization
 └──────────────────────────────────┘
           │
           ▼
   [final_movie.mp4]
