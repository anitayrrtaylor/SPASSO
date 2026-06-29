# Setting Up SPASSO on Any Computer

A step-by-step installation guide for **SPASSO** (Software Package for Adaptive
Satellite-based Sampling for Oceanographic cruises). Works on **Linux** and **macOS**;
on **Windows**, use **WSL2** (Ubuntu) and follow the Linux steps.

After installing, see `HOW_TO_USE_SPASSO.md` for how to *run* it and edit cruise configs.

---

## 0. What you'll need

| Requirement | Why | How to get it |
|---|---|---|
| **Python ≥ 3.9** (3.11 recommended) | Runs SPASSO | conda/miniforge (recommended) or system Python |
| **A LaTeX compiler** (`pdflatex` or `latexmk`) | Builds the PDF bulletin | TeX Live (Linux) / MacTeX (Mac) |
| **A Copernicus Marine account** (free) | Downloads all satellite data | https://data.marine.copernicus.eu (register) |
| **git** | Clone the code | preinstalled on most systems |
| **~5 GB free disk** | Code + bathymetry (1.6 GB) + downloaded data | — |

---

## 1. Install the system prerequisites

### Linux (Debian/Ubuntu)
```bash
sudo apt update
sudo apt install -y git texlive-latex-extra texlive-fonts-recommended latexmk
```

### macOS (with Homebrew)
```bash
# git usually present; install a LaTeX distribution:
brew install --cask mactex-no-gui     # or full "mactex"
# pdflatex then lives at /Library/TeX/texbin/pdflatex
```

### Windows
Install **WSL2 + Ubuntu** (`wsl --install` in PowerShell), then follow the **Linux** steps
inside the Ubuntu terminal.

**Verify LaTeX is installed and note its path** — you'll need it in Step 5:
```bash
which pdflatex      # e.g. /usr/bin/pdflatex  (Linux)  or  /Library/TeX/texbin/pdflatex (Mac)
```

---

## 2. Get the code

```bash
cd ~                      # or wherever you want it; the install location is NOT hardcoded
git clone https://github.com/OceanCruises/SPASSO.git
cd SPASSO
```

> SPASSO finds its own root folder as the **parent of wherever you launch it from** (you always
> launch from `src/`). So you can put the `SPASSO/` folder anywhere — no path editing needed for that.

---

## 3. Create the Python environment

