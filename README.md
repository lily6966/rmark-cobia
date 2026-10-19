# Running RMark with MARK on an Apple Silicon Mac

[RMark](https://cran.r-project.org/package=RMark) is an R interface to **MARK**, a separate program that does the actual model fitting. On Apple Silicon Macs (M1–M4), the macOS MARK binary is an Intel (`x86_64`) program. It runs under Rosetta but needs **Intel** versions of two gcc runtime libraries (`libquadmath` and `libgcc_s`).

Homebrew no longer installs on Intel/Rosetta, so these instructions get the Intel libraries from conda-forge instead.

## Prerequisites

- An Apple Silicon Mac with Rosetta installed:
  ```bash
  softwareupdate --install-rosetta --agree-to-license
  ```
- [Anaconda or Miniconda](https://www.anaconda.com/download)
- R, with the RMark package: `install.packages("RMark")`
- The macOS MARK executable from [phidot.org](http://www.phidot.org/software/mark/), saved as `MARK` in your working folder

> **Tip:** zsh does not treat `#` as a comment when you paste commands, so paste the commands below exactly as shown.

## Installation

### 1. Create an Intel-only conda environment with the libraries

```bash
CONDA_SUBDIR=osx-64 conda create -y -n mark-intel -c conda-forge libgfortran5
```

This environment only holds the libraries. You never need to activate it, but **do not delete it**, because MARK depends on its files.

### 2. Check that the libraries are Intel builds

```bash
SRC="$(conda info --base)/envs/mark-intel/lib"
ls "$SRC" | grep -E "quadmath|gcc_s"
lipo -info "$SRC/libquadmath.0.dylib" "$SRC/libgcc_s.1.1.dylib"
```

`lipo` should report `x86_64` for both files.

### 3. Link the libraries to where MARK looks for them

Run this in the same Terminal window as step 2, so that `SRC` is still set:

```bash
sudo mkdir -p /usr/local/opt/gcc/lib/gcc/current
sudo ln -sf "$SRC/libquadmath.0.dylib" /usr/local/opt/gcc/lib/gcc/current/
sudo ln -sf "$SRC/libgcc_s.1.1.dylib"  /usr/local/opt/gcc/lib/gcc/current/
```

### 4. Install MARK

From the folder containing the `MARK` binary:

```bash
sudo cp MARK /usr/local/bin/mark
sudo chmod +x /usr/local/bin/mark
sudo xattr -d com.apple.quarantine /usr/local/bin/mark   # "No such xattr" is fine
```

### 5. Test MARK

```bash
/usr/local/bin/mark
```

It should start without a `dyld: Library not loaded` error.

## Running the R code

Tell RMark where MARK is installed **before** the first call to `mark()`:

```r
library(RMark)
MarkPath = "/usr/local/bin"
```

Then, in RStudio, open `RMarkWorkshop.R` and step through it line by line with **Cmd + Enter**. You can also run the whole file from the R console:

```r
setwd("/path/to/this/repo")
source("RMarkWorkshop.R", echo = TRUE)
```

Notes on the workshop script:
- The data files `example.txt` and `example.inp` must be in the working directory.
- Run `install.packages("RMark")` only once.
- Chunk 72 needs `install.packages("Hmisc")`.
- Chunk 50 reads `SharedParameters.txt`; skip it if you don't have that file.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `Output file does not exist. Unable to find or run mark.exe` | MARK itself failed to start. Run `/usr/local/bin/mark` in Terminal to see the real error. |
| `dyld: Library not loaded: .../gcc/10/libquadmath.0.dylib` | You have an older MARK binary hard-wired to gcc 10. Replace it with the newer build (step 4). |
| `dyld: Library not loaded: .../gcc/current/...` | The links from step 3 are missing or broken. Re-run steps 2–3. |
| `lipo` reports `arm64` | The environment wasn't created as Intel. Remove it (`conda env remove -n mark-intel`) and re-run step 1 with `CONDA_SUBDIR=osx-64`. |
| `Homebrew on macOS is only supported on Apple Silicon processors!` | Expected; Homebrew can't install Intel libraries. Use the conda method above. |
| Checking which libraries MARK needs | `otool -L /usr/local/bin/mark` |

## Credits and citation

`RMarkWorkshop.R` and the example data come from Jeff Laake's **RMark Workshop Notes**, distributed in the RMark documentation bundle ([RMarkDocumentation.zip](http://www.phidot.org/software/mark/rmark/RMarkDocumentation.zip)) on phidot.org. The version here adds local setup lines (`setwd()`, `MarkPath`, `PATH`) for running on macOS.

If you use this code or RMark in your work, please cite:

> Laake, J.L. (2013). RMark: An R Interface for Analysis of Capture-Recapture Data with MARK. AFSC Processed Report 2013-01. Alaska Fisheries Science Center, NOAA, National Marine Fisheries Service, Seattle, WA. https://apps-afsc.fisheries.noaa.gov/Publications/ProcRpt/PR2013-01.pdf

> White, G.C. & Burnham, K.P. (1999). Program MARK: survival estimation from populations of marked animals. *Bird Study* 46(sup1): S120–S139.

In R, `citation("RMark")` prints the current recommended citation.
