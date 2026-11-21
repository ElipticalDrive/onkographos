# 🧠 Neural Radiance Fields (NeRF) - 3D Scene Reconstruction

This project demonstrates 3D reconstruction of a scene using **Neural Radiance Fields (NeRF)**. Given a set of input images, the model learns to synthesize novel views of the scene by predicting color and density along camera rays.

---

## 📂 Project Structure

├── final-project.ipynb # Main Jupyter Notebook
├── requirements.txt # List of dependencies
├── sample_images_input/ # Sample input images for training
├── training_images_video/ # Video used to extract training images
├── results/ # Rendered outputs (images/videos)

---

## 🚀 Features

- Load input images and generate rays.
- Implement NeRF from scratch using PyTorch.
- Positional encoding and hierarchical sampling.
- Volume rendering to synthesize novel views.
- Visualize reconstructed images from new camera angles.

---

## 🛠️ Requirements

Install all dependencies using:

```bash
pip install -r requirements.txt

If you don't have requirements.txt, manually install:
pip install torch numpy matplotlib imageio imageio-ffmpeg configargparse
```

<h2>🖼️ Sample Results</h2>
Check the results/ folder to see:

Rendered images from novel viewpoints

Output video

<h2>📥 How to Use</h2>
Clone the repository:
git clone https://github.com/lukky44-bit/3D-Reconstruction-using-NeRF.git
cd nerf-3d-reconstruction
Place your input images into the sample_images_input/ folder.

Run the notebook:

Open final-project.ipynb in Jupyter or Colab

Execute the cells step-by-step

<h2>📄 License</h2>
This project is open-sourced under the MIT License. Feel free to modify and use for educational purpose

<h2>🙋‍♂️ Author</h2>
Lakshan D.
REVA University | 6th Semester | B.Tech AI & ML
