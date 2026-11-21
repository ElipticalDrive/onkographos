# 3D CNN for Brain MRI Classification

A PyTorch pipeline for classifying **whole 3D brain MRI volumes** (NIfTI `.nii` / `.nii.gz`) with a volumetric convolutional network. It trains on Apple Silicon (MPS), CUDA or CPU.

> **Status: in progress.** Data loading, the model, training and evaluation work end to end. Next up: assembling a labeled dataset and adding augmentation. See the [roadmap](#roadmap).

## How it works

```
T1w MRI volume (D × H × W)
      │  nibabel load → z-score normalize → 1 × D × H × W tensor
      ▼
Conv3d(1→16)  → ReLU → MaxPool3d(2)
Conv3d(16→32) → ReLU → MaxPool3d(2)
Conv3d(32→64) → ReLU → MaxPool3d(2)
      │  flatten
      ▼
LazyLinear(→128) → ReLU → Linear(128→num_classes)
```

- **Shape-agnostic head.** `LazyLinear` infers the flattened size from the first volume, so the same model works across scanners and resolutions. A dry-run forward pass runs before the optimizer is created, so the head's weights actually get trained.
- **Full-resolution volumes** (e.g. 176 × 192 × 192), batch size 1 to fit in memory.
- **Training loop** reports loss and accuracy for train and validation each epoch and checkpoints the best model by validation loss.

## Data

Built and tested with T1-weighted anatomical scans from OpenNeuro [ds000228](https://openneuro.org/datasets/ds000228) ("pixar" subjects). Volumes aren't committed (see `.gitignore`).

```
data/
  train/  *.nii.gz + labels.csv
  val/    *.nii.gz + labels.csv
```

`labels.csv` has two columns, `filename,label` (integer class). Volumes without a label are skipped and reported.

## Usage

```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python train.py --data data --epochs 20 --lr 1e-4 --num-classes 2
```

## Roadmap

- [ ] Labeled task, e.g. child vs. adult classification from ds000228 participant metadata
- [ ] Skull stripping and resampling to a common voxel spacing
- [ ] 3D augmentation (flips, small rotations, intensity jitter)
- [ ] Batch norm / residual blocks; compare against a pretrained MedicalNet backbone
- [ ] Grad-CAM saliency maps to visualize which regions drive predictions

## Tech

Python · PyTorch · nibabel · NumPy