**Recommended — conda / miniforge** (handles the tricky `basemap`/`netCDF4` binaries cleanly).
Install [miniforge](https://github.com/conda-forge/miniforge) first if you don't have conda, then:

```bash
# from the SPASSO/ folder — uses the provided environment.yml (creates env named "test-env")
conda env create -f environment.yml
conda activate test-env
```

**Alternative — plain venv + pip** (works, but `basemap` can be harder to build):
```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

> Note: SPASSO also auto-installs anything missing from `requirements.txt` on first launch, but
> creating the environment yourself up front is more reliable.

---

## 4. Verify the Python packages

```bash
# with your environment activated, from the SPASSO/ folder:
python check_modules.py
```
This prints **installed** vs **missing** packages. Install any missing one with:
```bash
pip install <package_name>
```

> ⚠️ Keep **SciPy pinned to 1.13.1** (`scipy==1.13.1`). Newer SciPy removed `interp2d`, which SPASSO
> still uses — a newer version causes the run to crash mid-diagnostic.

---

## 5. Point the config at THIS machine's paths

Two paths in every cruise's config file are machine-specific and **must** be set. Open the example
config:
```bash
# edit with any text editor
nano Cruises/WMedSeaExample/config_WMedSeaExample.ini
```
Find the `## LIBRARY paths` section near the bottom and set:

```ini
[library]              # (section header in the file)
motulib        = /full/path/to/your/env/bin/      ← folder containing the copernicusmarine/motuclient binary (TRAILING SLASH required)
latexcompiler  = /usr/bin/pdflatex                ← the pdflatex path from Step 1
```

**How to find the right values on this machine:**
```bash
# motulib = the DIRECTORY of the copernicusmarine binary, WITH a trailing slash:
dirname $(which copernicusmarine)     # then add a trailing "/"

# latexcompiler = the full pdflatex path:
which pdflatex
```
Example (conda on Linux): `motulib = /home/you/miniforge3/envs/test-env/bin/`
Example (Mac):            `latexcompiler = /Library/TeX/texbin/pdflatex`

---

## 6. Add your Copernicus Marine credentials

In the same config file, set the `[userpwd]` section to **your own** account:
```ini
[userpwd]
userCMEMS = your_copernicus_username     ← your USERNAME, not your email
pwdCMEMS  = your_copernicus_password
```

Test the login works (with the environment activated):
```bash
copernicusmarine login --username your_copernicus_username
```

> If your password contains accented/non-ASCII characters, always launch SPASSO with `PYTHONUTF8=1`
> (see Step 8) so Python reads the config correctly.

---

## 7. Download the bathymetry file (one-time)

Only required for the `TIMEFROMBATHY` diagnostic, but recommended. Download the global NOAA ETOPO 2022
file into `Data/BATHY/`:
```bash
mkdir -p Data/BATHY
curl -L -o Data/BATHY/ETOPO_2022_v1_30s_N90W180_bed.nc \
  "https://www.ngdc.noaa.gov/thredds/fileServer/global/ETOPO2022/30s/30s_bed_elev_netcdf/ETOPO_2022_v1_30s_N90W180_bed.nc"
```
(~1.6 GB.) The filename must match the `bathyfile =` line in the config.

---

## 8. Run the test case

Always launch **from inside `src/`** (that's how SPASSO locates its root folder):
```bash
cd src
PYTHONUTF8=1 python Spasso.py WMedSeaExample
```
- Replace `WMedSeaExample` with another cruise name to run a different config.
- A full run takes a few minutes (it downloads data, then computes diagnostics).
- Success looks like: *"Successfully ended program… Thanks for using SPASSO."*

**Results land in** `Cruises/WMedSeaExample/`:
| Folder | Contents |
|---|---|
| `Figures/` | Output map images (`.png`) |
| `Bulletin/` | The PDF bulletin (and `.tex` source) |
| `Logs/` | Run log — **open this first if anything fails** |

---

## 9. Make a convenient shortcut (optional)

So you can just type `spasso WMedSeaExample` from anywhere:
```bash
# add to ~/.bashrc (Linux) or ~/.zshrc (Mac); adjust the path to your install + env
echo 'spasso() { cd ~/SPASSO/src && PYTHONUTF8=1 conda run -n test-env python Spasso.py "$1"; }' >> ~/.bashrc
source ~/.bashrc
```

---

## 10. Installation troubleshooting

| Symptom | Fix |
|---|---|
| `basemap` fails to build under pip | Use the **conda** environment (Step 3) — conda-forge ships prebuilt basemap. |
| `interp2d has been removed` / SciPy error | Pin SciPy: `pip install scipy==1.13.1` (or `conda install -n test-env scipy=1.13.1`). |
| `Invalid credentials` | Use your Copernicus **username**, not email. Re-check `[userpwd]`; reset at marine.copernicus.eu. |
| PDF bulletin step fails / `latexmk not found` | LaTeX not installed or `latexcompiler` path wrong (Steps 1 & 5). |
| `motuclient: command not found` during download | `motulib` path wrong or missing trailing slash (Step 5). |
| Password with accents read wrong | Launch with `PYTHONUTF8=1` (Step 8). |
| Run exits right after the package list | Config file must be named exactly `config_<FolderName>.ini` and its `[cruises] cruise =` line must match the name you pass. |
| `KeyError: 'longitude'` on a re-run | Stale files in `Cruises/<Name>/Wrk/` from a failed run — delete them and re-run. |

---

## Quick reference

```bash
# one-time setup
git clone https://github.com/OceanCruises/SPASSO.git && cd SPASSO
conda env create -f environment.yml && conda activate test-env
python check_modules.py
#  → edit Cruises/WMedSeaExample/config_WMedSeaExample.ini:
#       [library]  motulib + latexcompiler   (this machine's paths)
#       [userpwd]  userCMEMS + pwdCMEMS       (your Copernicus account)
#  → download bathymetry into Data/BATHY/

# every run
cd src
PYTHONUTF8=1 python Spasso.py WMedSeaExample
```
