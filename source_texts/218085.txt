# nf-imaris-mip
Compute maximum intensity projections (MIP) for Imaris images

## Contents

- `main.nf`: A simple Nextflow workflow that coordinates the execution of the ImageJ macro on a set of input files
- `nextflow.config`: The default parameters for that workflow
- `macros/make_mip.ijm`: The ImageJ macro being run
- `templates/make_mip.sh`: The shell script used to run the ImageJ macro inside a Docker container

## Parameters

- `inputs`: Comma delimited string with input Imaris (.ims) files (supports glob patterns)
- `outdir`: Path where output files will be published
- `container`: Docker container used to run the macro (i.e. `fiji/fiji:20220415`)
- `macro`: The file in `macros/` which will be run (i.e. `make_mip.ijm`)
- `make_MIP`: Whether or not to make a MIP (`"true"` or `"false"`)
- `make_slice`: Whether or not to export a slice (`"true"` or `"false"`)
- `slice_number`: The position of the slice (ranges from 0-100) (default: `50`)
- `make_JPG`: Whether or not to export a JPG (in addition to TIFF) for both the MIP and the slice (`"true"` or `"false"`)
- `include_scalebar`: Whether or not to include the scalebar on the JPG (`"true"` or `"false"`)
