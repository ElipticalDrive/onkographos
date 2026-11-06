# VTK Transfer Function Visualizer

VTK Transfer Function Visualizer is a web tool specifically designed for volume rendering applications using VTK (Visualization Toolkit). It allows users to visualize and fine-tune transfer function parameters for scalar opacity and color mapping, enabling enhanced visualization of volumetric datasets.

## Features

- Input VTK transfer function parameters in JSON format
- Visualize scalar opacity and color transfer functions
- Interactive line chart for exploring transfer function distribution
- Easy-to-use interface for adjusting transfer function parameters
- Instantaneous visualization updates for efficient parameter tuning

## Usage

1. Input VTK transfer function parameters in JSON format
2. Click "Plot Points" to visualize the transfer functions
3. Explore and adjust transfer function parameters using the interactive line chart
4. Fine-tune parameters to enhance the visualization of volumetric data sets
5. Export optimized transfer function parameters for use in VTK-based volume rendering applications

## Example Input

```json
{
  "name": "CT-Coronary-Arteries-3",
  "gradientOpacity": "4 0 1 255 1",
  "specularPower": "10",
  "scalarOpacity": "14 -2048 0 128.643 0 129.982 0.0982143 173.636 0.669643 255.884 0.857143 584.878 0.866071 3661 1",
  "specular": "0.2",
  "shade": "1",
  "ambient": "0.1",
  "colorTransfer": "28 -2048 0 0 0 128.643 0 0 0 129.982 0.615686 0 0.0156863 173.636 0.909804 0.454902 0 255.884 0.886275 0.886275 0.886275 584.878 0.968627 0.968627 0.968627 3661 1 1 1",
  "diffuse": "0.9",
  "interpolation": "1"
}
