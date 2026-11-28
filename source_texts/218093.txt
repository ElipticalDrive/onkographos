## General Description

This program automatically processes `.ims` microscopy image files by generating **Maximum Intensity Projections (MIP)** and saving them as compressed **.ome.tiff** files. It is designed to handle larger datasets, since it doesn't load the entire dataset into memory at once.

The algorithm loads each Z slice of dimensions (T,C,Y,X) incrementally into memory and registers the maximum value for each pixel in the tensor. The final result corresponds to the MIP of the dataset.

This approach ensures that large `.ims` files can be processed on most computers, since the full dataset is not loaded simultaneously into memory.

If, for any reason, the program cannot process a specific file, its name and error will be saved in the generated report, and the program will continue processing the remaining files.

A standalone executable that runs this program is also available as a Release.

---

## Running the code

You will need a Python installation.

**Windows:**

```bat
setup_venv.bat
.venv\Scripts\activate
python main.py
```
  

**macOS/Linux:**

```sh
./setup_venv.sh
source .venv/bin/activate
python main.py
```

---

## Program usage

1. Run the executable/code.

2. Wait for the program to open.

3. Select the folder containing the `.ims` files.

4. Wait while the program automatically:

   - Processes all `.ims` files.
   - Saves the maximum intensity projections.
   - Generates the `report.txt` file.

---

## Output

The generated projections and the `report.txt` file will be saved in:

    <selected folder>/maximum intensity projections/

Example:

    C:/Users/YourName/Desktop/IMS files/maximum intensity projections/

---

## Author

**Simão Peniche Seixas**

simao.seixas@i3s.up.pt  
simao.peniche.seixas@gmail.com  
i3S - Institute for Research and Innovation in Health
