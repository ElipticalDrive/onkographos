# Convert-.ply-point-cloud-to-.sog-Gaussian-Splatting

This utility processes standard Lidar or photogrammetry point clouds (`.ply`) and prepares them for 3D Gaussian Splatting web viewers by converting them into Spatially Ordered Gaussians (`.sog`). 

Standard point clouds often lack the necessary properties (like scale, rotation, and normalized opacity) required by Gaussian Splatting engines. Furthermore, coordinate systems between raw Lidar scans and WebGL environments usually mismatch, resulting in upside-down or misaligned models. 

This script bridges that gap by injecting required Gaussian properties, normalizing color spherical harmonics, applying coordinate system transformations, and triggering the PlayCanvas `splat-transform` tool to build the highly compressed `.sog` file.

You can also convert directly on https://superspl.at/convert 
---

## ✨ Features

* **Memory Safe Processing:** Uses `float64` processing for centering to prevent coordinate jitter on large-scale geographic models, while scaling down to `float32` for final output to save memory.
* **Coordinate Correction:** Automatically transforms Z-up (Lidar) coordinate systems to Y-up (WebGL), flipping the vertical axis so models appear upright in web viewers.
* **Property Injection:** Generates required Gaussian Splat attributes (`scale`, `rot`, `opacity`, `f_dc`) directly from raw vertex data.
* **Large File Support:** Automatically overrides Node.js memory limits (`--max-old-space-size`) to prevent heap crashes during the final `.sog` compression phase.

---

## 🛠️ Prerequisites

To run this converter, you will need:
* **Python 3.7+**
* **Node.js** (v14 or higher)

---

## 🚀 Installation & Setup

**1. Clone the repository**
```bash
git clone [https://github.com/yourusername/ply-to-sog-converter.git](https://github.com/yourusername/ply-to-sog-converter.git)
cd ply-to-sog-converter
```
---

**2. Install Python Dependencies
Install the required Python packages using pip:
```bash
pip install -r requirements.txt
```

**3. Install Node.js Dependencies
This script relies on the PlayCanvas splat-transform tool. Install it locally within the project folder using npm:
```bash
npm install
```

---

## ⚙️ Configuration
All configurable variables are located at the very top of convert.py in the CONFIG dictionary. You can edit these directly before running the script:

DENSITY_STEP: Controls point downsampling (1 = 100%, 2 = 50%, etc.).

Y_OFFSET: Adds depth to the Z-axis so the web camera starts a comfortable distance away from the object.

GAUSSIAN_SCALE: Controls the size of individual splats (-4.0 is tight, -3.0 is looser).

NODE_MAX_MEMORY: Megabytes allocated for the Node.js .sog conversion (default is 16384 for 16GB).

---

## 💻 Usage
Run the script via the command line, passing your .ply file as an argument:

```bash
python convert.py --input /path/to/your/model.ply
```

What happens next?

The script loads your .ply file.

It calculates the center point and applies coordinate fixes.

It saves a temporary, fixed .ply file in the same directory.

It calls splat-transform to compress the file.

A highly compressed .sog file is generated in the exact same folder as your input file, and the temporary file is deleted.

---

## requirements

numpy>=1.20.0
plyfile>=0.8.0
