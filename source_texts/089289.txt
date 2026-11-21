# Physics-Aware Cone Beam CT Reconstruction

A GPU-accelerated, physics-informed CT reconstruction pipeline combining:
- Classical filtered backprojection (FDK)
- Iterative reconstruction (SIRT via autodiff)
- Deep learning enhancement (U-Net)
- Synthetic CT simulation pipeline
- ImageJ-compatible export

## Motivation
Industrial imaging systems (X-ray CT, NDE) require high-quality reconstruction under noise, sparse views, and physics constraints. This project explores hybrid reconstruction combining physics and learning.

## Features
- Cone-beam style forward projector
- FDK implementation from scratch
- Iterative reconstruction using PyTorch autodiff
- Synthetic phantom generation
- DL-based post reconstruction enhancement
- Export to ImageJ raw volumes

## Example Results
| Ground Truth | FDK | FDK + DL | Iterative |
|--------------|-----|-----------|-------|
<img width="1716" height="427" alt="image" src="https://github.com/user-attachments/assets/5796d786-fb98-48ec-8a02-bd7b6251467f" />




## How to Run
```bash
pip install -r requirements.txt
python generate_data.py
python reconstruct.py
python export_imagej.py
